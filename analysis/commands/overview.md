# Claude Code コマンドシステム解析

## 1. 概要

Claude Code は **80以上のスラッシュコマンド** を内蔵しており、ユーザーが `/command` 形式で呼び出すことで各種操作を実行できる。

### コマンド型（3種類）

| 型 | 識別子 | 実行方式 | 戻り値 |
|---|---|---|---|
| **PromptCommand** | `prompt` | AIにプロンプトを送信して実行 | AI応答 |
| **LocalCommand** | `local` | ローカルで即時処理（AI不要） | `LocalCommandResult` |
| **LocalJSXCommand** | `local-jsx` | React UIコンポーネントをレンダリング | `React.ReactNode` |

### 条件付きロード

コマンドは以下の条件でロード/非表示が制御される:

- **`feature()` gate**: Feature Flag で有効/無効を切り替え
- **`process.env`**: 環境変数による条件分岐
- **`availability`**: 利用プラン（`claude-ai`, `console`）による制限

---

## 2. コマンド実行フロー

```
ユーザー入力 ("/<command> [args]")
       │
       ▼
  findCommand()  ── コマンド名からコマンド定義を検索
       │
       ▼
   type 分岐
       │
       ├─ "prompt"
       │     └─ getPromptForCommand() → プロンプト生成 → AI送信 → AI応答
       │
       ├─ "local"
       │     └─ load() → call() → LocalCommandResult
       │
       └─ "local-jsx"
             └─ load() → call() → React.ReactNode（UIレンダリング）
```

**補足**:
- `findCommand()` はビルトインコマンド → スキル → プラグインの順で検索する
- `prompt` 型はユーザー入力を含むプロンプトをAIに送信し、AIが応答・ツール実行を行う
- `local` 型はAIを介さずローカルで処理を完結させる
- `local-jsx` 型はInk（React for CLI）ベースのUIコンポーネントを返す

---

## 3. 全コマンド一覧

### コア操作

| コマンド | 型 | 説明 |
|---|---|---|
| `/help` | local-jsx | ヘルプ表示（コマンド一覧・使い方） |
| `/config` | local-jsx | 設定パネル（テーマ・モデル・権限等） |
| `/doctor` | local-jsx | 診断ツール（環境・認証・MCP等の問題検出） |
| `/init` | prompt | プロジェクト初期化（CLAUDE.md 生成） |
| `/compact` | local | 会話履歴を圧縮（トークン節約） |
| `/clear` | local | 会話履歴をクリア |
| `/exit` | local | CLIを終了 |
| `/quit` | local | CLIを終了（/exit のエイリアス） |
| `/session` | local | セッション管理（保存・復元・一覧） |

### Git / コードレビュー

| コマンド | 型 | 説明 |
|---|---|---|
| `/commit` | prompt | Git コミット（差分解析→メッセージ生成→コミット） |
| `/review` | prompt | コードレビュー（PR差分の品質チェック） |
| `/pr-comments` | prompt | PRコメントへの対応 |
| `/pr` | prompt | プルリクエスト作成 |

### 計画・分析

| コマンド | 型 | 説明 |
|---|---|---|
| `/plan` | local-jsx | 計画モード（要件整理→リスク評価→実装計画） |
| `/ultraplan` | local-jsx | 深い計画モード（feature gated） |
| `/think` | prompt | 深い思考モード（複雑な問題の段階的推論） |

### MCP / ツール管理

| コマンド | 型 | 説明 |
|---|---|---|
| `/mcp` | local-jsx | MCP サーバー管理（一覧・追加・削除・診断） |
| `/tools` | local-jsx | 利用可能ツール一覧の表示 |
| `/permissions` | local-jsx | 権限設定の管理 |
| `/allowed-tools` | local | 許可済みツールの一覧表示 |

### コスト・使用量

| コマンド | 型 | 説明 |
|---|---|---|
| `/cost` | local | 現セッションのトークン使用量・コスト表示 |
| `/tokens` | local | トークン数の詳細表示 |

### デバッグ・開発

| コマンド | 型 | 説明 |
|---|---|---|
| `/debug` | prompt | デバッグ支援（エラー解析・修正提案） |
| `/batch` | prompt | バッチ処理（複数ファイルへの一括操作） |
| `/loop` | prompt | 定期実行（指定間隔でコマンドを繰り返し） |
| `/verify` | prompt | 検証（実装の正当性確認） |
| `/build-fix` | prompt | ビルドエラーの検出と修正 |

### 記憶・学習

