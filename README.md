# claude-code-analysis

Claude Code (Anthropic CLI) のソースコード解析プロジェクト。

## 解析対象
- ソース: [instructkr/claude-code](https://github.com/instructkr/claude-code)
- 規模: ~1,900ファイル / 512,000行+ (TypeScript)
- ランタイム: Bun / React+Ink (ターミナルUI)

## 解析領域

| 領域 | ディレクトリ | 概要 |
|---|---|---|
| アーキテクチャ | `analysis/` | 全体構成・レイヤー・データフロー |
| ツールシステム | `analysis/tools/` | ~40個のツール実装 (Bash, Edit, Grep等) |
| コマンド | `analysis/commands/` | ~50個のスラッシュコマンド |
| エージェント | `analysis/agents/` | AgentTool, TeamCreate, Coordinator |
| 権限モデル | `analysis/permissions/` | default/plan/auto の判定ロジック |
| 起動シーケンス | `analysis/startup/` | 並列プリフェッチ・遅延読み込み |
| UI | `analysis/ui/` | React+Ink ターミナルUI |
| スクリプト | `scripts/` | 統計・依存関係グラフ等の解析ツール |
| 発見事項 | `findings/` | 解析メモ・知見 |

## 解析優先順位
1. ツールシステム — 内部実装の理解
2. エージェント機構 — マルチエージェント設計
3. 権限モデル — 許可判定ロジック
4. 起動シーケンス — パフォーマンス最適化手法
