# OpenAI Codex CLI vs Claude Code 比較分析

## 1. 概要テーブル

| 項目 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| ライセンス | Apache 2.0（OSS） | プロプライエタリ |
| 主要言語 | Rust 94.7% | TypeScript（ソースmap流出版） |
| ビルド | Bazel + pnpm | Bun |
| Stars | 69.9k | 16.5k（流出アーカイブ） |
| UI | Terminal(Rust) + Desktop + Web | React+Ink(TUI) + Desktop + Web |
| 認証 | ChatGPT Plan / API Key | Claude.ai / API Key / Bedrock / Vertex AI |

---

## 2. 技術スタック比較

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| コア言語 | Rust 94.7% | TypeScript |
| ランタイム | ネイティブバイナリ（Rust compiled） | Bun（JavaScriptランタイム） |
| ビルドシステム | Bazel（企業級モノレポ対応）+ pnpm | Bun built-in bundler |
| パッケージ管理 | pnpm（JS部分） | Bun（npm互換） |
| 環境再現性 | Nix Flakes（完全再現ビルド） | npm/Bun lockfile |
| パフォーマンス特性 | Rustネイティブ、低レイテンシ、低メモリ | V8ベース、GC依存、起動はBunで高速化 |
| TUIフレームワーク | Rust製独自Terminal UI | React + Ink（宣言的TUI） |
| バリデーション | 不明 | Zod v4（ランタイムスキーマ検証） |

**所感**: Codexは「パフォーマンスファースト」のRust選択。Claude Codeは「開発速度ファースト」のTypeScript + Bun選択。企業級ビルドにBazel + Nix Flakesを採用するCodexは再現性に強い。

---

## 3. ツールシステム比較

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| ツール数 | Skills標準（数は非公開） | 43個の内蔵ツール |
| 定義方式 | SKILL.md（Markdown宣言） | `buildTool()` 統一インターフェース（TypeScript） |
| 発動方式 | Explicit（明示呼出）/ Implicit（自動推論） | LLMによる自動選択 + Permission Rules制御 |
| スキル標準 | Open Agent Skills標準（オープン仕様） | 独自仕様（非公開） |
| 外部ツール | MCP servers bundle対応 | MCP完全実装 + Hooks |
| ツール検証 | 不明 | 5層セキュリティで各ツール呼び出しを検証 |

### Codex: Open Agent Skills

```
SKILL.md
├── name / description / version
├── triggers（発動条件）
├── instructions（実行手順）
└── explicit / implicit 発動モード
```

- **Explicit**: ユーザーが明示的にスキル名を指定して呼び出し
- **Implicit**: エージェントがコンテキストから自動推論して発動

### Claude Code: buildTool() 統一インターフェース

```typescript
buildTool({
  name: "Read",
  description: "...",
  inputSchema: zodSchema,
  isEnabled: (config) => boolean,
  isReadOnly: () => boolean,
  call: async (input, context) => ToolResult,
  userFacingName: () => string,
})
```

- 全43ツールが同一インターフェースで定義
- `isEnabled`/`isReadOnly` によるコンテキスト依存の動的制御
- Permission Rules と連動した実行時検証

---

## 4. エージェント機構

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| マルチエージェント | 不明確（AGENTS.mdはコード品質ガイド） | 完全マルチエージェント |
| エージェント定義 | AGENTS.md = コーディング規約ファイル | 6種の組み込みエージェント + Custom Agents |
| オーケストレーション | 不明 | Fork / Team / Coordinator パターン |
| サブエージェント | 不明 | TaskAgent（独立コンテキスト、権限継承） |
| 並行実行 | 不明 | Fork（並行）/ Team（協調）/ Coordinator（監督） |

### Claude Code のエージェント構成

| エージェント種別 | 役割 |
|---|---|
| **TaskAgent** | サブタスク実行。親から権限・コンテキストを継承 |
| **Fork** | 複数サブエージェントを並行実行。独立コンテキスト |
| **Team** | 協調型。共有コンテキストでの分業 |
| **Coordinator** | 監督型。サブエージェントの結果を統合・判断 |
| **SendMessage** | 実行中エージェントへのメッセージ送信 |
| **Custom Agents** | `.claude/agents/*.md` でユーザー定義 |

### Codex の AGENTS.md

