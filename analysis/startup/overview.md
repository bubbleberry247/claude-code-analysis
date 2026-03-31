# Claude Code 起動シーケンス解析

## 1. 起動フロー全体像

```
entrypoints/cli.tsx
  │
  ├─ 高速パス分岐（--version, --daemon-worker, bridge等）
  │
  └─ main.tsx（モジュール評価時に並列プリフェッチ開始）
       │
       ├─ preAction フック（MDM + Keychain 待機 → init() → sinks）
       │
       └─ action handler
            │
            ├─ setup()  ← 15フェーズの初期化
            │
            └─ launchRepl()
                 ├─ Trust dialog
                 ├─ テレメトリ初期化
                 ├─ SessionStart hooks
                 ├─ AppState + Store 作成
                 └─ REPL UI render（React INK）
```

---

## 2. 高速パス（cli.tsx）

CLI エントリポイントでは、重いモジュールをロードする前に軽量な分岐を行う。

| パス | 動作 | ロードコスト |
|---|---|---|
| `--version` | バージョン文字列を出力して即終了 | ゼロモジュールロード |
| `--daemon-worker` | `dynamic import("./daemon-worker")` | daemon 専用モジュールのみ |
| `--claude-in-chrome-mcp` | `dynamic import("./chrome-mcp")` | MCP 専用モジュールのみ |
| `bridge` サブコマンド | `dynamic import("./bridge")` | bridge 専用モジュールのみ |
| それ以外 | `main.tsx` をフルロード | 全モジュール |

**`feature()` 関数**: ビルドタイム DCE（Dead Code Elimination）で機能する。Bun bundler がコンパイル時に `feature('FLAG_NAME')` の結果を静的評価し、不要なコードパスを削除する。実行時コストはゼロ。

---

## 3. 並列プリフェッチ（main.tsx モジュール評価時）

`main.tsx` がインポートされた時点（モジュール評価時）で、メインの import チェーンと並行して以下が起動する:

```typescript
// モジュールトップレベルで即座に発火
startMdmRawRead()        // [1] MDM subprocess spawn（macOS/Windows 設定読み取り）
startKeychainPrefetch()  // [2] macOS keychain async prefetch

// ↑ これらは約135msかかるメインimportsと並行して実行される
// メインimportsが完了する頃にはプリフェッチも完了（または完了間近）
```

**設計意図**: MDM 設定読み取りと Keychain アクセスは I/O バウンドな操作。メインモジュールの import（CPU バウンド: パース・評価）と並列化することで、クリティカルパスから外す。

---

## 4. preAction フック

Commander.js の `.preAction()` で、コマンドハンドラ実行前に必須の初期化を完了させる。

```typescript
// ── ブロッキング（並列待機） ──
await Promise.all([
  ensureMdmSettingsLoaded(),        // ~15ms（§3 で spawn 済みの結果を回収）
  ensureKeychainPrefetchCompleted() // ~65ms on macOS（§3 で開始済みの結果を回収）
])

// ── 逐次初期化 ──
await init()          // §5 参照（memoize 済み、2回目以降は即座に返る）
initSinks()           // テレメトリシンクの初期化
runMigrations()       // データマイグレーション（設定ファイル形式の変更等）

// ── ノンブロッキング（fire-and-forget） ──
void loadRemoteManagedSettings()   // ~200ms、リモート設定の非同期取得
void loadPolicyLimits()            // ポリシー制限の非同期取得
```

---

## 5. init() 関数（memoize 済み、1回限り実行）

`init()` は `memoize` でラップされており、何度呼ばれても実際の初期化は1回だけ実行される。

| Phase | 処理 | 説明 |
|---|---|---|
| 1 | `enableConfigs()` | Feature flag / config システムの有効化 |
| 2 | `applySafeConfigEnvironmentVariables()` + `applyExtraCACertsFromConfig()` | 安全な環境変数の適用 + カスタム CA 証明書の設定 |
| 3 | `setupGracefulShutdown()` | SIGINT/SIGTERM ハンドラ登録 |
| 4 | Background prefetch（`Promise.all`） | Analytics SDK, GrowthBook, OAuth token, JetBrains 検出を並列起動 |
| 5 | `configureGlobalMTLS()` + `configureGlobalAgents()` | mTLS 設定 + HTTP/HTTPS グローバルエージェント設定 |
| 6 | `preconnectAnthropicApi()` | Anthropic API への TCP プリコネクト。**Phase 5 の後**でないと CA certs / proxy agents が未適用 |
| 7 | CCR upstream proxy | `CLAUDE_CODE_REMOTE` 環境のみ。リモート実行用の upstream proxy 設定 |
| 8 | Cleanup registry | 終了時クリーンアップ関数の登録 |
| 9 | Scratchpad 保証 | 一時作業ディレクトリの存在確認・作成 |

