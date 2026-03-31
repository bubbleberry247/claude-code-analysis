# .map ファイル経由ソースコード流出事件 — 検証結果

**検証日**: 2026-03-31
**対象**: instructkr/claude-code アーカイブ（GitHub Stars: 16,484 / Forks: 24,299）

## 背景

2026-03-31、セキュリティ研究者 Chaofan Shou が npm パッケージ `@anthropic-ai/claude-code` に含まれる `.map` ファイル（sourcemap）から Claude Code のソースコード全体が取得可能であることを報告した。

- **原因**: Bun がビルド時にデフォルトで生成する sourcemap を `.npmignore` から除外し忘れた
- **結果**: TypeScript の元ソースがそのまま復元可能な状態で npm に公開されていた
- **アーカイブ**: instructkr/claude-code として GitHub に保存済み

## 検証結果

記事で言及された未公開機能・コードネーム 13 項目について、ダウンロード済みソースで実在を検証した。

| # | 項目 | 記事の説明 | ソース検証 | ファイルパス |
|---|---|---|---|---|
| 1 | **KAIROS** | 常時オン・プロアクティブ支援 | 実在 | `bridge/bridgeMain.ts`, `commands.ts` |
| 2 | **ULTRAPLAN** | 30分計画セッション + Opus 4.6 | 実在 | `commands/ultraplan.tsx`（`ULTRAPLAN_TIMEOUT_MS = 30*60*1000`） |
| 3 | **BUDDY** | たまごっちペット、ガチャ、18種 | 実在 | `buddy/types.ts`（18種・5段階レアリティ）, `buddy/companion.ts` |
| 4 | **Coordinator Mode** | マルチエージェント統制 | 実在 | `coordinator/coordinatorMode.js`, `tools.ts` |
| 5 | **Dream System** | 記憶整理エンジン | 実在 | `services/autoDream/`（24h+5セッションで発動、三段階ゲート） |
| 6 | **Undercover Mode** | 内部情報漏えい防止 | 実在 | `utils/undercover.ts` |
| 7 | **Tengu** | Claude Code コードネーム | 実在 | `main.tsx`（`logTenguInit()`） |
| 8 | **Fennec** | Opus 旧コードネーム | 実在 | `migrations/migrateFennecToOpus.ts` |
| 9 | **Chicago** | Computer Use | 実在 | `utils/computerUse/gates.ts`（`ChicagoConfig`） |
| 10 | **Penguin Mode** | Fast Mode | 実在 | `utils/fastMode.ts`（`/api/claude_code_penguin_mode`） |
| 11 | **DANGEROUS_uncachedSystemPromptSection()** | キャッシュ破壊関数 | 実在 | `constants/systemPromptSections.ts:32` |
| 12 | **TungstenTool** | 内部社員専用ツール | 実在 | `tools/TungstenTool/`（`USER_TYPE === 'ant'`） |
| 13 | **YOLO classifier** | 自動承認 ML 判定 | 未検出 | — |

### 検証サマリー

- **実在確認**: 12 / 13 項目（92%）
- **未検出**: 1 項目（YOLO classifier）
- **結論**: 記事の信頼性は非常に高い

## ダメージ評価（記事の分析の要約）

| カテゴリ | 深刻度 | 説明 |
|---|---|---|
| **戦略的ダメージ** | 中〜高 | ロードマップ（未公開機能名・実装段階）の流出が最大の打撃。競合が開発方針を把握可能 |
| **セキュリティリスク** | 現実的だが限定的 | クライアント側コードの露出であり、API 側の検証・認証は健在。直接的な悪用は困難 |
| **競争上の露出** | 大きいが致命傷ではない | モデル品質が本質的優位であり、コードの露出だけでは競合が追従できない |

## 皮肉なポイント

**Undercover Mode**（内部情報漏えい防止機構）のソースコード自体が `.map` ファイル経由で漏えいした。情報漏えいを防ぐためのコードが、情報漏えいによって公開されるという構造的皮肉。
