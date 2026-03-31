# Claude Code ツールシステム解析

## 1. 概要

Claude Code のツールシステムは、LLM が外界と対話するための統一的なインターフェースを提供する。

- **規模**: 43 個のメインツールディレクトリ、184 ファイル
- **統一インターフェース**: `Tool<Input, Output, Progress>` ジェネリクス型で全ツールを抽象化
- **一貫性保証**: `buildTool()` ヘルパー関数により、ツール定義の構造・バリデーション・登録を標準化
- **スキーマ二重サポート**: Zod v4 スキーマ（TypeScript 型推論）と JSON Schema（LLM API 送信用）の両方を同時サポート

---

## 2. ツール一覧（全 43 ツール）

### コアファイル操作

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| BashTool | `BashTool/` | Shell コマンド実行（セキュリティ検証付き）（1,143 行） |
| FileEditTool | `FileEditTool/` | テキストファイル編集（625 行） |
| FileReadTool | `FileReadTool/` | ファイル読取（PDF 対応）（1,183 行） |
| FileWriteTool | `FileWriteTool/` | ファイル新規作成（434 行） |
| GlobTool | `GlobTool/` | ファイルパターン検索（198 行） |
| GrepTool | `GrepTool/` | 内容正規表現検索（577 行） |
| NotebookEditTool | `NotebookEditTool/` | Jupyter Notebook 編集（490 行） |
| PowerShellTool | `PowerShellTool/` | Windows PowerShell 実行（1,000 行） |

### エージェント・タスク管理

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| AgentTool | `AgentTool/` | エージェント実行・コーディネーター（1,397 行） |
| SendMessageTool | `SendMessageTool/` | エージェント間メッセージ（917 行） |
| TaskCreateTool | `TaskCreateTool/` | タスク作成 |
| TaskListTool | `TaskListTool/` | タスク一覧 |
| TaskGetTool | `TaskGetTool/` | タスク取得 |
| TaskUpdateTool | `TaskUpdateTool/` | タスク更新（406 行） |
| TaskOutputTool | `TaskOutputTool/` | タスク出力表示（583 行） |
| TaskStopTool | `TaskStopTool/` | 実行中タスク停止 |
| TeamCreateTool | `TeamCreateTool/` | エージェントチーム作成（240 行） |
| TeamDeleteTool | `TeamDeleteTool/` | チーム削除 |
| TodoWriteTool | `TodoWriteTool/` | Todo 管理 |

### ユーザーインタラクション

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| AskUserQuestionTool | `AskUserQuestionTool/` | ユーザー確認ダイアログ（265 行） |
| SkillTool | `SkillTool/` | スキル実行（1,108 行） |
| ToolSearchTool | `ToolSearchTool/` | 遅延ツール検索（471 行） |
| BriefTool | `BriefTool/` | 添付ファイル・コンテキスト管理（204 行） |

### プランニング・ワークツリー

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| EnterPlanModeTool | `EnterPlanModeTool/` | 計画モード開始 |
| ExitPlanModeTool | `ExitPlanModeTool/` | 計画モード終了 |
| EnterWorktreeTool | `EnterWorktreeTool/` | git ワークツリー管理 |
| ExitWorktreeTool | `ExitWorktreeTool/` | ワークツリー終了 |

### Web・外部連携

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| WebFetchTool | `WebFetchTool/` | URL 取得・分析（318 行） |
| WebSearchTool | `WebSearchTool/` | Web 検索（435 行） |

### MCP (Model Context Protocol)

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| MCPTool | `MCPTool/` | MCP 統合（77 行） |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | MCP リソース一覧 |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | MCP リソース読取 |
| McpAuthTool | `McpAuthTool/` | MCP 認証（215 行） |

### 設定・システム

| ツール名 | ディレクトリ | 説明 |
|---|---|---|
| ConfigTool | `ConfigTool/` | 設定管理（467 行） |
| LSPTool | `LSPTool/` | 言語サーバープロトコル統合（860 行） |
| REPLTool | `REPLTool/` | 仮想マシン環境 |
| SleepTool | `SleepTool/` | 待機制御 |
| SyntheticOutputTool | `SyntheticOutputTool/` | 出力生成 |
| RemoteTriggerTool | `RemoteTriggerTool/` | リモート定期実行 |
| ScheduleCronTool | `ScheduleCronTool/` | スケジュール管理 |

---

## 3. ツール登録方式（tools.ts）

### 5 つの登録パターン

