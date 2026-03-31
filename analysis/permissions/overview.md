# Claude Code 権限モデル解析

## 1. 概要

Claude Code のセキュリティは **5層の階層構造** で実装されている。
各層は独立して機能し、上位層で許可されても下位層で拒否される場合がある。

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: Permission Rules (allow / ask / deny)     │
│  Layer 2: Mode Gating (plan / acceptEdits / bypass) │
│  Layer 3: Tool Validation (Bash解析 / パス検証)      │
│  Layer 4: Classifier (Auto Mode LLM再分類)           │
│  Layer 5: Sandbox Enforcement (FS / Network / 隔離)  │
└─────────────────────────────────────────────────────┘
```

リクエストは Layer 1 から順に評価され、いずれかの層で拒否されれば実行されない。

---

## 2. 権限モード一覧

| モード | 説明 |
|---|---|
| `default` | 全ツール使用時にユーザー確認を要求 |
| `acceptEdits` | CWD内のファイル編集を自動承認（Bash等は引き続き確認） |
| `plan` | Plan Mode。読み取り専用操作のみ許可。書き込み・実行を一切拒否 |
| `bypassPermissions` | 権限チェックをバイパス（全ツール自動許可） |
| `dontAsk` | 確認プロンプトを抑止 |
| `auto` | Auto Mode。Anthropic内部のみ使用可能（`TRANSCRIPT_CLASSIFIER` 機能ゲート必須） |
| `bubble` | 内部使用。外部ユーザーには非表示 |

### モード切替サイクル

```
default → acceptEdits → plan → bypass/auto → default
                    （Shift+Tab で順次切替）
```

---

## 3. ルール評価順序

### 3段階の評価ロジック

```
1. deny  ルール（最優先）→ 即座に拒否。以降のルールは評価しない
2. ask   ルール          → ユーザーに確認プロンプトを表示
3. allow ルール          → 自動許可（確認なし）
```

該当ルールなし → モードのデフォルト動作に従う。

### ルールソース優先順位（上位が優先）

| 優先度 | ソース | 設定方法 |
|:---:|---|---|
| 1 | `flagSettings` | CLI引数 `--allow`, `--deny` |
| 2 | `policySettings` | 企業管理ポリシー（MDM等） |
| 3 | `localSettings` | `.claude/settings.local.json`（gitignore推奨） |
| 4 | `projectSettings` | `.claude/settings.json`（リポジトリ共有） |
| 5 | `userSettings` | `~/.claude/settings.json`（ユーザーグローバル） |
| 6 | `session` | セッション中の動的変更（"Always allow" 選択時等） |

上位ソースの `deny` は下位ソースの `allow` を常にオーバーライドする。

---

## 4. BashTool セキュリティ

### ファイル構成

| ファイル | 行数 | 責務 |
|---|---:|---|
| `bashPermissions.ts` | 2,621 | コマンド許可ルール照合。prefix / exact / wildcard パターンマッチング |
| `bashSecurity.ts` | 2,592 | 危険パターン検出（コマンド置換、IFS注入、Zsh危険命令） |
| `readOnlyValidation.ts` | 1,990 | 読み取り専用コマンド許可リスト管理 |
| `pathValidation.ts` | 1,303 | パストラバーサル防止、UNCパス検出、シンボリックリンク解決 |
| `shouldUseSandbox.ts` | 153 | サンドボックス適用判定ロジック |

### 危険パターン検出ID

| パターンID | 検出対象 | 例 |
|---|---|---|
| `COMMAND_SUBSTITUTION` | コマンド置換構文 | `$()`, `${}`, `<()` |
| `DANGEROUS_VARIABLES` | 危険な環境変数 | `LD_PRELOAD`, `DYLD_INSERT_LIBRARIES` |
| `IFS_INJECTION` | IFS変数悪用 | `$IFS` を使ったコマンド分割 |
| `ZSH_DANGEROUS_COMMANDS` | Zsh固有の危険コマンド | `zmodload`, `emulate`, `sysopen`, `ztcp` |
| `COMMENT_QUOTE_DESYNC` | コメントによる引用ずれ | `#` でクォートバランスを崩す攻撃 |

```
MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50  // ReDoS対策: 解析するサブコマンド数の上限
```

---

## 5. リスク分類

### LOW / MEDIUM / HIGH

| リスクレベル | 定義 | 例 |
|---|---|---|
| **LOW** | 安全な開発フロー。副作用なし | `git status`, `ls`, `cat`, `grep` |
| **MEDIUM** | 回復可能な変更。ファイルシステムへの書き込みを伴う | ファイル作成・削除、`git commit`, `npm install` |
| **HIGH** | 危険で不可逆的。システムやデータに重大な影響 | システムファイル削除、秘密情報漏洩、`rm -rf /` |

### Auto Mode セーフリスト

Auto Mode で確認なしに自動許可されるツール:

| カテゴリ | ツール |
|---|---|
| 読み取り専用 | `FileRead`, `Grep`, `Glob` |
| タスク管理 | `TodoWrite`, `TaskUpdate` |
| UI操作 | `AskUserQuestion` |
| チーム管理 | `SendMessage`, `TeamCreate` |

上記以外のツール（特に `Bash`, `FileWrite`）は Auto Mode でも Classifier 経由で評価される。

---

## 6. 保護ファイルリスト

### DANGEROUS_FILES

以下のファイルへの書き込みは特別な保護対象:

