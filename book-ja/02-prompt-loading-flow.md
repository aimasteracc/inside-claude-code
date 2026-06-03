# プロンプト読み込みフロー — 1つのメッセージがAPIリクエストになるまで

*入力したメッセージは、Anthropicに届く前に6つのエントリ関数と3つの組み立てレイヤーを通過する。そして、実際に届く内容のほとんどは、あなたが入力したものではない。*

## なぜ重要か

Claude Codeに「authモジュールのバグを直して」と頼むとき、モデルが受け取るのはその一文だけではない。役割定義、振る舞いのルール、ツールの説明、gitのスナップショット、あなたの`CLAUDE.md`、利用可能なスキルの一覧、トークン使用量の警告、その他十数個のコンテキストブロックを受け取る。これらはすべて、ターンごとに、特定の順序で、新たに組み立てられる。エージェントを構築する立場からすれば、この順序とキャッシュの境界線こそが、1ターンあたり数セントで済むシステムと、毎回187Kトークンの静的な指示を再課金されるシステムとを分ける決定的な差になる。本章では、その旅路をソースレベルで追い、あなたがこのアーキテクチャをそのまま流用できるようにする。

## 3つのレイヤー

Claude Codeはリクエストを概念的に3つのレイヤーで組み立て、それをソースレベルの6つのエントリ関数に対応させている。

1. **静的システムプロンプト** — セッションごとに一度だけ構築され、キャッシュされる。（`getSystemPrompt`）
2. **コンテキスト注入** — システムコンテキストを末尾に追加し、ユーザーコンテキストを先頭に追加する。（`buildEffectiveSystemPrompt`、`appendSystemContext`、`prependUserContext`）
3. **ターンごとのアタッチメント** — 1ターンごとに収集され、再注入される。（`getAttachments`）

そして最後の2ステップがすべてを接着する。ツールの説明（`toolToAPISchema`）と最終組み立て（`queryModel`）である。

```
            "fix the bug in the auth module"
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 1  Static system prompt   getSystemPrompt()          |
  | (built ONCE per session, then served from cache)          |
  |                                                           |
  |   (1) "You are Claude Code, Anthropic's official CLI..."  |
  |   (2) <system-reminder> tag explainer                     |
  |   (3) doing-tasks rules (8x no-* + 2 behavior)            |
  |   (4) executing-actions-with-care                         |
  |   (5) tool-usage policy (prefer dedicated tools > Bash)   |
  |   (6) tone-and-style + output-efficiency                  |
  |  ===== SYSTEM_PROMPT_DYNAMIC_BOUNDARY (cache split) =====  |
  |   (8) memory      <- loadMemoryPrompt()                   |
  |   (9) env_info    (OS / shell / git)                      |
  |  (10) language preference                                 |
  |  (11) mcp_instructions   [uncached]                       |
  |  (12) scratchpad path                                     |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 2  Context injection                                |
  |                                                           |
  |  buildEffectiveSystemPrompt()  -> picks WHICH prompt      |
  |  appendSystemContext()  -> git status snapshot + breaker  |
  |  prependUserContext()   -> CLAUDE.md + date, wrapped in   |
  |                            <system-reminder> USER message |
  +-----------------------------------------------------------+
                        |
                        v
  +-----------------------------------------------------------+
  | LAYER 3  Per-turn attachments   getAttachments()          |
  | (re-collected EVERY turn, event-driven)                   |
  |                                                           |
  |   skill_listing | agent_listing | changed_files |         |
  |   todo_reminders | new_diagnostics | nested_memory | ...  |
  |   each -> wrapInSystemReminder()                          |
  +-----------------------------------------------------------+
                        |
            toolToAPISchema()  ->  tools: [...]
                        |
                        v
            queryModel()  ->  POST /v1/messages
```

## レイヤー1 — 静的プロンプトとキャッシュ境界

`getSystemPrompt()`は文字列の配列を返す。最初の6つは事実上不変だ。役割を定義する行、`<system-reminder>`タグの説明、「タスクの遂行」ルール（8つの`no-*`禁止事項と2つの振る舞い）、慎重な操作のための安全ブロック、ツール使用ポリシー、トーン／出力効率のブロックである。

