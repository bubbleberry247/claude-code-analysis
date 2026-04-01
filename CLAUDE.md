# claude-code-analysis

## 概要
instructkr/claude-code のソースコード解析プロジェクト。
2026-03-31 の npm sourcemap 流出事件で公開された Claude Code (Anthropic CLI) の
内部アーキテクチャ・設計パターン・未公開機能を体系的に文書化する。

## 解析対象リポジトリ
- GitHub: https://github.com/instructkr/claude-code
- 言語: TypeScript (strict mode)
- ランタイム: Bun / React+Ink (ターミナルUI)
- 規模: 1,902ファイル / 188,314行
- ローカル参照: `source/` (.gitignore除外)

## ディレクトリ構成
```
analysis/          ← アーキテクチャ解析ドキュメント（Markdown）
  ├── architecture.md, system-prompts.md, mcp-integration.md, ...
  ├── agents/, commands/, permissions/, startup/, tools/, ui/
  ├── unreleased-features.md
  └── codex-comparison.md
findings/          ← 発見事項・セキュリティ監査
  ├── security-audit.md
  ├── sourcemap-leak-verification.md
  └── tacit-knowledge.md
scripts/           ← 解析用スクリプト
source/            ← 解析対象ソース（.gitignore除外、コミットしない）
logs/              ← 実行ログ（.gitignore除外推奨）
```

## ルール
- 解析結果は `analysis/` 配下にMarkdownで記録
- 発見事項・メモは `findings/` に記録
- 解析スクリプトは `scripts/` に配置
- ソースコードそのものはこのリポジトリにコミットしない（参照のみ）
- `logs/` はローカル実行ログ用。コミット不要

## 解析ワークフロー
1. `source/` 内の対象コードをReadで精読
2. 構造・パターン・設計意図を分析
3. `analysis/` または `findings/` にMarkdownで文書化
4. GPT-5.4 (Codex) によるクロスレビューで精度向上

## 解析済みトピック一覧
- 全体アーキテクチャ（レイヤー設計、依存DAG）
- 起動シーケンス（15Phase、200ms最適化）
- ツールシステム（43ツール、Tool<I,O,P>）
- コマンドシステム（80+スラッシュコマンド）
- エージェント機構（Fork/Team/Coordinator）
- 権限モデル（5層セキュリティ、YOLO Classifier）
- システムプロンプト（静的/動的分離、3層キャッシュ）
- MCP統合（4ツール、5トランスポート、OAuth）
- UI層（React+Ink、Vim、カスタムStore）
- 未公開機能（KAIROS, ULTRAPLAN, BUDDY, Dream）
- Codex CLI比較（10項目）
- 暗黙知ガイド（CLAUDE.md/Memory/Agents/Skills/Hooks）
- セキュリティ監査（悪意コード・バックドアなし判定）
- Sourcemap流出検証（13項目中12項目実在確認）
