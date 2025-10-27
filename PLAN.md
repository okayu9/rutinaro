# 実装計画：柔軟な周期ルーティン管理PWA

本計画は、仕様ドキュメントの要件を満たすための作業をフェーズ別・モジュール別に詳細化したものである。各タスクは、成果物、主担当モジュール、完了条件を明確にしている。チェックボックス形式で進捗を管理できるように再構成している。

---

## 1. プロジェクトセットアップ
- [ ] **開発環境の初期化**
  - [ ] `npm create vite@latest` で Svelte + TypeScript テンプレートを生成し、`strict: true` を tsconfig で確認。
  - [ ] エイリアス設定（`$lib`, `$components` など）を `tsconfig.json` と `vite.config.ts` 双方で統一。
- [ ] **依存パッケージ導入**
  - [ ] UI/スタイル: `tailwindcss`, `postcss`, `autoprefixer`, `flowbite-svelte`, `@tailwindcss/forms`, `@heroicons/svelte`。
  - [ ] 日時・繰り返し: `@js-temporal/polyfill`, `rrule`, `@date-fns/tz`（タイムゾーンリスト用）。
  - [ ] データ・検証: `idb`, `zod`, `uuid`（polyfill 不要を確認しつつ）。
  - [ ] テスト: `vitest`, `@testing-library/svelte`, `@testing-library/jest-dom`, `@playwright/test`, `@testing-library/dom`, `axe-core`, `vitest-axe`, `fast-check`。
- [ ] **Vite 設定**
  - [ ] `vite.config.ts` に tinro 向けの `hash` ベースルーティングを想定した base パス注入ロジックを追加（`VITE_PUBLIC_BASE` 環境変数）。
  - [ ] `vite-plugin-pwa` を設定し、`generateSW`, `registerType: 'prompt'`, `includeAssets`（アイコン）, `start_url: '.'`, `scope: '.'` を指定。
- [ ] **Tailwind/Flowbite 初期設定**
  - [ ] `tailwind.config.cjs` で `flowbite-svelte` のパス、`prefers-reduced-motion` の variants、`4.5:1` コントラストとなるカラーパレットを定義。
  - [ ] `src/app.css` にベーススタイル（フォーカスリング、スクリーンリーダー向けユーティリティ）を追加。
  - [ ] Flowbite のカスタムテーマファイルを `src/lib/theme.ts` に配置。
- [ ] **CI/CD ベース整備**
  - [ ] GitHub Actions で `npm install`, `npm run lint`, `npm run test`, `npm run build` を行うワークフロー雛形を追加。
  - [ ] ブランチ保護ルールとして `main` への PR には上記ワークフロー成功が必須とする。

---

## 2. データレイヤーと型設計
- [ ] **ドメインモデル定義** (`src/lib/domain/`)
  - [ ] 型: `Routine`, `Schedule`, `RRuleOptions`, `HolidayPolicy`, `OccurrencePreview` などを `Temporal` オブジェクトベースで定義。
  - [ ] ユーティリティ: `composeZonedDateTime(plainDate, plainTime, tz)`, `applyHolidayPolicy(schedule, holidays)` を実装。
  - [ ] ガード関数: RRULE の妥当性チェック（interval >= 1, count/until 排他, BYDAY/BYMONTHDAY 等の必須条件）。
- [ ] **DTO と変換ロジック** (`src/lib/dto/`)
  - [ ] Zod スキーマ: `RoutineDTO`, `ScheduleDTO`, `SettingsDTO`, `HolidayFileDTO` を strict モードで作成。
  - [ ] `mapDomainToDTO` / `mapDTOToDomain` 関数を用意し、`PlainDate`⇔`YYYY-MM-DD`, `PlainTime`⇔`HH:mm`, `ZonedDateTime`⇔ISO 変換を実装。
  - [ ] バージョン差分ハンドリング: `schemaVersion` を参照して不足フィールドを補完する関数を追加。
- [ ] **IndexedDB ストレージ** (`src/lib/storage/indexeddb.ts`)
  - [ ] `idb` ラッパを用いて `routines` ストアを作成（`keyPath: 'id'`、インデックス 3 種）。
  - [ ] CRUD: `listRoutines`, `getRoutine`, `saveRoutine`, `deleteRoutine`, `bulkImport` を実装。保存時に `updatedAt` を ISO で更新。
  - [ ] `schemaVersion` アップグレード: レコード読み込み時に補正を行う関数、インデックス追加時の IDB バージョンアップロジックを記述。
