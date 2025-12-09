# Laravel My Manual (Study Group版: Sail + npm)

> 受講者向けに、フロント依存も含めて **すべて Sail コンテナ内で npm を叩く**手順に統一しています。pnpm やホスト Node のセットアップは不要です。

## 先に知っておくこと（実行環境の前提）

- PHP/Composer/Artisan/Node/npm はすべて **Sail 内で実行**する。
- ホスト側は Docker Desktop（WSL2+Ubuntu でも可）と Git/curl だけ入っていればOK。Node/npm/pnpm をホストに入れる必要はない。
- Vite 開発時に外部デバイスから見るなら `./vendor/bin/sail npm run dev -- --host 0.0.0.0 -- --port 5173` のように `--host` を付ける。

## Laravel Sail の起動と停止

### プロジェクト作成

```bash
curl -s "https://laravel.build/anglers?php=84&node=22" | bash
cd {プロジェクト名}
./vendor/bin/sail up
```

ブラウザで `http://localhost` にアクセス。

### セッションエラーが出た場合

初回アクセス時にセッションエラーが出ることがある。その場合 `.env` を次に変更：

```dotenv
SESSION_DRIVER=file
```

### 停止

- ターミナルで `Ctrl + C`
- Docker Desktop から起動・停止も可能

---

## Vite + Inertia v2 セットアップ（npm を Sail 内で使用）

### Vue 3 をインストール

```bash
./vendor/bin/sail npm install vue@^3
```

### Vite に Vue プラグインを追加

```bash
./vendor/bin/sail npm install -D @vitejs/plugin-vue
```

`vite.config.js` を以下のように編集:

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
        tailwindcss(),
        vue({
          template: {
            transformAssetUrls: {
              base: null,
              includeAbsolute: false
            }
          }
        }),
    ],
});
```

### Inertia をインストール (サーバーサイド)

```bash
./vendor/bin/sail composer require inertiajs/inertia-laravel
```

`resources/views/welcome.blade.php` を `app.blade.php` にリネームし、以下のように編集:

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1">
      @vite('resources/js/app.js')
      @inertiaHead
  </head>
  <body>
    @inertia
  </body>
</html>
```

### ミドルウェアの登録

```bash
./vendor/bin/sail artisan inertia:middleware
```

`bootstrap/app.php` のミドルウェアのブロックに以下を追加:

```php
use App\Http\Middleware\HandleInertiaRequests;

$middleware->web(append: [
    HandleInertiaRequests::class,
]);
```

### Inertia をインストール (クライアントサイド)

```bash
./vendor/bin/sail npm install @inertiajs/vue3
```

`resources/js/app.js` を以下の内容に変更:

```js
import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';

createInertiaApp({
  resolve: name => {
    const pages = import.meta.glob('./Pages/**/*.vue', { eager: true })
    return pages[`./Pages/${name}.vue`]
  },
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
});
```

### 最初のページとルート

`resources/js/Pages/Index/Index.vue` を作成:

```vue
<template>Hello {{ counter }}!</template>
<script setup>
import { ref } from 'vue'
const counter = ref(0)
setInterval(() => counter.value++, 1000)
</script>
```

`routes/web.php` を編集:

```php
use Inertia\Inertia;

Route::get('/', function () {
    return Inertia::render('Index/Index');
});
```

### 確認

```bash
./vendor/bin/sail npm run dev
```

ブラウザで `http://localhost` を開いてカウンターが動けばOK。

---

## 多言語対応 (i18n)

### 言語ファイルの publish

```bash
./vendor/bin/sail artisan lang:publish
```

- `lang/en` が生成されるので、複製して `lang/ja` を作成する

### Vue i18n パッケージのインストール

```bash
./vendor/bin/sail npm install -D laravel-vue-i18n
```

### vite.config.js の設定

```js
import i18n from "laravel-vue-i18n/vite";

export default defineConfig({
  plugins: [
    laravel({
      input: ['resources/css/app.css', 'resources/js/app.js'],
      refresh: true,
    }),
    tailwindcss(),
    vue(),
    i18n(),
  ],
});
```

### app.js の設定

```js
import { i18nVue } from "laravel-vue-i18n";

createInertiaApp({
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .use(i18nVue, {
        resolve: async (lang) => {
          const langs = import.meta.glob('../../lang/*.json');
          return await langs[`../../lang/${lang}.json`]();
        },
      })
      .mount(el)
  },
});
```

### 実行

```bash
./vendor/bin/sail npm run dev
```

- `php_en.json` と `php_ja.json` が生成される
- Vite サーバ終了時に削除されるため `.gitignore` に追加すること

---

## デフォルトテンプレート設定

### MainLayout の作成

`resources/js/Layouts/MainLayout.vue` を作成

```js
<template>
    <slot>Default Layout</slot>
</template>
```

### app.js の設定変更

```js
import { resolvePageComponent } from "laravel-vite-plugin/inertia-helpers"
import MainLayout from '@/Layouts/MainLayout.vue'

const page = resolvePageComponent(
            `./Pages/${name}.vue`,
            import.meta.glob("./Pages/**/**/*.vue")
        )
        page.then((module) => {
            module.default.layout = module.default.layout || MainLayout
        })

        return page
```

## TailwindCSS（Laravel 12）

- Laravel 12 は **Tailwind v4 + @tailwindcss/vite** が最初から導入済み。追加セットアップ不要。
- `resources/css/app.css` に **v4 記法**で書く（`@import` / `@source` / `@theme`）。`@source` は v4 の `content` 置き換えなので削除しない。

### app.css の基本形（v4）

```css
@import 'tailwindcss';

/* 既存の @source は残す */
/* 例:
@source '../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php';
@source '../../storage/framework/views/*.php';
@source '../**/*.blade.php';
@source '../**/*.js';
*/

/* 既存の @theme もここに置く */
@theme {
  /* 例: --font-sans: ... */
}
```

### 共通コンポーネント（@layer components）

従来の v3 と同様に使える。`app.css` に追記。

```css
@layer components {
  .label { @apply block text-sm font-medium leading-6 text-gray-900; }
  .input-text { @apply block w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6; }
  .input-text-valid { @apply block w-full rounded-md border-0 py-1.5 pr-10 text-red-900 ring-1 ring-inset ring-red-300 placeholder:text-red-300 focus:ring-2 focus:ring-inset focus:ring-red-500 sm:text-sm sm:leading-6; }
  .btn-primary { @apply rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-semibold leading-6 text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600; }
  .btn-outline { @apply rounded-md bg-white px-2.5 py-1.5 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50; }
  .btn-cancel { @apply text-sm font-semibold leading-6 text-gray-900; }
  .input-error { @apply text-sm text-red-500; }
}
```

### 公式プラグイン（v4 の書き方）

```bash
./vendor/bin/sail npm install -D @tailwindcss/forms @tailwindcss/typography @tailwindcss/aspect-ratio
```

`app.css` に `@plugin` で読み込む：

```css
@plugin '@tailwindcss/forms';
@plugin '@tailwindcss/typography';
@plugin '@tailwindcss/aspect-ratio';
```

### Vue での使い方

Vue の `<template>` にユーティリティをそのまま書く。

```vue
<template>
  <div class="hidden lg:fixed lg:inset-y-0 lg:z-50 lg:flex lg:w-72 lg:flex-col">
    <!-- ... -->
  </div>
</template>
```

### tailwind.config.js について

- v4 では **必須ではない**。`@source` / `@theme` / `@plugin` を `app.css` に書く方式に移行。
- 外部 UI ライブラリや特殊なプラグイン設定が必要な場合のみ作成する。
