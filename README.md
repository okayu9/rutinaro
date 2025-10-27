# rutinaro

柔軟な周期ルーティン管理PWAのフロントエンドをSvelte + Viteで構築するためのリポジトリです。現在はViteテンプレートを基にした初期構成のみを含んでいます。

## 必要要件
- Node.js 18 以上
- npm / pnpm などのパッケージマネージャ

## セットアップ
1. 依存関係をインストールします。
   ```sh
   pnpm install
   ```
2. 開発サーバーを起動します。
   ```sh
   pnpm dev
   ```
3. ビルドを実行する場合。
   ```sh
   pnpm build
   ```
4. ビルド成果物をプレビューする場合。
   ```sh
   pnpm preview --open
   ```

## プロジェクト構成
- `src/` : Svelte コンポーネントとエントリポイント。
- `public/` : 静的アセット。
- `vite.config.ts` : Vite 設定（`$lib`, `$components` エイリアスを含む）。
- `tsconfig.json` : TypeScript 設定（`strict: true`、`@tsconfig/svelte` 拡張）。

今後、PLAN.md に基づきドメインモデルやUIなどを実装していきます。
