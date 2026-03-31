# Claude Code UI層 解析ドキュメント

## 1. 概要

| 指標 | 値 |
|---|---|
| 総ファイル数 | 611 |
| 総行数 | 121,782 |
| フレームワーク | React + Ink（カスタムターミナルUIフレームワーク） |

### ディレクトリ構成

| ディレクトリ | ファイル数 | 役割 |
|---|---|---|
| `ink/` | 96 | Ink TUI本体（レンダラ、レイアウトエンジン、入出力制御） |
| `components/` | 389 | Reactコンポーネント（UI部品） |
| `hooks/` | 104 | カスタムフック（状態・入力・通知・リモート等） |
| `state/` | 6 | グローバル状態管理（カスタムStore実装） |
| `keybindings/` | 14 | キーバインド定義・解決パイプライン |
| `screens/` | 3 | 画面レベルコンポーネント |
| `vim/` | 5 | Vimモード状態機械 |

---

## 2. Ink TUI アーキテクチャ

### レンダリングパイプライン

```
React Components
       |
       v
React Reconciler (custom)
       |
       v
DOM Tree (virtual)
       |
       v
Yoga Layout Engine (flexbox)
       |
       v
Screen Buffer (cell grid)
       |
       v
Terminal Output (ANSI escape sequences)
```

### エントリポイント: `ink.tsx`

- メイン入出力制御を担当
- 60FPS レンダリングループ
- alt-screen（代替画面バッファ）対応
- stdin/stdout のラップと制御

### Ink TUI コンポーネント（17個）

| コンポーネント | 役割 |
|---|---|
| `Box` | Flexboxコンテナ（Yogaレイアウト） |
| `Text` | テキスト表示（ANSI スタイル付き） |
| `Button` | インタラクティブボタン |
| `ScrollBox` | スクロール可能コンテナ |
| `Link` | ハイパーリンク（OSC 8） |
| `AlternateScreen` | 代替画面バッファ切替 |
| `Static` | 静的コンテンツ（再レンダリング抑制） |
| `Spacer` | フレキシブルスペーサー |
| `Transform` | テキスト変換ラッパー |
| `Newline` | 改行挿入 |
| その他7個 | 内部レイアウト・描画用ユーティリティ |

---

## 3. React コンポーネントカテゴリ

### メッセージ表示（30+）

ユーザー・アシスタント・システムの各種メッセージを表示する。

| コンポーネント | 説明 |
|---|---|
| `AssistantTextMessage` | アシスタントのテキスト応答 |
| `UserPromptMessage` | ユーザー入力プロンプト |
| `SystemAPIErrorMessage` | APIエラー表示 |
| `ToolResultMessage` | ツール実行結果 |
| `AssistantToolUseMessage` | ツール使用表示 |
| `VirtualMessageList` | メッセージ一覧（仮想化スクロール） |
| その他24+ | コンテキスト表示、添付ファイル、差分表示等 |

### 入力・プロンプト（21）

| コンポーネント | 説明 |
|---|---|
| `PromptInput` | メインプロンプト入力 |
| `ShimmeredInput` | シマー付き入力フィールド |
| `HistorySearchInput` | 履歴検索入力（Ctrl+R） |
| `AutocompleteInput` | オートコンプリート付き入力 |
| `MultilineEditor` | 複数行エディタ |
| その他16 | ファイル選択、コマンドパレット等 |

### パーミッションダイアログ（12+）

| コンポーネント | 説明 |
|---|---|
| `BashPermissionRequest` | Bash実行許可 |
| `FileEditPermissionRequest` | ファイル編集許可 |
| `FileWritePermissionRequest` | ファイル書き込み許可 |
| `McpToolPermissionRequest` | MCPツール実行許可 |
| `WebFetchPermissionRequest` | Web取得許可 |
| その他7+ | 各ツール固有の許可ダイアログ |

### タスク（11）

| コンポーネント | 説明 |
|---|---|
| `BackgroundTask` | バックグラウンドタスク表示 |
| `RemoteSessionProgress` | リモートセッション進捗 |
| `TaskProgressBar` | タスクプログレスバー |
| `SpinnerWithLabel` | ラベル付きスピナー |
| その他7 | タスクキュー、ステータス等 |

### チーム（8）

