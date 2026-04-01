# Claude Code 暗黙知ガイド — CLAUDE.md / Memory / Agents / Skills / Hooks の書き方

実際のソースコード解析から抽出した、ドキュメントに書かれていない設計知識。

---

## 1. CLAUDE.md の暗黙知

### 読み込み順序と優先度

| 順序 | ソース | 説明 |
|------|--------|------|
| 1 | Managed | 組織管理者が設定した指示 |
| 2 | User (`~/.claude/`) | ユーザーのグローバル設定 |
| 3 | Project (root → CWD) | プロジェクトルートからCWDに向かって走査 |
| 4 | Local | `.local` 付きファイル |

**CWDに近いほど優先度が高い**（後に読み込まれるため上書きされる）。

各ディレクトリ内の読み込み順:

```
CLAUDE.md → .claude/CLAUDE.md → .claude/rules/*.md → CLAUDE.local.md
```

`CLAUDE.local.md` は同ディレクトリ内で最高優先度を持つ。

### サイズ制限

| 項目 | 値 |
|------|-----|
| MAX_MEMORY_CHARACTER_COUNT | **40,000文字** |
| プロンプト注入位置 | ユーザーコンテキストとして毎ターン冒頭にprepend |
| 固定プレフィックス | `"IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written."` |

### キャッシュ挙動

- `memoize` でセッション全体キャッシュされる
- `/clear` でリセットされる
- **実行中にCLAUDE.mdを編集しても反映されない**（セッション再起動が必要）

### @include 構文

テキストノードのみから抽出される（コードブロック内では無視）。

```markdown
# 以下は有効（テキストノード）
@./path/to/file

# 以下は無視（コードブロック内）
```
@./path/to/file
```
```

| 構文 | 意味 |
|------|------|
| `@./path/to/file` | プロジェクト内ファイルを参照 |
| `@~/path` | ホームディレクトリ参照 |

User memory（`~/.claude/`配下）は `includeExternal: true` が有効で、プロジェクト外ファイルも参照可能。

### .claude/rules/ の条件付きルール

YAML frontmatter で適用条件を指定できる:

```yaml
---
paths:
  - src/components/**
  - tests/**
---
# このルールはマッチするファイル操作時のみ適用される
```

`paths` に glob パターンを指定すると、そのパターンにマッチするファイルが操作対象のときだけルールが有効化される。

### /init が生成する推奨コンテンツ

**書くべき:**

- 非標準の build / test / lint コマンド
- デフォルトと異なるコード規約
- テストの quirks（特殊な振る舞い）
- ブランチ命名 / PR規約 / commit スタイル
- 必須環境変数
- 非明白な gotchas・アーキテクチャ決定

**書くべきでない:**

- ファイル一覧（Claude が自力で発見可能）
- 標準言語規約（Claude が既知の知識として持っている）
- `"write clean code"` 的なジェネリックアドバイス
- 詳細 API docs → `@path` で参照に留める
- 頻繁に変わる情報 → `@path` で最新を読み込ませる

---

## 2. Memory の暗黙知

### ディレクトリ構造

```
~/.claude/projects/{sanitized-git-root}/memory/
├── MEMORY.md          ← インデックス（コンテンツを直接書かない）
├── feedback_xxx.md    ← 個別メモリファイル（frontmatter付き）
├── reference_xxx.md
└── ...
```

### MEMORY.md の制限

| 項目 | 値 |
|------|-----|
| MAX_ENTRYPOINT_LINES | **200行** |
| MAX_ENTRYPOINT_BYTES | **25,000バイト** |
| 各行の文字数目安 | ~150文字以内 |
| 各行の形式 | `- [Title](file.md) — one-line hook` |
| frontmatter | **なし**（MEMORY.md自体にはfrontmatterを書かない） |

### メモリファイルの frontmatter

```yaml
---
name: メモリ名
description: 1行説明（関連性判定に使用、具体的に書く）
type: user|feedback|project|reference
---
```