**Phase 4 → 6 の順序が重要**: API プリコネクトは CA 証明書とプロキシエージェントが設定された後に行う必要がある。逆順だと TLS エラーや接続失敗の原因になる。

---

## 6. setup() 関数

REPL 起動前の環境構築。15 フェーズで構成される。

| Phase | 処理 | 説明 |
|---|---|---|
| 1 | Node.js version check | 18+ 必須。未満ならエラー終了 |
| 2 | カスタムセッション ID | `--session-id` switch の処理 |
| 3 | UDS messaging server start | Unix Domain Socket サーバー起動（プロセス間通信） |
| 4 | Teammate snapshot capture | チームメイト状態のスナップショット取得 |
| 5 | Terminal backup restoration | ターミナル状態のバックアップ復元 |
| 6 | `setCwd()` | カレントディレクトリ設定。**hook config snapshot 前に必須**（hook はパス解決に cwd を使う） |
| 7 | Hook configuration snapshot + file watcher | Hook 設定のスナップショット取得 + ファイル変更監視の開始 |
| 8 | Worktree creation | Git worktree の作成（必要な場合） |
| 9 | Background jobs | Skills / plugins キャッシュのウォームアップ（非同期） |
| 10 | Plugin sync & hook hot reload | プラグイン同期 + hook のホットリロード設定 |
| 11 | Attribution hooks | リポジトリ分類（`feature('COMMIT_ATTRIBUTION')` 有効時のみ） |
| 12 | テレメトリシンク | テレメトリデータの送信先設定 |
| 13 | API key helper prefetch | API キーヘルパーの事前取得 |
| 14 | リリースノート準備 | 新バージョンのリリースノート表示準備 |
| 15 | 権限モード safety check | 権限モードの安全性チェック |

**Phase 6 → 7 の依存**: Hook の設定スナップショットはファイルパスを解決する必要があるため、`setCwd()` が先に完了している必要がある。

---

## 7. REPL 起動フロー

`launchRepl()` は以下の順序で REPL を立ち上げる:

```
1. setup()
   │  15フェーズの初期化（§6）
   │
2. Trust dialog 表示
   │  ユーザーがプロジェクトを信頼するか確認
   │  （初回 or 設定変更時のみ表示）
   │
3. テレメトリ初期化
   │  Trust 承認後に開始（拒否時はテレメトリ無効）
   │
4. SessionStart hooks 実行
   │  .claude/settings.json の hooks.SessionStart を実行
   │
5. AppState + Store 作成
   │  アプリケーション状態とデータストアの初期化
   │
6. REPL UI render（React INK）
      ターミナル UI の描画開始
      ユーザー入力受付開始
```

---

## 8. 遅延インポート一覧

実行時にすべてのモジュールをロードするのではなく、条件付きで遅延ロードされるモジュール:

| モジュール | 条件 | タイミング | 理由 |
|---|---|---|---|
| `assistant/index.js` | `feature('KAIROS')` | モジュール評価時 | ビルドタイム DCE。フラグ無効時はバンドルから除外 |
| `coordinator/coordinatorMode.js` | `feature('COORDINATOR_MODE')` | モジュール評価時 | ビルドタイム DCE。フラグ無効時はバンドルから除外 |
| `services/analytics/` | Always（常時） | `init()` の `Promise.all` 内 | ~400KB の OpenTelemetry を起動クリティカルパスから除外 |
| `upstreamproxy/` | `CLAUDE_CODE_REMOTE` | `init()` の `await` | CCR（Claude Code Remote）環境でのみ必要 |
| `utils/attributionHooks.js` | `feature('COMMIT_ATTRIBUTION')` | `setup()` の `setImmediate` | bare repo では不要。非同期で遅延実行 |

---

## 9. パフォーマンス計測ポイント（profileCheckpoint）

コード中に埋め込まれた `profileCheckpoint()` により、起動パフォーマンスを計測できる。

### 典型タイムライン

```
T+0ms      main_tsx_entry
           │  モジュール評価開始
           │  startMdmRawRead() + startKeychainPrefetch() 発火
           │
T+135ms    imports 完了
           │  preAction フック開始
           │  MDM/Keychain の Promise.all 待機
           │
T+160ms    preAction_after_mdm
           │  init() 開始
           │
T+180ms    preAction_after_init
           │  initSinks() + runMigrations()
           │
T+200ms    action_handler_start
           │  setup() 開始（15フェーズ）
           │  ... Trust dialog, hooks, UI render ...
           │
T+1500ms+  launchRepl 完了
           │  ユーザー入力受付開始
```

### 各区間の内訳

