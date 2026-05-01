# 第1章：composer.json と package.json の読み方

この章では、  
**Laravel プロジェクトのパッケージ管理をどこで読むか**を整理する。

ポイントは次の3つ。

- PHP 側の依存関係は `composer.json` で見る
- JavaScript / Vite 側の依存関係は `package.json` で見る
- 実際にインストールされるバージョンは lockfile で固定される

---

## 1-1 今日見るファイル

今回見るファイルは次の4つ。

- `anglers/composer.json`
- `anglers/composer.lock`
- `anglers/package.json`
- `anglers/package-lock.json`

追加で、npm の設定として次も見る。

- `anglers/.npmrc`

このプロジェクトでは `.npmrc` に次の設定を入れている。

```ini
min-release-age=7
```

これは、公開から7日以内の新しい npm パッケージを避けるための設定。

---

## 1-2 composer.json は PHP 側の設計図

`composer.json` は、PHP / Laravel 側のパッケージ管理ファイル。

このファイルを見ると、次のことが分かる。

- このプロジェクトが何のプロジェクトか
- どの PHP バージョンを前提にしているか
- Laravel や FileMaker 連携など、実行時に必要なパッケージ
- 開発やテストでだけ使うパッケージ
- クラスの自動読み込みルール
- Composer 実行時に走る script
- パッケージ管理の細かい設定

---

## 1-3 まず require を見る

`require` は、アプリケーションの実行に必要なパッケージ。

```json
"require": {
    "php": "^8.2",
    "gearbox-solutions/eloquent-filemaker": "^2.6",
    "inertiajs/inertia-laravel": "^2.0",
    "intervention/image": "^3.11",
    "laravel/framework": "^12.0",
    "laravel/tinker": "^2.10.1",
    "tightenco/ziggy": "^2.6"
}
```

ここを見ると、このプロジェクトの中心が分かる。

- `php`
  - PHP 8.2 以上を前提にしている
- `laravel/framework`
  - Laravel 本体
- `gearbox-solutions/eloquent-filemaker`
  - FileMaker を Eloquent 風に扱うためのパッケージ
- `inertiajs/inertia-laravel`
  - Laravel と Vue を Inertia でつなぐためのパッケージ
- `intervention/image`
  - 画像処理用のパッケージ
- `tightenco/ziggy`
  - Laravel のルート情報を JavaScript 側で使うためのパッケージ
- `laravel/tinker`
  - 対話的に Laravel のコードを試すためのツール

> まず `require` を見ると、  
> このアプリが何を土台にして動いているかが分かる。

---

## 1-4 require-dev は開発用

`require-dev` は、開発時やテスト時に使うパッケージ。

```json
"require-dev": {
    "fakerphp/faker": "^1.23",
    "itsgoingd/clockwork": "^5.3",
    "laravel/pail": "^1.2.2",
    "laravel/pint": "^1.24",
    "laravel/sail": "^1.41",
    "mockery/mockery": "^1.6",
    "nunomaduro/collision": "^8.6",
    "phpunit/phpunit": "^11.5.3"
}
```

ここには、本番動作そのものではなく、開発を助けるものが入る。

- `laravel/sail`
  - Docker 開発環境
- `phpunit/phpunit`
  - PHP のテスト
- `laravel/pint`
  - PHP コード整形
- `fakerphp/faker`
  - テストデータ生成
- `mockery/mockery`
  - テスト用のモック
- `laravel/pail`
  - ログ確認用ツール
- `clockwork`
  - 開発時のデバッグ支援
- `collision`
  - エラー表示を見やすくする

### require と require-dev の違い

- `require`
  - アプリを動かすために必要
- `require-dev`
  - 開発、テスト、デバッグで必要

本番環境では、通常 `require-dev` は入れない。

```bash
composer install --no-dev
```

---

## 1-5 バージョン指定の読み方

Composer では、次のような指定がよく出てくる。

```json
"laravel/framework": "^12.0"
```

`^12.0` は、ざっくり言うと、

> Laravel 12 系の範囲で、互換性がある新しいバージョンを許可する

という意味。

たとえば `^12.0` は、通常 `12.x` 系の更新を許可する。

一方で、`^2.6` のような指定なら、`2.x` 系の範囲で更新される。

