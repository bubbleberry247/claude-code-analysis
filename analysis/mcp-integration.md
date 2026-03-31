# Claude Code MCP統合 解析ドキュメント

## 1. 概要

Model Context Protocol（MCP）は、Claude Codeが外部ツール・リソースと統合するための標準プロトコル。
LLMが外部システムの機能を「ツール」として呼び出し、「リソース」としてデータを読み取る仕組みを提供する。

### コアコンポーネント

| コンポーネント | 役割 |
|---|---|
| **MCPTool** | 外部サーバーのツールをClaude Codeツールとして公開 |
| **ListMcpResourcesTool** | サーバーが公開するリソース一覧を取得 |
| **ReadMcpResourceTool** | 特定リソースの内容を読み取り |
| **McpAuthTool** | 未認証サーバーへのOAuth認証フローを開始 |

### トランスポート

```
stdio   — ローカルプロセス（stdin/stdout）。最も一般的
sse     — Server-Sent Events（HTTP長時間接続）
http    — Streamable HTTP（sse の後継、セッション管理付き）
ws      — WebSocket
sdk     — SDK内蔵サーバー（インプロセス）
```

### 設定スコープ

```
local       — .mcp.json（プロジェクトローカル）
user        — ~/.claude/settings.json（ユーザーグローバル）
project     — .claude/settings.json（プロジェクト共有）
dynamic     — ランタイム中に追加
enterprise  — 組織管理者設定
claudeai    — Claude AI組み込み（Gmail, Calendar等）
managed     — マネージド環境用
```

---

## 2. MCPツール群

### MCPTool

外部MCPサーバーが公開するツールをClaude Codeのツールとしてラップする。

```
特徴:
- isMcp: true フラグで識別
- スキーマはサーバーから取得したものをそのままpassthrough
- mcpClient.ts でツール呼び出しをオーバーライド（callTool → JSON-RPC）
- ツール実行結果は content[] 配列（text / image / resource）
```

### ListMcpResourcesTool

```
特徴:
- shouldDefer: true（遅延ロード、必要時にスキーマ取得）
- LRUキャッシュでリソース一覧をキャッシュ
- サーバーからの resources/list_changed 通知でキャッシュ無効化
- レスポンス: uri, name, mimeType の一覧
```

### ReadMcpResourceTool

```
特徴:
- URI指定でリソース内容を読み取り
- テキストリソース → そのまま返却
- バイナリblob → base64デコード後ディスクに一時保存 → ファイルパスを返却
- mimeType に基づく適切なハンドリング
```

### McpAuthTool

```
特徴:
- 未認証（needs-auth）状態のサーバーに対して擬似ツールを表示
  例: mcp__github__authenticate
- ユーザーがツールを呼び出すとOAuthフロー開始
- 認証完了後、擬似ツールを削除し本物のサーバーツール群に入れ替え
- セッション中のシームレスな認証体験を提供
```

---

## 3. サーバー接続状態

MCPサーバーは以下の5つの状態を持つ:

| 型 | 状態 | 説明 | 保持するデータ |
|---|---|---|---|
| `ConnectedMCPServer` | 接続済み | 正常稼働中 | client, capabilities, cleanup関数 |
| `NeedsAuthMCPServer` | 認証必要 | OAuth未完了 | authUrl, serverId |
| `FailedMCPServer` | 接続失敗 | エラーで接続不可 | error, retryCount |
| `PendingMCPServer` | 接続中 | 初期化処理中 | connectPromise |
| `DisabledMCPServer` | 無効化 | ユーザーが明示的に無効化 | disabledReason |

```
状態遷移図:

  Pending ──成功──> Connected
    │                  │
    │              401/期限切れ
    │                  │
    ├──失敗──> Failed   └──> NeedsAuth ──認証完了──> Connected
    │            │
    │          リトライ
    │            │
    │            └──> Pending
    │
    └──ユーザー無効化──> Disabled
```

---

## 4. ツール名前空間