| パターン | 方式 | 例 |
|---|---|---|
| **1. 直接インポート** | 即座に有効 | BashTool, FileReadTool 等の基本ツール |
| **2. 条件付きロード** | `process.env` による分岐 | REPLTool → `USER_TYPE === 'ant'` |
| **3. フィーチャー条件** | feature flag による分岐 | `feature('PROACTIVE')`, `feature('AGENT_TRIGGERS')` |
| **4. 環境変数チェック** | 有効化フラグ | `ENABLE_LSP_TOOL`, `isWorktreeModeEnabled()`, `isAgentSwarmsEnabled()` |
| **5. 遅延ロード** | 循環依存回避 | `lazy require` パターン |

### 4 つの登録関数

```
getAllBaseTools()
  → 全ツールを返却（フィルタなし）

getTools(permissionContext)
  → 権限コンテキストに基づきフィルタリング

assembleToolPool(permissionContext, mcpTools)
  → 組み込みツール + MCP ツールを統合した完全なツールプール

filterToolsByDenyRules(tools, permissionContext)
  → ツール名マッチングによる拒否ルール適用
```

---

## 4. Tool インターフェース定義

```typescript
Tool<Input, Output, Progress> = {
  // 識別
  name: string,
  aliases?: string[],
  searchHint?: string,

  // コア機能
  call(input: Input, context: ToolContext): AsyncGenerator<Progress, Output>,
  description(context: DescriptionContext): string,

  // スキーマ
  inputSchema: ZodSchema<Input>,   // Zod v4
  outputSchema?: ZodSchema<Output>,

  // 特性宣言
  isConcurrencySafe(): boolean,    // 並行実行可能か
  isEnabled(): boolean,            // 有効か
  isReadOnly(): boolean,           // 読取専用か

  // オプション特性
  isDestructive?(): boolean,       // 破壊的操作か
  isSearchOrReadCommand?(): boolean,
  isOpenWorld?(): boolean,         // 外部アクセスか

  // 実行制御
  interruptBehavior?(): InterruptBehavior,
  requiresUserInteraction?(): boolean,

  // 分類フラグ
  isMcp?: boolean,
  isLsp?: boolean,
  shouldDefer?: boolean,           // 遅延ロード対象
  alwaysLoad?: boolean             // 常時ロード
}
```

---

## 5. 各ツールディレクトリの標準構成

```
ToolName/
├── ToolName.ts/tsx   — 実装本体（call(), inputSchema, description 等）
├── UI.tsx            — React コンポーネント（Ink による TUI 表示）
├── prompt.ts         — LLM に渡すプロンプトテンプレート
├── constants.ts      — 定数定義
└── [helpers].ts      — 補助機能（バリデーション、変換等）
```

全ツールがこの規約に従うことで、新規ツール追加時の認知負荷を最小化している。

---

## 6. セキュリティ分層

### BashTool のセキュリティスタック

BashTool は最も複雑なセキュリティ検証を持ち、5 層の防御で構成される。

| ファイル | 行数 | 責務 |
|---|---|---|
| `bashPermissions.ts` | 2,621 行 | コマンドの許可/拒否ルール検証 |
| `bashSecurity.ts` | 2,592 行 | 総合セキュリティチェック |
| `readOnlyValidation.ts` | 1,990 行 | 読取専用モードでの書込み検出 |
| `pathValidation.ts` | 1,303 行 | パストラバーサル・ディレクトリ外アクセス防止 |
| `sedValidation.ts` | 684 行 | sed コマンドによるファイル変更検証 |
| **合計** | **9,190 行** | |

### PowerShellTool のセキュリティスタック

Windows 環境向けに同等の保護を提供（合計 7,014 行）。

### セキュリティ検証フロー

```
ユーザー入力
  → コマンドパース
    → パス検証 (pathValidation)
      → 読取専用チェック (readOnlyValidation)
        → 許可ルール照合 (bashPermissions)
          → セキュリティポリシー適用 (bashSecurity)
            → 実行 or 拒否
```

---

## 7. 組み込みエージェント（6 種）

AgentTool は、用途別に事前定義された 6 種のエージェントプロファイルを持つ。

| エージェント | 用途 | 説明 |
|---|---|---|
| **General Purpose** | 汎用作業 | デフォルトのサブエージェント。ファイル操作・コード変更等の一般タスク |
| **Plan** | 計画策定 | 実装前の計画・設計フェーズ。コード変更は行わない |
| **Explore** | コード探索 | コードベース調査・理解。読取専用ツールのみ使用 |
| **Verification** | 検証・テスト | テスト実行・結果検証。品質保証フェーズ |
| **Claude Code Guide** | ガイダンス | Claude Code の使い方・設定に関する質問対応 |
| **Statusline Setup** | ステータス行設定 | ターミナルのステータスライン設定支援 |

各エージェントは独自のシステムプロンプトとツールアクセス権限を持ち、タスクの性質に応じて適切なエージェントが選択される。