ここで大事なのは、`composer.json` だけでは実際の細かいバージョンまでは決まらないこと。

実際に入っているバージョンは `composer.lock` に記録される。

### PHP 8.2 に固定したい場合

PHP 8.2 系だけを許可したい場合は、次のように書く。

```json
"php": "8.2.*"
```

これは、PHP 8.2 系だけを許可する指定。

```txt
8.2.0  OK
8.2.27 OK
8.3.0  NG
8.4.0  NG
```

一方で、今の設定は次のようになっている。

```json
"php": "^8.2"
```

これは、PHP 8.2 以上、9.0 未満を許可する指定。

```txt
8.2.x OK
8.3.x OK
8.4.x OK
9.0.0 NG
```

Laravel 12 は PHP 8.2 以上を前提にしているため、`^8.2` は自然な書き方。

ただし、本番環境や検証環境を PHP 8.2 系にそろえたい場合は、`8.2.*` のように書く。

### PHP のバージョンは少し重く考える

PHP 本体のバージョンは、通常のライブラリ更新よりも影響が大きい。

たとえば、`8.2` から `8.3`、`8.4` に上げる場合は、次のような点が関係する。

- 言語仕様や標準関数の変更
- 非推奨になった書き方
- PHP 拡張の対応状況
- Docker / Sail / 本番サーバーの PHP バージョン
- Composer パッケージ側の対応状況
- テストが通るかどうか

そのため、`composer.json` に

```json
"php": "^8.2"
```

と書けるからといって、すぐに PHP 8.3 や 8.4 へ上げてよい、という意味ではない。

`^8.2` は Composer 上の許可範囲であり、  
実際にどの PHP バージョンで動かすかは、Dockerfile、Sail、CI、本番環境の設定と合わせて決める。

PHP 8.2 系で運用すると決めているなら、`composer.json` でも次のように固定しておくと、環境差に気づきやすくなる。

```json
"php": "8.2.*"
```

### Sail が PHP 8.5 だった場合

実際の Sail コンテナが PHP 8.5 だった場合でも、

```json
"php": "^8.2"
```

という指定なら Composer 上は許可範囲に入る。

`^8.2` は、PHP 8.2 以上、9.0 未満を許可する指定だから。

```txt
8.2.x OK
8.3.x OK
8.4.x OK
8.5.x OK
9.0.0 NG
```

つまり、Sail が PHP 8.5 でも Composer がすぐに止まるとは限らない。

ここで分けて考える必要がある。

- `composer.json` の `"php"` 指定
  - このプロジェクトが許可する PHP バージョンの範囲
- Sail コンテナの PHP バージョン
  - 実際に開発環境で動いている PHP バージョン
- 本番サーバーの PHP バージョン
  - 実際に公開先で動く PHP バージョン

開発環境が PHP 8.5 で、本番環境が PHP 8.2 のようにずれていると、開発中は動いていたコードが本番で動かない可能性がある。

そのため、PHP バージョンは `composer.json` だけでなく、Sail、CI、本番環境も含めてそろえる。

確認するには、Sail 内で次を実行する。

```bash
./vendor/bin/sail php -v
```

本番を PHP 8.2 系で運用するなら、Sail も PHP 8.2 系にそろえるか、少なくとも Composer 側で PHP 8.2 として依存解決する設定を検討する。

```json
"config": {
    "platform": {
        "php": "8.2.0"
    }
}
```

`config.platform.php` は、Composer に対して「この PHP バージョンとして依存関係を解決する」と伝える設定。

ただし、これは実際の PHP 実行バージョンを変えるものではない。  
Sail の PHP を変えるには、Sail の Docker 設定を変更する必要がある。

Composer と npm に共通するバージョン指定や依存要件の確認は、`package.json` まで見た後でまとめて整理する。

---