| コマンド | 型 | 説明 |
|---|---|---|
| `/remember` | prompt | 情報を記憶（CLAUDE.md への書き込み） |
| `/learn` | prompt | 再利用可能なパターンの抽出 |

### UI・表示

| コマンド | 型 | 説明 |
|---|---|---|
| `/theme` | local-jsx | テーマ切り替え（ダーク/ライト/カスタム） |
| `/vim` | local | Vimキーバインドの切り替え |
| `/terminal-setup` | local-jsx | ターミナル設定ガイド |
| `/status-bar` | local | ステータスバーの表示切替 |

### 音声・AI対話

| コマンド | 型 | 説明 |
|---|---|---|
| `/voice` | local-jsx | 音声入出力モード |
| `/buddy` | local-jsx | AIコンパニオン（feature gated） |
| `/assistant` | local-jsx | アシスタントモード（feature gated） |

### モデル・プロバイダ

| コマンド | 型 | 説明 |
|---|---|---|
| `/model` | local-jsx | モデル選択・切り替え |
| `/providers` | local-jsx | プロバイダ一覧・設定 |

### スキル・拡張

| コマンド | 型 | 説明 |
|---|---|---|
| `/simplify` | prompt | コードの簡素化・リファクタリング |
| `/schedule` | prompt | リモートエージェントのスケジュール管理 |
| `/claude-api` | prompt | Claude API / SDK の利用支援 |

### ログイン・認証

| コマンド | 型 | 説明 |
|---|---|---|
| `/login` | local-jsx | 認証（Anthropic / GitHub OAuth） |
| `/logout` | local | ログアウト |

### その他

| コマンド | 型 | 説明 |
|---|---|---|
| `/bug` | local-jsx | バグ報告（GitHubへのIssue作成支援） |
| `/feedback` | local-jsx | フィードバック送信 |
| `/tips` | local | 使い方のヒント表示 |
| `/context-pack` | prompt | コンテキストパック生成（関連ファイルの要約） |
| `/checkpoint` | prompt | チェックポイント作成 |
| `/e2e` | prompt | E2Eテスト生成・実行（Playwright） |
| `/eval` | prompt | 評価コマンド |
| `/tdd` | prompt | TDD ワークフロー |
| `/test-coverage` | prompt | テストカバレッジ確認 |
| `/refactor-clean` | prompt | リファクタリング |
| `/code-review` | prompt | コードレビュー（詳細版） |
| `/resume-session` | prompt | セッション復元 |
| `/save-session` | prompt | セッション保存 |
| `/brief` | local-jsx | 簡潔応答モード（feature gated） |
| `/proactive` | local-jsx | プロアクティブモード（feature gated） |
| `/fork` | local-jsx | 会話フォーク（feature gated） |
| `/peers` | local-jsx | ピアエージェント連携（feature gated） |
| `/notebook` | local-jsx | ノートブック操作 |
| `/worktree` | local-jsx | Git Worktree管理 |

---

## 4. Feature Gate 付きコマンド

Feature Flag によって制御され、全ユーザーには公開されていないコマンド群。

| コマンド | Feature Flag | 説明 | 状態 |
|---|---|---|---|
| `/brief` | `brief_mode` | 簡潔応答モード | 限定公開 |
| `/assistant` | `assistant_mode` | アシスタントモード | 限定公開 |
| `/proactive` | `proactive_mode` | プロアクティブ提案モード | 限定公開 |
| `/ultraplan` | `ultraplan` | 深い計画モード（複数エージェント連携） | 限定公開 |
| `/buddy` | `buddy` | AIコンパニオン（常駐型アシスタント） | 限定公開 |
| `/fork` | `fork` | 会話の分岐・並列探索 | 限定公開 |
| `/peers` | `peers` | 複数エージェント間のピア連携 | 限定公開 |
| `/voice` | `voice` | 音声入出力 | 限定公開 |

**注意**: Feature Gate の有効化は Anthropic サーバー側で制御されるため、ユーザーが手動で切り替えることはできない。

---

## 5. スキルシステム統合

コマンド検索時の**優先順位**（上位が優先）:

| 優先度 | カテゴリ | 説明 | 例 |
|---|---|---|---|
| 1 | **Bundled Skills** | コンパイル時に組み込まれたスキル | `/batch`, `/debug`, `/verify` |
| 2 | **Built-in Plugin Skills** | 組み込みプラグインのスキル | `/claude-api`, `/e2e` |
| 3 | **Skill Dir Commands** | `.claude/skills/` ディレクトリのスキル | ユーザー定義スキル |
| 4 | **Workflow Commands** | ワークフロー定義コマンド | カスタムワークフロー |
| 5 | **Plugin Commands** | プラグインコマンド | サードパーティ拡張 |
| 6 | **Plugin Skills** | プラグインスキル | サードパーティスキル |
| 7 | **Built-in Commands** | 組み込みコマンド（最低優先度） | `/help`, `/config` |