- `description` が関連性判定の**鍵**。Sonnet がこのフィールドを見てメモリの選択を行う
- `type` が無効な値でもエラーにはならず、`undefined` として処理される

### 4つの型の使い分け

| 型 | 用途 | 共有 |
|----|------|------|
| **user** | 役割・スキル・ドメイン知識・設定 | 常に private |
| **feedback** | 修正指示・確認パターン・理由とコンテキスト | private |
| **project** | 期限・決定・リスク・外部制約 | team で共有推奨 |
| **reference** | 外部システムへのポインタ | private |

### 保存禁止（ハード制約）

以下の情報はメモリに**保存してはならない**:

- コードパターン・規約・アーキテクチャ・ファイルパス（discoverable）
- Git 履歴・最近の変更（`git log` / `blame` で取得可能）
- デバッグ解法（修正はコード内、コンテキストは commit message で管理）
- CLAUDE.md に既に書かれている情報（二重管理の禁止）
- 一時的な状態・進行中の作業

### 自動抽出の発動条件

3つのゲートを順に通過する必要がある:

| ゲート | 条件 | 性質 |
|--------|------|------|
| Gate 1 | トークンしきい値を超過 | **必須** |
| Gate 2 | ツール呼び出ししきい値を超過 | 追加条件 |
| Gate 3 | 安全なタイミング（最終ターンにツール呼び出しがない） | タイミング制約 |

主エージェントが既にメモリ書き込み済みの場合、抽出はスキップされる。

### メモリの鮮度管理

1日以上古いメモリには自動的に警告が付与される:

```
"This memory is N days old. Verify against current code before asserting as fact."
```

---

## 3. Agents の暗黙知

### エージェント定義の構造

- ファイル: `~/.claude/agents/agent-name.md`
- frontmatter (YAML) = メタデータ
- Markdown 本文 = **そのままシステムプロンプトになる**

### frontmatter 全フィールドとデフォルト

| フィールド | デフォルト | 効果 |
|------------|------------|------|
| `name` | **必須** | agentType（完全一致で検索される） |
| `description` | **必須** | UI 表示・AI 選択基準 |
| `tools` | undefined (全許可) | `['*']` も全許可。具体名で制限可能 |
| `disallowedTools` | undefined | `tools` から差し引く |
| `model` | undefined | `'inherit'` で親継承。`'sonnet'`, `'opus'`, `'haiku'` 指定可 |
| `permissionMode` | undefined | `'plan'`, `'acceptEdits'`, `'bypassPermissions'` 等 |
| `effort` | undefined | `'quick'`, `'medium'`, `'thorough'`, `'max'` or 数値 |
| `maxTurns` | undefined | 正の整数。ターン数制限 |
| `memory` | undefined | `'user'`, `'project'`, `'local'`。メモリスコープ |
| `background` | undefined | `true` でバックグラウンド実行 |
| `isolation` | undefined | `'worktree'` で git worktree 分離 |
| `mcpServers` | undefined | 文字列参照 or インライン定義 |
| `hooks` | undefined | エージェント固有フック |
| `skills` | undefined | プリロードスキル名 |
| `initialPrompt` | undefined | 最初のターンに自動前置 |

### マッチングルール

- **完全一致のみ**（部分一致では検索されない）
- 同名時の優先度: `Managed > CLI args > Project > User > Plugin > Built-in`

### ツール制限の暗黙ルール

| ルール | 詳細 |
|--------|------|
| AgentTool 使用不可 | カスタムエージェントは再帰防止のため AgentTool を呼べない |
| memory 指定時の自動追加 | `Read` / `Write` / `Edit` が自動的にツールリストに追加される |
| MCP ツール | `mcp__*` は常に許可される |
| プラグインエージェント制限 | `permissionMode` / `hooks` / `mcpServers` は設定不可 |

### CLAUDE.md の継承

