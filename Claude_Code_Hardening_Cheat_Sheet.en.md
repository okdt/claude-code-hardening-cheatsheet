# Claude Code Hardening Cheatsheet

> Version 1.2 (2026-08-02)
>
> Verified against: Claude Code v2.1.220 (as of 2026-08-01), macOS. Setting keys and behavior change between versions. Everything here has been checked against both the official documentation and a live install.

---

## 1. Introduction

Claude Code runs shell commands, reads and writes files, and talks to external services on your behalf. What makes it convenient is exactly that it runs with *your* privileges — and the flip side is that anything you didn't intend can run with those same privileges too. This cheat sheet is about controlling that swing. Enabling sandboxing alone already buys you a lot. If you're a technical lead setting a team baseline, start building from here.

### Risks: Unexpected Agentic Collapse — Why Hardening Is Needed

- **Well-intentioned overreach** — Claude Code may take actions that are technically correct but go beyond what you intended: deleting files to "clean up," force-pushing to "fix" a branch, or installing packages you didn't ask for. ([OWASP LLM09: Overreliance](https://genai.owasp.org/llm-top-10/))
- **Excessive permissions** — By default, Claude Code can do anything your user account can do. Without deny rules, a single "yes" can grant access to destructive commands, credential files, or remote systems. ([OWASP LLM06: Excessive Agency](https://genai.owasp.org/llm-top-10/))
- **Indirect prompt injection** — The content Claude Code processes (source code, documents, web pages) can contain instructions that influence its behavior. An attacker can embed malicious prompts in files or dependencies that Claude reads as part of normal operation. ([OWASP LLM01: Prompt Injection](https://genai.owasp.org/llm-top-10/))
- **More places for secrets to sit** — configuration files, environment variables, and the conversation transcript. Using Claude Code introduces places where an API key or token can be exposed in plaintext, and if your home directory syncs to the cloud, it's copied to every machine you own the moment you write it. Every other risk here stops once you fix the setting. **A leaked secret doesn't.** Rotation is the only way out, which is why this one gets handled more carefully than the rest (Section 6). ([OWASP LLM02: Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/))
- **Limiting damage in an already-compromised environment** — If your machine is hit by RCE (Remote Code Execution), malware, or a supply chain attack, Claude Code falls under that compromise too. Hardening shrinks the blast radius — the scope of damage — so that even if an attacker uses Claude Code as a stepping stone, what they can do through it is limited.

These are not hypothetical. They are the reason guardrails exist: to ensure that when things go wrong — and they will — the damage is contained.
This cheatsheet provides a practical guide to hardening your Claude Code environment through `~/.claude/settings.json` — applying the principle of least privilege and Human-in-the-Loop (HITL) controls.

### Approach

1. **Sandboxing** — OS-level process isolation that restricts Claude Code's (and its child processes') file and network access. This operates at the kernel level, so the AI cannot circumvent it. The most fundamental and strongest defense layer.
2. **Permissions (allow/deny/ask/default)** — Rules that control what happens when a tool (Bash command, file edit, etc.) is invoked from Claude Code's console: "always allow / always ask / always deny / default (not configured)." Permissions provide fine-grained access control per tool invocation. Using `ask` enables Human-in-the-Loop — systematic visual review.
3. **Hooks (hooks + PreToolUse)** — A mechanism to automatically run shell scripts before and after tool invocations. Lets you inject fine-grained pattern matching and environment-specific custom checks that permissions' allow/deny alone can't handle.
4. **Logging** — Sometimes needed in enterprise environments, or for debugging this hardening setup itself. We touch on it briefly.
5. **Secrets management** — Where API keys and tokens go, and where they must not. The tooling that keeps plaintext out of your configuration files, your environment variables, and your transcripts. This one differs in kind from 1–4: a wrong setting can be fixed, a leaked secret cannot.
6. **Data retention** — How long conversation transcripts and session logs stay on your disk. Not what Claude can do, but how much trace it leaves. Decide the retention period explicitly.

> Stacking defenses several deep, rather than leaning on any single one, is what's called **defense in depth**. When one layer is breached, the next still stands.

### What about CLAUDE.md?

`CLAUDE.md` (at the project root or `~/.claude/CLAUDE.md`) is meant for recording the purpose, context, and outline of your environment and work. While listing permission policies like "don't push directly to main" or "don't commit until tests pass" isn't entirely meaningless, CLAUDE.md has a limited capacity (around 200 lines is recommended), and while it's fine for context, it's not well suited as a place for deny policies. On top of that, even if you do write them there, they're merely "requests" at best — and they're often forgotten.

When something should not happen, stop it by force rather than by asking — figuring out how to do that is the starting point of this document.

### About the examples

Deny rules stop the basic cases, the sandbox restricts access based on boundary principles, and hooks let you add more detailed conditions on top. The three layers work in combination.

This document covers what to block, what to allow, what to always ask about, and what to do when deny rules aren't enough. The deny list examples here are samples, not a complete list. They start from the perspective of what risks to suppress. This cheat sheet is primarily written and tested on macOS, but should be useful for Linux and Windows (WSL: Windows Subsystem for Linux) as well.

One more thing: you can hand this whole document to Claude Code. The work of hardening your own Claude Code environment is something that Claude Code can think through with you. A prompt for running that check-and-improve pass is included ([`Claude_Code_Hardening_Audit_Prompt.en.md`](Claude_Code_Hardening_Audit_Prompt.en.md)).

Customize for your own environment and risk profile.

---

## 2. Sandboxing

The sandbox isolates Claude Code's file and network access at the OS level. Even if a deny rule is bypassed, the sandbox prevents access to resources outside defined boundaries. It's the strongest defense you can configure — enable it first.

Supported on macOS (Seatbelt), Linux, and WSL2 (bubblewrap). WSL1 and native Windows are not supported: bubblewrap needs kernel features that only WSL2 has, so on Windows you run Claude Code inside WSL2.

### How to enable

- Run `/sandbox` in Claude Code's interactive mode. This opens a menu where you can enable sandboxing and configure its mode.

- Doing this manually each time is tedious and error-prone, so configure it in settings instead.

`settings.json` configuration:

```json
"sandbox": {
  "enabled": true,
  "autoAllowBashIfSandboxed": true,
  "filesystem": {
    "denyRead": ["~/.ssh", "~/.gnupg", "~/.aws", "~/.config/gcloud"]
  },
  "network": {
    "allowedDomains": ["github.com", "registry.npmjs.org", "pypi.org"]
  }
}
```

| Setting | Why |
|---------|-----|
| `enabled: true` | Isolates file and network access at the OS level. **Writes** are limited to the current working directory and paths you explicitly allow. **Reads**, by contrast, default to "everything is allowed except what `denyRead` blocks" — so credential protection depends on the `denyRead` list below. |
| `autoAllowBashIfSandboxed` | Skips the permission prompt for Bash commands while the sandbox is active. Safe because the OS constrains their scope. What gets skipped is the default prompt and a bare `Bash` (or `Bash(*)`) ask rule — a content-scoped ask like `Bash(curl *)`, and every deny rule, still apply inside the sandbox. |
| `filesystem.denyRead` | Paths that stay unreadable even inside the sandbox. Since reads are allowed by default, this is where you explicitly seal off credential stores. The example covers SSH keys, GPG keys, AWS credentials, and GCP config. **Because the OS enforces it, whatever process tries to open that path is stopped.** A `Read()` deny in the permission system reaches as far as the commands Claude Code recognizes, `cat` among them, but it can't follow a script that opens the file itself. That's where the two layers differ. |
| `network.allowedDomains` | Domains a sandboxed Bash command is allowed to reach. By default nothing is pre-allowed, so the first connection to a new domain triggers an approval prompt. List the ones you use often to skip the prompts. To block a specific domain, add it to `deniedDomains` (it overrides a broader allow). |

> **Note:** Filesystem and network isolation only mean something *together*. Leave the network wide open and a single over-permissive read can ship your SSH keys straight out. The reverse is true too. One side alone is a hole.

### Handling the escape hatch (dangerouslyDisableSandbox)

Even with the sandbox on, if a command fails *inside* it, Claude may retry it *outside* the sandbox using `dangerouslyDisableSandbox`. This is by design — it kicks in for sandbox-incompatible tools (like Docker) or when a command needs a host you haven't allowed. The retry goes through the normal permission flow by default, so it isn't "silently escaping," but when you're in a hurry it's easy to reflexively approve it, and your strongest layer quietly erodes. Handle it in three steps:

**1. Fix the root cause in settings (do this first).** The escape almost always comes down to one of: the destination isn't in the allowlist, the write target is outside the working directory, or the tool can't run sandboxed. Address each in turn — add the domain to `network.allowedDomains`, add the path to `filesystem.allowWrite`, or list *just that tool* in `excludedCommands`. Then there's no need to escape at all. `excludedCommands` runs one command outside the sandbox rather than turning the whole thing off, so the blast radius stays small.

**2. Record the escapes and feed them back into the settings.** Keep allowing the escapes for now, and grow the configuration from what you observe. The next subsection covers this in full.

**3. Seal the escape hatch.** Set `allowUnsandboxedCommands: false` (Strict sandbox mode) and `dangerouslyDisableSandbox` is ignored entirely. A failure stays a failure, and only commands you listed in `excludedCommands` run outside. Good for teams or CI that need a hard guarantee that nothing escapes.

> Heads-up: in versions before the April 2026 fix, `dangerouslyDisableSandbox` ran outside the sandbox *without* a permission prompt. If you're on an older build, update first.

> **The state to aim for:** the best setup keeps the sandbox **on** — no escapes needed — *and* still **refuses** dangerous access and commands. You get the first by tuning the allowlists (this section), and the second with deny rules (Section 4). If escaping becomes routine, that's a sign the config isn't tight enough yet — use the log to steer toward a shape where you never need to drop the sandbox.

### Turning escape records into better settings

Bringing the number of sandbox escapes down isn't something you can do by intuition. Only once you have what escaped, when, and through which command in front of you does it become clear which setting to add. This is where the configuration actually grows.

**Record.** A `PreToolUse` hook picks out the calls that carry `dangerouslyDisableSandbox: true` and appends the timestamp and the command. It doesn't stop anything — stopping the work means it doesn't get done, and the log never fills up. The script and how to register it are in Use case 4 in Section 5. What you end up with is a `timestamp \t command` TSV.

**Read.** Even a weekly look, sorted by frequency, reveals your habits.

```bash
# what dropped the sandbox, and how often, most frequent first
cut -f2 ~/.claude/logs/sandbox-bypass.tsv | sort | uniq -c | sort -rn | head
```

Work down from the top. You don't have to fix everything at once — clearing the few most frequent entries visibly cuts the number of prompts.

**Put it into the settings.** Each pattern you read has a place it belongs.

| What the log shows | Where it goes |
|---|---|
| Repeated escapes to reach a specific host | Add the domain to `network.allowedDomains` |
| Escapes from writing outside the working directory | Add the path to `filesystem.allowWrite` |
| A specific tool (e.g. docker) escaping every time | List just that tool in `excludedCommands` |
| Something escaping that should never run at all | Block it with `permissions.deny` instead of excluding it |
| Escapes that are sporadic and hard to justify | Promote to Strict mode (`allowUnsandboxedCommands: false`) |

Look at the fourth row. **An escape doesn't automatically mean "allow it."** Reading the log sometimes tells you that this is something you never wanted run in the first place, and what belongs there is a deny rule, not a wider allowlist. The log isn't only material for opening things up.

**Check that it goes quiet.** After you widen an allowlist, run for a while and see whether the log stops. If it doesn't, that's a signal there's still a hole. This back-and-forth *is* the work of shaping a configuration you don't have to escape from.

### Tightening further (optional)

If you want to go all the way on hardening, consider these too.

- **`failIfUnavailable: true`** — when a dependency like bubblewrap is missing, or the platform is unsupported, Claude Code by default **just prints a warning and starts up with no sandbox** (your strongest layer drops silently). Set this to `true` and it refuses to start unless the sandbox comes up. For teams or CI that need "sandbox required" to actually mean it.
- **`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` (environment variable)** — sandboxed Bash inherits the parent process's environment as-is, which means any API key or cloud credential sitting in an env var is visible to child processes. Setting this strips Anthropic and cloud-provider credentials from subprocesses. The value is just `"1"`, and it goes in the `env` block of `settings.json`:

  ```json
  "env": {
    "CLAUDE_CODE_SUBPROCESS_ENV_SCRUB": "1"
  }
  ```

  Anything in `env` reaches both the session and the child processes it starts. Exporting `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` in your shell before launching `claude` does the same thing.

### Know the limits

The sandbox is strong, but it isn't a complete isolation boundary.

- The network proxy **decides by hostname alone and does not inspect TLS contents**. So allowing a broad domain like `github.com` leaves room to reach hosts outside the allowlist via domain fronting and similar tricks (Anthropic says so explicitly). Broad allows are convenient, but they're also an exfiltration path — keep the allowlist minimal.
- The built-in Read / Edit / Write tools don't go through the sandbox; they're governed by the permission system instead. So "reading a file" hits a different defense layer depending on whether it goes through Bash (sandbox applies) or a tool (permissions apply). Worth keeping straight.

> Designing defenses that minimize the scope of impact is known as the **principle of least privilege**.

---

## 3. Permission System

Use Claude Code's permission system to control what is allowed when a command or tool is invoked. Understand that there are four levels of rules.

### Permission levels

| Permission | Behavior | Purpose |
|------------|----------|---------|
| `deny` | Always blocked (no prompt) | Guardrails |
| `ask` | Always prompted (even if previously approved with "don't ask again") | Human-in-the-Loop — usually trusted, but create a checkpoint for review |
| `allow` | Always permitted (no prompt) | Convenience — for trusted operations where you want to skip confirmation |
| _(default / not configured)_ | Prompted on first use; "don't ask again" makes it permanent | The default judgment is in your head — you may say yes too quickly when busy, or can't tell safe from unsafe when unsure. (Everyone occasionally doubts their own reliability.) |

> `deny` is for things that should **never** happen. `allow` is for things you **always** trust. `ask` is for things you **usually** trust but want to verify.

### Where to put rules

These rules can be set under your HOME directory or per project (folder/directory/repository) shared with your team.

| File | Who | Scope | Git |
|------|-----|-------|-----|
| `~/.claude/settings.json` | You only | All your projects on this machine | Not in any repo |
| `<project>/.claude/settings.json` | Entire team | This project only | Committed and shared with everyone |
| `<project>/.claude/settings.local.json` | You only | This project only | Lives in the repo directory but gitignored — never pushed |

---

## 4. Thinking About Rules: Deny / Ask / Allow

This section lists specific rules organized by threat category. Each rule includes the rationale so you can decide whether it applies to your environment.

See [`settings_example.jsonc`](settings_example.jsonc) for a single file containing all rules below, with `allow` and `ask` examples and commentary in comments. Note that comments will cause JSON errors if left in place.

### 4.1 Deny — Destructive Git Operations

Prevent irreversible changes to your repository and its history.

```json
"Bash(git push -f *)",
"Bash(git push --force *)",
"Bash(git reset --hard *)",
"Bash(git checkout .)",
"Bash(git restore *)",
"Bash(git clean -f *)",
"Bash(git add -A)"
```

| Rule | Risk |
|------|------|
| `git push -f / --force` | Overwrites remote history. Can destroy teammates' work. |
| `git reset --hard` | Discards all uncommitted changes irreversibly. |
| `git checkout .` | Silently reverts all working tree changes. |
| `git restore` | The successor to `git checkout --`, and what modern git points you to. Reverts working tree changes the same way. Note it also stops `--staged`, which only unstages. |
| `git clean -f` | Deletes untracked files permanently. |
| `git add -A` | Stages everything — may accidentally include `.env`, credentials, or large binaries. (`git add .` is an everyday command, so it sits in `ask` in Section 4.12.) |

> **What you're really protecting is "don't break the repo":** these git denies can be sidestepped with `gh` or a raw `curl` to the API, so they aren't a wall — and piling on more deny rules to chase each bypass just makes the list unreadable. Take them for what they are: a few light, legible guardrails that say "nothing destructive here." (Blocking `github.com` at the network layer is a non-starter — you'd lose clone and pull and just cut the developer off.)
>
> The wall that actually holds lives on the remote: branch protection. Forbid force-pushes and deletion on `main`, require PRs, and the server turns them away whether they came from git, gh, or curl — while normal work carries on. Git is forgiving anyway: the reflog retains history for about 90 days, and remote-tracking branches and other clones are copies. The only things you truly can't get back are uncommitted changes (`reset --hard` or `checkout .` on a dirty tree) and untracked files (`git clean -fdx`) — so put just those behind `ask`.

### 4.2 Deny — Destructive File Operations

Prevent bulk file deletion that could wipe out project trees.

```json
"Bash(rm -rf *)",
"Bash(rm -r *)"
```

| Rule | Risk |
|------|------|
| `rm -rf` | Recursively deletes directories without confirmation. A wrong path can wipe out entire project trees. |
| `rm -r` | Same as above, but prompts in some configurations. Still too dangerous to allow unconditionally. |

One thing to know: even with both rules above, `rm -fr`, `rm -Rf`, and `rm --recursive` get through — those spellings simply don't match the patterns. The same operation has several spellings, and chasing every one of them is a losing game. The two above are the spellings you'll actually run into, though, so they earn their place. The day one of them fires, you'll be glad it was there.

Here's something I do myself: ban `rm` outright, and have anything you want gone moved to a trash folder (`trash/`, say) instead.

```json
"Bash(rm *)"
```

The ban isn't the point — the alternative is. Put "delete by moving to `trash/`, not with `rm`" in your `CLAUDE.md,` and Claude will offer that route from the start.

This works because **the sandbox does not protect the inside of your working directory**. Writes are permitted in the current directory and the session temp directory, so `rm -rf ./src` is entirely legal as far as the OS is concerned. Your own workspace is guarded by deny rules or by nothing.

### 4.3 Deny — Dangerous System Operations

Prevent permission changes and process kills that could destabilize your environment.

```json
"Bash(chmod 777 *)",
"Bash(chmod -R *)",
"Bash(chown -R *)",
"Bash(killall *)",
"Bash(pkill *)",
"Bash(kill -9 *)"
```

| Rule | Risk |
|------|------|
| `chmod 777` | Makes files world-readable/writable/executable. A common security anti-pattern. |
| `chmod -R / chown -R` | Recursive permission/ownership changes can break system directories or expose sensitive files. |
| `killall / pkill` | Terminates processes by name. Can kill unrelated critical processes. |
| `kill -9` | Force-kills without cleanup. Can cause data corruption in running applications. |

### 4.4 Deny — Privilege Escalation

Prevent Claude Code from running commands as root.

```json
"Bash(sudo *)",
"Bash(su *)"
```

An AI assistant should never escalate privileges. `sudo` requires a password, but it's more reliable to deny the attempt itself.

### 4.5 Deny — Remote Code Execution via Pipe

Piping a remote script straight into a shell (`curl ... | sh`) is a classic supply chain entry point. The tricky part is how naturally Claude Code suggests it, as a routine "install" step — and because it *looks like* an ordinary install, you reflexively hit "yes."

Stopping it takes a small trick. Writing the pipe into the pattern, as in `Bash(curl *|*sh)`, does nothing at all: deny rules split a command at the pipe and match each part on its own, so a pattern containing a pipe matches neither half.

Block the interpreter instead of the fetcher.

```json
"Bash(sh *)",
"Bash(bash *)",
"Bash(zsh *)"
```

The right-hand side of the pipe is matched by itself, so this stops `curl ... | sh` and `wget -O- ... | bash` alike — and it doesn't care which fetcher was used. The download-then-run form, `curl -o /tmp/a.sh && sh /tmp/a.sh`, is stopped for the same reason. The `sudo bash` variant is covered by `Bash(sudo *)` in Section 4.4.

`curl` and `wget` themselves stay out of the deny list and go to the `ask` list in Section 4.12. Fetching is too routine to block outright — deny it and you'll end up deleting the rule. With `ask`, the prompt shows the full command, including the pipe, so you can catch it right there.

Two limits are worth stating. `curl ... | python -` is not stopped by the rules above; enumerating interpreters is that losing game again, and from this point on it's the network allowlist in Section 2 that does the work, since an unapproved domain is unreachable to begin with. And legitimate runs like `bash build.sh` are stopped too, so move `bash` to `ask` if that gets in your way.

The surest move is still not to let Claude run these at all: take the command it proposes and type it yourself.

### 4.6 Deny — Remote Access

Prevent Claude Code from initiating connections to remote hosts. An AI assistant should not initiate remote connections. These commands can transfer files or execute commands on remote hosts.
When sandbox mode is enabled, network access is blocked at the OS level, but Claude Code can still attempt to execute these commands — that's why they're listed here.

```json
"Bash(ssh *)",
"Bash(scp *)",
"Bash(rsync *)"
```

If you need remote access, allow specific targets rather than a blanket allow.

### 4.7 Deny — Package Publishing & Deployment

Prevent unintended package publishing and deployment, even in CI/CD contexts.
Publishing packages or triggering deployments should be a deliberate human action. Unless explicitly designed into your workflow, an AI should not do this autonomously.

```json
"Bash(npm publish *)",
"Bash(yarn publish *)",
"Bash(pnpm publish *)",
"Bash(*deploy*)"
```

### 4.8 Deny — Easy to Approve, Hard to Undo (macOS)

Some macOS commands look harmless but can cause serious damage. Users tend to approve these without a second thought — that's exactly what makes them risky.

```json
"Bash(osascript *)",
"Bash(defaults write *)"
```

| Rule | Why it's easy to approve | Actual risk |
|------|-------------------------|-------------|
| `osascript` | "Just automating Finder" | AppleScript can send emails, control apps, access the keychain, and much more. |
| `defaults write` | "Just changing a setting" | Can modify security-critical macOS preferences, disable Gatekeeper, or alter app behavior. |

> **About `open`:** It sits in the `ask` list in Section 4.12 (prompt every time) rather than in deny. Opening files and URLs is a legitimate everyday need, and a blanket block just gets in the way. Check the target at the prompt before approving, and you can still stop a suspicious URL or a downloaded file right there.

* These rules are not needed on other operating systems, but consider the same approach for your platform's equivalents (contributions welcome).

### 4.9 Deny — Infrastructure

Prevent autonomous changes to cloud infrastructure.

```json
"Bash(terraform apply *)",
"Bash(terraform destroy *)",
"Bash(kubectl apply *)",
"Bash(kubectl delete *)",
"Bash(helm install *)",
"Bash(helm upgrade *)",
"Bash(docker push *)"
```

| Rule | Risk |
|------|------|
| `terraform apply / destroy` | Creates, modifies, or destroys cloud infrastructure. |
| `kubectl apply / delete` | Deploys or removes workloads on Kubernetes clusters. |
| `helm install / upgrade` | Installs or upgrades Kubernetes packages with broad cluster impact. |
| `docker push` | Publishes container images to registries. |

Cloud CLIs (`aws`, `gcloud`) aren't listed here — they're in the `ask` list in Section 4.12 instead. There are too many destructive subcommands to enumerate, and a pattern can't tell `aws s3 ls` from `aws s3 rm`. Rules that target flags, like `--no-cli-pager` or `--quiet`, run into the same wall: they only match when the flag lands at the end, and "no pager" says nothing about how destructive a command is. Narrowing the permissions themselves on the IAM side is the real fix.

### 4.10 Deny — Sensitive File Access

Prevent Claude Code from reading files that contain secrets.

```json
"Read(**/.env)",
"Read(**/.env.*)",
"Edit(**/.env)",
"Edit(**/.env.*)"
```

`.env` files typically contain API keys, database passwords, and other secrets.

A `Read` deny only closes off reading. To stop the file being rewritten as well, you need the `Edit` deny too — and since `.env` usually isn't in git, an overwrite isn't something you get back.

Consider adding more patterns for your environment:

```json
"Read(**/*.pem)",
"Read(**/*.key)",
"Read(**/credentials*)"
```

### 4.11 Deny — MCP (Model Context Protocol) Actions: Preventing Impersonation Messages

Prevent Claude Code from sending messages on your behalf (impersonation).
An AI assistant reading messages for context is different from it **sending** messages — the latter should require your explicit action.

```json
"mcp__claude_ai_Slack__slack_send_message",
"mcp__claude_ai_Slack__slack_schedule_message"
```

### 4.12 Ask — Human-in-the-Loop

It's hard to make everything black or white.
For useful, legitimate commands where **a human should still review each invocation**, use `ask`.

When you approve a command with "Yes, don't ask again", it becomes permanently allowed for that project. `ask` rules override this — Claude Code will **always** prompt you, even if you previously said "don't ask again". This is one way to implement the Human-in-the-Loop (HITL) approach.

```json
{
  "permissions": {
    "ask": [
      "Bash(git add .)",
      "Bash(git commit *)",
      "Bash(git push *)",
      "Bash(npm install *)",
      "Bash(pip install *)",
      "Bash(brew install *)",
      "Bash(psql *)",
      "Bash(mysql *)",
      "Bash(mongosh *)",
      "Bash(sqlite3 *)",
      "Bash(open *)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(aws *)",
      "Bash(gcloud *)"
    ]
  }
}
```

| Rule | Why `ask`, not `allow` or `deny` |
|------|----------------------------------|
| `git commit` | You want to review the commit message and what's being committed each time. |
| `git push` | Branch and remote operations should be verified each time. |
| `npm/pip/brew install` | Adding dependencies carries the risk of introducing vulnerabilities. Keep a review checkpoint. |
| `git add .` | Stages every file at once — handy, but easily sweeps in `.env` or large binaries. Confirm what's included each time. |
| `psql / mysql / mongosh / sqlite3` | Pattern matching can't distinguish `SELECT` from `DROP TABLE`. Read the SQL at the prompt and decide. |
| `open` (macOS) | Opening files and URLs is often legitimate, but the same command can launch arbitrary apps or open phishing URLs. Check the target at the prompt before letting it through. |
| `curl` / `wget` | Fetching is an everyday need. The prompt shows the entire command, so you can see whether it's being piped into a shell (see Section 4.5). |
| `aws` / `gcloud` | A pattern can't tell `s3 ls` from `s3 rm`. Read the subcommand at the prompt (Section 4.9). |

#### A note on database commands

Commands like `psql`, `mysql`, `mongosh`, and `sqlite3` can be destructive (`DROP TABLE`, `DELETE FROM`), but Claude Code's deny rules can't distinguish the content of command arguments. Denying `Bash(psql *)` blocks all usage, including harmless `SELECT` queries needed for analysis.

**Recommendation:** Don't block database commands wholesale with `deny` — catch them with `ask` instead. When the prompt appears, you can sort the harmless `SELECT` from the irreversible `DROP TABLE` with your own eyes.

### 4.13 Allow — Trusted Operations

Allow rules skip the permission prompt for commands you trust.
Useful for reducing prompt fatigue during safe, frequently used operations.

For example, test/build commands, safe git operations, MCP read-only actions, etc.

We've included samples, so see the commented-out `allow` section in [`settings_example.jsonc`](settings_example.jsonc).

---

## 5. Hooks — When Deny Rules Aren't Enough

Deny rules are enforced by the Claude Code harness (not the AI model), so Claude cannot "choose" to ignore them.
But database operations and access to sensitive files can't be judged from the surface of a command string. That's past the reach of deny's pattern matching.

Deny rules use glob pattern matching, which has inherent limitations:

Examples:
- `Bash(sudo *)` blocks `sudo rm -rf /` but not a script that internally calls `sudo`
- `Bash(sh *)` blocks `curl url | sh` but not `curl url | python -`
- `Bash(rm -rf *)` blocks `rm -rf /tmp` but not a Makefile target that runs `rm -rf` internally

This is where **Hooks** come in — shell scripts that run at specific points in Claude Code's lifecycle. The most useful one is `PreToolUse` — right before a tool call executes. Deny rules only do pattern matching on the command string, so they can only judge whether the surface text matches. Hooks receive the full command as JSON, allowing much finer-grained decisions. See the examples below.

### Detailed explanation

From the [official documentation](https://code.claude.com/docs/en/permissions):

> Read and Edit deny rules apply to Claude's built-in file tools and to file commands Claude Code recognizes in Bash, such as `cat`, `head`, `tail`, and `sed`. They don't apply to arbitrary subprocesses that read or write files indirectly, like a Python or Node script that opens files itself.

So `Read(**/.env)` in your deny list does stop the straightforward reads, `cat .env` included. What gets through is everything past that: a script that opens the file from the inside, or a tool Claude Code doesn't read as a file-reading command. A deny list checks the command string, not what the process actually opens.

### Use case 1: Block destructive SQL in database commands

**Problem:** Denying `Bash(psql *)` blocks `SELECT` too. But you want to catch `DROP TABLE` or `DELETE FROM` before they execute.

**Hook script** — save as `~/.claude/hooks/block-destructive-sql.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# Check if the command contains destructive SQL keywords
if echo "$CMD" | grep -iqE '(DROP\s|DELETE\s+FROM|TRUNCATE\s|ALTER\s+TABLE.*DROP)'; then
  echo "Blocked: destructive SQL detected in command: $CMD" >&2
  exit 2
fi

exit 0
```

```bash
chmod +x ~/.claude/hooks/block-destructive-sql.sh
```

**Settings** — add to your `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-destructive-sql.sh"
          }
        ]
      }
    ]
  }
}
```

Now `psql -c "SELECT * FROM users"` proceeds normally, but `psql -c "DROP TABLE users"` is blocked with a reason.

### Use case 2: Block Bash from reading sensitive files

**Problem:** A `Read(**/.env)` deny does stop `cat .env`. What's left is the routes Claude Code doesn't read as file-reading commands. `xxd`, `strings`, `base64`, and `od` will happily print the contents, and once something is opened from inside `python -c` you have nothing to match on.

**Hook script** — save as `~/.claude/hooks/block-sensitive-reads.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

SENSITIVE_PATTERNS='\.env|\.pem|\.key|id_rsa|id_ed25519|credentials'

# catch the read paths the deny rules don't look at
READERS='xxd|strings|base64|od|hexdump|dd|python3?|node|perl|ruby'

if echo "$CMD" | grep -iqE "(${READERS})\s.*(${SENSITIVE_PATTERNS})"; then
  echo "Blocked: reading sensitive file via Bash: $CMD" >&2
  exit 2
fi

exit 0
```

This is still enumerating command names, so it's not airtight. Write `python3.12` instead of `python,` and it slips by; write the script to a file first, and there's nothing in the command string to match at all. The way to actually seal a path is `filesystem.denyRead` in Section 2 — the OS stops the process that used to open the file. Treat this hook as the fallback for environments where you can't run the sandbox.

**Settings** — same structure, same matcher:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-sensitive-reads.sh"
          }
        ]
      }
    ]
  }
}
```

### Use case 3: Block push to main/master branch

**Problem:** `git push` is useful so you've set it to `ask`, but an accidental approval can push directly to main. The deny rule `Bash(git push *)` can't distinguish branches.

**Hook script** — save as `~/.claude/hooks/block-push-to-main.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# Skip if not a git push command
echo "$CMD" | grep -qE '^\s*git\s+push' || exit 0