### バンドル済みスキル（14個）

コンパイル時に組み込まれ、常に利用可能なスキル群:

| スキル | 説明 |
|---|---|
| `/batch` | 複数ファイルへの一括操作 |
| `/claude-api` | Claude API / Anthropic SDK 利用支援 |
| `/debug` | デバッグ支援（エラー解析・修正提案） |
| `/loop` | 定期実行（指定間隔でコマンドを繰り返し） |
| `/remember` | 情報を CLAUDE.md に記憶 |
| `/schedule` | リモートエージェントのスケジュール管理 |
| `/simplify` | コードの簡素化・リファクタリング |
| `/verify` | 実装の検証 |
| `/build-fix` | ビルドエラーの検出と修正 |
| `/checkpoint` | チェックポイント作成 |
| `/code-review` | コードレビュー（詳細版） |
| `/eval` | 評価コマンド |
| `/learn` | 再利用可能パターンの抽出 |
| `/plan` | 計画モード（要件整理→実装計画） |

**スキル呼び出し**: `Skill` ツール経由で呼び出される。ユーザーが `/skill-name` と入力すると、コマンドシステムがスキルを検索し、`Skill` ツールで実行する。

---

## 6. Remote / Bridge Mode 安全性

リモート実行やブリッジモードでは、実行可能なコマンドが厳しく制限される。

### REMOTE_SAFE_COMMANDS

リモートモード（Webベースのセッション等）で許可されるコマンド:

| コマンド | 理由 |
|---|---|
| `session` | セッション管理は安全 |
| `exit` | 終了は常に許可 |
| `clear` | 履歴クリアは安全 |
| `help` | ヘルプ表示は安全 |
| `cost` | コスト表示は安全 |
| `compact` | 圧縮は安全 |
| `tokens` | トークン表示は安全 |
| `model` | モデル切替は安全 |

### BRIDGE_SAFE_COMMANDS

ブリッジモード（VS Code 拡張等）で許可されるコマンド:

| コマンド | 理由 |
|---|---|
| `compact` | 圧縮は安全 |
| `clear` | 履歴クリアは安全 |
| `cost` | コスト表示は安全 |
| `tokens` | トークン表示は安全 |
| `model` | モデル切替は安全 |
| `help` | ヘルプ表示は安全 |
| `exit` | 終了は常に許可 |
| `session` | セッション管理は安全 |
| `status-bar` | UI表示は安全 |

### isBridgeSafeCommand() の判定ロジック

```
isBridgeSafeCommand(command):
  if command.type == "local-jsx":
    return false  ← JSXレンダリングはブリッジ非対応
  if command.type == "prompt":
    return true   ← AI実行はブリッジでも許可
  if command.name in BRIDGE_SAFE_COMMANDS:
    return true   ← ホワイトリストに含まれるローカルコマンド
  return false
```

**設計意図**: ブリッジモードではReact UIのレンダリングができないため `local-jsx` を禁止し、`prompt` 型はAI側で処理されるため許可している。

---

## 7. Availability Requirements

コマンドの利用可否はユーザーのアクセス経路（プラン）によって決まる。

| 値 | 対象ユーザー | 説明 |
|---|---|---|
| `claude-ai` | Pro / Max / Team 購読者 | claude.ai 経由でアクセスするユーザー |
| `console` | API 直接利用者 | Anthropic Console / API Key でアクセスするユーザー |
| *(指定なし)* | **全員** | プランに関係なく利用可能 |

### 制限の適用例

| コマンド | availability | 説明 |
|---|---|---|
| `/schedule` | `claude-ai` | リモートエージェントのスケジュール（Pro/Max/Team のみ） |
| `/peers` | `claude-ai` | ピアエージェント連携（Pro/Max/Team のみ） |
| `/fork` | `claude-ai` | 会話フォーク（Pro/Max/Team のみ） |
| `/config` | *(なし)* | 全ユーザー利用可能 |
| `/help` | *(なし)* | 全ユーザー利用可能 |
| `/commit` | *(なし)* | 全ユーザー利用可能 |

**注意**: `availability` は `feature()` gate と組み合わせて使用されることがある。その場合、Feature Flag が有効 **かつ** availability 条件を満たすユーザーのみがコマンドを利用できる。
