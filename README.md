# claude-code-hardening-cheatsheet

**[日本語版はこちら (Japanese)](README.ja.md)**

A security hardening cheatsheet for [Claude Code](https://code.claude.com/) `~/.claude/settings.json`.

Claude Code is powerful — it can run shell commands, read files, and interact with external services. These settings restrict what it's **not allowed** to do, so you can focus on what it **should** do.

## Quick Start

**Option A: Run the script**

```bash
git clone https://github.com/okdt/claude-code-hardening-cheatsheet.git
cd claude-code-hardening-cheatsheet
chmod +x hardening-claude-code-env.sh
./hardening-claude-code-env.sh
```

**Option B: Use the example as reference**

Open [`settings-example.jsonc`](settings-example.jsonc), pick the rules you need, and add them to your `~/.claude/settings.json`. The example uses JSONC (comments), but `settings.json` requires plain JSON — don't copy comments into it.

## Cheatsheet

Read the full cheatsheet for detailed explanations of each rule:

- [Claude Code Hardening Cheatsheet (English)](ClaudeCodeHardeningCheatSheet.md)
- [Claude Code Hardening Cheatsheet (日本語)](ClaudeCodeHardeningCheatSheet.ja.md)

## Files

| File | Description |
|------|-------------|
| [`ClaudeCodeHardeningCheatSheet.md`](ClaudeCodeHardeningCheatSheet.md) | Full cheatsheet — deny/allow/ask rules explained with rationale |
| [`settings-example.jsonc`](settings-example.jsonc) | Example `settings.json` with all rules and commented-out allow/ask examples |
| [`hardening-claude-code-env.sh`](hardening-claude-code-env.sh) | Interactive script — applies a baseline set of deny rules |

## Author

[okdt](https://github.com/okdt)

## References

- [Claude Code Security Best Practices](https://code.claude.com/docs/en/security)
- [Claude Code Settings](https://code.claude.com/docs/en/settings)
- [Claude Code Permissions](https://code.claude.com/docs/en/permissions)
- [Claude Codeの設定でやるべきセキュリティ対策](https://qiita.com/dai_chi/items/f6d5e907b9fee791b658) (Japanese)

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Free to use, share, and adapt with attribution.
