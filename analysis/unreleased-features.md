# Claude Code 未公開機能 個別解析

> 解析日: 2026-03-31
> 対象: Claude Code バンドル内の未公開（ant-only / feature-gated）機能群

---

## 1. KAIROS — セッション連続性システム

### 概要

セッション間の状態を維持し、以前の作業コンテキストをシームレスに復帰させるシステム。
`feature('KAIROS')` ゲートで制御され、ant-only ビルドでのみ有効。

### エントリポイント

| 要素 | 説明 |
|---|---|
| Feature Gate | `feature('KAIROS')` — ant-only |
| CLI フラグ | `--continue` / `--session-id <id>` でセッション復帰 |
| 状態管理 | `kairosActive` フラグで有効/無効を追跡 |

### セッション復帰メカニズム

1. `--continue` または `--session-id` フラグを検出
2. `bridge-pointer.json` を読み込み
3. bridge-pointer から **worktree 兄弟ディレクトリ** を検索
4. 一致するセッション環境を復元

### 環境保持

- シャットダウン時に環境を **生きたまま残す**（destroy しない）
- **TTL: 4 時間** — 4 時間経過後に自動クリーンアップ
- これにより `--continue` での即座復帰が可能

### PROACTIVE フラグ

- `PROACTIVE` フラグの存在は確認されている
- ただし **実装ファイルは未発見** — スタブまたは将来実装予定の可能性

### 他機能との制約

- **Dream System と相互排除**: `kairosActive === true` の場合、Dream（自動記憶統合）は無効化される

---

## 2. ULTRAPLAN — マルチエージェント計画システム

### 概要

複雑なタスクを検出し、自動的にマルチエージェント計画モードに移行するシステム。
リモートコンテナ上でエージェントを実行し、計画の承認フローを管理する。

### 定数・設定

| 定数 | 値 | 説明 |
|---|---|---|
| `ULTRAPLAN_TIMEOUT_MS` | `30 * 60 * 1000` (30 分) | 計画モードのタイムアウト |
| デフォルトモデル | Opus 4.6 | GrowthBook 未設定時のフォールバック |

### モデル選択

```
getUltraplanModel():
  1. GrowthBook feature flag: tengu_ultraplan_model を参照
  2. 値が設定されていればそのモデルを使用
  3. 未設定 → デフォルト Opus 4.6
```

### 実行基盤

- **CCR v2 SDK URL** でリモートコンテナを起動
- ローカルではなくクラウド上でエージェントが計画を策定

### 状態遷移 (ExitPlanModeScanner)

```
running → needs_input → plan_ready
```

| 状態 | 説明 |
|---|---|
| `running` | 計画策定中 |
| `needs_input` | ユーザー入力待ち |
| `plan_ready` | 計画完成、承認待ち |

### 承認ポーリング

```
pollForApprovedExitPlanMode():
  - ポーリング間隔: 3 秒
  - 最大連続失敗: 5 回 → 中止
  - 承認されるまでループ
```

### 自動起動トリガー

- `findUltraplanTriggerPositions()` がユーザー入力からキーワードを検出
- 特定のキーワードパターンに一致すると自動的に ULTRAPLAN モードを起動

---

## 3. BUDDY — AI コンパニオンシステム

### 概要

たまごっち風の AI コンパニオン。ユーザーごとに決定論的に生成され、
レアリティ・種族・外見パーツがランダム割り当てされる。

### 生物一覧 (18 種)

| # | Species | # | Species | # | Species |
|---|---|---|---|---|---|
| 1 | duck | 7 | owl | 13 | cactus |
| 2 | goose | 8 | penguin | 14 | robot |
| 3 | blob | 9 | turtle | 15 | rabbit |
| 4 | cat | 10 | snail | 16 | mushroom |
| 5 | dragon | 11 | ghost | 17 | chonk |
| 6 | octopus | 12 | axolotl | 18 | capybara |

### レアリティシステム

| レアリティ | 確率 |
|---|---|
| Common | 60% |
| Uncommon | 25% |
| Rare | 10% |
| Epic | 4% |
| Legendary | 1% |

**Shiny（色違い）**: 全レアリティ共通で **1%** の確率で出現

### 決定論的生成

```
入力: userId + SALT('friend-2026-401')
  ↓
Mulberry32 PRNG（疑似乱数生成器）
  ↓
出力: CompanionBones {
  rarity,
  species,
  eye,    // 6 種
  hat,    // 8 種
  shiny,  // boolean
  stats   // 5 能力値
}
```

同一ユーザーは常に同じコンパニオンを取得する。SALT が変わらない限り結果は不変。

### ステータス (STAT_NAMES)

