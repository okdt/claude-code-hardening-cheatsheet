# Claude Code Hardening Cheatsheet

> バージョン 1.2（2026-08-02）
>
> 検証環境: Claude Code v2.1.220（2026-08-01 時点）、macOS。設定キーと挙動は版によって変わります。本文の記述は、公式ドキュメントと実機の双方で確認しています。

---

## 1. はじめに

Claude Code は、あなたに代わってシェルコマンドを実行し、ファイルを読み書きし、外部サービスとつなぎます。この便利さの正体は「あなたの権限をそのまま使えること」です。裏を返せば、あなたが意図していない操作も、同じ権限でやれてしまう、ということでもあります。このチートシートは、その振れ幅を制御するために書きました。まずはサンドボックスだけでも入れておくと効きます。チームのベースラインを決めたい人は、ここを土台にしてください。

### リスク：望まれないエージェントの崩壊 — なぜハードニング（セキュリティ堅牢化）設定が必要なのか

- **エージェントの善意の暴走** — Claude Code は技術的には正しくても、あなたの意図を超えた操作をすることがあります。「整理」のためにファイルを削除したり、「修正」のために force-push したり、頼んでいないパッケージをインストールしたりします。（[OWASP LLM09: Overreliance](https://genai.owasp.org/llm-top-10/)）
- **大きすぎる権限** — デフォルトでは、Claude Code はあなたのユーザーアカウントでできることは何でもできます。deny ルールがなければ、たった一度の「はい」で破壊的なコマンド、認証情報ファイル、リモートシステムへのアクセスを許してしまいます。（[OWASP LLM06: Excessive Agency](https://genai.owasp.org/llm-top-10/)）
- **間接的プロンプトインジェクション** — Claude Code が処理するコンテンツ（ソースコード、ドキュメント、Webページ）に、動作に影響を与える指示が紛れ込んでいることがあります。攻撃者は、Claude が通常の作業中に読むファイルや依存関係に悪意あるプロンプトを埋め込むことができます。（[OWASP LLM01: Prompt Injection](https://genai.owasp.org/llm-top-10/)）
- **秘密の置き場所が増える** — Claude Code を使うと、API キーやトークンが平文で残りうる場所が増えます。設定ファイル、環境変数、そして会話の transcript です。ホームディレクトリの配下を——`~/Documents` だけ、といった一部でも——クラウド同期していれば、書いた瞬間に全端末へ複製されます。ほかのリスクは設定を直せば止まりますが、**漏れた秘密だけは直せません**。ローテーションするしかありません。それで、この一点は他より慎重に扱います（§6）。（[OWASP LLM02: Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/)）
- **すでに侵害された環境でも被害を抑える** — マシンが RCE（Remote Code Execution）やマルウェア、サプライチェーン攻撃にやられた場合、Claude Code もその影響下に入ります。ハードニングでブラスト半径 — 被害の及ぶ範囲 — を小さくしておけば、攻撃者が Claude Code を踏み台にしても、やれることを絞れます。

これらは仮定の話ではありません。ガードレールが存在する理由です。問題が起きたとき — いずれ必ず起きますが... — 被害を封じ込めたい、そんな自分とみなさんのために書きました。
このチートシートは `~/.claude/settings.json` を通じて Claude Code 環境をハードニング（堅牢化）するための実践的なガイドです。最小権限の原則や Human-In-The-Loop（HITL）をどのように実践するかを考えるのに良い題材でしょう。

### 基本的なアプローチ

1. **サンドボックス(sandbox)** - サンドボックスとは、OS レベルのプロセス隔離技術を使って、Claude Code（とそこから起動される子プロセス）のファイル・ネットワークアクセスを制限する仕組みです。これはOSカーネルレベルなので、AI側から迂回できません。基本的な、かつ最強の防御層です。
2. **パーミッション(allow/deny/ask/default)** - Claude Code のコンソールからツール（Bash コマンド、ファイル編集など）が呼び出されたときに「常に許可 / 毎回確認 / 拒否 / デフォルト（設定しない）」を制御するルールです。パーミッションはツール呼び出し単位のきめ細かいアクセス制御を担います。ask の活用で **Human-In-The-Loop** - 目視確認の仕組み化が可能になります。
3. **フック(hooks + PreToolUse)** — ツール呼び出しの前後にシェルスクリプトを自動的に実行する仕組みです。パーミッションの allow/deny では対応しきれない、よりきめ細かいパターンマッチや環境固有のカスタムチェックを差し込めます。
4. **ログ** - エンタープライズの現場や、この仕組み自体のデバッグで要ることがあります。簡単に触れています。
5. **シークレット管理(secrets management)** - API キーやトークンをどこに置き、どこに置かないか——設定ファイル・環境変数・transcript のどこにも平文を残さないための道具立てです。1〜4 とは性質が違います。設定を間違えても直せますが、漏れた秘密は直せません。
6. **データ保持(data retention)** - 会話の transcript やセッションログを、どれだけの期間ローカルに残すか——何をさせるかではなく、どれだけ痕跡を残すかの話です。保持期間は明示的に決めておきます。

> **ポイント:** 防御の手立てを1段ではなく、何段も重ねて仕込む。この考え方を **多層防御** といいます。一枚が破られても、次の層が残る。

### CLAUDE.md に書いておけばいいのでは

`CLAUDE.md`（プロジェクトルートや `~/.claude/CLAUDE.md`）は、環境に関する目的やコンテキスト、作業のあらましを記載しておくところです。「`main` に直接 push しないで」「テストが通るまでコミットしないで」といった許可・不許可のポリシーを列挙することに意味がないとまでは言えませんが、そもそも CLAUDE.md に書ける内容には量的な限界があり（200行程度が推奨されています）、コンテキストの記載ならまだしも拒否ポリシーの置き場としては向いていません。加えて、書いたとしてもせいぜい「お願い」レベルで、よく忘れられます。

「起きてほしくないこと」に対しては、お願いではなく強制で止める——ではどうしたら良いのか、という点を見ていきましょう。

### サンプルについてのおことわり

3層の組み合わせで保護するイメージです。Deny ルールで基本的なケースをブロックし、サンドボックスが境界の原則に基づいてアクセスを制限し、Hooks でさらに詳細な条件を設定していきます。

ここで、何をブロックし、何を許可し、何を常に確認すべきか、そして deny ルールだけでは不十分な場合にどうすべきかなどを解説します。なお、本ドキュメントの deny リストはサンプルであり、コンプリートリストではありません。リスクから見て抑制したい視点からスタートしています。また、主に macOS 環境で執筆・検証していますが、Linux や Windows（WSL: Windows Subsystem for Linux）でも参考になるはずです。

ちなみに、この文書はまるごと Claude Code に読ませて使えます。自分の Claude Code 利用環境を堅牢化する作業を、まさにそのあなたの Claude Code に検討させられるわけです。環境のチェックと改善を進めるためのプロンプトを同梱してあります（[`Claude_Code_Hardening_Audit_Prompt.ja.md`](Claude_Code_Hardening_Audit_Prompt.ja.md)）。

自分の環境や操作リスクに合わせてカスタマイズしてください。

---

## 2. サンドボックス

サンドボックスは Claude Code のファイル・ネットワークアクセスを OS レベルで分離します。deny ルールがバイパスされた場合でも、定義された境界外のリソースへのアクセスを防ぎます。設定できる対策の中で一番堅いので、まず有効にしておいてください。

macOS（Seatbelt）、Linux・WSL2（bubblewrap）に対応しています。WSL1 とネイティブ Windows は未対応です。bubblewrap が要求するカーネル機能が WSL2 にしか無いためで、Windows では WSL2 の中で動かすことになります。

### 有効化の方法

- Claude Code の対話モードで `/sandbox` を実行してください。メニューからサンドボックスの有効化とモード設定ができます。

- 毎回入力するのは面倒ですし抜け漏れが起きますので、以下の設定をしてください。

`settings.json` で直接設定する: 

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

| 設定 | 理由 |
|------|------|
| `enabled: true` | OS レベルでファイル・ネットワークアクセスを分離する。**書き込み**はカレントディレクトリと明示的に許可したパスのみ。一方で**読み取り**は「全体が許可、`denyRead` で塞いだ場所だけ不可」がデフォルト。だから認証情報の保護は下の `denyRead` が要 |
| `autoAllowBashIfSandboxed` | サンドボックスが有効な間は Bash コマンドの許可プロンプトを省略。スコープが OS で制約されているため安全。省略されるのは既定のプロンプトと `Bash` 単体（`Bash(*)`）の ask だけで、`Bash(curl *)` のように中身を指定した ask と、すべての deny はサンドボックス中でも効く |
| `filesystem.denyRead` | サンドボックス内でも読み取りを禁じるパス。読み取りはデフォルト全許可なので、認証情報ストアはここで明示的に塞ぐ。例では SSH 鍵・GPG 鍵・AWS 認証情報・GCP 設定が対象。**OS レベルの強制なので、そのパスを開こうとするプロセスは何であれ止まる**。permission の `Read()` deny も `cat` のような認識済みコマンドまでは効くが、自作スクリプトが自力でファイルを開く経路までは追えない。そこが層の違い |
| `network.allowedDomains` | サンドボックス内の Bash が接続してよいドメイン。デフォルトは何も許可されておらず、新しいドメインへの初回接続ごとに確認プロンプトが出る。常用先を列挙すればプロンプトを省ける。逆に個別に塞ぎたいドメインは `deniedDomains` に入れる（広い許可を上書きしてブロック） |

> **ポイント:** ファイル分離とネットワーク分離は**対**で意味を持ちます。ネットワークを開けっ放しにすると、ファイル読み取りを一箇所許してしまっただけで SSH 鍵などをそのまま外部へ送信されかねません。逆もまた然り。片方だけでは穴になります。

### サンドボックスを一時的に外して抜けてくる問題（dangerouslyDisableSandbox）

サンドボックスを有効にしても、Claude はコマンドがサンドボックス内で失敗すると、`dangerouslyDisableSandbox` を付けてサンドボックス**外**で再試行することがあります。これは仕様で、サンドボックス非対応のツール（docker など）や、許可していないホストへの接続が必要なときに起きます。再試行はデフォルトで許可プロンプトを通るので「黙って抜ける」わけではありませんが、急いでいると反射的に Yes を押しがちで、そこから最強の層がなし崩しに骨抜きになります。

さばき方は次の3段階で考えてください。

**1. 根本原因を設定で潰す（まず推奨）。** 抜け出しの理由はたいてい「接続先が許可リストにない」「書き込み先がカレントディレクトリの外」「ツールがサンドボックス非対応」といったところでしょう。順に判断し、 `network.allowedDomains` にドメインを足す / `filesystem.allowWrite` にパスを足す / `excludedCommands` にそのツール**だけ**を列挙するなどで対処します。こうすれば、そもそも抜け出す必要がなくなります。`excludedCommands` は「サンドボックスを丸ごとオフ」ではなく「このコマンドだけ外で動かす」ので、影響範囲を最小にできます。最初はこのチューニングをこつこつやっていくことが良いでしょう。

**2. 抜けたことを記録して、設定に反映する。** 抜け出しをその場では都度許容しつつ、設定を育てていく段階です。次の項で詳しく書きます。

**3. エスケープハッチ自体を封じる。** `allowUnsandboxedCommands: false`（= Strict sandbox mode）にすると、`dangerouslyDisableSandbox` は完全に無視されます。失敗は失敗のまま、`excludedCommands` に明示したものだけが外で動きます。「抜け出しは一切させない」を担保したいチームや CI に向いています。

> 補足: 2026年4月の修正より前のバージョンには、`dangerouslyDisableSandbox` が**許可プロンプトを出さずに**実行されるバグがありました。古いバージョンを使っているなら、まず更新してください。

> **目指すべき姿：** 最善は「サンドボックスを外さずに使えて、しかも危険なアクセス・コマンドは通さない」の両立です。前者は許可リストの調整（本セクション）で、後者は deny ルール（§4）で詰めます。外すのが常態化していたら、それは設定がまだ甘いというサインです。記録を見ながら、外さずに済む形へ寄せていくのがサンドボックス設定の基本方針です。

### 無効化の記録を設定改善につなげる

サンドボックスを外した回数を減らしていく作業は、勘ではできません。何が・いつ・どのコマンドで外れたかが手元に残っていて初めて、どの設定を足せばよいかが決まります。ここが、設定を育てる本体です。

**記録する。** `PreToolUse` フックで `dangerouslyDisableSandbox: true` が付いた呼び出しだけを検出し、日時とコマンドを追記します。実行は止めません。止めてしまうと作業が進まず、記録も溜まらないからです。スクリプトと登録の仕方は §5 の活用例4にあります。残るのは「日時 \t コマンド」の TSV です。

**読む。** 週に一度でも頻度順に眺めれば、癖が見えます。

```bash
# 何が何回サンドボックスを外したか、多い順に
cut -f2 ~/.claude/logs/sandbox-bypass.tsv | sort | uniq -c | sort -rn | head
```

上位に来たものから順に手を打ちます。全部を一度に直す必要はありません。頻度の高い数件を潰すだけで、プロンプトの数は目に見えて減ります。

**設定に落とす。** 読み取ったパターンには、それぞれ対応する置き場所があります。

| ログから読み取れること | 反映先 |
|---|---|
| 特定ホストへの接続で繰り返し外している | `network.allowedDomains` にそのドメインを追加 |
| カレントディレクトリ外への書き込みで外している | `filesystem.allowWrite` にそのパスを追加 |
| 特定ツール（docker 等）が毎回外れる | `excludedCommands` にそのツールだけ列挙 |
| そもそも実行させたくないものが外れている | 外出しではなく `permissions.deny` で拒否 |
| 外す理由が散発的で正当性も薄い | `allowUnsandboxedCommands: false`（Strict mode）に格上げ |

4 行目に注意してください。**「外れているから許可する」とは限りません。** ログを見て「これはそもそもやらせたくない操作だ」と気づくこともあります。その場合に足すべきは許可リストではなく deny ルールです。ログは許可を広げるためだけの材料ではありません。

**効いたか確かめる。** 許可リストを追加したら、しばらく使ってみて、同じ理由で外れなくなったかを見ます。まだ外れるなら、設定が足りていません。

調整しおわったら、このログ（`~/.claude/logs/sandbox-bypass.tsv`）の内容は不要です。捨ててしまって構いません。

### さらに厳格にする（任意）

ハードニングを徹底するなら次も検討してください。

- **`failIfUnavailable: true`** — サンドボックスは、Linux・WSL2 では `bubblewrap` などのパッケージが入っていて初めて動きます（macOS は標準機能なので追加インストール不要）。これらが入っていなかったり、そもそも対応していないプラットフォーム（ネイティブ Windows など）だと、サンドボックスは起動できません。その場合、Claude Code はデフォルトでは**警告を出すだけで、サンドボックス無しのまま起動します**（いちばん強い層が、気づかないうちに外れた状態です）。`true` にすると、サンドボックスが立ち上がらない限り Claude Code 自体を起動しません。「サンドボックスが効いていない状態では動かしたくない」というチームや CI に向いています。
- **`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`（環境変数）** — 通常、サンドボックス内の Bash は親プロセスの環境変数をそのまま継承します。環境変数に入れた API キーやクラウド認証情報は、子プロセスから見えます。この環境変数を設定すると、Anthropic やクラウドプロバイダの認証情報を子プロセスから剥がせます。値は `"1"` を入れるだけで、`settings.json` の `env` ブロックに書きます。

  ```json
  "env": {
    "CLAUDE_CODE_SUBPROCESS_ENV_SCRUB": "1"
  }
  ```

  `env` に書いた変数は、セッションと、そこから起動する子プロセスの両方に渡ります。シェルで `export CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` してから `claude` を起動しても同じです。

### 限界も知っておく

サンドボックスは強力ですが「完全な隔離」ではありません。

- ネットワークのプロキシは**ドメイン名で許可を判断するだけで、TLS の中身は検査しません**。そのため `github.com` のような広いドメインを許可すると、domain fronting などで許可リスト外のホストへ抜ける余地が残ります（公式も明言しています）。広い許可は便利ですが、データ持ち出しの経路にもなります。許可リストは最小に保ってください。
- Read / Edit / Write の組み込みツールはサンドボックスを通らず、permission system 側で制御されます。同じ「ファイルを読む」でも、Bash 経由（sandbox が効く）かツール経由（permission が効く）かで防御層が違う、という点は押さえておいてください。

> **ポイント:** 影響範囲を最小限にとどめることで防御を設計する考え方（最小権限の原則）

---

## 3. パーミッションの仕組み

Claude Code のパーミッションの仕組みを使って、コマンドなどツールが呼び出されたときの動作許可を指示します。ルールには4段階あることを理解しておきましょう。

### パーミッションレベル

| パーミッション | 動作 | 利用する意味  | 
|--------------|---|---|
| `deny` | 常にブロック（プロンプトなし） |ガードレール |  
| `ask` | 常に確認（「今後聞かない」で許可済みでも毎回聞く） | Human-In-The-Loop。おおむね信頼できるが、確認の段階を挟みたい操作に |
| `allow` | 常に許可（プロンプトなし） | 利便性のため。信頼できる操作で、毎回確認を省きたい | 
| _（デフォルト/設定していない場合）_ | 初回は確認、「今後聞かない」で以降は永続許可 | デフォルトの判断基準は、あなたの頭の中 — 急いでいると安易にYesを押しがちですし、よくわからなくてYes/Noを判断できないこともあります。(自分の信頼性を時々信じられなくなることは誰にでもあります)|

> **ポイント:** `deny` は**絶対に起きてはならない**こと。`allow` は**常に信頼できる**こと。`ask` は**たいてい信頼できるが確認したい**こと 


### ルールの設定場所
このルールは、HOMEディレクトリの下にでも、チームと共有するプロジェクト(フォルダ/ディレクトリ/リポジトリ)単位でも設定できます。

| ファイル | 誰の設定か | スコープ | Git |
|---------|----------|---------|-----|
| `~/.claude/settings.json` | 自分だけ | このマシンの全プロジェクト | どのリポジトリにも含まれない |
| `<project>/.claude/settings.json` | チーム全員 | このプロジェクトのみ | コミットされ全員に共有される |
| `<project>/.claude/settings.local.json` | 自分だけ | このプロジェクトのみ | リポジトリ内に置くが gitignore で push されないようにしましょう |


---

## 4. ルールの考え方 Deny / Ask / Allow 

このセクションでは脅威カテゴリ別に具体的なルールを列挙します。各ルールには根拠を記載していますので、自分の環境に適用すべきかどうかを判断できます。

すべてのルールを1ファイルにまとめたものは [`settings_example.jsonc`](settings_example.jsonc) を参照してください。`allow` と `ask` の例や解説はコメントとして記載されています。ただし、コメントはそのまま残すとJSONとしてエラーになるので、お気をつけください。

### 4.1 Deny — 破壊的な Git 操作

リポジトリと履歴に対する不可逆な変更を防ぎます。

```json
"Bash(git push -f *)",
"Bash(git push --force *)",
"Bash(git reset --hard *)",
"Bash(git checkout .)",
"Bash(git restore *)",
"Bash(git clean -f *)",
"Bash(git add -A)"
```

| ルール | リスク |
|--------|--------|
| `git push -f / --force` | リモート履歴の上書き。チームメンバーの作業を壊す恐れがある。 |
| `git reset --hard` | コミットされていない変更をすべて不可逆的に破棄する。 |
| `git checkout .` | ワーキングツリーの変更を無言で巻き戻す。 |
| `git restore` | `git checkout --` の後継で、今の git はこちらを勧める。同じく変更を巻き戻す。`--staged`（ステージから戻すだけ）も一緒に止まる。 |
| `git clean -f` | 追跡されていないファイルを完全に削除。 |
| `git add -A` | すべてをステージング。`.env`、認証情報、巨大なバイナリを誤って含みかねない。（`git add .` は日常的に使うので、§4.12 の `ask` に置いている） |

> **これらの deny は「壁」ではなく「ガードレール」です。** `git` を止めても、同じ操作は `gh` や `curl` でも実行できるため、完全には防げません。抜け道を塞ごうと deny リストを伸ばし続けるより、「破壊的な操作はしない」という方針を示す数本にとどめるのが実用的です。なお、`github.com` をネットワークごと遮断するのは避けてください。clone も pull もできなくなり、通常の作業が止まります。
>
> リポジトリの破壊を確実に防ぐには、リモート側の**ブランチ保護**が有効です。`main` への force-push と削除を禁止し、PR を必須にすれば、git・gh・curl のいずれもサーバー側で拒否されます。通常の開発はそのまま続けられます。
>
> また、git の操作の多くは復元できます（reflog は約90日保持。リモートや他のクローンからも複製できます）。本当に取り返しがつかないのは次の2つだけです。ここは `ask` で毎回確認すると確実です。
>
> - コミットしていない変更（作業ツリーに対する `reset --hard` や `checkout .`）
> - 追跡外ファイルの削除（`git clean -fdx`）

### 4.2 Deny — 破壊的ファイル操作

プロジェクトツリーを丸ごと消しかねない一括削除を防ぎます。

```json
"Bash(rm -rf *)",
"Bash(rm -r *)"
```

| ルール | リスク |
|--------|--------|
| `rm -rf` | 確認なしでディレクトリを再帰削除。パスを間違えるとプロジェクト全体が消える。 |
| `rm -r` | 同上。設定次第で確認が入るが、無条件に許可するには危険すぎる。 |

注意が一点あります。上の 2 本でも、バッチリマッチしない `rm -fr`、`rm -Rf`、`rm --recursive` は通ってしまうのです。同じ操作に綴りが複数あるためで、追いかけ始めるときりがないのですが、それでも上記 2 つはよく出てくるパターンなので入れておくと良いでしょう。止まった時に、入れておいて良かったと思うはずです。

ここで、わたしが実践しているアイデアですが、`rm` そのものを禁止し、消したいものは退避フォルダ（`trash/` など）へ `mv` させるという方法です。

この場合、

```json
"Bash(rm *)"
```

これを入れることになります。

禁止するだけでなく、代替路を用意するのが肝です。`CLAUDE.md` にも「削除は `rm` ではなく退避フォルダ（`trash/`）へ」と書いておけば、Claude はそう振る舞います。

これが効くのは、**サンドボックスは作業ディレクトリの中を守らない**という点にあります。自分の作業環境は deny でしか守れない、これを覚えておきましょう。

### 4.3 Deny — 危険なシステム操作

パーミッション変更やプロセス強制終了による環境の不安定化、無防備化を防ぎます。

```json
"Bash(chmod 777 *)",
"Bash(chmod -R *)",
"Bash(chown -R *)",
"Bash(killall *)",
"Bash(pkill *)",
"Bash(kill -9 *)"
```

| ルール | リスク |
|--------|--------|
| `chmod 777` | 全ユーザに読み書き実行を許可。セキュリティのアンチパターン。 |
| `chmod -R / chown -R` | 再帰的な権限・所有者変更。システムディレクトリの破壊や機密ファイルの露出につながる。 |
| `killall / pkill` | 名前でプロセスを終了。無関係な重要プロセスを巻き込む恐れがある。 |
| `kill -9` | クリーンアップなしの強制終了。実行中アプリのデータ破損を起こしうる。 |

### 4.4 Deny — 権限昇格

Claude Code が root 権限でコマンドを実行することを防ぎます。

```json
"Bash(sudo *)",
"Bash(su *)"
```

AIアシスタント自身が権限昇格すべきではありません。`sudo` はパスワードを要求しますが、そもそも試みること自体を deny で防ぐ方が確実です。

### 4.5 Deny — パイプ経由のリモートコード実行

リモートスクリプトをそのままシェルに流し込むパイプ（`curl ... | sh`）は、サプライチェーン攻撃の典型的な入り口です。Claude Code はこれを「インストール手順」として自然に提案してきますし、普通のインストールに見えるため、つい反射的に許可してしまいます。

止め方には少しコツが要ります。`Bash(curl *|*sh)` のようにパイプごとパターンに書いても効きません。deny ルールはコマンドをパイプで分割してから、それぞれを別々に照合するので、パイプを含んだ文字列はどの断片にも一致しないのです。

代わりに、**取ってくる側ではなく、実行する側を止めます**。

```json
"Bash(sh *)",
"Bash(bash *)",
"Bash(zsh *)"
```

パイプの右側は独立に照合されるので、これで `curl ... | sh` も `wget -O- ... | bash` も止まります。取ってくる側の綴り（`curl` か `wget` か、それ以外か）に依存しないのが利点です。いったん落としてから実行する `curl -o /tmp/a.sh && sh /tmp/a.sh` も、同じ理由で止まります。`sudo bash` の形は §4.4 の `Bash(sudo *)` が受け持ちます。

`curl` と `wget` そのものは deny にせず、§4.12 の `ask` に置きます。API を叩く正当な用途が多く、一律に止めると結局ルールごと外すことになるからです。ask ならプロンプトにコマンド全文が出るので、パイプが付いていればそこで気づけます。

限界も書いておきます。`curl ... | python -` は上のルールでは止まりません。インタプリタを数え上げ始めるときりがないので、ここから先は §2 のネットワーク許可リストの仕事です（許可していないドメインには、そもそも到達できません）。また `bash build.sh` のような正当な実行も止まるので、煩わしければ `bash` は ask に落としてください。

いちばん確実なのは、こうしたコマンドを Claude に実行させないことです。提案された内容は、自分の手で打ちます。ひと手間で、この種の事故はほぼ避けられます。

### 4.6 Deny — リモートアクセス

Claude Code がリモートホストへ接続することを防ぎます。AIアシスタントにリモート接続を許可すべきではありません。これらのコマンドはリモートホストへのファイル転送やコマンド実行が可能です。
サンドボックスモードが有効な場合、ネットワークアクセス自体はブロックされますが、Claude Code がこれらのコマンドを実行しようとすること自体は防げないので、記載しています。

```json
"Bash(ssh *)",
"Bash(scp *)",
"Bash(rsync *)"
```

なお、リモートアクセスが必要な場合は、全面的に許可するのではなく、特定のターゲットだけを許可してください。

### 4.7 Deny — パッケージ公開とデプロイ

CI/CDとはいえ、意図しないソフトウェアパッケージの公開やデプロイを防ぎます。
パッケージの公開やデプロイのトリガーは、人間が意図的に行うべきアクションです。明示的に設計していない限り、AIが自律的に行うべきではありません。

```json
"Bash(npm publish *)",
"Bash(yarn publish *)",
"Bash(pnpm publish *)",
"Bash(*deploy*)"
```

### 4.8 Deny — つい許可しがちだが取り返しがつかないもの (macOS)

無害に見えるんですが、深刻な被害を引き起こしうる macOS コマンドがありますので、これらもブロックするという考え方です。

```json
"Bash(osascript *)",
"Bash(defaults write *)"
```

| ルール | つい許可してしまう理由 | 実際のリスク |
|--------|---------------------|------------|
| `osascript` | 「Finderの操作を自動化するだけ」 | AppleScript はメール送信、アプリ制御、キーチェーンアクセスなど、ほぼ何でもできる。 |
| `defaults write` | 「設定を変えるだけ」 | macOS のセキュリティ設定変更、Gatekeeper 無効化、アプリ動作の改変が可能。 |

> **`open` について:** deny ではなく §4.12 の `ask`（毎回確認）に置いています。ファイルや URL を開く正当な用途が多く、一律ブロックは不便だからです。プロンプトで対象を見てから通せば、怪しい URL やダウンロードファイルの実行はその場で止められます。

* 他のOSでは、これらのルールは不要ですが、考え方を参考に、それぞれ検討してください（情報お寄せください）


### 4.9 Deny — インフラストラクチャ

クラウドインフラへの自律的な変更を防ぎます。

```json
"Bash(terraform apply *)",
"Bash(terraform destroy *)",
"Bash(kubectl apply *)",
"Bash(kubectl delete *)",
"Bash(helm install *)",
"Bash(helm upgrade *)",
"Bash(docker push *)"
```

| ルール | リスク |
|--------|--------|
| `terraform apply / destroy` | クラウドインフラの作成・変更・破壊。 |
| `kubectl apply / delete` | Kubernetes クラスタ上のワークロードのデプロイ・削除。 |
| `helm install / upgrade` | Kubernetes パッケージのインストール・アップグレード。クラスタ全体に影響しうる。 |
| `docker push` | コンテナイメージをレジストリに公開。 |

クラウド CLI（`aws`、`gcloud`）はここに並べず、§4.12 の `ask` に回しています。破壊的なサブコマンドが多すぎて deny では列挙しきれませんし、パターンでは `aws s3 ls` と `aws s3 rm` を区別できないからです。`--no-cli-pager` や `--quiet` のようなフラグを狙うルールも同じです。フラグが末尾に来たときしか一致しませんし、そもそも「ページャなし」は破壊性を指していません。権限そのものを IAM 側で絞るのが本筋です。

### 4.10 Deny — 機密ファイルへのアクセス

シークレットを含むファイルの読み取りを防ぎます。

```json
"Read(**/.env)",
"Read(**/.env.*)",
"Edit(**/.env)",
"Edit(**/.env.*)"
```

`.env` ファイルには通常、APIキー、データベースパスワードなどの秘密情報が含まれます。

`Read` の deny は読み取りしか塞ぎません。書き換えまで止めるなら `Edit` の deny も要ります。`.env` は git に入っていないことが多く、上書きされると戻せません。

以下も、環境に応じて追加を検討してください：

```json
"Read(**/*.pem)",
"Read(**/*.key)",
"Read(**/credentials*)"
```


### 4.11 Deny — MCP（Model Context Protocol）アクションによるなりすましメッセージの防止

Claude Code があなたになりすまして（名義で）メッセージを送信することを防ぎます。
AIアシスタントがコンテキスト把握のためにメッセージを**読む**ことと、**送信する**ことは別問題です — 後者はあなたの明示的なアクションであるべきです。

```json
"mcp__claude_ai_Slack__slack_send_message",
"mcp__claude_ai_Slack__slack_schedule_message"
```

### 4.12 Ask — Human-in-the-Loop

すべてに白黒をつけることは困難ですよね。
便利で正当なコマンドでも、**毎回人間が確認すべき** ときには、`ask` を使います。

一度「はい、今後は聞かない」で承認すると、そのコマンドはプロジェクト内で永続的に許可されますが、`ask` ルールはこれを上書きし、以前「聞かない」にしていても**常にプロンプトを表示します**。これは Human-in-the-Loop（HITL）手法のひとつのやりかたです。

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

| ルール | なぜ `allow` でも `deny` でもなく `ask` なのか |
|--------|------------------------------------------|
| `git commit` | コミットメッセージと対象を毎回確認したい |
| `git push` | ブランチとリモートへの操作は、毎回確認すべき |
| `npm/pip/brew install` | 依存関係の追加は脆弱性を持ち込むリスク。確認できるようにしておきましょう |
| `git add .` | 全ファイルを一括ステージング。便利だが `.env` や巨大バイナリを巻き込みやすい。何が入るか毎回確認する |
| `psql / mysql / mongosh / sqlite3` | パターンでは `SELECT` と `DROP TABLE` を区別できない。プロンプトで SQL 全文を見て判断する |
| `open` (macOS) | ファイル・URL を開く正当な用途は多いが、任意アプリ起動やフィッシング URL の実行にも使える。毎回プロンプトで対象を確かめてから通す |
| `curl` / `wget` | 取得そのものは日常的。パイプでシェルに渡していないかをプロンプトの全文で確かめる（§4.5） |
| `aws` / `gcloud` | パターンでは `s3 ls` と `s3 rm` を区別できない。プロンプトでサブコマンドを見て判断する（§4.9） |


#### データベースコマンドについて

`psql`、`mysql`、`mongosh`、`sqlite3` などは破壊的な操作（`DROP TABLE`、`DELETE FROM`）が可能ですが、Claude Code の deny ルールはコマンドの引数の中身までは区別できません。`Bash(psql *)` を deny にすると、分析に必要な `SELECT` クエリも含めてすべてブロックされてしまいます。

**推奨:** データベースコマンドは `deny` で一律に止めるのではなく、`ask` で一度受け止めます。プロンプトが出たその場で、無害な `SELECT` と取り返しのつかない `DROP TABLE` を、自分の目で振り分けられます。

### 4.13 Allow — 信頼できる操作

allow ルールは信頼するコマンドの許可プロンプトを省略します。
安全で頻繁に使う判断操作のプロンプト疲れを軽減するのに有効です。

例えば、テスト・ビルドコマンド、安全な git 操作、MCP の読み取り専用アクションなどですね。

サンプルを書いておきましたので、こちら [`settings_example.jsonc`](settings_example.jsonc) のコメントアウトされた `allow` セクションを参照してください。


---

## 5. Hooks — Deny ルールだけでは不十分なとき

deny ルールは Claude Code のハーネス（CLI ツール本体）が強制するもので、AI モデルが「無視する」ことはできません。
しかし、データベース操作や機微な情報へのアクセスは、コマンド文字列の表面だけでは判別できません。deny のパターンマッチでは届かない領域です。

deny ルールは glob パターンマッチングを使うため、本質的に限界があります：

例：
- `Bash(sudo *)` は `sudo rm -rf /` をブロックしますが、内部で `sudo` を呼ぶスクリプトは通る
- `Bash(sh *)` は `curl url | sh` をブロックしますが、`curl url | python -` は通る
- `Bash(rm -rf *)` は `rm -rf /tmp` をブロックしますが、Makefile 内の `rm -rf` は通る

そこで、ここで紹介する **Hooks** は Claude Code のライフサイクルの特定のタイミングで実行されるカスタムシェルスクリプトです。最も重要なのは**ツール呼び出しの実行前**（`PreToolUse`）です。コマンドパターンしかマッチできない deny ルールとは異なり、Hook スクリプトはコマンド全体を JSON として受け取り、任意のロジックを適用できます。引数の検査、ファイル内容の確認、外部システムへの問い合わせや呼び出しのブロックなどです。


### 詳細な説明

[公式ドキュメント](https://code.claude.com/docs/ja/permissions)より：

> `Read` と `Edit` の deny ルールは、Claude の組み込みファイルツールに加えて、Claude Code が認識する Bash のファイルコマンド（`cat`、`head`、`tail`、`sed` など）にも適用されます。ファイルを間接的に読み書きする任意のサブプロセス、たとえば自分でファイルを開く Python や Node のスクリプトには適用されません。

`Read(**/.env)` を deny に入れておけば、`cat .env` のような素直な読み取りは止まります。抜けるのはその先です。スクリプトを書いて中でファイルを開いたり、Claude Code が「ファイルを読むコマンド」と認識しないツールを使ったりする場合です。deny リストが見ているのはコマンド文字列であって、プロセスが実際に何を開くかではありません。


### 活用例1: データベースコマンドの破壊的SQLをブロック

**課題:** `Bash(psql *)` を deny にすると `SELECT` もブロックされてしまいます。でも `DROP TABLE` や `DELETE FROM` は、実行前に止めたいところです。

**Hook スクリプト** — `~/.claude/hooks/block-destructive-sql.sh` として保存：

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# 破壊的なSQLキーワードが含まれているか検査
if echo "$CMD" | grep -iqE '(DROP\s|DELETE\s+FROM|TRUNCATE\s|ALTER\s+TABLE.*DROP)'; then
  echo "Blocked: destructive SQL detected in command: $CMD" >&2
  exit 2
fi

exit 0
```

```bash
chmod +x ~/.claude/hooks/block-destructive-sql.sh
```

**設定** — `settings.json` に追加：

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

これで `psql -c "SELECT * FROM users"` は通常通り実行されますが、`psql -c "DROP TABLE users"` は理由付きでブロックされます。


### 活用例2: Bash 経由での機密ファイル読み取りをブロック

**課題:** `Read(**/.env)` の deny は `cat .env` までは止めます。残るのは、Claude Code が「ファイルを読むコマンド」と認識しない経路です。`xxd`、`strings`、`base64`、`od` あたりは中身をそのまま吐きますし、`python -c` で開かれたら手も足も出ません。

**Hook スクリプト** — `~/.claude/hooks/block-sensitive-reads.sh` として保存：

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

SENSITIVE_PATTERNS='\.env|\.pem|\.key|id_rsa|id_ed25519|credentials'

# deny ルールが見ていない読み取り経路を拾う
READERS='xxd|strings|base64|od|hexdump|dd|python3?|node|perl|ruby'

if echo "$CMD" | grep -iqE "(${READERS})\s.*(${SENSITIVE_PATTERNS})"; then
  echo "Blocked: reading sensitive file via Bash: $CMD" >&2
  exit 2
fi

exit 0
```

これも結局はコマンド名の数え上げなので、完全ではありません。`python` を `python3.12` と書かれれば外れますし、スクリプトをファイルに書いてから実行されたらパターンには何も現れません。確実に塞ぐなら §2 の `filesystem.denyRead` です。あちらは OS がプロセスごと止めるので、何で開こうと関係ありません。このフックは、サンドボックスを使えない環境での次善策と考えてください。

**設定** — 同じ構造、同じ matcher：

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

### 活用例3: main/master ブランチへの push をブロック

**課題:** `git push` は便利なので `ask` にしていますが、うっかり承認すると main に直接 push できてしまいます。deny ルールの `Bash(git push *)` では、ブランチを区別できません。

**Hook スクリプト** — `~/.claude/hooks/block-push-to-main.sh` として保存：

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# git push コマンドでなければスキップ
echo "$CMD" | grep -qE '^\s*git\s+push' || exit 0

# 明示的に main/master が指定されている場合
if echo "$CMD" | grep -qE '\b(main|master)\b'; then
  echo "Blocked: push to main/master is not allowed: $CMD" >&2
  exit 2
fi

# 引数なし or リモート名のみの push → 現在のブランチを確認
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

**設定** — `settings.json` に追加：

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

これで `git push origin main` や引数なしの `git push`（現在のブランチが main の場合）をブロックします。作業用ブランチへの push はそのまま通ります。

> **Note:** `feature/main-cleanup` のようにブランチ名に "main" や "master" という文字列を含むケースにもマッチしてしまうので、気をつけてください。

### 活用例4: サンドボックスの一時無効化を記録する

**課題:** サンドボックスを有効にしても、Claude は失敗したコマンドを `dangerouslyDisableSandbox` を付けて外で再試行することがあります（§2 参照）。完全に封じる（`allowUnsandboxedCommands: false`）ほどではありませんが、「いつ・どのコマンドで外に出たか」は把握して、後で許可リストを見直したいところです。

**考え方:** ブロックはせず、記録だけします。`PreToolUse(Bash)` で `dangerouslyDisableSandbox: true` が付いた呼び出しを検出し、タイムスタンプとコマンドを TSV に追記します。終了コードは 0 のままにします（実行は止めません）。溜まったログを定期的に眺めれば、「このドメインを `allowedDomains` に足せば抜けずに済む」「このツールは `excludedCommands` に入れればいい」といった調整に気づけます。

**Hook スクリプト** — `~/.claude/hooks/log-sandbox-bypass.sh` として保存：

```bash
#!/bin/bash
INPUT=$(cat)
LOG=~/.claude/logs/sandbox-bypass.tsv

# dangerouslyDisableSandbox が true のときだけ記録（それ以外は素通り）
DISABLED=$(echo "$INPUT" | jq -r '.tool_input.dangerouslyDisableSandbox // false')
if [ "$DISABLED" = "true" ]; then
  CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

  # 秘密が載りうる形を落としてから書く
  CMD=$(printf '%s' "$CMD" | sed -E \
    -e 's/([A-Za-z_]*(KEY|SECRET|TOKEN|PASSWORD|PASSWD)[A-Za-z_]*=)[^ ]*/\1***/g' \
    -e 's/(--(password|token|api-key|apikey|secret)[= ])[^ ]*/\1***/g' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1***/g' \
    -e 's#(https?://)[^/ :]+:[^/ @]+@#\1***:***@#g' \
    -e 's/([?&](access_token|token|sig|signature|key)=)[^ &]*/\1***/g')

  mkdir -p "$(dirname "$LOG")"
  umask 077                      # 本人だけが読める権限でログを作る
  printf '%s\t%s\n' "$(date -Iseconds)" "$CMD" >> "$LOG"
fi

exit 0   # 記録だけ。実行はブロックしない
```

```bash
chmod +x ~/.claude/hooks/log-sandbox-bypass.sh
```

**設定** — 同じ構造、同じ matcher（他の Bash フックと併用可）：

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

これで `~/.claude/logs/sandbox-bypass.tsv` に「サンドボックスを外した瞬間」が「日時 \t コマンド」の TSV で残ります。ブロックする活用例1〜3とは逆に、止めずに**観測する**フックです。

このログはコマンド全文なので、秘密が載ることがあります。効くのは順に、**本人しか読めない権限にすること**と、**古い行を捨てて直近だけ残すこと**です。上のマスキングはその手前の気休めで、知らない形のトークンは素通りします。やるならこの程度は要る、という見本と考えてください。

`cleanupPeriodDays`（§7）が効くのは `~/.claude/projects/` の transcript だけで、このファイルは対象外です。自分で切り詰めます。

```bash
tail -n 500 ~/.claude/logs/sandbox-bypass.tsv > /tmp/sb.tsv && mv /tmp/sb.tsv ~/.claude/logs/sandbox-bypass.tsv
```

調整に使うのは直近の数百行で足ります。古い記録を残しても、得られるものより残る秘密の方が大きくなります。

溜まったログの読み方と、そこから設定へ落とすところまでは §2 の「無効化の記録を設定改善につなげる」に書いてあります。

### 複数の Hook を組み合わせる

同じイベントに複数の Hook スクリプトを登録できます。すべて実行され、どれか1つでも終了コード 2 を返せば呼び出しはブロックされます：

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

### 記録する — 拒否ログと監査証跡

deny で止めた操作は、その場のセッションには表示されますが、デフォルトではファイルに残りません。「いつ・何を止めたか」を振り返ることは、ポリシーを育てることにつながります。

手立ては 2 つあります。

**自前のフックで残す。** 活用例4のように `PreToolUse` でログを書けば手軽で、`grep` ですぐ読めます。ただし `PreToolUse` はパーミッション判定より**前**に走るので、残るのは「Claude が何を呼ぼうとしたか」であって「deny で止まった」という結果ではありません。一方、ブロックする側のフック（活用例1〜3）なら、止める瞬間が自分のスクリプトの中なので、`exit 2` の手前に 1 行足せば結果まで残せます。

**OpenTelemetry で残す。** 判定の結果まで含めるならこちらです（[公式ドキュメント](https://code.claude.com/docs/ja/monitoring-usage)）。`claude_code.tool_decision` イベントが `decision`（accept / reject）と `source`（`config` = 設定の deny ルール / `hook` / `user_reject` など）を持つので、**何が落としたのか**まで区別できます。チームやエンタープライズで集約・分析するなら、これが本筋です。

---

## 6. シークレットと認証情報

> **この節だけは、設定を間違えても後から直せません。** サンドボックスや承認は、間違えたら設定を直せば済みます。漏れたシークレットは直せません。ローテーションするしかない。ここは他の節より慎重に読んでください。

混ざりやすい 3 つの話を分けます。

**(1) Claude Code 自身の資格情報**

macOS では利用可能ならキーチェーンに、Windows と Linux ではファイル権限で保護された形で保存されます。ここは既定のままで構いません。

API キーを使う構成なら、値を設定に書かず、取り出すコマンドを書けます。

```json
{ "apiKeyHelper": "/usr/local/bin/get-my-key" }
```

Bedrock や Vertex 経由なら `awsAuthRefresh` / `awsCredentialExport` / `gcpAuthRefresh` が同じ役割を果たします。いずれも「値ではなく、値の取り方」を書く受け口です。

**(2) あなたの API キーをどう渡すか**

原則は一つです。

> **`settings.json` の `env` ブロックに値を書かない。**

例外はありません。「あとで消すから」「ローカルだけだから」で書いた行が、そのまま同期・バックアップ・スクリーンショット・画面共有に乗ります。ホームディレクトリの配下を——ホームまるごとでなくても、`~/Documents` の一部だけでも——クラウド同期しているなら、書いた瞬間に複製されます。`.claude/settings.local.json` も同じです。gitignore されていても、同期からは守られません。

MCP サーバに渡すしかない場合は、起動時にだけ注入して、そのプロセスに閉じ込めます。

```bash
eval $(get-secret --export API_KEY MY_TOKEN) && exec my-mcp-server
```

対話シェルへ `export` したまま放置しません。`.env` にも書き戻しません。保管には OS キーチェーン、[1Password CLI](https://developer.1password.com/docs/cli/)、[Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)、[HashiCorp Vault](https://developer.hashicorp.com/vault)、各クラウドの Secret Manager、[`pass`](https://www.passwordstore.org/)、[SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age) といった選択肢があります。どれを選んでも狙いは同じで、**設定ファイルにもシェル履歴にも平文の値を残さず、必要なときにだけ取り出す**ことです。

**(3) 環境変数を子プロセスへ流さない**

サンドボックス内の Bash は、既定で親プロセスの環境変数をそのまま受け取ります。環境変数に入れたキーは、そこから見えます。

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

`mode: "deny"` は、そのファイルを読めなくし、その環境変数をサンドボックス内のコマンドから消します。組み込みの除外リストはないので、**自分で挙げたものだけが守られます**。Anthropic とクラウドの資格情報をまとめて剥がすなら、§2 の `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` の方が手早いです。

なお、サンドボックスは `settings.json` 自体への書き込みを全スコープで拒否します。設定を書き換えて自分の制限を広げる、という経路は塞がれています。

---

## 7. データ保持を最小化する（transcript / セッションログ）

サンドボックス・パーミッション・Hooks は「何をさせるか」の制御でした。ここからは「**どれだけ痕跡を残すか**」の話です。Claude Code は会話の transcript／セッションログをローカル（`~/.claude/projects/<プロジェクト>/*.jsonl`）に保存します。ここにはソースコード、設計、ファイル内容、そして稀に貼り付けた認証情報が乗ります。

保持期間はデフォルト依存にせず、**明示的に設定**してください。データ最小化の基本であり、万一 transcript に機密が乗った場合の「ローカルに残る窓」を意図した長さに固定できます。

### 設定方法

`settings.json` に `cleanupPeriodDays`（日数）を設定します。**未設定だとデフォルト 30 日**で自動削除されます。

```jsonc
{
  // transcript / セッションログのローカル保持日数。これを過ぎると自動削除される。
  // 未設定時のデフォルトは 30 日。デフォルト依存にせず明示する。
  "cleanupPeriodDays": 30
}
```

### 何日にすべきか — 考え方

長くするほど復旧や分析に使える反面、機密の痕跡が長く残ります。次の3点で決めます。

- **復旧の時間軸は短い**: 中断・クラッシュしたセッションを後から復元する用途は、せいぜい数日〜1週間先まで遡れれば足ります。
- **恒久的に必要な知見は別管理にする**: 結論・決定事項・運用ルールは、transcript ではなく別のメモリ／ドキュメントに蒸留しておきます。そうすれば transcript は「揮発してよい作業ログ」になり、短めの保持でも知識を失いません。
- **露出窓を詰めたいか**: 値を短くするほど、transcript に乗った機密がローカルから消えるのが早くなります。

> **目安**: 多くの個人・小規模運用では 30 日前後が無難です（復旧に十分で、分析にも 1 か月分ある）。痕跡を詰めたいなら 14 日も実用的。2 週間あれば復旧用途でまず困りません。60〜90 日へ延ばす積極的な理由は、古い transcript を実際に掘り返す運用がない限り薄いはずです。

### 関連: 送信されるデータも絞る

ローカル保持と合わせて、外部送信されるデータも環境変数で絞れます（同じ「データ最小化」の文脈）。

```jsonc
{
  "env": {
    "DISABLE_TELEMETRY": "1",
    "DISABLE_ERROR_REPORTING": "1"
  }
}
```

> 自動アップデートを止める `DISABLE_AUTOUPDATER` まで切ると更新（セキュリティ修正含む）が止まるので、そこは目的次第。**保持の最小化（`cleanupPeriodDays`）と送信の最小化（telemetry/error-reporting）は別レイヤ**として、両方を意図的に設定するのが望ましい。

---

## 8. 設定が効いているか確かめる

書いただけでは信用できません。設定は複数のファイルに分かれていて、上位が下位を覆します。最後に一度、効いているかを見てください。

- **`/sandbox`** — Config タブに、マージ後の解決済みサンドボックス設定が出ます。書いたつもりのキーがここに出ていなければ、効いていません。Overrides タブでは、エスケープハッチを封じたか（Strict モードか）も確認できます
- **`/permissions`** — いま効いている allow / ask / deny の一覧です。Recently denied タブには、直近で止まったものが並びます
- **`claude doctor`** — セッションの外から実行できます。インストールの健全性を確認し、カレントディレクトリの設定ファイルを信頼プロンプトなしで読みます

そのうえで、実際にコマンドを一つ走らせてください。許可していないドメインへの `curl` が止まるか、作業ディレクトリの外へ書けないかを確かめます。**ハードニングで最も多い事故は「設定したつもり」です。**

なお、Claude Code には「サンドボックスの中だけで試しにコマンドを走らせる」専用のサブコマンドはありません。確認は、解決済みの設定を目で見ることと、実際に動かしてみることの 2 つで行います。

---

## 参考資料

### 公式ドキュメント

- [セキュリティ](https://code.claude.com/docs/ja/security)
- [Claude Code の設定](https://code.claude.com/docs/ja/settings)
- [権限を設定する](https://code.claude.com/docs/ja/permissions)
- [サンドボックス](https://code.claude.com/docs/ja/sandboxing)
- [hooks でワークフローを自動化する](https://code.claude.com/docs/ja/hooks-guide)


## 関連チートシート・参考文献

- [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) — AIエージェントシステムの主要リスクとベストプラクティス：ツール権限の最小化、プロンプトインジェクション対策、Human-in-the-Loop など
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Prompt_Injection_Prevention_Cheat_Sheet.html) — プロンプトインジェクション攻撃への防御に関する技術ガイダンス
- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) — LLM アプリケーションにおける脅威の全体像
