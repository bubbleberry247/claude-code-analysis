# claude-code-analysis

Claude Code (Anthropic CLI) のソースコード解析プロジェクト。
2026-03-31 の npm sourcemap 流出事件で公開されたソースコードの体系的な解析。

## 解析対象
- ソース: [instructkr/claude-code](https://github.com/instructkr/claude-code) (Stars: 16.5k)
- 規模: 1,902ファイル / 188,314行 (TypeScript)
- ランタイム: Bun / React+Ink (ターミナルUI)
- ビルド時DCE: `feature()` ゲートで未使用コード削除

## 解析ドキュメント一覧

### アーキテクチャ解析 (`analysis/`)

| ドキュメント | 内容 |
|---|---|
| [architecture.md](analysis/architecture.md) | 全体アーキテクチャ — 1,884ファイル、レイヤー設計、エントリーポイント、依存関係DAG |
| [startup/overview.md](analysis/startup/overview.md) | 起動シーケンス — 並列プリフェッチ、15Phase初期化、200ms起動最適化 |
| [tools/overview.md](analysis/tools/overview.md) | ツールシステム — 43ツール、Tool\<I,O,P\>インターフェース、5パターン登録方式 |
| [commands/overview.md](analysis/commands/overview.md) | コマンドシステム — 80+スラッシュコマンド、3コマンド型、スキル統合 |
| [agents/overview.md](analysis/agents/overview.md) | エージェント機構 — Fork/Team/Coordinator、6種組み込み、Mailbox通信 |
| [permissions/overview.md](analysis/permissions/overview.md) | 権限モデル — 5層セキュリティ、YOLO Classifier、サンドボックス |
| [system-prompts.md](analysis/system-prompts.md) | システムプロンプト — 静的/動的分離、3層キャッシュ、Undercover Mode |
| [mcp-integration.md](analysis/mcp-integration.md) | MCP統合 — 4ツール、5トランスポート、OAuth、Chrome連携 |
| [ui/overview.md](analysis/ui/overview.md) | UI層 — 611ファイル/121K行、React+Ink、Vim、カスタムStore |
| [unreleased-features.md](analysis/unreleased-features.md) | 未公開機能 — KAIROS、ULTRAPLAN、BUDDY、Dream System |
| [codex-comparison.md](analysis/codex-comparison.md) | Codex比較 — OpenAI Codex CLI vs Claude Code（10項目比較） |

### 発見事項 (`findings/`)

| ドキュメント | 内容 |
|---|---|
| [sourcemap-leak-verification.md](findings/sourcemap-leak-verification.md) | .map流出事件検証 — 記事で言及された13項目中12項目(92%)の実在確認 |
| [security-audit.md](findings/security-audit.md) | セキュリティ監査 — 悪意あるコード・バックドア・情報窃取なし判定 |

## 主要な発見

### アーキテクチャ
- **レイヤー設計**: UI(React+Ink) → ビジネスロジック(query/QueryEngine) → ツール(43種) → 状態管理(Zustand風カスタム) → ユーティリティ(180K行)
- **起動最適化**: MDM+Keychain並列プリフェッチ、feature()ビルドタイムDCE、API preconnect
- **プロンプトキャッシュ**: 静的/動的境界マーカーで3層キャッシュ、0.1x課金で再利用

### セキュリティ
- **5層権限モデル**: Permission Rules → Mode Gating → Tool Validation → YOLO Classifier → Sandbox
- **BashTool**: 9,190行のセキュリティコード（コマンド解析、IFS注入防止、パストラバーサル防止）
- **Undercover Mode**: 公開リポでの内部コードネーム漏洩防止（皮肉にも.mapで流出）

### 未公開機能
- **KAIROS**: セッション連続性（--continue、4時間TTL）
- **ULTRAPLAN**: 30分深い計画（CCRリモートコンテナ、Opus 4.6）
- **BUDDY**: AIコンパニオンペット（18種、ガチャ、shiny 1%）
- **Dream System**: 自動記憶統合（24h+5セッション三段階ゲート）

## ディレクトリ構成
```
claude-code-analysis/
├── analysis/
│   ├── architecture.md
│   ├── startup/overview.md
│   ├── tools/overview.md
│   ├── commands/overview.md
│   ├── agents/overview.md
│   ├── permissions/overview.md
│   ├── system-prompts.md
│   ├── mcp-integration.md
│   ├── ui/overview.md
│   ├── unreleased-features.md
│   └── codex-comparison.md
├── findings/
│   ├── sourcemap-leak-verification.md
│   └── security-audit.md
├── source/                 ← 解析対象ソース（.gitignore除外）
├── CLAUDE.md
└── README.md
```