| 区間 | 所要時間（目安） | ボトルネック |
|---|---|---|
| モジュール評価 | ~135ms | JS パース + 評価。並列プリフェッチで隠蔽 |
| MDM 待機 | ~15ms | subprocess 完了待ち（プリフェッチ済み） |
| Keychain 待機 | ~65ms（macOS） | セキュリティフレームワーク呼び出し |
| init() | ~20ms | 大半は非同期で fire-and-forget |
| setup() + REPL | ~1300ms | Trust dialog, hook 実行, React INK 描画 |

---

## 10. 初期化の依存関係 DAG

```
                    ┌─────────────┐
                    │  cli.tsx     │
                    │ (entrypoint) │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │--version │ │--daemon  │ │ main.tsx  │
        │(即終了)  │ │(動的imp) │ │(フルロード)│
        └──────────┘ └──────────┘ └─────┬─────┘
                                        │
                          ┌─────────────┼─────────────┐
                          │ (並列)      │ (並列)      │
                          ▼             ▼             │
                   ┌────────────┐ ┌────────────┐     │
                   │ MDM Read   │ │ Keychain   │     │
                   │ (spawn)    │ │ (prefetch) │     │
                   └─────┬──────┘ └─────┬──────┘     │
                         │              │             │
                         └──────┬───────┘             │
                                │                     │
                                ▼                     ▼
                      ┌──────────────────┐   ┌──────────────┐
                      │  Promise.all     │   │ imports完了   │
                      │  (MDM+Keychain)  │   │ (~135ms)     │
                      └────────┬─────────┘   └──────┬───────┘
                               │                     │
                               └──────────┬──────────┘
                                          │
                                          ▼
                               ┌─────────────────┐
                               │   preAction      │
                               └────────┬────────┘
                                        │
                                        ▼
                          ┌──────────────────────────┐
                          │        init()            │
                          │  (memoize: 1回限り)      │
                          └────────────┬─────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │ enableConfigs│  │ CA certs +   │  │ graceful     │
           │              │  │ env vars     │  │ shutdown     │
           └──────┬───────┘  └──────┬───────┘  └──────────────┘
                  │                 │
                  │                 ▼
                  │        ┌──────────────────┐
                  │        │ mTLS + global    │
                  │        │ agents (Phase 5) │
                  │        └────────┬─────────┘
                  │                 │
                  │                 ▼  ← Phase 5 完了が前提条件
                  │        ┌──────────────────┐
                  │        │ preconnect API   │
                  │        │ (Phase 6)        │
                  │        └─────────────────┘
                  │
                  ▼
           ┌──────────────────┐
           │ Background       │
           │ prefetch (Ph.4)  │
           │ ・Analytics      │
           │ ・GrowthBook     │
           │ ・OAuth          │
           │ ・JetBrains検出  │
           └──────────────────┘
                                        │
                               ┌────────┘
                               ▼
                    ┌─────────────────────┐
                    │    initSinks()      │
                    │    runMigrations()  │
                    └────────┬────────────┘
                             │
                    ┌────────┘     ┌──────────────────────┐
                    ▼              │ void (non-blocking)  │
             ┌────────────┐       │ ・remoteManagedSettings│
             │  setup()   │       │ ・policyLimits        │
             │ (15 phases)│       └──────────────────────┘
             └─────┬──────┘
                   │
        ┌──────────┼──────────────────────┐
        │          │                      │
        ▼          ▼                      ▼
  ┌──────────┐ ┌──────────────┐   ┌────────────┐
  │ setCwd() │ │ Node version │   │ UDS server │
  │ (Ph.6)   │ │ check (Ph.1) │   │ (Ph.3)     │
  └────┬─────┘ └──────────────┘   └────────────┘
       │
       ▼  ← setCwd() 完了が前提条件
  ┌──────────────────┐
  │ Hook snapshot +  │
  │ file watcher     │
  │ (Ph.7)           │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 残りのPhases     │
  │ (8-15)           │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │  launchRepl()    │
  │                  │
  │  Trust dialog    │
  │       ↓          │
  │  Telemetry init  │
  │       ↓          │
  │  SessionStart    │
  │  hooks           │
  │       ↓          │
  │  AppState+Store  │
  │       ↓          │
  │  React INK       │
  │  render          │
  └──────────────────┘
```

### 重要な依存関係（まとめ）

| 上流 | 下流 | 理由 |
|---|---|---|
| MDM spawn + Keychain prefetch | `Promise.all` 待機 | I/O 完了を保証 |
| `enableConfigs()` | 他の全 Phase | Feature flag が未確定だと分岐不可 |
| CA certs + mTLS + global agents | `preconnectAnthropicApi()` | TLS / proxy 設定なしに接続すると失敗 |
| `setCwd()` | Hook snapshot | Hook のパス解決に cwd が必要 |
| Trust dialog | テレメトリ初期化 | ユーザー同意なしにテレメトリ送信不可 |
| `init()` | `setup()` | グローバル状態（config, shutdown handler, API接続）が前提 |
