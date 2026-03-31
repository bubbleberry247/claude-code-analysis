# Claude Code エージェント機構 解析ドキュメント

## 1. 概要

Claude Code のエージェント機構は**階層型・非同期・マルチコンテキスト設計**で構築されている。
コアは `runAgent()` generator で、streaming API に対応し、親エージェントからのコンテキスト継承、
非同期バックグラウンド実行、チーム協調を統一的に処理する。

主要な設計原則:
- **Generator ベース**: `runAgent()` は async generator として実装され、streaming レスポンスを逐次 yield
- **コンテキスト継承**: 親エージェントのシステムプロンプト・MCP サーバー・権限を子に伝播
- **多様な実行モード**: 同期・非同期・チーム・リモートの 4 パターンを単一の `AgentTool` で統合

---

## 2. AgentTool 入力スキーマ

```typescript
{
  description: string           // 3-5語のタスク説明（必須）
  prompt: string                // 実行プロンプト（必須）
  subagent_type?: string        // エージェント種別（built-in/custom）
  model?: 'sonnet'|'opus'|'haiku'  // モデルオーバーライド
  run_in_background?: boolean   // 非同期実行フラグ
  isolation?: 'worktree'|'remote'  // 分離モード
  name?: string                 // アドレス可能な名前（チーム通信用）
  mode?: string                 // 権限モード（plan/trust等）
  team_name?: string            // 所属チーム名
}
```

| パラメータ | 必須 | デフォルト | 説明 |
|---|---|---|---|
| `description` | Yes | - | UIに表示されるタスク概要 |
| `prompt` | Yes | - | サブエージェントへの指示全文 |
| `subagent_type` | No | (general) | 組み込みエージェント種別を指定 |
| `model` | No | (親と同じ) | 使用モデルの明示指定 |
| `run_in_background` | No | `false` | `true` で非同期実行 |
| `isolation` | No | (なし) | `worktree`: git worktree分離、`remote`: CCR環境 |
| `name` | No | (自動生成) | チーム内でのアドレス名 |
| `mode` | No | (親と同じ) | 権限モード上書き |
| `team_name` | No | (なし) | チーム所属を指定 |

---

## 3. 実行フロー

### 処理ステップ

```
1. MCP サーバー初期化
   └─ 親のMCPサーバー設定を継承 + エージェント固有設定をマージ

2. システムプロンプト構築
   └─ 親プロンプトベース + エージェント定義の prompt/rules を注入

3. メッセージ準備
   ├─ Fork検出 → 親の会話履歴を継承（forkSubagent）
   └─ Worktree継承 → パス変換指示を付加

4. API Query 実行
   └─ streaming generator で逐次レスポンスを yield

5. トランスクリプト記録
   └─ 会話ログを保存（デバッグ・監査用）
```

### 出力パターン

| パターン | 条件 | 返却値 |
|---|---|---|
| `completed` | 同期実行が正常終了 | 最終テキスト出力 |
| `async_launched` | `run_in_background: true` | タスクID（後で `TaskOutput` で取得） |
| `teammate_spawned` | `team_name` 指定あり | Tmux セッション情報 |
| `remote_launched` | `isolation: 'remote'` | CCR 環境の接続情報 |

---

## 4. Fork メカニズム（forkSubagent.ts）

Fork は親エージェントの完全なコンテキストをサブエージェントに引き継ぐ機構。

```
親エージェント会話
  │
  ├─ feature('FORK_SUBAGENT') ゲート判定
  │
  ├─ 親の完全な会話コンテキスト継承
  │   └─ system prompt + message history + tool results
  │
  ├─ FORK_PLACEHOLDER_RESULT 挿入
  │   └─ プロンプトキャッシュヒット率を最大化
  │       （親と同一プレフィックスを維持）
  │
  ├─ buildChildMessage()
  │   ├─ 自己完結ルール強制
  │   │   ├─ サブエージェント spawn 禁止
  │   │   └─ コミット必須（変更がある場合）
  │   └─ タスク固有の指示を注入
  │
  └─ buildWorktreeNotice()
      └─ git worktree 使用時のパス変換指示
          （/original/path → /worktree/path）
```

**設計意図**: Fork により、サブエージェントは親の文脈を完全に理解した状態でタスクを開始できる。`FORK_PLACEHOLDER_RESULT` は API のプロンプトキャッシュ機構を活用し、共通プレフィックス部分のトークン消費を削減する。

---

## 5. 組み込みエージェント（6種）