| コンポーネント | 説明 |
|---|---|
| `TeamMemberList` | チームメンバー一覧 |
| `CoordinatorTaskPanel` | コーディネータータスクパネル |
| `SwarmStatusBar` | Swarmステータスバー |
| その他5 | チーム操作、メッセージ、進捗 |

---

## 4. カスタムフック分類（104個）

### 状態管理

| フック | 説明 |
|---|---|
| `useAppState` | AppStateからの選択的読み取り |
| `useSetAppState` | AppStateの更新 |
| `useMessages` | メッセージ配列の管理 |
| `useStore` | Storeインスタンスへのアクセス |

### 入力

| フック | 説明 |
|---|---|
| `useArrowKeyHistory` | 矢印キーによる入力履歴ナビゲーション |
| `useSearchInput` | 検索入力の状態管理 |
| `useTextInput` | テキスト入力のカーソル・変換制御 |
| `useVimMode` | Vimモード状態の管理 |

### 通知（17個）

| フック | 説明 |
|---|---|
| `useStartupNotification` | 起動時通知 |
| `useRateLimitWarningNotification` | レートリミット警告 |
| `useDeprecationNotification` | 非推奨通知 |
| `useUpdateAvailableNotification` | アップデート通知 |
| `usePermissionModeNotification` | パーミッションモード通知 |
| その他12 | セッション、エラー、ヒント等 |

### リモート

| フック | 説明 |
|---|---|
| `useReplBridge` | REPLブリッジ通信 |
| `useRemoteSession` | リモートセッション管理 |
| `useDirectConnect` | ダイレクト接続制御 |
| `useWebSocket` | WebSocket通信 |

### パーミッション

| フック | 説明 |
|---|---|
| `useCanUseTool` | ツール使用可否判定 |
| `useSwarmPermissionPoller` | Swarmパーミッションポーリング |
| `usePermissionRequestHandler` | パーミッションリクエスト処理 |

### キーバインド

| フック | 説明 |
|---|---|
| `useGlobalKeybindings` | グローバルキーバインド登録 |
| `useKeybinding` | 個別キーバインド登録 |
| `useKeybindingContext` | キーバインドコンテキスト取得 |

### UI

| フック | 説明 |
|---|---|
| `useTerminalSize` | ターミナルサイズ監視 |
| `useFpsMetrics` | FPSメトリクス収集 |
| `useMoreRight` | 右方向スクロール制御 |
| `useScrollPosition` | スクロール位置管理 |

---

## 5. 状態管理

### カスタムStore実装

Zustandなどの外部ライブラリは不使用。約35行の軽量Store実装。

```typescript
// 概念的な構造（簡略化）
class Store<T> {
  private state: T;
  private listeners: Set<() => void>;

  getState(): T;
  setState(partial: Partial<T>): void;
  subscribe(listener: () => void): () => void;
}
```

- `useSyncExternalStore` で React と統合
- 選択的サブスクリプション（セレクタパターン）で不要な再レンダリングを防止

### AppState 主要フィールド

| フィールド | 型概要 | 説明 |
|---|---|---|
| `settings` | `Settings` | ユーザー設定 |
| `messages` | `Message[]` | 会話メッセージ履歴 |
| `tasks` | `Task[]` | 実行中タスク |
| `mcp` | `McpState` | MCPサーバー接続状態 |
| `plugins` | `PluginState` | プラグイン状態 |
| `teamContext` | `TeamContext` | チーム/Swarm状態 |
| `notifications` | `Notification[]` | 通知キュー |
| `toolPermissionContext` | `PermissionContext` | ツール許可の状態・キャッシュ |
| `speculation` | `SpeculationState` | 投機的実行の状態 |

---

## 6. キーバインド

### パイプライン

```
ユーザーキー入力
       |
       v
parseKeypress()          ← 生キーイベントを構造化
       |
       v
KeybindingContext         ← 現在のモード・フォーカス状態
       |
       v
resolver()               ← バインド定義テーブルを検索
       |
       v
match                    ← 一致するバインドを特定
       |
       v
useKeybinding callback   ← 登録済みアクションを実行
       |
       v
Action                   ← 状態変更・コマンド実行
```

### デフォルトバインド（`defaultBindings.ts`）

