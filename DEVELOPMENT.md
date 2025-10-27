# 開発者向けドキュメント：柔軟な周期ルーティン管理PWA

このドキュメントは、柔軟な周期ルーティン管理PWAの実装時に参照する開発者向け情報をまとめたものである。設計仕様に基づき、システム構造、データモデル、UI方針、技術的制約、品質要件を整理する。

## 1. プロジェクト概要
- **目的**：多様な周期（毎朝／毎日／3日ごと／毎週火曜／毎月29日／第3木曜など）を柔軟に設定・管理できるローカルファーストPWAを提供する。
- **範囲**：単一ページアプリケーション（SPA）として構築し、通知機能は提供しない。ユーザーがアクセスしたタイミングでスケジュール計算を行う。
- **ホスティング**：GitHub Pages。ハッシュルーティングにより直リンク404を回避する。

## 2. 技術スタック
- **フレームワーク**：Svelte + Vite（TypeScript/strict）。
- **ルーティング**：tinro（`#/...`）。
- **スタイリング**：Tailwind CSS、Flowbite-Svelte、Heroicons。
- **日時処理**：Temporal Polyfill（PlainDate/PlainTime/ZonedDateTime、IANAタイムゾーン）。
- **繰り返し処理**：`rrule`（RFC 5545）。
- **データ永続化**：IndexedDB（`idb` ラッパ使用）。
- **状態管理**：Svelte標準ストア（`writable`/`derived`）。
- **PWA**：`vite-plugin-pwa`（`generateSW`、`registerType: 'prompt'`、`start_url`/`scope`は`.`）。
- **テスト**：Vitest、@testing-library/svelte、fast-check、Playwright、axe。

## 3. アプリケーション構造
推奨ディレクトリ構成と主要モジュールの責務は次の通り。

| パス | 役割 |
| --- | --- |
| `src/app.html` | PWAシェル、基本メタタグ、manifest参照の定義。 |
| `src/routes/` | tinroによるルート定義および各ページコンポーネント。|
| `src/lib/components/` | UIコンポーネント（タブ、フォーム、カレンダー、プレビューなど）。|
| `src/lib/domain/` | ドメインモデル、Temporalユーティリティ、RRULE補助関数。|
| `src/lib/dto/` | DTOスキーマ、Zod検証、ドメインとの変換関数。|
| `src/lib/storage/` | IndexedDBアクセス、バージョニング補正、BroadcastChannel通知。|
| `src/lib/stores/` | ルーティン・設定・祝日データのSvelteストア。|
| `src/lib/ics/` | ICSインポート用軽量パーサと正規化ロジック。|
| `src/manifest.json` | Web App Manifest（`start_url`/`scope`は`.`）。|
| `public/` | PWAアイコン群、静的アセット。|

## 4. ドメインモデル
- **Routine**：`id`、`title`、`description`、`timezone`、`enabled`、`schedules`、`exDates`（PlainDate[]）、`rDates`（ZonedDateTime[]）、`updatedAt` など。
- **Schedule**：`rrule`（構造体が正）、`time`（PlainTime）、`startDate`、`endDate` など。RRULEテキストは冗長保持するが、解析不能な場合は保存不可。
- **HolidayPolicy**：`none`／`prev_bday`／`next_bday`／`skip`。
- **MissingDatePolicy**：`skip`／`roll_back`／`roll_forward`。
- **OccurrencePreview**：プレビュー表示用に生成する次回5件のZonedDateTime。
- **設定情報**：タイムゾーン、祝日データメタ情報、バックアップ履歴など。

RRULEは日付パターンのみで展開し、PlainTimeとの合成で最終的なZonedDateTimeを得る。複数スケジュールは個別展開後にマージして重複排除する。

## 5. DTOと検証
- DTOは文字列ベースで保持する（例：日付は`YYYY-MM-DD`、時刻は`"HH:mm"`、日時はISO文字列）。
- ZodスキーマはStrictモードで定義し、未知フィールドは拒否する。
- `mapDomainToDTO` / `mapDTOToDomain` でドメイン⇔DTO変換を行う。
- スキーマ進化は `schemaVersion` に基づきオンザフライで補正する。IndexedDBのバージョンはインデックス追加時のみ上げる。

## 6. 永続化
- IndexedDBストア：`routines`（`keyPath: 'id'`）。
- 格納内容：ルーティン1件あたりに `schedules`・`exDates`・`rDates` を含めたドキュメント指向保存。
- 派生フィールド：`title_lc`（小文字化）、`updatedAt`（ISO UTC文字列）。
- インデックス：`idx_routines_by_title_lc`、`idx_routines_by_updatedAt`、`idx_routines_by_enabled`。
- バージョニング：既存レコード読み込み時に不足フィールドを補い、必要な場合のみIDBバージョンを更新。
- マルチタブ対応：`BroadcastChannel` で保存完了イベントを通知し、競合時はバナー表示でリロードを促す。