- [ ] **マルチタブ通知**
  - [ ] `BroadcastChannel` を `src/lib/channels/storage.ts` に定義し、保存完了イベント発火と他タブ側のリロード/バナー通知を実装。
  - [ ] 未保存変更が存在するタブが通知を受けた場合、軽微な競合解決フロー（リロード促進バナー）を表示する仕組みを整える。

---

## 3. ルーティング・ストア・ページ骨組み
- [ ] **tinro ルーター設定**
  - [ ] `src/routes/+layout.svelte` で tinro を初期化し、`#/routines`, `#/routines/new`, `#/routines/:id/edit`, `#/calendar`, `#/settings` を定義。
  - [ ] 404/リダイレクト処理を tinro の fallback で実装。
- [ ] **グローバルストア設計** (`src/lib/stores/`)
  - [ ] `routinesStore`: Writable ストアで IndexedDB との同期、ソート（`updatedAt` 降順）、`load`, `refresh`, `upsert`, `remove` API を提供。
  - [ ] `settingsStore`: タイムゾーン・祝日ポリシー・バックアップ設定を保持。初期化時に `Intl.DateTimeFormat().resolvedOptions().timeZone` を使用。
  - [ ] `holidaysStore`: ICS インポート結果（PlainDate の Set）とメタ情報を保持。
- [ ] **ページスケルトン**
  - [ ] 一覧ページ：空データ時のカード（新規作成導線）、既存データのカードビュー（タイトル、次回発生、更新日時）。
  - [ ] 新規/編集ページ：タブ構造と保存ボタン配置のみ先行実装、後続でフォーム詳細を埋める。
  - [ ] カレンダー：月グリッドの土台（ヘッダーナビゲーション、曜日列、セルのレンダリングループ）。
  - [ ] 設定：タイムゾーンセレクト、バックアップ/復元セクション、PWA 情報プレースホルダを配置。

---

## 4. ルーティン編集 UI 詳細
- [ ] **フォームステート管理**
  - [ ] `useRoutineForm` カスタムフック（Svelte ストア）でフォーム値・検証状態・未保存フラグを管理。
  - [ ] `beforeunload` リスナー、tinro の `goto` フックを利用して離脱ガードを実装。
- [ ] **基本タブ**
  - [ ] 入力項目: タイトル、説明、開始日 (`PlainDate`)、開始時刻 (`PlainTime`)、タイムゾーン（`<select>` + 検索）、有効/無効トグル。
  - [ ] Flowbite コンポーネントのアクセシビリティ属性確認（`aria-label`, `aria-describedby`）。
- [ ] **繰り返しタブ**
  - [ ] Schedule リスト: 最大 8 件まで追加可能、各 Schedule は accordion/cards で折り畳み式編集。
  - [ ] RRULE フォーム: `freq`, `interval`, `byDay`, `byMonthDay`, `bySetPos`, `count`, `until`、`wkst` 等。未入力禁止条件に対するエラーメッセージ表示。
  - [ ] RRULE テキスト保持: ユーザー編集不可の hidden フィールドに保持し、解析不能なら保存不可。`BYHOUR/BYMINUTE` が含まれる場合は警告を表示しフォームの時刻を優先。
- [ ] **例外タブ**
  - [ ] EXDATE: カレンダーグリッドで各日ボタン化 (`aria-pressed`)、today/highlight のスタイルを Tailwind で調整。
  - [ ] RDATE: ISO/TZ 入力フォーム（`datetime-local` ではなくテキスト + TZ セレクト）、追加・削除の操作ログを表示。
  - [ ] 追加時に重複排除・昇順ソートを実施。
- [ ] **上級タブ**
  - [ ] RRULE テキストエディタ（textarea）とバリデーション結果（警告メッセージ領域）。
  - [ ] 祝日ポリシー選択（none / prev_bday / next_bday / skip）、存在しない日付ポリシー（skip / roll_back / roll_forward）。
  - [ ] RRULE OR 合成プレビューの設定（最大 18 ヶ月、COUNT <= 5000 で打ち切り）。