| エージェント | モデル | ツール制限 | 用途 |
|---|---|---|---|
| **General Purpose** | default | `['*']` 全許可 | 複雑なマルチステップタスク。汎用的な実装・調査 |
| **Explore** | haiku（速度重視） | Agent/Edit/Write 禁止 | コードベース高速探索。読み取り専用で安全 |
| **Plan** | default | 読み取り専用 | 実装計画の設計・構造化 |
| **Verification** | default | - | 実装結果の検証・テスト実行 |
| **Claude Code Guide** | - | - | Claude Code 自体の使用方法説明 |
| **Statusline Setup** | - | - | ターミナルステータスライン設定 |

**選定ガイドライン**:
- 探索のみ → `Explore`（haiku で高速・低コスト）
- 計画立案 → `Plan`（書き込み不可で安全）
- 実装タスク → `General Purpose`（全ツール利用可能）
- 品質確認 → `Verification`（実装後の検証に特化）

---

## 6. チーム機構（TeamCreate / TeamDelete）

### チーム作成フロー

```
TeamCreate
  │
  ├─ TeamFile 生成
  │   ├─ name: string           // チーム表示名
  │   ├─ description: string    // チームの目的
  │   ├─ leadAgentId: string    // リードエージェントID
  │   └─ members: Member[]      // メンバー一覧
  │
  ├─ ユニーク名生成
  │   └─ generateWordSlug()     // 例: "brave-falcon", "swift-river"
  │
  ├─ Task list 初期化
  │   └─ チーム共有のタスクボード作成
  │
  └─ Session cleanup 登録
      └─ セッション終了時にチームリソースを自動解放
```

### チーム削除

```
TeamDelete
  │
  ├─ メンバーエージェントの停止
  ├─ タスクリストのクリーンアップ
  └─ TeamFile の削除
```

---

## 7. エージェント間通信（SendMessageTool）

### Mailbox システム

エージェント間の通信は Mailbox パターンで実装される。

| 送信先指定 | 形式 | 説明 |
|---|---|---|
| **Single** | `to: "teammate_name"` | 特定のチームメイトに直接送信 |
| **Broadcast** | `to: "*"` | チーム全員に一斉送信 |
| **Protocol (UDS)** | `to: "uds:<socket>"` | Unix Domain Socket 経由 |
| **Protocol (Bridge)** | `to: "bridge:<session-id>"` | セッションブリッジ経由 |

### 構造化メッセージ型

```
メッセージ型              用途
─────────────────────────────────────────
shutdown_request         エージェント停止要求
shutdown_response        停止応答（ACK）
plan_approval_response   計画承認の結果通知
```

### 通信フロー例

```
Agent-A                    Agent-B
  │                          │
  ├─ SendMessage ────────────>│
  │   to: "agent-b"          │
  │   body: "タスク完了"       │
  │                          │
  │<──────────── SendMessage ─┤
  │   to: "agent-a"          │
  │   body: "確認済み"         │
```

---

## 8. Coordinator Mode

```
feature('COORDINATOR_MODE') ゲート
  │
  ├─ 有効化条件
  │   └─ Feature flag が ON
  │
  ├─ 役割
  │   ├─ Worker エージェントの指揮
  │   ├─ 研究結果の synthesis（統合）
  │   └─ task-notification 形式で完了通知
  │
  └─ Worker Tools 制限
      ├─ 許可: Read, Grep, Glob, Bash（読取系）
      └─ 禁止: AgentTool, TeamCreate, TeamDelete 等
           （Worker がさらに Agent を spawn することを防止）
```

**設計意図**: Coordinator は「指揮者」として振る舞い、自身は実装を行わない。Worker に制限付きツールセットを与えることで、エージェントの無限再帰を防止する。

---

## 9. タスク管理

### タスクライフサイクル

```
TaskCreate ──> TaskUpdate ──> TaskOutput
    │              │              │
    │              │              └─ 非同期agent出力ファイル読込
    │              │
    │              ├─ status: pending → in_progress → completed / failed
    │              └─ owner: 自動割当（実行エージェントID）
    │
    ├─ ID生成（ユニークタスクID）
    └─ hooks実行（タスク作成時フック）
```

| ツール | 機能 |
|---|---|
| `TaskCreate` | タスク生成。ID自動採番 + hooks 実行 |
| `TaskUpdate` | ステータス更新 + owner 自動割当 |
| `TaskList` | チーム内タスク一覧取得 |
| `TaskGet` | 特定タスクの詳細取得 |
| `TaskOutput` | 非同期エージェントの出力ファイル読込 |
| `TaskStop` | 実行中タスクの abort |

---

## 10. エージェント定義ロード（loadAgentsDir.ts）

### 優先度（高→低）

```
Built-in  >  Plugin  >  User  >  Project  >  Flag  >  Policy
```

同名のエージェント定義が複数ソースに存在する場合、上位の定義が優先される。

### Frontmatter スキーマ

エージェント定義ファイル（`.md`）の YAML frontmatter で指定可能なフィールド:

