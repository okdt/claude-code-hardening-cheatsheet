# claude-code-hardening-cheatsheet

**[English version](README.md)**

[Claude Code](https://code.claude.com/) の `~/.claude/settings.json` に適用するセキュリティ強化チートシートです。

Claude Code はシェルコマンドの実行、ファイルの読み取り、外部サービスとの連携が可能です。これらの設定は、Claude Code に**やらせるべきでないこと**を明確に制限し、安心して**やらせたいこと**に集中できるようにします。

## チートシート

各ルールの詳しい解説はチートシート本体を参照：

- [Claude Code Hardening Cheatsheet (English)](ClaudeCodeHardeningCheatSheet.md)
- [Claude Code Hardening Cheatsheet (日本語)](ClaudeCodeHardeningCheatSheet.ja.md)

## ファイル構成

| ファイル | 説明 |
|---------|------|
| [`ClaudeCodeHardeningCheatSheet.md`](ClaudeCodeHardeningCheatSheet.md) | チートシート本体 — deny/allow/ask ルールの解説と根拠 |
| [`settings-example.jsonc`](settings-example.jsonc) | `settings.json` のサンプル — 全ルールと allow/ask の例をコメントアウトで収録 |

## 作者

Riotaro OKADA ([okdt](https://github.com/okdt))

## ライセンス

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ja) — 帰属表示をすれば、自由に利用・改変・再配布できます。