# Explicit main/master in the command
if echo "$CMD" | grep -qE '\b(main|master)\b'; then
  echo "Blocked: push to main/master is not allowed: $CMD" >&2
  exit 2
fi

# Implicit push (no args or remote only) — check current branch
if echo "$CMD" | grep -qE '^\s*git\s+push\s*$' || \
   echo "$CMD" | grep -qE '^\s*git\s+push\s+(-[a-zA-Z]+\s+)*[a-zA-Z0-9_.-]+\s*$'; then
  CURRENT=$(git branch --show-current 2>/dev/null)
  if [ "$CURRENT" = "main" ] || [ "$CURRENT" = "master" ]; then
    echo "Blocked: currently on $CURRENT — push to main/master is not allowed" >&2
    exit 2
  fi
fi

exit 0
```

```bash
chmod +x ~/.claude/hooks/block-push-to-main.sh
```

**Settings** — add to your `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-push-to-main.sh"
          }
        ]
      }
    ]
  }
}
```

This blocks `git push origin main` and bare `git push` when on the main branch. Pushes to feature branches go through normally.

> **Note:** This will also match branch names that contain "main" or "master" as a substring, such as `feature/main-cleanup`. Adjust the pattern if needed.

### Use case 4: Record sandbox escapes

**Problem:** Even with the sandbox enabled, Claude may retry a failed command outside it with `dangerouslyDisableSandbox` (see Section 2). You don't necessarily want to seal it shut (`allowUnsandboxedCommands: false`), but you do want to know *when* and *which command* escaped, so you can revisit the allowlists later.

**Idea:** Don't block — just record. A `PreToolUse(Bash)` hook detects calls carrying `dangerouslyDisableSandbox: true` and appends a timestamp and the command to a TSV. The exit code stays 0 (the command still runs). Skim the log periodically and you'll spot things like "add this domain to `allowedDomains` and it stops escaping" or "this tool belongs in `excludedCommands`."

**Hook script** — save as `~/.claude/hooks/log-sandbox-bypass.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
LOG=~/.claude/logs/sandbox-bypass.tsv