- デフォルトでは親の CLAUDE.md 階層が継承される
- `omitClaudeMd` は built-in エージェント限定（Explore / Plan で使用、トークン節約目的）

---

## 4. Skills の暗黙知

### スキルファイルの構造

```
~/.claude/skills/
├── skill-name/
│   └── SKILL.md              ← ディレクトリ形式のみ有効
├── group/
│   └── subgroup/
│       └── skill-name/
│           └── SKILL.md      ← スキル名: "group:subgroup:skill-name"
```

**重要:** 単一ファイル `~/.claude/skills/skill-name.md` は**不可**。必ずディレクトリ形式にする。

### frontmatter 全フィールド

| フィールド | デフォルト | 効果 |
|------------|------------|------|
| `name` | ディレクトリ名 | 表示用別名 |
| `description` | Markdown 最初の行 | AI 選択基準（**250文字制限**） |
| `allowed-tools` | `[]` | スキル実行中のみ一時許可されるツール |
| `when_to_use` | undefined | AI 自動選択の判定基準 |
| `disable-model-invocation` | `false` | `true` で AI が呼べなくなる |
| `user-invocable` | `true` | `false` でユーザーが `/` で呼べなくなる |
| `context` | `'inline'` | `'fork'` で独立サブエージェント実行 |
| `agent` | undefined | fork 時のエージェント型 |
| `model` | undefined | モデルオーバーライド |
| `effort` | undefined | 思考努力レベル |
| `paths` | undefined | glob 条件（マッチ時のみ有効化） |
| `shell` | `'bash'` | `'powershell'` 指定可 |

### AI がスキルを選ぶ仕組み

1. `description` + `when_to_use` が `" - "` で結合される
2. 結合後の文字列は **250文字制限** が適用される
3. SkillTool プロンプト予算: ウィンドウの **1%**（200K window で約 8,000文字）
4. bundled スキルは予算に関係なくフルテキスト表示される
5. **`when_to_use` が具体的なほど AI の選択精度が上がる**

### `!`command`` 構文

スキル内でシェルコマンドを埋め込み実行できる:

| 形式 | 構文 | 結果 |
|------|------|------|
| インライン | `` !`pwd` `` | コマンド結果に置換 |
| ブロック | ` ```! find . -name "*.ts" ``` ` | 複数行結果 |

注意事項:
- MCP スキルではセキュリティ上**実行されない**
- 権限チェックあり（Bash ツールの権限ルールが適用される）

### 環境変数

| 変数 | 値 |
|------|-----|
| `${CLAUDE_SKILL_DIR}` | スキルディレクトリのパス |
| `${CLAUDE_SESSION_ID}` | セッション ID |

### `context: 'fork'` を使う基準

- 大量トークン消費の可能性がある時
- 独立した思考ステップが必要な時
- メインと異なる effort / model で実行したい時

### `allowed-tools` のスコープ

- **スキル実行中のみ有効**。終了後は自動 revert される
- 内部的には `alwaysAllowRules` に一時追加される仕組み

---

## 5. Hooks の暗黙知

### 全27イベント（主要なものを抜粋）

| イベント | マッチ対象 | 用途 |
|----------|------------|------|
| `PreToolUse` | tool_name | ツール実行前。exit 2 でブロック |
| `PostToolUse` | tool_name | ツール実行後。結果を修正可能 |
| `UserPromptSubmit` | - | ユーザー入力フィルタ |
| `SessionStart` | source (startup/resume/clear) | セッション初期化 |
| `SessionEnd` | reason | クリーンアップ |
| `Stop` | - | Claude 応答終了前。exit 2 で追加指示 |
| `Notification` | notification_type | 通知連携 |
| `SubagentStart/Stop` | agent_type | サブエージェント監視 |

### 4種類の Hook タイプ

| タイプ | 説明 |
|--------|------|
| **command** | シェルスクリプト実行 |
| **prompt** | LLM に判定を委託（Haiku 等で軽量判定） |
| **agent** | サブエージェントで複雑な検証 |
| **http** | Webhook POST |

