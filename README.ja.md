# claude-code-hardening-cheatsheet

**[English version](README.md)**

[Claude Code](https://code.claude.com/) の `~/.claude/settings.json` に適用するセキュリティ強化チートシートです。

Claude Code はシェルコマンドの実行、ファイルの読み取り、外部サービスとの連携が可能です。これらの設定は、Claude Code に**やらせるべきでないこと**を明確に制限し、安心して**やらせたいこと**に集中できるようにします。

## クイックスタート

**方法A: スクリプトで適用**

```bash
git clone https://github.com/okdt/claude-code-hardening-cheatsheet.git
cd claude-code-hardening-cheatsheet
chmod +x hardening-claude-code-env.sh
./hardening-claude-code-env.sh
```

**方法B: サンプルを参考にする**

[`settings-example.jsonc`](settings-example.jsonc) を開き、必要なルールを選んで `~/.claude/settings.json` に追加してください。サンプルは JSONC（コメント付き）ですが、`settings.json` は素の JSON のみ対応なので、コメントはコピーしないこと。

## チートシート

各ルールの詳しい解説はチートシート本体を参照：

- [Claude Code Hardening Cheatsheet (English)](ClaudeCodeHardeningCheatSheet.md)
- [Claude Code Hardening Cheatsheet (日本語)](ClaudeCodeHardeningCheatSheet.ja.md)

## ファイル構成

| ファイル | 説明 |
|---------|------|
| [`ClaudeCodeHardeningCheatSheet.md`](ClaudeCodeHardeningCheatSheet.md) | チートシート本体 — deny/allow/ask ルールの解説と根拠 |
| [`settings-example.jsonc`](settings-example.jsonc) | `settings.json` のサンプル — 全ルールと allow/ask の例をコメントアウトで収録 |
| [`hardening-claude-code-env.sh`](hardening-claude-code-env.sh) | 対話型スクリプト — ベースラインの deny ルールを適用 |

## 作者

[okdt](https://github.com/okdt)

## 参考資料

- [セキュリティ](https://code.claude.com/docs/ja/security)
- [Claude Code の設定](https://code.claude.com/docs/ja/settings)
- [権限を設定する](https://code.claude.com/docs/ja/permissions)
- [Claude Codeの設定でやるべきセキュリティ対策](https://qiita.com/dai_chi/items/f6d5e907b9fee791b658)

## ライセンス

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ja) — 帰属表示をすれば、自由に利用・改変・再配布できます。