| Stat | 説明 |
|---|---|
| DEBUGGING | デバッグ力 |
| PATIENCE | 忍耐力 |
| CHAOS | 混沌度 |
| WISDOM | 知恵 |
| SNARK | 皮肉力 |

### UI レンダリング (CompanionSprite.tsx)

| 要素 | 仕様 |
|---|---|
| ティック間隔 | 500ms |
| アイドルシーケンス | フレームアニメーション |
| 吹き出し | テキスト表示 |
| ハート浮揚 | アニメーション演出 |

### 未実装要素

- **たまごっち的育成**: 未実装。餌やり・成長・進化などの仕組みはない
- **Stats**: UI 表示のみ。ゲームプレイへの影響なし

---

## 4. Dream System — 自動記憶統合

### 概要

バックグラウンドで CLAUDE.md（メモリファイル）を自動統合・整理するシステム。
`/dream` コマンドの自動版として動作する。

### ファイル構成

```
services/autoDream/    — Dream サービス本体
tasks/DreamTask/       — UI タスク表示
consolidationPrompt.ts — 統合プロンプト（4 フェーズ）
```

### 三段階ゲート

統合実行には以下の **全条件** を満たす必要がある:

| # | Gate | 条件 | 詳細 |
|---|---|---|---|
| 1 | **Time Gate** | 前回統合から **24 時間**経過 | `lastConsolidatedAt` で判定 |
| 2 | **Sessions Gate** | **5 セッション**以上記録 | 10 分間隔でスキャン |
| 3 | **Lock Gate** | consolidation-lock 未取得 | PID ベース、**1 時間**で stale 判定 |

### Feature Gate

- GrowthBook gate: `tengu_onyx_plover`

### 無効化条件

| 条件 | 理由 |
|---|---|
| `kairosActive === true` | KAIROS とは相互排除 |
| Remote mode | リモート実行中は干渉を避ける |
| Memory dir 未設定 | 統合対象がない |

### 統合プロンプト 4 フェーズ (consolidationPrompt.ts)

```
Phase 1: Orient   — 現在のメモリ構造を把握
Phase 2: Gather   — 直近セッションから重要情報を収集
Phase 3: Consolidate — 重複排除・統合・構造化
Phase 4: Prune    — 不要エントリの削除
```

### 実行メカニズム

- **forked agent** として実行（メインエージェントとは別プロセス）
- `/dream` コマンド相当の処理を自動実行

### DreamTask UI

| 要素 | 仕様 |
|---|---|
| 表示範囲 | 最新 30 ターン |
| Phase 遷移 | `starting` → `updating` |

### Lock ファイルの二重目的

- `lockファイルの mtime` = `lastConsolidatedAt` として利用
- ロールバック対応: lock ファイルの mtime を巻き戻すことで再統合を強制可能

---

## 5. 機能間の制約マトリクス

### 相互排除・依存関係

```
┌──────────┐     相互排除     ┌──────────┐
│  KAIROS  │ ←──────────────→ │  Dream   │
└──────────┘                  └──────────┘
                                  │
                                  │ 無効化
                                  ▼
                          ┌──────────────┐
                          │ Remote Mode  │
                          └──────────────┘
```

| 制約 | 説明 |
|---|---|
| KAIROS ↔ Dream | 相互排除。KAIROS active 時に Dream は実行されない |
| Remote Mode → Dream | Remote mode では Dream 無効 |
| Memory dir → Dream | Memory dir が存在しない場合、Dream は起動しない |
| ant-only ゲート | KAIROS 含む多数の機能が ant-only。外部ビルドでは tree-shake される |

### ゲート一覧

| 機能 | Feature Gate | GrowthBook Key | ant-only |
|---|---|---|---|
| KAIROS | `feature('KAIROS')` | — | Yes |
| ULTRAPLAN | — | `tengu_ultraplan_model` | — |
| BUDDY | — | — | — |
| Dream | — | `tengu_onyx_plover` | — |

---

## 補足: 用語集

| 用語 | 説明 |
|---|---|
| ant-only | Anthropic 内部ビルドでのみ有効な機能フラグ |
| GrowthBook | フィーチャーフラグ管理サービス。A/B テストやロールアウト制御に使用 |
| tree-shake | 外部ビルド時に未使用コード（ant-only 機能）を除去する最適化 |
| CCR v2 | Claude Code Remote v2。リモートコンテナ実行基盤 |
| Mulberry32 | 32 ビット整数ベースの高速 PRNG アルゴリズム |
| forked agent | メインプロセスからフォークされた独立エージェントプロセス |
| consolidation-lock | 排他制御用ロックファイル。PID を記録し多重実行を防止 |
