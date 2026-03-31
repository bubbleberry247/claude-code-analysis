# claude-code-analysis

## 概要
instructkr/claude-code のソースコード解析プロジェクト。
Claude Code (Anthropic CLI) の内部アーキテクチャ・設計パターンを文書化する。

## 解析対象リポジトリ
- GitHub: https://github.com/instructkr/claude-code
- 言語: TypeScript (厳密モード)
- 規模: ~1,900ファイル / 512,000行+

## ルール
- 解析結果は `analysis/` 配下にMarkdownで記録
- 発見事項・メモは `findings/` に記録
- 解析スクリプトは `scripts/` に配置
- ソースコードそのものはこのリポジトリにコミットしない（参照のみ）