### Exit Code の意味

| コード | 意味 | 動作 |
|--------|------|------|
| `0` | 成功 | 続行 |
| `2` | **ブロッキングエラー** | ツール実行を阻止 |
| `1`, `3+` | 非ブロッキングエラー | 警告のみ、続行 |

### Hook 出力（JSON）でできること

```jsonc
{
  // 処理を停止
  "continue": false,

  // 権限判定
  "hookSpecificOutput": {
    "permissionDecision": "allow" | "deny" | "ask",

    // ツール入力を修正
    "updatedInput": { /* ... */ },

    // Claude に追加情報を渡す
    "additionalContext": "追加のコンテキスト文字列"
  }
}
```

### CLAUDE_ENV_FILE の活用

`SessionStart` hook で書き込むと、以降の全 hook とツールで環境変数が利用可能になる:

```bash
echo "export MY_VAR=value" >> "$CLAUDE_ENV_FILE"
```

### if 条件フィルタ

権限ルール構文と同じ形式で、発動条件をフィルタリングできる:

```jsonc
{
  "if": "Bash(git *)"     // git コマンドのみ発動
}
{
  "if": "Write(*.ts)"     // .ts ファイル書き込みのみ
}
```

### 優先度（設定ソース順）

| 順位 | ソース |
|------|--------|
| 1 | ローカル設定 (`.claude/settings.local.json`) |
| 2 | プロジェクト設定 (`.claude/settings.json`) |
| 3 | ユーザー設定 (`~/.claude/settings.json`) |
| 4 | プラグイン Hook |
| 5 | セッション Hook（一時） |

### タイムアウト

| 対象 | デフォルト | 備考 |
|------|------------|------|
| PreToolUse 等 | **10分** | 通常の Hook |
| SessionEnd | **1.5秒** | 高速終了必須 |
| 個別指定 | `"timeout": 10` | 秒単位で指定 |

---

## 6. 相互作用の暗黙知

### CLAUDE.md と Memory の関係

| CLAUDE.md | Memory |
|-----------|--------|
| 不変ルール | 可変メモ |
| 常時ロード | 関連性ベースで選択ロード |
| ここに書いた情報はメモリに**保存禁止** | CLAUDE.md にない情報のみ保存 |

### Agent と Skills の関係

- エージェント frontmatter の `skills:` でスキルをプリロードできる
- エージェント内から SkillTool で動的にスキルを呼び出すことも可能

### Hooks と Agent / Skills の関係

- エージェント frontmatter に `hooks:` を設定可能
- スキル frontmatter に `hooks:` を設定可能
- いずれもセッションスコープで有効

### Memory と Dream の関係

| 項目 | 詳細 |
|------|------|
| Dream System | `memory/*.md` を自動統合する仕組み |
| 発動ゲート | 24時間経過 + 5セッション蓄積 + Lock 取得の三段階 |
| KAIROS との排他 | KAIROS active 時は Dream 無効化される |

---

## 7. GPT-5.4 深層レビューによる修正・追加（2026-04-01）

GPT-5.4が16ファイルを実際に読み、行番号付きで指摘した修正と追加の暗黙知。

### 修正（私の誤り8件）

| 当初の主張 | 修正 | 根拠 |
|---|---|---|
| @includeはinclude先が優先 | **親→子順**で積まれる | claudemd.ts:663 |
| autoMemoryDirectoryはproject設定で上書き可能 | **projectSettings意図的除外**（セキュリティ） | paths.ts:172 |
| SessionEnd Hookは通常と同じタイムアウト | **既定1,500ms**（高速終了必須） | hooks.ts:175 |
| MCPスキルでも!command実行される | **loadedFrom !== 'mcp'で遮断** | loadSkillsDir.ts:374 |
| extractMemoriesは毎ターン必ず実行 | **直書き検出でスキップ+coalesce** | extractMemories.ts:348,557 |
| Hook trust制御は一部イベントのみ | **インタラクティブで全Hook依存** | hooks.ts:286 |
| policy設定変更もHookでブロック可能 | **blocked:falseに強制** | hooks.ts:4232 |
| システムプロンプト節は毎回再計算 | **節名キャッシュで再利用、cacheBreakのみ再計算** | systemPromptSections.ts:50 |

