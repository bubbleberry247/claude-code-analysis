# Claude Code システムプロンプト構成の解析

## 1. 概要

Claude Code のシステムプロンプトは**段階的セクション化** + **プロンプトキャッシュ戦略**で設計されている。

- **静的セクション**（キャッシュ可能）: モデルの振る舞い・ツール定義など、セッション間で不変の部分
- **動的セクション**（セッション毎に再計算）: 環境情報・ユーザー設定・MCP接続など

両者は `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` マーカーで分離され、API レベルでキャッシュ制御される。

```
┌─────────────────────────────────────────┐
│  静的セクション（キャッシュ可能）         │
│  ─ intro, system, tasks, actions, tools │
│  ─ tone, output_efficiency              │
├─── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──────┤
│  動的セクション（セッション毎）          │
│  ─ session_guidance, memory, env_info   │
│  ─ language, mcp_instructions, etc.     │
└─────────────────────────────────────────┘
```

---

## 2. プロンプト構成の全体構造

### 静的セクション（キャッシュ可能）

| セクション | 関数名 | 役割 |
|---|---|---|
| Intro | `getSimpleIntroSection()` | 自己紹介 + `CYBER_RISK_INSTRUCTION` 埋め込み |
| System | `getSimpleSystemSection()` | ツール許可リスト、`system-reminder` 処理、フック処理ルール |
| Tasks | `getSimpleDoingTasksSection()` | タスク実行原則、コード品質基準、セキュリティガイドライン |
| Actions | `getActionsSection()` | 可逆性・爆発範囲（blast radius）検討、リスク管理方針 |
| Tools | `getUsingYourToolsSection()` | 各ツール固有の使用ルール（後述） |
| Tone | `getSimpleToneAndStyleSection()` | 応答トーン・スタイル指針 |
| Output | `getOutputEfficiencySection()` | 出力効率化（冗長性回避、トークン節約） |

### 動的セクション（セッション毎に再計算）

| セクション | 説明 | キャッシュ |
|---|---|---|
| `session_guidance` | セッション固有のガイダンス | 通常キャッシュ |
| `memory` | CLAUDE.md 等のメモリファイル内容 | 通常キャッシュ |
| `env_info_simple` | CWD, git status, OS, shell 情報 | 通常キャッシュ |
| `language` | ユーザー指定言語 | 通常キャッシュ |
| `output_style` | 出力スタイル設定 | 通常キャッシュ |
| `mcp_instructions` | MCP サーバー接続情報 | **DANGEROUS_uncached** |
| `scratchpad` | 内部メモ領域 | 通常キャッシュ |
| `frc` | FRC（Feature Release Control）フラグ | 通常キャッシュ |
| `summarize_tool_results` | ツール結果要約設定 | 通常キャッシュ |
| `token_budget` | トークンバジェット制約 | 通常キャッシュ |
| `brief` | 簡潔応答モード | 通常キャッシュ |

---

## 3. セクション管理（systemPromptSections.ts）

### データ構造

```typescript
type SystemPromptSection = {
  name: string;        // セクション識別子
  compute: () => Promise<string>;  // 内容生成関数
  cacheBreak: boolean; // true = キャッシュ無効化
};
```

### セクション生成関数

| 関数 | cacheBreak | 用途 |
|---|---|---|
| `systemPromptSection()` | `false` | 通常セクション。キャッシュ可能 |
| `DANGEROUS_uncachedSystemPromptSection()` | `true` | 毎ターン再計算。MCP接続情報等 |

### 解決フロー

```
resolveSystemPromptSections()
  │
  ├── Promise.all() で全セクションを並列実行
  │
  ├── キャッシュチェック
  │   ├── cacheBreak: false → 前回結果と比較、変更なければ再利用
  │   └── cacheBreak: true  → 常に再計算
  │
  └── セクション結合 → 最終システムプロンプト
```

---

## 4. キャッシュ戦略と API 課金

### 3層キャッシュ構造

```
┌──────────────────────────────────────────────┐
│ 層1: グローバルキャッシュ                      │
│   全ユーザー共有の静的プロンプト部分           │
│   課金: 0.1x（通常の1/10）                    │
│   対象: intro, system, tasks, actions, tools  │
├──────────────────────────────────────────────┤
│ 層2: ツールキャッシュ                          │
│   セッション単位で分離                         │
│   課金: 0.1x（通常の1/10）                    │
│   対象: ツール定義JSON、memory内容             │
├──────────────────────────────────────────────┤
│ 層3: セクション内メモライズ                    │
│   compute() の計算結果をメモリ保持             │
│   課金: なし（API送信前の最適化）              │
│   対象: 全セクションの compute() 結果          │
└──────────────────────────────────────────────┘
```

### DANGEROUS_uncached のコスト問題

| 問題 | 詳細 |
|---|---|
| 発生条件 | MCP サーバーの接続・切断 |
| 影響範囲 | `mcp_instructions` セクション全体がキャッシュ無効化 |
| コスト | 約 50-100K トークンが毎ターン再計算（キャッシュ割引なし） |
| 対策（進行中） | delta attachments 方式への移行（差分のみ送信） |

---

## 5. 主要ツールのプロンプト要約

### BashTool