# Only record when dangerouslyDisableSandbox is true; otherwise pass through
DISABLED=$(echo "$INPUT" | jq -r '.tool_input.dangerouslyDisableSandbox // false')
if [ "$DISABLED" = "true" ]; then
  CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')
  mkdir -p "$(dirname "$LOG")"
  printf '%s\t%s\n' "$(date -Iseconds)" "$CMD" >> "$LOG"
fi

exit 0   # record only — do not block execution
```

```bash
chmod +x ~/.claude/hooks/log-sandbox-bypass.sh
```

**Configuration** — same structure, same matcher (combines fine with other Bash hooks):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/log-sandbox-bypass.sh"
          }
        ]
      }
    ]
  }
}
```

Now `~/.claude/logs/sandbox-bypass.tsv` holds a chronological record of every moment the sandbox was dropped, as a `timestamp \t command` TSV. Unlike Use cases 1–3, which block, this one **observes** without stopping anything.

Reading the log, and turning what you find into settings, is covered in "Turning escape records into better settings" in Section 2.

### Combining multiple hooks

You can register multiple hook scripts under the same event. They all run, and if any one exits with code 2, the call is blocked:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/block-destructive-sql.sh"
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/block-sensitive-reads.sh"
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/block-push-to-main.sh"
          }
        ]
      }
    ]
  }
}
```

### Keeping a record — denial logs and audit trails

Operations blocked by deny rules appear in the session, but by default nothing is written to a file. Being able to look back at *when* and *what* you stopped is what lets the policy grow. Two ways to record it.

**Log it from your own hook.** Writing a log from `PreToolUse`, the way Use case 4 does, is quick and readable with a plain `grep`. Note though that `PreToolUse` runs *before* the permission decision, so what you get is "here's what Claude tried to run," not "this one was denied." The blocking hooks (Use cases 1–3) are different: the moment of the block happens inside your own script, so one extra line before `exit 2` records the outcome as well.

**Send it to OpenTelemetry.** For the verdict itself, this is the route ([official docs](https://code.claude.com/docs/en/monitoring-usage)). The `claude_code.tool_decision` event carries `decision` (accept/reject) and `source` (`config` for a deny rule in settings, `hook`, `user_reject`, and so on), so you can tell **what blocked it**. For a team or an enterprise that aggregates and analyzes this, it's the real answer.

---

## 6. Secrets and Credentials

> **This is the one section where a mistake can't be undone later.** Get the sandbox or an approval rule wrong and you fix the setting. A leaked secret doesn't work that way — rotation is the only way out. Read this one more carefully than the rest.

Three things get tangled together here. Take them apart.

**(1) Claude Code's own credentials**

On macOS, these go into the Keychain when it's available; on Windows and Linux, they're protected by file permissions. Leave this at the default.

If you authenticate with an API key, you can store the fetch method in your settings instead of the value itself.

```json
{ "apiKeyHelper": "/usr/local/bin/get-my-key" }
```

Through Bedrock or Vertex, `awsAuthRefresh`, `awsCredentialExport`, and `gcpAuthRefresh` play the same role. All of them are places to write *how to get the value*, not the value.

**(2) How you hand over your own API keys**

There's one rule.

> **Never put a value in the `env` block of `settings.json`.**

No exceptions. The line you added because "I'll delete it later" or "it's only local" rides along into your sync, your backups, your screenshots, and your screen shares. If your home directory syncs to the cloud, it's replicated the instant you save. The same goes for `.claude/settings.local.json` — gitignored isn't the same as protected, and sync doesn't care what git thinks.

When something has to reach an MCP server, inject it at launch and keep it confined to that process.

```bash
eval $(get-secret --export API_KEY MY_TOKEN) && exec my-mcp-server
```

Don't `export` it into an interactive shell and walk away. Don't write it back into a `.env`. For storage, you have the OS keychain, [1Password CLI](https://developer.1password.com/docs/cli/), [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/), [HashiCorp Vault](https://developer.hashicorp.com/vault), your cloud's Secret Manager, [`pass`](https://www.passwordstore.org/), or [SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age). Whichever you pick, the aim is the same: **no plaintext value left in a config file or your shell history, and the value retrieved only when it's needed.**

**(3) Keep environment variables out of child processes**

Bash inside the sandbox inherits the parent process environment by default. A key you put in an environment variable is visible from there.

```json
{
  "sandbox": {
    "credentials": {
      "files": [{ "path": "~/.aws/credentials", "mode": "deny" }],
      "envVars": [{ "name": "GITHUB_TOKEN", "mode": "deny" }]
    }
  }
}
```

`"mode": "deny"` makes the file unreadable and unsets the variable before each sandboxed command runs. There's no built-in deny list, so **only what you list is protected**. To strip Anthropic and cloud provider credentials in one move instead, `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` from Section 2 is the faster route.

One more thing worth knowing: the sandbox refuses writes to `settings.json` at every scope. Rewriting the configuration to widen its own limits isn't a path that's open.

---

## 7. Minimize Data Retention (transcripts and session logs)

Sandboxing, permissions, and hooks control *what Claude can do*. From here on it's about **how much trace it leaves behind**. Claude Code stores conversation transcripts and session logs locally, under `~/.claude/projects/<project>/*.jsonl`. Source code, design discussions, file contents — and occasionally a credential someone pasted — all land there.

Don't leave the retention period to the default. **Set it explicitly.** It's basic data minimization, and it fixes the window during which a secret that made it into a transcript stays on your disk.

### How to set it

Set `cleanupPeriodDays` (in days) in `settings.json`. **Unset, it defaults to 30 days**, after which files are deleted automatically.

```jsonc
{
  // How many days transcripts and session logs stay on disk before automatic deletion.
  // The default when unset is 30 days. Don't rely on the default — say what you want.
  "cleanupPeriodDays": 30
}
```

### Choosing a number

A longer window gives you more to recover and analyze from, and leaves traces of secrets around longer. Three things decide it:

- **Recovery has a short horizon.** Restoring an interrupted or crashed session rarely needs to reach back more than a few days to a week.
- **Keep the durable knowledge somewhere else.** Distill conclusions, decisions, and operating rules into notes or memory rather than leaving them in transcripts. Do that and the transcript becomes a work log you're free to lose, which is what makes a short retention safe.
- **How tight do you want the exposure window?** The shorter the value, the sooner anything sensitive in a transcript is gone from your disk.

> **A rule of thumb**: for most individuals and small teams, around 30 days is a comfortable place to sit — enough for recovery, and a month's worth for analysis. If you want traces gone sooner, 14 days is practical; two weeks covers recovery in almost every case. Stretching to 60 or 90 days is hard to justify unless you actually go back and dig through old transcripts.

### Related: trim what gets sent

Alongside local retention, you can narrow what leaves the machine, with environment variables. Same data-minimization thinking.

```jsonc
{
  "env": {
    "DISABLE_TELEMETRY": "1",
    "DISABLE_ERROR_REPORTING": "1"
  }
}
```

> Going as far as `DISABLE_AUTOUPDATER` also stops updates, security fixes included, so that one depends on what you're after. Treat minimizing retention (`cleanupPeriodDays`) and minimizing transmission (telemetry, error reporting) as separate layers, and set both deliberately.

---

## 8. Check That Your Settings Are Actually in Effect

Writing a setting doesn't mean it's working. Settings live in several files and the higher scope overrides the lower one. Take one pass at the end to confirm.

- **`/sandbox`** — the Config tab shows the resolved sandbox settings after every scope is merged. If a key you thought you set isn't there, it isn't in effect. The Overrides tab tells you whether the escape hatch is sealed (Strict mode).
- **`/permissions`** — the allow/ask/deny rules currently in force. The Recently denied tab shows what was stopped lately.
- **`claude doctor`** — runs outside a session. It checks the health of your installation and reads the settings files in the current directory without a trust prompt.

Then actually run something. Does a `curl` to a domain you didn't allow get stopped? Can anything write outside the working directory? **The most common accident in hardening is assuming a setting took effect.**

Note that Claude Code has no dedicated subcommand to run a one-off command in the sandbox. Verification comes down to two things: reading the resolved settings, and trying it for real.

---

## References

### Official Documentation

- [Claude Code Security Best Practices](https://code.claude.com/docs/en/security)
- [Claude Code Settings](https://code.claude.com/docs/en/settings)
- [Claude Code Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Code Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide)

## Related Cheat Sheets & Further Reading

- [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) — Covers key risks and best practices for AI agent systems: tool permission minimization, prompt injection prevention, human-in-the-loop controls, and more.
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Prompt_Injection_Prevention_Cheat_Sheet.html) — Technical guidance on defending against prompt injection attacks.
- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) — The broader threat landscape for LLM-powered applications.