- [ ] **プレビューコンポーネント** (`src/lib/components/PreviewPane.svelte`)
  - [ ] Temporal + rrule を組み合わせて次回 5 件の `ZonedDateTime` を生成。
  - [ ] EXDATE/RDATE を適用、祝日ポリシーによる日付調整結果も反映。
  - [ ] 表示形式（曜日、日付、時刻、タイムゾーン）と重複排除ロジックをテストで保証。

---

## 5. カレンダー画面
- [ ] **月グリッドコンポーネント** (`src/lib/components/calendar/`)
  - [ ] `CalendarGrid.svelte`: 月初の曜日に合わせて前後の空セルを生成、`button` でセルを描画し `aria-pressed`・`aria-label` を付与。
  - [ ] キーボードナビゲーション: 矢印キーで前後移動、Home/End で週先頭/末尾、PageUp/Down で月切り替え。
  - [ ] `prefers-reduced-motion` に応じたアニメーション抑制。
- [ ] **祝日ハイライトと EXDATE トグル**
  - [ ] `holidaysStore` から当月の祝日を取得し、セルに `data-holiday` 属性を付与してスタイル適用。
  - [ ] EXDATE トグルはクリック/Enter/Space で切り替え、更新結果を `routinesStore` へ書き戻し。
- [ ] **月切り替えロジック**
  - [ ] ルートクエリ `rid` から対象ルーティンを特定し、Temporal OR 合成で指定月の発生イベントを算出。
  - [ ] 別月移動時に再計算し、ロード中インジケーターを表示。

---

## 6. 設定・バックアップ機能
- [ ] **タイムゾーン設定**
  - [ ] `Intl.supportedValuesOf('timeZone')`（未対応ブラウザの場合は静的リスト）で候補を取得し、検索付きセレクトを実装。
  - [ ] 選択結果は `settingsStore` に保存し、ルーティン作成時のデフォルト値に反映。
- [ ] **バックアップ/復元 UI**
  - [ ] `Blob` ダウンロードを `download` 属性付き `<a>` で提供。ファイル名に `exportedAt` を含める。
  - [ ] 復元時は確認ダイアログ後に IndexedDB を全削除→JSON インポート→ストア再同期。
  - [ ] フォーマット検証（`formatVersion` や必須フィールド）に失敗した場合は詳細なエラーメッセージを表示。
- [ ] **ICS インポート**
  - [ ] 軽量パーサを `src/lib/ics/parser.ts` に実装し、RRULE/RDATE/EXDATE を `PlainDate` 集合に正規化。
  - [ ] ICS 取り込み後、祝日ポリシーに依存する画面へ通知し、必要なら再計算。
  - [ ] インポート履歴（ファイル名・件数・最終更新日）を settings に保持。

---

## 7. アクセシビリティとスタイル強化
- [ ] **アクセシビリティ QA**
  - [ ] フォーカスリングが Tailwind カスタムクラスで全要素に適用されることを Storybook もしくはコンポーネントデモで確認。
  - [ ] モーダル（ICS インポートなど）にフォーカストラップと `Esc` 閉じる機能を実装。
  - [ ] タブコンポーネントは `role="tablist"` と矢印キー移動、`aria-controls` と `aria-selected` を管理。
- [ ] **カラー/テーマ**
  - [ ] コントラスト比を `tailwind.config.cjs` のプラグインまたは手動チェックで検証。
  - [ ] `prefers-reduced-motion` メディアクエリでアニメーションを無効化。トランジションが必要な箇所は `motion-safe` クラスで制御。
- [ ] **自動テスト**
  - [ ] `vitest-axe` を用いて主要コンポーネント（一覧、フォーム、カレンダー、設定）に対するアクセシビリティ検証を追加。

---

## 8. PWA 対応
- [ ] **Service Worker 構成**
  - [ ] vite-plugin-pwa の `workbox` 設定で静的ファイルのキャッシュ戦略を `StaleWhileRevalidate` に。`navigateFallback` を `index.html` に設定。
  - [ ] `self.skipWaiting()` と `clients.claim()` を更新通知フローに組み込み、`registerType: 'prompt'` で更新トーストを表示。