| キー | アクション |
|---|---|
| `Ctrl+C` | 中断 / キャンセル |
| `Ctrl+T` | テーマ切替 |
| `Ctrl+R` | 履歴検索 |
| `Escape` | モード切替 / キャンセル |
| `g j` (コード) | Vim風ナビゲーション |

### カスタマイズ

`~/.claude/keybindings.json` でユーザー定義のキーバインドを設定可能。
デフォルトバインドを上書き、または新規バインドを追加できる。

---

## 7. Vim モード

5ファイルで構成される状態機械。

### モード

| モード | 説明 |
|---|---|
| `INSERT` | 通常のテキスト入力（デフォルト） |
| `NORMAL` | Vim風コマンドモード |

### CommandState（コマンド解析状態）

```
idle
  |
  +---> count          (数値プレフィックス: 3dw)
  |       |
  +---> operator       (オペレータ待ち: d, c, y)
  |       |
  |       +---> operatorFind     (f/t + 文字: df.)
  |       |
  |       +---> operatorTextObj  (テキストオブジェクト: diw, ci")
  |
  +---> find           (f/F/t/T + 文字)
  |
  +---> g              (g プレフィックスコマンド)
  |
  +---> replace        (r + 文字)
  |
  +---> indent         (>>/<<)
```

### PersistentState（永続状態）

| フィールド | 説明 |
|---|---|
| `lastChange` | 最後の変更操作（`.` ドットリピート用） |
| `lastFind` | 最後の検索（`;` / `,` リピート用） |
| `register` | ヤンク/デリートレジスタ |

---

## 8. パフォーマンス最適化

### フレームバッファキャッシュ

| 手法 | 説明 |
|---|---|
| セルプール | 描画セルオブジェクトを再利用、GC負荷軽減 |
| スタイルプール | ANSIスタイル文字列をキャッシュ、重複生成防止 |

### 差分レンダリング

前フレームとの差分を計算し、変更されたセルのみターミナルに出力。
全画面再描画を回避してレンダリングコストを最小化。

### VirtualMessageList（メッセージ仮想化）

- 可視領域のメッセージのみレンダリング
- 長い会話でも一定のメモリ使用量を維持
- スクロール位置に基づく動的マウント/アンマウント

### Yoga レイアウトキャッシュ

- レイアウト計算結果をキャッシュ
- プロパティ未変更時はレイアウト再計算をスキップ
- ツリー全体ではなく変更サブツリーのみ再計算

---

## 9. データフロー図

```
+----------------------------------------------------------+
|                        REPL.tsx                           |
|                    (トップレベル)                          |
+---------------------------+------------------------------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
   +-------------------+      +---------------------+
   |  AppState (Store)  |      |   Ink Terminal      |
   |  - settings        |      |   - stdin/stdout    |
   |  - messages        |      |   - 60FPS render    |
   |  - tasks           |      |   - alt-screen      |
   |  - mcp             |      +----------+----------+
   |  - teamContext      |                 |
   +--------+-----------+                 |
            |                             |
            v                             v
   +-------------------+      +---------------------+
   |  Hooks (104)       |      |  Ink Components (17)|
   |  - useAppState     |      |  - Box, Text        |
   |  - useMessages     |      |  - ScrollBox        |
   |  - useKeybinding   |      |  - Button, Link     |
   |  - useNotification |      +----------+----------+
   +--------+-----------+                 |
            |                             |
            v                             |
   +-------------------+                  |
   | Components (389)   |                 |
   |  - Messages        |                 |
   |  - Inputs          +-----------------+
   |  - Permissions     |    (レンダリング)
   |  - Tasks           |
   |  - Team            |
   +--------+-----------+
            |
            v
   +-------------------+
   | Terminal Output    |
   | (ANSI sequences)  |
   +-------------------+
```

### データフロー要約

1. **REPL.tsx** がアプリケーション全体を初期化
2. **AppState** がグローバル状態を保持し、Storeパターンで配信
3. **Hooks** が状態とUIロジックを接続（104個のカスタムフック）
4. **Components** がHooksから受け取ったデータをUIに変換（389個）
5. **Ink Components** がReactツリーをターミナル描画命令に変換（17個）
6. **Terminal Output** がANSIエスケープシーケンスとして最終出力