MCPツールはグローバルに一意な名前で識別される。

### 命名パターン

```
mcp__<normalized_server_name>__<normalized_tool_name>
```

### 正規化ルール

- スペース、ハイフン、ドット → アンダースコア
- 連続アンダースコア → 単一アンダースコア
- 先頭/末尾のアンダースコア除去

### 例

| サーバー名 | ツール名 | 正規化後 |
|---|---|---|
| github | search_code | `mcp__github__search_code` |
| claude-ai/Slack | post_message | `mcp__claude_ai_Slack__post_message` |
| my.db-server | run_query | `mcp__my_db_server__run_query` |
| browser-tools | takeScreenshot | `mcp__browser_tools__takeScreenshot` |

---

## 5. ツール統合（assembleToolPool）

Claude Codeが利用可能なツール一覧を構築するパイプライン:

```
Step 1: 組み込みツール取得
  getTools() → Read, Write, Edit, Bash, Grep, Glob, ...

Step 2: MCPツール取得 + フィルタ
  全接続サーバーのツール取得
  → denyルール適用（settings.json の deny リスト）
  → 許可されたツールのみ通過

Step 3: 名前デデュプ（重複排除）
  組み込みツール名とMCPツール名が衝突した場合
  → 組み込みツールが優先（MCPツールは除外）

Step 4: ソート
  プロンプトキャッシュの安定性のため、ツール名でアルファベット順ソート
  → キャッシュヒット率向上
```

```
┌─────────────┐   ┌──────────────┐   ┌───────────┐   ┌──────────┐
│ Built-in    │   │ MCP Servers  │   │ Deny      │   │ Dedupe   │
│ Tools       │──>│ Tools        │──>│ Filter    │──>│ & Sort   │──> Final Pool
│ (優先)      │   │ (全サーバー) │   │ (除外)    │   │ (安定順) │
└─────────────┘   └──────────────┘   └───────────┘   └──────────┘
```

---

## 6. 設定階層

設定は複数のソースからマージされ、より具体的なスコープが優先される。

```
優先度（低 → 高）:

  Global Settings          ~/.claude/settings.json
       ↓
  Project Settings         .claude/settings.json
       ↓
  Local MCP Config         .mcp.json
       ↓
  Plugin Config            プラグインが提供する設定
       ↓
  Enterprise Config        組織管理者の強制設定（最優先）
```

### .mcp.json の構造例

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "..."
      }
    },
    "slack": {
      "type": "sse",
      "url": "https://mcp.slack.example.com/sse"
    }
  }
}
```

### settings.json のMCP関連設定

```json
{
  "mcpServers": { ... },
  "permissions": {
    "deny": ["mcp__dangerous_server__*"]
  }
}
```

---

## 7. OAuth認証フロー

MCPサーバーが認証を要求する場合のフロー:

```
1. メタデータ発見
   GET /.well-known/oauth-authorization-server
   → authorization_endpoint, token_endpoint, scopes 取得

2. 利用可能ポート探索
   localhost でリダイレクト用HTTPサーバーを起動するポートを探索
   → ランダムポート or 設定済みポートから利用可能なものを選択

3. SDK authフロー開始
   PKCE付きAuthorization Code Flowを開始
   → ブラウザでauthorization_endpointを開く
   → ユーザーが認可
   → リダイレクトでauthorization_codeを受信
   → token_endpointでアクセストークン取得

4. キーチェーン保存
   取得したトークンをOSのキーチェーン/資格情報ストアに保存
   → 次回起動時に自動利用