CodexのAGENTS.mdは「エージェント」の名前に反して、実際にはコーディング規約・品質ガイドラインファイル。マルチエージェントオーケストレーション機能は現時点で不明確。

---

## 5. 権限モデル

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| サンドボックス | 言及あり（詳細は developers.openai.com に委譲） | 5層セキュリティモデル |
| 権限粒度 | 不明 | ツール単位 × ファイルパス × モード |
| モード | 不明 | plan / autoEdit / fullAuto |
| ユーザー制御 | 不明 | Permission Rules（allow/deny、glob対応） |

### Claude Code: 5層セキュリティモデル

```
Layer 1: Permission Rules
  └─ .claude/settings.json で allow/deny パターン定義
       例: "Bash(npm test:*)" → allow

Layer 2: Mode Gating
  └─ plan: 読み取り専用
  └─ autoEdit: 編集許可、コマンド制限
  └─ fullAuto: 全ツール許可

Layer 3: Tool Validation
  └─ 各ツールの isEnabled() / isReadOnly() による動的判定
  └─ 入力スキーマの Zod バリデーション

Layer 4: Classifier
  └─ 危険コマンド検出（rm -rf, git push --force 等）
  └─ 秘密情報漏洩防止

Layer 5: Sandbox
  └─ macOS: Apple Seatbelt (sandbox-exec)
  └─ Linux: Docker / iptables
  └─ ファイルシステム・ネットワークの制限
```

### Codex: サンドボックス

- Sandboxの存在は言及されているが、詳細な実装ドキュメントは `developers.openai.com` に委譲
- OSSであるため、ソースコードから実装の監査が可能

---

## 6. MCP（Model Context Protocol）対応

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| MCP対応 | 不明確（MCP servers bundleの記述あり） | 完全実装 |
| MCPツール数 | 不明 | 4ツール（use_mcp_tool, access_mcp_resource 等） |
| トランスポート | 不明 | 5種（stdio, sse, streamable-http, docker, url） |
| OAuth対応 | 不明 | OAuth 2.1 PKCE + Dynamic Client Registration |
| Chrome連携 | 不明 | Chrome DevTools Protocol 経由のMCPブリッジ |
| サーバー管理 | MCP servers bundleとしてバンドル | `.claude/settings.json` で宣言的設定 |

### Claude Code の MCP 実装詳細

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

- **5トランスポート**: stdio（ローカルプロセス）、SSE（HTTP Server-Sent Events）、streamable-http（双方向HTTP）、docker（コンテナ内MCP）、url（リモートMCP + OAuth）
- **OAuth**: RFC準拠の2.1 PKCE + Dynamic Client Registration。トークンの自動更新対応
- **Chrome DevTools**: `--mcp-server-chrome-devtools` フラグでChromeのDevToolsをMCPサーバーとして利用可能

---

## 7. 拡張性

| 拡張メカニズム | OpenAI Codex CLI | Claude Code |
|---|---|---|
| Skills / Commands | Open Agent Skills（SKILL.md） | Skills（スラッシュコマンド） |
| Plugins | Plugins対応 | Plugins対応 |
| MCP | MCP servers bundle | 完全MCP実装（5トランスポート） |
| Hooks | 不明 | PreToolUse / PostToolUse / Notification等 |
| Agent SDK | 不明 | Claude Agent SDK（プログラマティック制御） |
| Custom Agents | AGENTS.md（コーディング規約） | `.claude/agents/*.md`（カスタムエージェント定義） |
| フォーク可能性 | **Apache 2.0で完全フォーク可能** | プロプライエタリ（フォーク不可） |
| コミュニティ貢献 | PR受付、Issue対応 | 非公開リポジトリ |

### 拡張アーキテクチャの違い

**Codex**: OSSベースの拡張モデル。Apache 2.0ライセンスにより、コア部分のフォーク・改変・再配布が自由。コミュニティ主導の拡張エコシステムが期待される。

**Claude Code**: プロプライエタリだが、多層の公式拡張ポイントを提供。Agent SDK によるプログラマティック制御、Hooks による実行フロー介入、Custom Agents によるドメイン特化エージェント定義など、コードを書かずに拡張できる仕組みが充実。

---

## 8. UI / フロントエンド対応