## 1-6 autoload は名前空間とファイル配置の対応ルール

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Database\\Factories\\": "database/factories/",
        "Database\\Seeders\\": "database/seeders/"
    }
}
```

これは、PHP の **名前空間** と **ファイル配置** の対応を決めている。

たとえば、

```php
App\Models\User
```

というクラスは、

```txt
app/Models/User.php
```

にある、という読み方になる。

この対応は、`App\` という名前空間の下にあるクラスは、`app/` ディレクトリの中から探す、という意味。

### 名前空間とは

名前空間は、同じクラス名が衝突しないようにするための仕組み。

たとえば、同じ `User` というクラス名でも、名前空間が違えば別のクラスとして扱える。

```php
App\Models\User
App\Services\User
```

どちらも最後の名前は `User` だが、  
`App\Models` と `App\Services` という名前空間が違うため、別のクラスとして扱われる。

Laravel では、`app/Models/User.php` のクラスは次のように書かれている。

```php
namespace App\Models;

class User
{
}
```

この `namespace App\Models;` によって、  
このクラスの正式な名前は `App\Models\User` になる。

つまり名前空間は、クラスに付ける住所のようなもの。

Composer の `autoload` は、この名前空間を見て、どのファイルを読み込めばよいかを判断している。

> 名前空間でクラスの正式な名前を決め、  
> autoload でその名前空間とファイルの場所を対応させる。

---

## 1-7 scripts は Composer のショートカット

`scripts` には、Composer コマンドから実行できる処理が書かれている。

このプロジェクトには、たとえば次の script がある。

```json
"test": [
    "@php artisan config:clear --ansi",
    "@php artisan test"
]
```

これは、次のコマンドで実行できる。

```bash
composer test
```

中では、

1. Laravel の config cache をクリアする
2. テストを実行する

という順番で動く。

### setup

```json
"setup": [
    "composer install",
    "@php -r \"file_exists('.env') || copy('.env.example', '.env');\"",
    "@php artisan key:generate",
    "@php artisan migrate --force",
    "npm install",
    "npm run build"
]
```

これは、初期セットアップの流れをまとめたもの。

- PHP パッケージを入れる
- `.env` を作る
- APP_KEY を作る
- migration を実行する
- npm パッケージを入れる
- フロントエンドを build する

このように scripts を見ると、  
**このプロジェクトを動かすための手順** が見えてくる。

---

## 1-8 package.json は JavaScript 側の設計図

`package.json` は、JavaScript / Node.js / Vite 側のパッケージ管理ファイル。

このファイルを見ると、次のことが分かる。

- フロントエンドの実行コマンド
- Vite や Vue などの依存パッケージ
- 開発時に使うパッケージ
- ESM として扱うかどうか

---

## 1-9 scripts は npm の入口

```json
"scripts": {
    "build": "vite build",
    "dev": "vite"
}
```

ここに書かれているものは、`npm run` で実行できる。

```bash
npm run dev
npm run build
```

### dev

```bash
npm run dev
```

開発用の Vite サーバーを起動する。

画面を作っている間は、このコマンドを使う。

### build

```bash
npm run build
```

本番公開用のファイルを作る。

Laravel + Vite では、ビルド結果が `public/build` などに出力される。

---

## 1-10 dependencies と devDependencies

このプロジェクトの `package.json` には、`dependencies` と `devDependencies` がある。

### dependencies

```json
"dependencies": {
    "@inertiajs/vue3": "^2.2.19",
    "vue": "^3.5.25",
    "ziggy-js": "^2.6.0"
}
```

アプリケーション側で使う主要なライブラリ。

- `vue`
  - Vue 本体
- `@inertiajs/vue3`
  - Vue で Inertia を使うためのパッケージ
- `ziggy-js`
  - JavaScript 側で Laravel の route を扱うためのパッケージ

### devDependencies

```json
"devDependencies": {
    "@tailwindcss/vite": "^4.0.0",
    "@vitejs/plugin-vue": "^6.0.2",
    "axios": "^1.11.0",
    "concurrently": "^9.0.1",
    "laravel-vite-plugin": "^2.0.0",
    "laravel-vue-i18n": "^2.8.0",
    "tailwindcss": "^4.0.0",
    "vite": "^7.0.7"
}
```

開発やビルドで使うパッケージ。

- `vite`
  - フロントエンドの開発サーバーとビルドツール
- `@vitejs/plugin-vue`
  - Vite で Vue を扱うための plugin
- `laravel-vite-plugin`
  - Laravel と Vite をつなぐ plugin
- `tailwindcss`
  - CSS フレームワーク
- `@tailwindcss/vite`
  - Tailwind CSS を Vite で使うための plugin
- `axios`
  - HTTP 通信用ライブラリ
- `concurrently`
  - 複数コマンドを同時に動かすためのツール
- `laravel-vue-i18n`
  - Laravel の翻訳ファイルを Vue 側で使うためのパッケージ

---

## 1-11 package.json の type

```json
"type": "module"
```

これは、このプロジェクトの JavaScript を ESM として扱う設定。

ESM では、次のような `import` / `export` を使う。

```js
import { createApp } from 'vue'
```

Laravel + Vite + Vue の構成では、この形がよく使われる。

---

## 1-12 lockfile の役割

`composer.json` と `package.json` は、依存関係の希望を書くファイル。

一方で、lockfile は実際に入ったバージョンを固定するファイル。

### PHP 側

- `composer.json`
  - 入れたいパッケージとバージョン範囲を書く
- `composer.lock`
  - 実際に入ったバージョンを記録する

### JavaScript 側

- `package.json`
  - 入れたいパッケージとバージョン範囲を書く
- `package-lock.json`
  - 実際に入ったバージョンを記録する

lockfile があることで、別の環境でも同じ依存関係を再現しやすくなる。

---

## 1-13 install と update の違い

パッケージ管理で大事なのは、`install` と `update` を分けて考えること。

### Composer

```bash
composer install
```

`composer.lock` に書かれたバージョンを入れる。

```bash
composer update
```

`composer.json` の範囲内で、新しいバージョンを探して `composer.lock` を更新する。

### npm

```bash
npm install
```

`package.json` と `package-lock.json` を見て依存関係を入れる。

```bash
npm ci
```

CI や本番ビルド向け。`package-lock.json` に固定された内容をそのまま入れる。

```bash
npm update
```

`package.json` の範囲内で、新しいバージョンに更新する。

---

## 1-14 今回の .npmrc

このプロジェクトでは、`anglers/.npmrc` に次の設定を入れている。

```ini
min-release-age=7
```

これは、公開直後の新しいパッケージを避けるための設定。

公開直後のパッケージには、次のようなリスクがある。

- 不具合がまだ見つかっていない
- 乗っ取られたパッケージの影響をすぐ受ける可能性がある
- 悪意あるパッケージが混入していても、発覚前に入れてしまう可能性がある

そのため、公開から少し時間を置いたバージョンを使うことで、  
サプライチェーン攻撃や公開直後の事故を踏みにくくする。

---

## 1-15 Composer と npm を見た後に確認すること

ここまでで、PHP 側の `composer.json` と JavaScript 側の `package.json` を見た。

最後に、両方に共通するバージョン指定と、依存要件の確認を整理する。

### `^` は package.json でも composer.json でも出てくる

`^` は、Composer でも npm でもよく使われるバージョン指定。

どちらも基本的には、

> 互換性がある範囲で、新しいバージョンを許可する

という意味で読める。

たとえば、

```json
"laravel/framework": "^12.0"
```

は Laravel 12 系の範囲で更新を許可する。

```json
"axios": "^1.11.0"
```

は axios 1 系の範囲で更新を許可する。

つまり `^` が付いている場合、`composer install` や `npm install` のたびに無制限に最新版が入るわけではない。

ただし、lockfile を更新する `composer update` や `npm update` を実行すると、その範囲内で新しいバージョンに変わる可能性がある。

細かい判定ルールは Composer と npm で異なる部分もあるが、最初は

> メジャーバージョンをまたがない範囲で更新されることが多い

と理解しておく。

### よく見るバージョン指定

Composer や npm では、`^` 以外にもいくつかの書き方がある。

| 書き方 | 意味 | 例 |
| --- | --- | --- |
| `1.2.3` | 完全固定 | `1.2.3` だけ |
| `1.2.*` | パッチだけ許可 | `1.2.0` から `1.2.x` |
| `^1.2.3` | 互換性のある範囲で許可 | 多くの場合 `1.x` 系 |
| `~1.2.3` | より狭い範囲で許可 | 多くの場合 `1.2.x` 系 |
| `>=1.2.0` | 指定以上を許可 | `1.2.0` 以上 |
| `>=1.2.0 <2.0.0` | 範囲指定 | `1.2.0` 以上、`2.0.0` 未満 |
| `*` | 何でも許可 | 制限なし |

### 固定と範囲指定の違い

完全に固定する場合は、バージョンをそのまま書く。

```json
"axios": "1.11.0"
```

これは `1.11.0` だけを許可する。

範囲で許可する場合は、`^` や `~` を使う。

```json
"axios": "^1.11.0"
```

これは、互換性がある範囲で新しいバージョンを許可する。

ただし、どちらの場合でも、実際にインストールされたバージョンは lockfile に記録される。

### Composer は依存要件を確認してくれる

Composer は、パッケージをインストールするときに依存要件を確認してくれる。

たとえば、PHP 8.2 の環境で、あるパッケージが PHP 8.3 以上を要求している場合、`composer install` や `composer update` の時点で止まる。

エラーは次のような内容になる。

```txt
package/example requires php ^8.3 -> your php version (8.2.x) does not satisfy that requirement.
```

Composer は、主に次のようなものを見ている。

- 今動いている PHP のバージョン
- `composer.json` の `"php"` 指定
- 各パッケージが要求している PHP バージョン
- 必要な PHP 拡張
- 各パッケージ同士の依存関係

つまり、PHP バージョンや必要な拡張が合わない場合は、Composer が教えてくれる。

### check-platform-reqs で確認する

現在の環境が、lockfile に記録されたパッケージの実行要件を満たしているか確認するには、次のコマンドを使う。

```bash
composer check-platform-reqs
```

Laravel Sail を使っている場合は、Sail コンテナ内の PHP で確認する。

```bash
./vendor/bin/sail composer check-platform-reqs
```

ここで大事なのは、Composer が見るのは **Composer を実行している環境の PHP** だということ。

- ホストで Composer を実行する
  - ホストの PHP バージョンを見る
- Sail 内で Composer を実行する
  - Sail コンテナ内の PHP バージョンを見る

Laravel Sail を前提にしているプロジェクトでは、基本的に Sail 内で確認した方が実行環境に近い。

```bash
./vendor/bin/sail composer install
./vendor/bin/sail composer check-platform-reqs
```

### npm 側との違い

npm にも Node.js のバージョン要件を見る仕組みはある。

たとえば `package.json` に `engines` が書かれている場合、Node.js のバージョンが合わないと警告が出る。

```json
"engines": {
    "node": ">=20"
}
```

ただし npm は、設定によっては警告だけで止まらないことがある。

Composer の PHP バージョン確認や拡張確認の方が、Laravel 開発では強く意識しやすい。

---

## 1-16 どこから読むか

初めてプロジェクトを見るときは、次の順番で読むと分かりやすい。

### PHP 側

1. `composer.json` の `require`
2. `composer.json` の `require-dev`
3. `composer.json` の `scripts`
4. `composer.lock`

### JavaScript 側

1. `package.json` の `scripts`
2. `package.json` の `dependencies`
3. `package.json` の `devDependencies`
4. `.npmrc`
5. `package-lock.json`

最初から lockfile を細かく読む必要はない。

まずは `composer.json` と `package.json` で全体像をつかみ、  
実際に固定されているバージョンを確認したいときに lockfile を見る。

---

## 1-17 この章で押さえること

- `composer.json` は PHP / Laravel 側の依存関係を見るファイル
- `package.json` は JavaScript / Vite 側の依存関係を見るファイル
- `require` は実行時に必要な PHP パッケージ
- `require-dev` は開発時に必要な PHP パッケージ
- `dependencies` はアプリ側で使う JavaScript パッケージ
- `devDependencies` は開発やビルドで使う JavaScript パッケージ
- lockfile は実際に入るバージョンを固定する
- `.npmrc` では npm の動作をプロジェクト単位で設定できる

---

## 第1章のまとめ

> `composer.json` と `package.json` は、  
> **このプロジェクトが何に依存しているかを見るための入口**。

パッケージ管理では、単にインストールするだけでなく、

- 何が入っているか
- どこまで更新される可能性があるか
- 本番で必要なものか、開発だけで必要なものか
- lockfile で何が固定されているか
- 公開直後のパッケージを避ける設定があるか

を読むことが重要になる。

この読み方が分かると、Dependabot の通知や脆弱性対応も、  
ただの警告ではなく **依存関係の変更として判断** できるようになる。