## 7. 日時とタイムゾーン
- 時刻は「壁時計の時刻」を真実とみなし、`PlainDate` + `PlainTime` + `timezone` から `ZonedDateTime` を生成する。
- 全ての演算はTemporal（IANA TZ）で行い、DSTによる欠落・重複を安全に処理する。
- 入力粒度は1分刻み。フォームでは24時間表記を使用する。

## 8. 繰り返しルール
- `rrule` は `BYHOUR` / `BYMINUTE` を使用しない。時間情報はフォームで指定したPlainTimeを利用する。
- RRULE展開結果（PlainDate）はEXDATE/RDATE/休日ポリシーを適用後にソートし、重複を除去する。
- `count`と`until`の同時指定は禁止。`interval >= 1`。頻度ごとの必須フィールド（例：WEEKLYは`byDay`必須）を検証する。
- 入力上限：`schedules ≤ 8`、`EXDATE ≤ 1000`、プレビュー範囲は18ヶ月以内、`COUNT ≤ 5000`。

## 9. 休日・営業日処理
- ICSファイルを手動インポートする方式。終日イベントのRRULE/RDATE/EXDATEを展開し、`YYYY-MM-DD`集合に正規化する。
- 祝日ポリシー：`none`／`prev_bday`（前営業日へ）／`next_bday`（翌営業日へ）／`skip`。
- 存在しない日付（例：毎月31日）は `skip`／`roll_back`（末日へ）／`roll_forward`（翌月1日へ）のいずれかを適用する。

## 10. ユーザーインターフェイス
- **ルート構成**：`#/routines`（一覧）／`#/routines/new`（新規）／`#/routines/:id/edit`（編集）／`#/calendar?rid=:id`（月ビュー）／`#/settings`（設定）。
- **一覧画面**：空状態カードから新規作成へ誘導。カードにはタイトル、次回発生、更新日時を表示。
- **編集画面**：タブ（基本／繰り返し／例外／上級）とフッター常設の「次5回プレビュー」。保存は明示的なボタン操作で行い、未保存離脱ガードを設ける。
- **カレンダー**：自作の月グリッドで祝日ハイライト、EXDATEトグル、キーボード移動を実装。
- **設定画面**：タイムゾーン選択、バックアップ／復元、PWA情報、祝日インポート履歴を提供する。

## 11. 入力コンポーネント
- 日付／時刻ピッカーは Flowbite-Svelte を使用し、時刻は1分刻みの24時間表記。
- EXDATEはカレンダー格子でトグル。RDATEはISO＋タイムゾーン必須で追加／削除する。
- RRULEの上級編集ではテキスト入力も許可し、未サポート項目は警告表示を行う。

## 12. アクセシビリティ
- フォーカスリング、フォーカストラップ（モーダル）、Esc/Enter/Space操作対応を全UIで確保する。
- タブは `role="tablist"` と矢印キー移動に対応する。
- カレンダーセルは `button` 化し `aria-pressed` を付与。今日・祝日はスクリーンリーダーで適切にアナウンスする。
- コントラスト比は4.5:1以上。`prefers-reduced-motion` に合わせてアニメーションを制御する。
- 自動a11y検証としてaxeをユニット／E2Eテストに組み込む。

## 13. PWA要件
- `vite-plugin-pwa`の`generateSW`構成でアプリシェルをプリキャッシュし、更新は`prompt`型でユーザーに提示する。
- `start_url`と`scope`は`.`とし、GitHub Pagesの配信パスに追随する。
- オフライン対応：初回インストール後はオフラインでも起動・閲覧可能。
- Service Workerは`self.skipWaiting()`と`clients.claim()`を使い、更新時に通知バナーを表示する。

## 14. セキュリティとプライバシー
- 通信は発生せず、データは端末内IndexedDBに留まる。
- バックアップJSONは平文でエクスポートされるため、取り扱いはユーザー責任。将来的な暗号化拡張余地を残す。

## 15. バックアップ／復元
- 形式：単一JSON（`formatVersion`, `exportedAt`, `settings`, `holidays.dates`, `routines[]`）。
- バックアップはBlobを生成し、`<a download>`で保存する。復元時は全消去後にインポートを実施し、ID衝突の迷いを避ける。
- フォーマット検証に失敗した場合は詳細なエラーメッセージを表示し、保存しない。

## 16. テストと品質保証
- **ユニットテスト**：営業日ポリシー、`generateRange`、RRULE入出力の警告、Temporal変換などをVitestで検証。
- **プロパティテスト**：`fast-check` でRRULE展開の単調増加・重複なしを保証。
- **E2Eテスト**：新規作成～プレビュー確認、カレンダーEXDATEトグル、ICSインポートによる祝日反映、バックアップ往復、PWAオフライン起動をPlaywrightで検証。
- **アクセシビリティ検証**：axeをユニット／E2Eテストに組み込み、主要画面に違反がないことを自動確認する。

---

仕様変更や追加要件が発生した場合は、本ドキュメントを更新し、ドメインモデル・永続化・テスト方針への影響範囲を明確にすること。