### 追加の暗黙知（GPT-5.4発見16件）

| # | 発見 | ファイル | 重要性 |
|---|---|---|---|
| 1 | @includeの深さ上限=5 | claudemd.ts | 循環参照・深い入れ子を防止 |
| 2 | MEMORY.mdはバイト数25KBでも切られる | memdir.ts:82 | 長文1行だと行数に余裕があっても失われる |
| 3 | autoMemoryDirectoryはprojectSettings除外 | paths.ts:172 | ~/.ssh誘導攻撃防止 |
| 4 | メモリスキャンはallSettledで部分失敗耐性 | memoryScan.ts:45 | 大量メモリでも全滅しない |
| 5 | 関連メモリ選択は前ターン除外後に5枠選別 | findRelevantMemories.ts:35 | 長セッションで新規発見促進 |
| 6 | 抽出エージェントは検証禁止・2ターン並列最適化 | prompts.ts:39 | grep/コード確認を禁止 |
| 7 | スキル安全プロパティはallowlist方式 | SkillTool.ts:525 | 新規プロパティは既定で不許可 |
| 8 | bundledスキルは予算不足時も説明保持 | prompt.ts:93 | ユーザースキルは名前のみになりうる |
| 9 | 全Hookがインタラクティブ時Trust依存 | hooks.ts:286 | Trust未承認で全Hookスキップ |
| 10 | asyncRewakeがスキーマで正式サポート | hooks.ts:55 | exit 2でモデル再起動可能 |
| 11 | policy設定変更はHookでブロック不可 | hooks.ts:4232 | blocked:falseに強制 |
| 12 | CCRでlocalスコープもプロジェクト名前空間付き | agentMemory.ts:30 | 端末ローカルと思うとデータ境界誤り |
| 13 | gitignoreされたディレクトリのスキルはスキップ | loadSkillsDir.ts | node_modules内偽スキル混入防止 |
| 14 | InstructionsLoaded Hookがload_reason単位で発火 | hooks設定 | compact/include/session_start別追跡 |
| 15 | SYSTEM_PROMPT_DYNAMIC_BOUNDARYを跨ぐとキャッシュ急落 | prompts.ts:106 | 性能とコスト |
| 16 | Skill予算は2%にスケール済み（CHANGELOG v2.1.32） | SkillTool | 1%は古い情報 |

### FDE実用Tips（GPT-5.4提案8件）

| # | 問題 | 暗黙知 | 具体的実装 |
|---|---|---|---|
| 1 | MEMORY.md肥大化 | 200行+25KB二重上限 | 各行150文字。詳細は個別ファイルへ |
| 2 | メモリ保存先の乗っ取り | projectSettings除外 | ~/.claude/settings.jsonでautoMemoryDirectory指定 |
| 3 | 25スキルの選択ノイズ | paths:で条件付き休眠 | OCRスキルにpaths: ["**/vision_ocr.py"]追加 |
| 4 | 偽スキル混入 | gitignoreスキップ | .gitignoreにnode_modules/vendor/明記 |
| 5 | エージェント間メモリリーク | memory分離 | 調査=user、実装=project、デバッグ=local |
| 6 | Hook全発火で重い | if条件フィルタ | "if": "Write(*.gs)|Edit(*.gs)"でGASのみ |
| 7 | 指示ファイル追跡 | InstructionsLoaded Hook | JSONL監査ログ出力 |
| 8 | MCPスキルで!command不動 | loadedFrom:mcp遮断 | 実行系=ローカル、宣言系=MCPに分離 |
