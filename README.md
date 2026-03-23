# claude-code-hardening-cheatsheet

**[日本語版はこちら (Japanese)](README.ja.md)**

A security hardening cheatsheet for [Claude Code](https://code.claude.com/) `~/.claude/settings.json`.

Claude Code is powerful — it can run shell commands, read files, and interact with external services. These settings restrict what it's **not allowed** to do, so you can focus on what it **should** do.

## Cheatsheet

Read the full cheatsheet for detailed explanations of each rule:

- [Claude Code Hardening Cheatsheet (English)](ClaudeCodeHardeningCheatSheet.md)
- [Claude Code Hardening Cheatsheet (日本語)](ClaudeCodeHardeningCheatSheet.ja.md)

## Files

| File | Description |
|------|-------------|
| [`ClaudeCodeHardeningCheatSheet.md`](ClaudeCodeHardeningCheatSheet.md) | Full cheatsheet — deny/allow/ask rules explained with rationale |
| [`settings-example.jsonc`](settings-example.jsonc) | Example `settings.json` with all rules and commented-out allow/ask examples |

## Author

Riotaro OKADA ([okdt](https://github.com/okdt))

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Free to use, share, and adapt with attribution.
