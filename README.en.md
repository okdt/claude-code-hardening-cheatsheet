# Claude Code Hardening Cheatsheet

**[日本語版 (Japanese)](README.md)**

A cheatsheet and configuration samples for running Claude Code on the safer side.

This repository has two goals:

- Distill general hardening principles into practical, day-to-day Claude Code settings
- Provide ready-to-use configuration examples covering `sandbox` / `permissions` / `hooks`

This is not official Anthropic documentation. Always verify against the official docs and your current Claude Code version before applying to production.

**Version 1.2 (2026-08-02) / Verified against Claude Code v2.1.220, macOS.** Everything here has been checked against both the official documentation and a live install.

## Included Files

- [Claude_Code_Hardening_Cheat_Sheet.en.md](./Claude_Code_Hardening_Cheat_Sheet.en.md)
  Main cheatsheet — hardening principles, recommended settings, and operational notes
- [Claude_Code_Hardening_Cheat_Sheet.ja.md](./Claude_Code_Hardening_Cheat_Sheet.ja.md)
  The Japanese original, which the English version is kept aligned with
- [settings_example.jsonc](./settings_example.jsonc)
  Commented `settings.json` template — all rules with allow/ask examples in comments
- [Claude_Code_Hardening_Audit_Prompt.en.md](./Claude_Code_Hardening_Audit_Prompt.en.md) / [.ja.md](./Claude_Code_Hardening_Audit_Prompt.ja.md)
  A prompt for handing the cheatsheet to Claude Code and working through your own environment with it
- [EDITORIAL.md](./EDITORIAL.md)
  Editorial policy for revising this cheatsheet (for contributors, written in Japanese)

## What's new in 1.2 (2026-08-02)

- Three new sections: secrets and credentials, data retention, and checking that your settings are in effect
- A prompt for running a check-and-improve pass on your own environment
- Revised deny / ask rules

## How To Use

This document is structured for progressive adoption, from beginners looking for safe defaults to advanced users fine-tuning settings for their specific needs.

- **Beginners:** Start by enabling the sandbox (Section 2). This alone makes a significant difference
- **Practitioners:** Design project-appropriate permissions with deny / ask / allow rules (Sections 3–4)
- **Advanced:** Add custom checks with Hooks (Section 5) for cases that pattern matching cannot handle

The template [`settings_example.jsonc`](settings_example.jsonc) contains all rules with commented allow/ask examples. Pick the rules you need and copy them into your `settings.json` (comment lines must be removed first).

### Keep it locally and hand it to Claude Code

With the cheatsheet and the audit prompt on your machine, you can have your own Claude Code work through your environment with you.

```bash
curl -O https://raw.githubusercontent.com/okdt/claude-code-hardening-cheatsheet/main/Claude_Code_Hardening_Cheat_Sheet.en.md
curl -O https://raw.githubusercontent.com/okdt/claude-code-hardening-cheatsheet/main/Claude_Code_Hardening_Audit_Prompt.en.md
```

Start Claude Code in that directory and paste the contents of `Claude_Code_Hardening_Audit_Prompt.en.md`.

## Scope

This repository covers:

- Claude Code sandbox configuration
- Permission policies (deny / ask / allow)
- Advanced custom checks via Hooks
- Logging denied operations

The following are out of scope:

- Security quality of generated code (that concerns what Claude Code writes, not how it behaves)
- Enterprise-specific DLP / SIEM / EDR designs
- Replacement for official Anthropic documentation
- One-size-fits-all configurations

## Notes

- Configuration keys and behavior may change across Claude Code versions
- Primarily written and tested on macOS, but most rules apply equally to Linux and Windows (WSL). Platform-specific rules are marked as such
- The deny lists are samples, not exhaustive. They are organized by risk perspective
- The cheatsheet references OWASP GenAI / Prompt Injection resources and covers secure design principles including Human-In-The-Loop, least privilege, and defense in depth

## Author

Riotaro OKADA ([okdt](https://github.com/okdt))

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) — Free to use, share, and adapt with attribution. Derivatives must use the same license.

## References

- [Claude Code official documentation](https://code.claude.com/docs/en)

## Related Document

- [Codex CLI Hardening Cheatsheet](https://github.com/okdt/codex-cli-hardening-cheatsheet) — Hardening cheatsheet for OpenAI Codex CLI