```

### XAA（Cross-App Access）

```
- Claude AI組み込みサーバー（Gmail, Calendar等）で使用
- Anthropicアカウントと連携済みのOAuthトークンを利用
- 通常のOAuthフローをバイパスし、XAAログインで認証
```

### headersHelper

```
- シェルスクリプトで動的にHTTPヘッダーを生成
- サーバー設定に headersHelper コマンドを指定
- リクエスト毎にスクリプトを実行し、出力をヘッダーとして付与
- 用途: 短寿命トークン、カスタム認証スキーム
```

---

## 8. 結果折りたたみ（classifyForCollapse）

MCPツールの実行結果が長大な場合、UIで自動折りたたみを行う仕組み。

### 分類ロジック

```
classifyForCollapse(toolName, serverName):
  1. サーバー名を正規化マッチング（100+パターン）
  2. ツール名からアクション種別を推定
  3. isSearch / isRead フラグを設定
  4. フラグに応じて折りたたみ表示を決定
```

### 対応サーバー（20+）

| カテゴリ | サーバー |
|---|---|
| コミュニケーション | Slack, Gmail, Discord |
| 開発 | GitHub, GitLab, Bitbucket |
| プロジェクト管理 | Linear, Jira, Asana, Notion |
| 監視 | Datadog, Sentry, PagerDuty |
| データ | PostgreSQL, MongoDB, BigQuery |
| その他 | Confluence, Zendesk, Intercom |

### 表示制御

```
isSearch: true  → 検索結果を折りたたみ、件数のみ表示
isRead: true    → 読み取り結果を折りたたみ、サマリー表示
どちらもfalse   → 結果をそのまま展開表示
```

---

## 9. エラー処理

### McpAuthError

```
発生条件: HTTP 401 レスポンス
処理:
  1. サーバー状態を needs-auth に遷移
  2. McpAuthTool（擬似認証ツール）を生成
  3. ユーザーに認証を促すメッセージ表示
```

### McpSessionExpiredError

```
発生条件: HTTP 404 + JSON-RPC エラーコード -32001
処理:
  1. セッション期限切れを検知
  2. 自動再接続を試行
  3. 新セッションでリクエストを再送
  4. 再接続失敗時は Failed 状態に遷移
```

### McpToolCallError

```
発生条件: ツール実行中のエラー（タイムアウト、サーバーエラー等）
処理:
  1. _meta フィールドを保持（デバッグ情報）
  2. テレメトリ用に安全なメッセージを生成（秘密情報を除外）
  3. ユーザーにはエラー概要のみ表示
  4. リトライはLLMの判断に委ねる
```

### エラー処理フロー図

```
ツール呼び出し失敗
       │
       ├── 401 → McpAuthError → needs-auth状態 → 認証ツール表示
       │
       ├── 404 + -32001 → McpSessionExpiredError → 自動再接続
       │
       ├── タイムアウト → McpToolCallError → エラー応答
       │
       └── その他 → McpToolCallError → エラー応答 + テレメトリ
```

---

## 10. Chrome連携

Claude CodeがChromeブラウザと連携するための2つのチャネル。

### WebSocket Bridge

```
接続先: wss://bridge.claudeusercontent.com
用途:   リモート環境（SSH等）からのChrome連携
フロー:
  1. Chrome拡張がBridge WebSocketに接続
  2. Claude CodeもBridge WebSocketに接続
  3. DeviceIDベースのペアリングでマッチング
  4. Bridge経由でメッセージ中継
```

### Native Socket

```
接続先: ~/.claude/chrome.sock（Unix Domain Socket）
用途:   ローカル環境でのChrome連携
フロー:
  1. Chrome拡張がローカルソケットに接続
  2. Claude Codeが同じソケットに接続
  3. 直接通信（低レイテンシ）
```

### ペアリング

```
- DeviceIDをローカルに保存（~/.claude/device_id）
- Chrome拡張とClaude Codeが同じDeviceIDを共有
- Bridge経由の場合、DeviceIDでセッションをマッチング
- ペアリング確立後はツール呼び出し（スクリーンショット取得等）が可能
```

### 接続選択ロジック

```
if (Native Socket 利用可能):
    → Native Socket（優先、低レイテンシ）
else if (WebSocket Bridge 利用可能):
    → WebSocket Bridge（リモート環境向け）
else:
    → Chrome連携なし
```