| フィールド | 型 | 説明 |
|---|---|---|
| `tools` | `string[]` | 許可ツールリスト |
| `disallowedTools` | `string[]` | 禁止ツールリスト |
| `prompt` | `string` | エージェント固有プロンプト |
| `model` | `string` | 使用モデル指定 |
| `effort` | `string` | reasoning effort レベル |
| `permissionMode` | `string` | 権限モード（plan/trust等） |
| `mcpServers` | `object` | MCP サーバー設定 |
| `hooks` | `object` | ライフサイクルフック定義 |
| `maxTurns` | `number` | 最大ターン数制限 |
| `skills` | `string[]` | 利用可能スキル |
| `initialPrompt` | `string` | 初期プロンプト（起動時自動実行） |
| `memory` | `object` | メモリスコープ設定 |
| `background` | `boolean` | バックグラウンド実行フラグ |
| `isolation` | `string` | 分離モード（worktree/remote） |

---

## 11. メモリ管理

### スコープ構成

| スコープ | パス | 用途 |
|---|---|---|
| **User** | `~/.claude/agent-memory/{agentType}/MEMORY.md` | ユーザー固有・プロジェクト横断の記憶 |
| **Project** | `.claude/agent-memory/{agentType}/MEMORY.md` | プロジェクト固有の記憶（Git管理可） |
| **Local** | `.claude/agent-memory-local/{agentType}/` | ローカル専用（Git除外） |

### Snapshot 同期機構

```
Agent実行中
  │
  ├─ メモリ書き込み（MEMORY.md更新）
  │
  ├─ Snapshot作成
  │   └─ タイムスタンプ付きバックアップ
  │
  └─ 同期
      └─ User ↔ Project 間で必要に応じて伝播
```

**設計意図**: エージェントごとに独立したメモリ空間を持つことで、異なるエージェント種別（Explore, Plan 等）が互いの記憶を汚染しない。Snapshot により、メモリ破損時のロールバックが可能。

---

## 12. データフロー図

```
┌──────────────────────────────────────────────────────────────────────┐
│                        User Input                                    │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       v
              ┌────────────────┐
              │  AgentTool.tsx  │  ← パラメータ解析・バリデーション
              └───────┬────────┘
                      │
          ┌───────────┼───────────┬──────────────┐
          │           │           │              │
          v           v           v              v
     ┌────────┐ ┌─────────┐ ┌────────┐  ┌───────────┐
     │  Sync  │ │  Async  │ │  Team  │  │  Remote   │
     │ (直接) │ │ (BG)    │ │(Tmux)  │  │  (CCR)    │
     └───┬────┘ └────┬────┘ └───┬────┘  └─────┬─────┘
         │           │          │              │
         └───────────┼──────────┼──────────────┘
                     │          │
                     v          v
              ┌────────────────────┐
              │    runAgent()      │  ← async generator（コア）
              │  ┌──────────────┐  │
              │  │ MCP Init     │  │
              │  │ Prompt Build │  │
              │  │ Msg Prepare  │  │
              │  └──────┬───────┘  │
              └─────────┼──────────┘
                        │
                        v
              ┌────────────────────┐
              │    query() API     │  ← Claude API streaming call
              └────────┬───────────┘
                       │
                       v
              ┌────────────────────┐
              │  Tool Execution    │  ← Bash, Read, Edit, Grep, etc.
              │  (recursive loop)  │
              └────────┬───────────┘
                       │
                       v
              ┌────────────────────────┐
              │  finalizeAgentTool()   │  ← 結果集約・クリーンアップ
              └────────┬───────────────┘
                       │
          ┌────────────┼────────────┬──────────────┐
          │            │            │              │
          v            v            v              v
     ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐
     │completed│ │async_    │ │teammate_│ │remote_    │
     │         │ │launched  │ │spawned  │ │launched   │
     │(テキスト)│ │(タスクID) │ │(Tmux)   │ │(CCR info) │
     └─────────┘ └──────────┘ └─────────┘ └───────────┘
                       │
                       v
              ┌────────────────────┐
              │    User Output     │
              └────────────────────┘
```

---

## 付録: 用語集

| 用語 | 説明 |
|---|---|
| **CCR** | Claude Code Remote。リモート環境でのエージェント実行基盤 |
| **MCP** | Model Context Protocol。ツール・リソースの標準化プロトコル |
| **Worktree** | Git worktree を利用した分離実行環境 |
| **Fork** | 親エージェントのコンテキストを完全継承してサブエージェントを生成する機構 |
| **Mailbox** | エージェント間の非同期メッセージングパターン |
| **Coordinator** | Worker を指揮するだけで自身は実装しないエージェントモード |
| **Slug** | `generateWordSlug()` で生成されるユニークな短い識別子（例: "brave-falcon"） |