| プラットフォーム | OpenAI Codex CLI | Claude Code |
|---|---|---|
| CLI（ターミナル） | Rust製Terminal UI | React + Ink TUI |
| VS Code | 対応 | 対応 |
| Cursor | 対応 | 対応 |
| Windsurf | 対応 | 不明 |
| JetBrains | 不明 | 対応（IntelliJ, PyCharm等） |
| Desktop App | 対応 | 対応 |
| Web | 対応 | 対応 |
| iOS | 不明 | 対応 |
| Chrome DevTools | 不明 | 対応（MCP経由） |

**所感**: Claude Codeは JetBrains / iOS / Chrome DevTools への対応で幅広いプラットフォームカバレッジ。Codexは Windsurf 対応が確認されている。

---

## 9. オープンソース度合い

| 観点 | OpenAI Codex CLI | Claude Code |
|---|---|---|
| ライセンス | Apache 2.0 | プロプライエタリ |
| ソースコード公開 | 全コード公開（GitHub） | 非公開（バイナリ配布） |
| コミュニティ貢献 | PR受付、Issue対応 | 不可 |
| セキュリティ監査 | ソースコードから直接監査可能 | 不可（.mapファイル経由で事実上解析可能） |
| フォーク | 自由（Apache 2.0） | 不可 |
| ビルド再現性 | Nix Flakes で完全再現 | 不可（ビルドシステム非公開） |
| 流出状況 | N/A（公式OSS） | .map ファイル経由でTypeScriptソース事実上流出 |

### OSS戦略の違い

**OpenAI Codex CLI**: 完全OSSとして公開。企業・個人問わず自由にフォーク・改変・商用利用が可能。セキュリティ監査、カスタムビルド、独自拡張がすべてソースレベルで可能。Nix Flakesによるビルド再現性も企業導入に有利。

**Claude Code**: プロプライエタリだが、npm配布のバイナリに含まれる `.map` ファイルからTypeScriptソースが事実上復元可能な状態で流出。意図的なOSS化ではないため、ライセンス上はソースの利用・再配布は不可。Anthropicはこの状態を認識しつつも、公式OSS化の予定は不明。

---

## 10. 総合評価

### 選定基準別推奨

| 選定基準 | 推奨 | 理由 |
|---|---|---|
| OSS重視 | **Codex** | Apache 2.0、全コード公開、フォーク自由 |
| マルチエージェント | **Claude Code** | Fork/Team/Coordinator + 6種組み込みエージェント |
| MCP統合 | **Claude Code** | 5トランスポート、OAuth、Chrome連携の完全実装 |
| IDE多様性 | **Claude Code** | JetBrains / iOS / Chrome DevTools 対応 |
| セキュリティ監査 | **Codex** | ソースコード完全公開、Nix再現ビルド |
| カスタム拡張性 | **Claude Code** | Agent SDK + Hooks + Custom Agents の多層拡張 |
| パフォーマンス | **Codex** | Rustネイティブ、低レイテンシ、低メモリ |
| 企業導入（認証） | **Claude Code** | Bedrock / Vertex AI 経由のエンタープライズ認証 |
| コミュニティ | **Codex** | OSS PR受付、Issue対応、コミュニティ主導の進化 |
| ビルド再現性 | **Codex** | Bazel + Nix Flakes の企業級ビルドパイプライン |

### 総括

**OpenAI Codex CLI** と **Claude Code** は異なる設計思想に基づくコーディングエージェント。

- **Codex** は「OSSファースト・パフォーマンスファースト」。Rust + Bazel + Nix Flakes の技術選択は企業のセキュリティ監査・ビルド再現性要件に強い。Apache 2.0 によりフォーク・カスタマイズが完全に自由。ただしマルチエージェント機構やMCP統合の成熟度はClaude Codeに劣る。

- **Claude Code** は「拡張性ファースト・統合ファースト」。Agent SDK / Hooks / Custom Agents / MCP完全実装により、プログラマティックな拡張と外部ツール統合に優れる。5層セキュリティモデルとBedrock/Vertex AI認証でエンタープライズ要件もカバー。ただしプロプライエタリであり、ソースレベルの監査やカスタムビルドは公式にはサポートされない。

選定は「OSS・監査可能性」を重視するか「拡張性・統合性」を重視するかで分かれる。