そして、肝心の行が続く。`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`だ。これより前のものはすべて安定しており、これより後のもの — メモリ、環境情報、言語設定、MCP命令、scratchpadのパス — はターン間で変化しうる。動的な後半部分は`resolveSystemPromptSections()`が埋める。

この境界が存在する理由はただ一つ、**Anthropicのプロンプトキャッシュがプレフィックス一致で動作する**からである。キャッシュヒットには同一のプレフィックスが必要だ。揮発性のコンテンツをすべて安定したコンテンツの*後ろ*に置くことで、Claude Codeはターンをまたいでプレフィックスを一定に保ち、キャッシュにヒットさせ、静的ブロックに対しては毎ターン満額を払う代わりに入力トークンコストのおよそ10%で済ませている。MCP命令のセクションは、サーバーの接続・切断のたびに変化するため、明示的にキャッシュ対象外としてマークされている。

## レイヤー2 — コンテキスト注入、そしてなぜCLAUDE.mdはシステムプロンプトに入らないのか

`buildEffectiveSystemPrompt()`は、優先度の高い順に、*どの*システムプロンプトを使うかを決定する。

| 優先度 | ソース | 採用される条件 |
|----------|--------|-----------|
| 1 | オーバーライドプロンプト | あるモード（例：loop）がすべてを置き換える |
| 2 | コーディネータープロンプト | コーディネーターモードで実行中 |
| 3 | エージェントプロンプト | サブエージェントが独自の`systemPrompt`を供給する |
| 4 | `--system-prompt`フラグ | ユーザーがカスタムプロンプトを渡した |
| 5 | デフォルト | レイヤー1の結果にフォールバックする |

続いて2つのインジェクターが動く。`appendSystemContext()`は、読み取り専用の`git status --short`と`git log --oneline -5`のスナップショット、そして`cacheBreaker`をシステムプロンプトの末尾に追加する。注意してほしいのは、このgitスナップショットは一度だけ取得され、会話の途中で更新*されない*点だ。

`prependUserContext()`はもっと巧妙なことをする。あなたの`CLAUDE.md`（作業ディレクトリからディレクトリツリーを上にたどって発見される）と現在の日付を取り、それらを`<system-reminder>`ブロックで包み、システムプロンプトにではなく**ユーザーメッセージ**として注入する。

```
<system-reminder>
As you answer the user's questions,
you can use the following context:
[CLAUDE.md contents + current date]
</system-reminder>
```

なぜユーザーメッセージなのか。それは`CLAUDE.md`が頻繁に変わるからだ。もしこれがシステムプロンプトに置かれていたら、編集のたびにキャッシュされたプレフィックスが無効化され、静的ブロック全体が再課金される。ユーザーメッセージに留めておくことで、その変動をシステムプロンプトのキャッシュから切り離せる。これはパイプライン全体の中でも、最も重要でありながら最も気づかれにくいアーキテクチャ上の判断の一つである。

## レイヤー3 — アタッチメントは読み込まれるのではなく、イベント駆動である

`getAttachments()`は**ターンごとに**実行され、直前に起きたことに基づいてコンテキストを再収集する。

| アタッチメント | トリガー |
|-----------|---------|
| `skill_listing` | 常に注入される（スキルとその説明） |
| `agent_listing` | 常に注入される（エージェントとwhenToUse） |
| `mcp_instructions` | MCPサーバーが接続または切断された |
| `dynamic_skill` | ファイル操作が新しいパスに触れた |
| `todo_reminders` | TodoWriteがしばらく使われていない |
| `changed_files` | 前のターンでファイルが変更された |
| `new_diagnostics` | コンパイラ／型チェッカーが新しいエラーを報告した |
| `nested_memory` | ネストされたメモリファイルが読み込まれた |
| `token_usage` | トークン消費量が変化した |