- [ ] **Web App Manifest**
  - [ ] `src/manifest.json` を作成し、`name`, `short_name`, `description`, `theme_color`, `background_color`, `display: 'standalone'`, アイコン群を指定。
  - [ ] GitHub Pages の `base` パスに対応するように相対パスを使用。
- [ ] **オフラインテスト**
  - [ ] Playwright で Service Worker 登録後にネットワークオフで `start_url` を読み込むテストを作成。
  - [ ] 主要ページのキャッシュヒットを確認（console log や `workbox` debug）。

---

## 9. テスト戦略詳細
- [ ] **ユニットテスト (Vitest)**
  - [ ] `generateOccurrences`（RRULE 展開 + PlainTime 合成）: 2/29, 月末, 第 n 曜日, 不存在日付処理など境界ケースを網羅。
  - [ ] `applyHolidayPolicy`: `prev_bday`, `next_bday`, `skip` の挙動を営業日カレンダーの例で検証。
  - [ ] DTO 変換: `mapDomainToDTO` / `mapDTOToDomain` の round-trip を fast-check でプロパティテスト。
- [ ] **コンポーネントテスト**
  - [ ] Svelte Testing Library でフォームバリデーション、タブ操作、EXDATE カレンダーのトグル、未保存ガードをテスト。
  - [ ] axe による自動 a11y チェックをスナップショット化。
- [ ] **E2E (Playwright)**
  - [ ] シナリオ: 新規作成→プレビュー確認、カレンダーで EXDATE トグル → プレビュー更新、ICS インポート → 祝日反映確認、バックアップ出力→初期化→復元、PWA オフラインスモーク。
  - [ ] CI 上で headless 実行、`pages` ブランチにデプロイする前の gating とする。

---

## 10. リリース準備・運用
- [ ] **デプロイ**
  - [ ] GitHub Actions で `npm run build` 後、`dist/` を `gh-pages` ブランチにデプロイ。`pages` もしくは `deployments` を使用して公開。
  - [ ] `VITE_PUBLIC_BASE` を GitHub Actions の環境変数で `/<repo>/` に設定。
- [ ] **ドキュメント整備**
  - [ ] `README.md` にセットアップ手順、PWA インストール方法、バックアップ/復元注意事項、開発者向けコマンド一覧を詳細記載。
  - [ ] `docs/` 以下にユーザーガイド（ルーティン作成、カレンダー操作、設定・バックアップ手順）を作成。
- [ ] **メンテナンス**
  - [ ] バージョンタグ (`v0.1.0`) を発行し、`CHANGELOG.md` を `Keep a Changelog` 形式で初期化。
  - [ ] 既知の制約（通知未対応、ローカル専用など）を issue テンプレートに明記。

---

## 11. マイルストーンと優先度
- [ ] **M1: 基盤整備 (Week 1)**
  - [ ] プロジェクト初期化、データモデルの雛形、ルーティング骨組み。
- [ ] **M2: ルーティン編集体験 (Week 2-3)**
  - [ ] フォーム、RRULE 処理、プレビュー機能。
- [ ] **M3: カレンダー & 例外処理 (Week 4)**
  - [ ] 月グリッド、EXDATE/RDATE 操作、祝日ポリシー適用。
- [ ] **M4: 設定・バックアップ・PWA (Week 5)**
  - [ ] ICS インポート、バックアップ、Service Worker。
- [ ] **M5: テスト整備 & リリース (Week 6)**
  - [ ] テストカバレッジ確保、CI/CD、ドキュメント整備、GitHub Pages 公開。

---

## 12. 完了条件チェックリスト
- [ ] 仕様で求められるルート全てが tinro で動作し、ハッシュルーティングの直接アクセスに耐える。
- [ ] Routine 保存が IndexedDB 上で正しく永続化され、マルチタブでも矛盾しない。
- [ ] RRULE 解析とプレビュー生成が仕様通り（時間はフォーム値、日付は rrule OR 合成）に動作。
- [ ] カレンダーで EXDATE トグル・祝日ハイライト・キーボード操作が成立。
- [ ] バックアップ JSON のエクスポート/インポートが往復可能。
- [ ] axe / Vitest / Playwright / fast-check の各テストが CI で成功。
- [ ] PWA としてオフライン起動が可能、更新フローが `prompt` で提示される。
- [ ] README と docs がユーザー・開発者双方に必要情報を提供。