| 項目 | ルール |
|---|---|
| コマンド禁止 | `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, `echo` → 専用ツール使用 |
| Git 安全プロトコル | `--force`, `reset --hard` 等は明示的指示がない限り禁止 |
| コミット | 新規コミット優先（`--amend` は明示的指示時のみ） |
| Undercover Mode | 有効時はコードネーム・内部情報をコミットメッセージに含めない |
| タイムアウト | デフォルト 120,000ms、最大 600,000ms |
| バックグラウンド | `run_in_background` パラメータで非同期実行可 |

### FileEditTool

| 項目 | ルール |
|---|---|
| 前提条件 | **Read 必須**（先にファイルを読んでからでないと Edit 不可） |
| old_string | ファイル内で**一意**でなければエラー。周辺コンテキストを含めて一意にする |
| 行番号 | Read 出力の行番号プレフィックスを `old_string` に含めない |
| replace_all | 変数リネーム等、全置換が必要な場合に使用 |
| 絵文字 | ユーザーの明示的要求がない限り含めない |

### FileReadTool

| 項目 | ルール |
|---|---|
| パス | 絶対パス必須 |
| 行数制限 | デフォルト 2,000行（`limit` パラメータで調整可） |
| 対応形式 | テキスト、画像（PNG, JPG 等）、PDF（最大20ページ/リクエスト）、Jupyter Notebook |
| PDF | 10ページ超は `pages` パラメータ必須 |
| 出力形式 | `cat -n` 形式（行番号付き、1始まり） |

### GrepTool

| 項目 | ルール |
|---|---|
| エンジン | ripgrep ベース |
| コマンド禁止 | `grep`, `rg` の Bash 直接実行は禁止 |
| 出力モード | `content`（行内容）、`files_with_matches`（ファイルパスのみ、デフォルト）、`count`（件数） |
| マルチライン | `multiline: true` で複数行パターンマッチ可能 |
| 制限 | `head_limit` デフォルト 250（0 で無制限、大規模結果は非推奨） |

### AgentTool

| 項目 | ルール |
|---|---|
| `subagent_type` 指定あり | **新規エージェント**を生成（クリーンなコンテキスト） |
| `subagent_type` 省略 | **現在のエージェントをフォーク**（コンテキスト継承） |
| 用途 | 並列タスク実行、独立した調査、長時間処理の委譲 |

---

## 6. Undercover Mode

### 判定ロジック

```
isUndercover()
  ├── ant-only ビルド？ → Yes
  ├── 内部リポジトリ許可リスト外？ → Yes
  └── それ以外 → No（通常モード）
```

### 制約事項

| 禁止項目 | 例 |
|---|---|
| コードネーム | "Capybara", "Pangolin" 等のモデルコードネーム |
| 未発表モデル番号 | 内部的なモデルバージョン識別子 |
| 内部リポジトリ名 | Anthropic 社内リポジトリへの参照 |
| Slack 短縮リンク | 社内 Slack のリンク |
| Co-Authored-By | Undercover 時はコミットに含めない |

### 正規化の例

```
❌ "Fix bug with Claude Capybara"
✅ "Fix race condition in file watcher"

❌ "Update from internal repo xyz"
✅ "Update dependency configuration"
```

---

## 7. セキュリティ指示（CYBER_RISK_INSTRUCTION）

### 配置

全システムプロンプトの**冒頭**（`getSimpleIntroSection()` 内）に埋め込み。

### 許可される活動

| カテゴリ | 説明 |
|---|---|
| CTF | Capture The Flag の演習・解法支援 |
| 教育セキュリティ | セキュリティ概念の学習・教育目的 |
| 認可ペネテスト | 明示的な許可があるペネトレーションテスト |
| 防御的セキュリティ | 脆弱性検出、パッチ適用、セキュリティ強化 |

### 拒否される活動

| カテゴリ | 説明 |
|---|---|
| 非認可ペネテスト | 許可のないシステムへの侵入試行 |
| DoS 攻撃 | サービス妨害攻撃の作成・実行 |
| サプライチェーン compromise | 依存関係への悪意あるコード注入 |
| マルウェア検出回避 | アンチウイルス・EDR の回避手法 |

---

## 8. /security-review コマンド

### 検査対象

| カテゴリ | 検査内容 |
|---|---|
| Injection | SQL Injection, Command Injection, XXE, Template Injection |
| 認証・認可 | Auth bypass, 権限昇格, セッション管理 |
| 暗号化 | 弱い暗号アルゴリズム, 不適切な鍵管理 |
| RCE | リモートコード実行の可能性 |
| XSS | クロスサイトスクリプティング |
| データ漏洩 | 機密情報の意図しない露出 |

### 報告基準

```
信頼スコア ≥ 8.0 → 報告対象
信頼スコア < 8.0 → 抑制（誤検知回避）
```

### 明示的な除外項目

| 除外項目 | 理由 |
|---|---|
| DoS | 可用性の問題であり、コードレビューの範囲外 |
| ディスク上のシークレット | 別途 secret scanning ツールの責務 |
| メモリ安全性 | 言語ランタイムの責務（Rust/C 以外では低優先） |
| AI プロンプトインジェクション | 別カテゴリの問題として扱う |