各アタッチメントは`wrapInSystemReminder()`を通され、`<system-reminder>`ブロックとして出力される。これが、あなたも耳にしたことがあるかもしれない「110以上のプロンプトファイル」の正体だ。それらは一度にすべて読み込まれるわけではない。セッション内のイベントに駆動され、必要に応じて発火する。ほとんどのターンでは、ほんの一握りしか注入されない。

## 最終組み立て

`toolToAPISchema()`はアクティブな全ツールを巡回し、それぞれの`.prompt()`メソッド — `BashTool.prompt()`、`ReadTool.prompt()`、`EditTool.prompt()`など — を呼び出して`description`フィールドを生成する。独立した「ツール説明」ファイルは存在しない。各ツールのプロンプトはコード内で組み立てられる（`BashTool.prompt()`だけでも30以上の制約フラグメントを連結している）。

続いて`queryModel()`が送出リクエストを構築する。

```jsonc
POST /v1/messages
{
  "model": "claude-sonnet-4-6",
  "system": [
    { "type": "text", "text": "You are Claude Code...",
      "cache_control": { "type": "ephemeral" } },   // cache breakpoint
    { "type": "text", "text": "...dynamic sections..." }
  ],
  "messages": [
    { "role": "user", "content": [
      "<system-reminder>CLAUDE.md...</system-reminder>",
      "<system-reminder>skill listing...</system-reminder>",
      "<system-reminder>token usage...</system-reminder>",
      "fix the bug in the auth module"            // your actual input, last
    ]},
    { "role": "assistant", "content": [ ... ] }     // prior turns
  ],
  "tools": [ /* Phase-5 schemas */ ],
  "max_tokens": 16384,
  "thinking": { "type": "enabled", "budget_tokens": 31999 }
}
```

system配列は、静的／動的の分割点に`cache_control: ephemeral`のブレークポイントを持つ。あなたが入力したメッセージは、注入されたすべてのリマインダーの後、ユーザーコンテンツ配列の*最後*に着地する。モデルはまずコンテキストを読み、あなたのリクエストを最後に読むのだ。

## 使えるテクニック

あなた自身のエージェントのプロンプト組み立てを、この同じ3つのレイヤーとして設計しよう。

- **レイヤー1 — 静的でキャッシュされたプレフィックス。** 役割定義、振る舞いのルール、ツールポリシー、トーンを、セッション内で決して変化しない1つのブロックにまとめる。その末尾にキャッシュのブレークポイントを置く。安定したものはすべてブレークポイントの*前*に、揮発性のものはすべて*後*に置く。この一つの判断だけで、指示ブロックの入力コストを約90%削減できる。
- **レイヤー2 — 揮発性のプロジェクトコンテキストはユーザーメッセージに隔離する。** 頻繁に変わるもの（あなたにとっての`CLAUDE.md`相当のもの、日付、gitスナップショット）はシステムプロンプトに*属さない* — キャッシュのプレフィックスを壊してしまう。区切り文字（`<system-reminder>`や独自のタグ）で包み、代わりにユーザーメッセージとして注入する。
- **レイヤー3 — コンテキストは常時ではなく、イベント駆動にする。** 考えうるすべての指示を毎ターン読み込んではいけない。何が変化したか — 触れられたファイル、発生したエラー、アイドル状態のツール — に基づいて、ターンごとにアタッチメントを収集する。*今この*ターンで関連するものだけを、それぞれ明確に区切られたブロックで注入する。
- **順序が重要 — コンテキストが先、リクエストが後。** ユーザーの実際の要求は、注入されたすべてのコンテキストの後、コンテンツ配列の一番最後に置く。そうすればモデルはタスクに取りかかる前にブリーフィングを読む。
- **ツールの説明はファイルではなくコードで生成する。** ツールごとの`.prompt()`メソッドがあれば、制約を動的に組み立てられ、単一の信頼できる情報源を保てる。

**ユーザーは一文を入力する。だが、モデルが実際に何を読むかを決めるのはアーキテクチャだ — そしてキャッシュの境界をどこに引くかが、そのコストを決める。**

---