```
.gitconfig
.gitmodules
.bashrc
.bash_profile
.zshrc
.zprofile
.profile
.ripgreprc
.mcp.json
.claude.json
```

### DANGEROUS_DIRECTORIES

以下のディレクトリ配下は書き込み制限:

```
.git/
.vscode/
.idea/
.claude/
```

### 保護メカニズム

- **シンボリックリンク解決**: 書き込み先パスを `realpath` で解決し、保護対象と照合
- **ケースインセンシティブ照合**: Windows環境では大文字小文字を区別せず保護
- CWD外への書き込みは `acceptEdits` モードでも確認を要求

---

## 7. サンドボックス実装

### SandboxRuntimeConfig

```typescript
interface SandboxRuntimeConfig {
  filesystem: {
    allowWrite: string[];   // 書き込み許可パス
    denyWrite: string[];    // 書き込み拒否パス
    denyRead: string[];     // 読み取り拒否パス
    allowRead: string[];    // 読み取り許可パス
  };
  network: {
    allowedDomains: string[];    // 通信許可ドメイン
    allowUnixSockets: boolean;   // UNIXソケット許可
    allowLocalBinding: boolean;  // ローカルバインド許可
  };
  ignoreViolations: boolean;     // 違反を無視（ログのみ）
}
```

### サンドボックス判定フロー

```
              isSandboxingEnabled()
                      │
               ┌──────┴──────┐
               │ false        │ true
               ▼              ▼
          サンドボックス   dangerouslyDisableSandbox?
          無効（素通し）        │
                        ┌──────┴──────┐
                        │ true        │ false
                        ▼             ▼
                   バイパス    containsExcludedCommand?
                                     │
                               ┌─────┴─────┐
                               │ true       │ false
                               ▼            ▼
                           除外         サンドボックス
                          （素通し）       適用
```

### autoAllowBashIfSandboxed

```
autoAllowBashIfSandboxed: true
```

サンドボックスが有効な場合、Bash コマンドは全て自動許可される（サンドボックスが安全境界を担保するため）。

---

## 8. Auto Mode（YOLO Classifier）

### アーキテクチャ

| 項目 | 値 |
|---|---|
| モデル | Claude Haiku（軽量・高速） |
| 入力 | 実行予定コマンド + 現在の許可ルール |
| 出力 | 構造化スコア: `{ matches, confidence, reason }` |
| 目的 | 許可ルールに基づきコマンドの安全性をLLMで再分類 |

### Denial Tracking（拒否追跡）

```
連続拒否: 3回 → ask モードにフォールバック
累積拒否: 20回 → ask モードにフォールバック
```

フォールバック後はユーザーが明示的に許可するまで Classifier をスキップする。

### 危険Bashパターン自動削除

Auto Mode では以下のコマンドプレフィックスを含むBash実行は Classifier を通さず即座に `ask` 扱い:

```
python, node, bash, ssh, eval, sudo,
curl, wget, nc, telnet, ncat
```

---

## 9. セキュリティ階層図

```
┌───────────────────────────────────────────────────────────────────┐
│                    ユーザーリクエスト                                │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 1: Permission Rules                                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                          │
│  │  deny   │→ │  ask    │→ │  allow  │                           │
│  │ (即拒否) │  │ (確認)  │  │ (自動)  │                           │
│  └─────────┘  └─────────┘  └─────────┘                          │
│  Sources: flag > policy > local > project > user > session       │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 2: Mode Gating                                             │
│  ┌──────────┐ ┌────────────┐ ┌──────┐ ┌────────┐                │
│  │ default  │ │acceptEdits │ │ plan │ │ bypass │                 │
│  │ (全確認) │ │(編集自動)  │ │(R/O) │ │(全許可)│                 │
│  └──────────┘ └────────────┘ └──────┘ └────────┘                │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 3: Tool Validation                                         │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────────┐            │
│  │ Bash解析      │ │ パス検証      │ │ 保護ファイル   │            │
│  │ (コマンド分解)│ │ (トラバーサル)│ │ (書込禁止)    │            │
│  └──────────────┘ └──────────────┘ └───────────────┘            │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 4: Classifier (Auto Mode のみ)                             │
│  ┌──────────────────────────────────────┐                        │
│  │ Claude Haiku によるLLM再分類          │                        │
│  │ Input: command + rules               │                        │
│  │ Output: { matches, confidence }      │                        │
│  │ Fallback: 3連続 or 20累積拒否 → ask  │                        │
│  └──────────────────────────────────────┘                        │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│  Layer 5: Sandbox Enforcement                                     │
│  ┌──────────────┐ ┌──────────────┐ ┌───────────────┐            │
│  │ ファイル      │ │ ネットワーク  │ │ 環境隔離       │            │
│  │ システム制限  │ │ ドメイン制限  │ │ (プロセス分離) │            │
│  └──────────────┘ └──────────────┘ └───────────────┘            │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
                    ┌───────────────┐
                    │   実行許可     │
                    └───────────────┘
```

---

## 補足: 設計原則

1. **Defense in Depth**: 単一の防御層に依存しない。5層すべてを通過しなければ実行されない
2. **Fail-Closed**: 判定不能な場合は拒否をデフォルトとする
3. **Least Privilege**: 必要最小限の権限のみ付与。Auto Mode でもセーフリスト外は評価が必要
4. **Auditability**: 各層の判定結果はログに記録され、後から追跡可能
