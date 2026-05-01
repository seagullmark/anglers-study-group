# 第3章：Laravel アプリをアップデートするときの流れ

## 3-1 この章でやること

一度動いた Laravel アプリを、あとから安全に更新する流れを整理する。

アプリの公開・更新にはいくつかやり方がある。

```txt
ローカルでビルドしたものをサーバーに置く
公開サーバー上で pull / install / build する
CodePipeline などの CI/CD を使う
```

今回の勉強会では、公開サーバー上で手順を確認しながら更新する流れを扱う。

理由は、ビルドやキャッシュ、依存パッケージ、nginx、`.env` の関係が見えやすいから。

ここを理解しておくと、CodePipeline などの CI/CD も組みやすくなる。

CI/CD は特別な魔法ではなく、手作業で行っている次の流れを自動化するもの。

```txt
コードを取得する
依存パッケージを入れる
Ziggy を生成する
ビルドする
キャッシュを整理する
公開先に反映する
動作確認する
```

手作業の意味が分かっていないと、CI/CD で失敗したときにどこを見ればよいか分からなくなる。

今回見るもの。

```txt
git pull
composer install
npm install
Ziggy 再生成
npm run build
Laravel キャッシュクリア
メンテナンスモード
Dependabot
npm audit / composer audit
失敗したときの戻し方
```

---

## 3-2 まず現在の状態を確認する

アップデート前に、今どのブランチで、差分があるか確認する。

ここで悩みになるのが、どのブランチを公開対象にするか。

`main` をそのまま公開対象にすると、流れは分かりやすい。
しかし、万が一不具合があったときに、すぐ戻しにくい。

この不安は自然なもの。
公開するものと開発の基準をどう分けるかは、運用でかなり重要になる。

考え方の例。

```txt
main
deploy/staging
```

`main` は開発の基準にする。
`deploy/staging` は公開確認用にする。

流れ。

```txt
main で開発する
deploy/staging に反映する
公開環境では deploy/staging を pull する
動作確認する
安定したら main に戻す、または production 用に反映する
```

今回のように、しばらく動かしてから安定判断したい場合は、`main` を直接公開するより、公開確認用のブランチを分けた方が戻しやすい。

```bash
git status
git branch --show-current
```

差分がある状態で `git pull` すると、更新と自分の変更が混ざる。

確認すること。

```txt
今のブランチ
未コミットの変更
pull してよい状態か
```

今回の更新作業を `main` ブランチで行う場合。

```txt
main
```

実際の公開更新では、どのブランチを公開対象にするかを決めておく。
不安がある場合は、`deploy/staging` のような公開確認用ブランチを使う。

---

## 3-3 最新コードを取り込む

```bash
git pull
```

ここで見ること。

```txt
どのファイルが変わったか
composer.lock が変わったか
package-lock.json が変わったか
resources/js/ziggy.js が混ざっていないか
```

`resources/js/ziggy.js` は環境ごとの URL が入るため、git 管理しない方針にする。

```txt
/resources/js/ziggy.js
```

---

## 3-4 PHP 側の依存を更新する

ここで混同しやすいのが、`install` と `update` の違い。

基本は、まず開発環境で動作確認する。
動いたら、その状態を Pull Request として確認し、取り込む側では `composer update` ではなく `composer install` を使う。

### `composer install`

```bash
composer install
```

`composer.lock` に書かれているバージョンを入れる。

つまり、Pull Request で確認済みの依存バージョンを再現する。

### `composer update`

```bash
composer update
```

`composer.json` の範囲内で、新しい依存バージョンを探して `composer.lock` を更新する。

つまり、依存パッケージのバージョンを動かす操作。

### Composer 更新確認の考え方

```txt
依存を上げる人:
composer update
動作確認
composer.lock を含めて Pull Request

取り込む人:
git pull
composer install
動作確認
```

`composer.lock` が変わった場合は、PHP 側の依存を入れ直す。

```bash
composer install
```

本番寄りにする場合。

```bash
composer install --no-dev --optimize-autoloader
```

オプションの意味。

```txt
--no-dev
require-dev のパッケージを入れない

--optimize-autoloader
クラスの読み込み情報を最適化する
```

`--no-dev` を付けると、テスト用や開発補助用のパッケージは入らない。

例。

```txt
phpunit
laravel/pint
mockery
faker
```

公開サーバーで実行に不要なものを減らせる。

`--optimize-autoloader` は、Composer の autoload を本番向けに最適化する。

Laravel は多くのクラスを読み込むため、公開環境では付けることが多い。

PHP や拡張の条件も確認する。

```bash
composer check-platform-reqs
```

`composer check-platform-reqs` は、今の実行環境が `composer.lock` の要求を満たしているか確認するコマンド。

ここで見るのは、パッケージの更新ではない。

見る対象。

```txt
PHP のバージョン
PHP 拡張
Composer runtime
```

例。

```txt
php          8.4.11 success
ext-dom      20031129 success
ext-mbstring * success
ext-xml      8.4.11 success
```

`success` なら、その条件は満たしている。

足りない場合は `failed` になる。

例。

```txt
ext-curl missing
```

その場合は、必要な PHP 拡張を入れる。

```bash
sudo apt install php-curl
```

この確認をしておくと、`composer install` は通ったが実行時に PHP 拡張が足りない、という事故を避けやすい。

見ること。

```txt
php が success か
ext-* が success か
FileMaker 関連パッケージが壊れていないか
```

---

## 3-5 JavaScript 側の依存を更新する

JavaScript 側も同じ。

理由を確認せずに `npm update` しない。

### `npm install`

```bash
npm install
```

`package-lock.json` を見ながら依存を入れる。

Pull Request で確認済みの依存関係を再現する。

### `npm update`

```bash
npm update
```

`package.json` の範囲内で、依存パッケージを更新する。

つまり、`package-lock.json` を動かす操作。

### `npm ci`

```bash
npm ci
```

`package-lock.json` を厳密に使って、クリーンに入れ直す。
CI や本番ビルドではこちらが向いている。

`npm install` と `npm ci` は、どちらも `node_modules` を作る点では近い。
ただし目的が違う。

```txt
npm install
開発中や確認中に使うことが多い
package-lock.json が更新されることがある

npm ci
CI/CD や本番ビルドで使いやすい
package-lock.json をそのまま再現する
package.json と package-lock.json がズレていると失敗する
既存の node_modules を消して入れ直す
package-lock.json は更新しない
```

今回のように勉強用環境で進める場合は、まず `npm install` で流れを確認する。

### npm 更新確認の考え方

```txt
依存を上げる人:
npm update または package.json の変更
npm install
動作確認
package-lock.json を含めて Pull Request

取り込む人:
git pull
npm install
動作確認
```

`package-lock.json` が変わった場合は、npm 側の依存を入れ直す。

---

## 3-6 Ziggy を再生成する

これはすべての Laravel アプリに必要な作業ではない。

このアプリではフロント側で `route()` を使う。
そのため、Ziggy のルート定義を JavaScript 側に渡している。

Ziggy の生成ファイルには `APP_URL` が入る。

```txt
resources/js/ziggy.js
```

そのため、環境を変えたら build 前に再生成する。

```bash
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
```

確認する。

```bash
grep -n "url\\|port" resources/js/ziggy.js
```

`localhost:8000` のような古い URL が残っていると、ログイン送信先が別の環境に向く。

---

## 3-7 フロントをビルドする

```bash
npm run build
```

確認する。

```bash
ls -la public/build
```

期待するもの。

```txt
assets/
manifest.json
```

---

## 3-8 `public/hot` を確認する

通常は `public/hot` を強く意識しなくてもよい。

`npm run dev` を正常に止めれば、通常は `public/hot` も消える。

今回つまづいたのは、初回ビルド時に `npm run dev` していた環境の `public` をそのまま配置したため。

`public/hot` が残っていると、Laravel はビルド済みの `public/build` ではなく Vite の開発サーバーを見に行く。

確認する。

```bash
test -f public/hot && cat public/hot || echo "public/hot なし"
```

残っていたら削除する。

```bash
rm public/hot
```

今回出たエラー。

```txt
GET http://localhost:5173/@vite/client net::ERR_CONNECTION_RESET
GET http://localhost:5173/resources/js/app.js net::ERR_CONNECTION_RESET
```

これは Vite dev server を見に行って失敗している状態。

---

## 3-9 `.env` を公開確認用にする

公開確認では `local` のままにしない。

今回の確認環境ではこうする。

```env
APP_ENV=staging
APP_DEBUG=false
APP_URL=http://192.168.139.209
```

確認する。

```bash
grep -n "APP_ENV\\|APP_DEBUG\\|APP_URL\\|VITE\\|SESSION_DOMAIN\\|SANCTUM_STATEFUL_DOMAINS" .env
```

`.env` を変えたらキャッシュを消す。

```bash
php artisan optimize:clear
```

---

## 3-10 Laravel のキャッシュを整理する

まず安全に全部消す。

```bash
php artisan optimize:clear
```

必要に応じて作る。

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

ただし、まずは `optimize:clear` で動作確認する。

理由。

```txt
古い .env
古い route
古い config
古い view
```

これらが残っていると、直したはずの設定が反映されない。

---

## 3-11 メンテナンスモード

更新中に利用者へ触らせたくない場合は、メンテナンスモードにする。

Laravel 公式。

```txt
https://laravel.com/docs/13.x/configuration#maintenance-mode
```

```bash
php artisan down
```

メンテナンスモード中でも、自分だけ確認したい場合は secret を使う。

```bash
php artisan down --secret="deploy-preview"
```

この状態で次の URL にアクセスすると、メンテナンスモードをバイパスする Cookie がブラウザに発行される。

```txt
http://192.168.139.209/deploy-preview
```

以後、そのブラウザではメンテナンスモード中でもアプリを確認できる。

secret を Laravel に生成させることもできる。

```bash
php artisan down --with-secret
```

ここでいう secret は、ブラウザのシークレットウィンドウのことではない。
メンテナンスモードを一時的に通過するための秘密URL。

更新後に戻す。

```bash
php artisan up
```

流れ。

```bash
php artisan down --secret="deploy-preview"
git pull
composer install --no-dev --optimize-autoloader
npm install
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
php artisan up
```

更新中は通常アクセスにはメンテナンス画面を出す。
確認者は secret URL から入って動作確認する。

---

## 3-12 nginx / PHP-FPM を確認する

nginx 設定を確認する。

```bash
sudo nginx -t
```

反映する。

```bash
sudo systemctl reload nginx
```

PHP-FPM を確認する。

```bash
systemctl status php8.4-fpm
```

nginx の Laravel 設定は公式をベースにする。

```txt
https://laravel.com/docs/13.x/deployment#nginx
```

---

## 3-13 ブラウザで確認する

確認する URL。

```txt
http://192.168.139.209/login
```

見ること。

```txt
ページが表示される
CSS / JS が読み込まれる
ログイン POST が同じホストに飛ぶ
localhost:5173 を見に行っていない
localhost:8000 を見に行っていない
```

開発者ツールで見るところ。

```txt
Network
Console
```

---

## 3-14 Dependabot で依存パッケージの更新を確認する

Dependabot は「ディペンダボット」と読む。

作成したファイル。

```txt
.github/dependabot.yml
```

内容。

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 5
    groups:
      npm-dependencies:
        patterns:
          - "*"

  - package-ecosystem: "composer"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:30"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 5
    groups:
      composer-dependencies:
        patterns:
          - "*"
```

見る対象。

```txt
package.json
package-lock.json
composer.json
composer.lock
```

Dependabot は更新 PR を作る。
安全確認は開発者が行う。

メジャーアップデートの PR を出したくない場合は、`ignore` を使う。

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 5
    ignore:
      - dependency-name: "*"
        update-types:
          - "version-update:semver-major"
    groups:
      npm-dependencies:
        patterns:
          - "*"

  - package-ecosystem: "composer"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:30"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 5
    ignore:
      - dependency-name: "*"
        update-types:
          - "version-update:semver-major"
    groups:
      composer-dependencies:
        patterns:
          - "*"
```

この設定では、すべての依存パッケージについてメジャーアップデートを無視する。

```txt
version-update:semver-major
```

メジャーアップデートは破壊的変更を含むことがある。
そのため、通常の定期更新では避け、必要なときに個別に確認する。

---

## 3-15 Dependabot PR が来たら見ること

Dependabot PR を `git pull` しただけでは、実行環境の `vendor/` や `node_modules/` は更新されない。

PR に含まれる lockfile の内容を、実際の実行環境へ反映する必要がある。

Composer の PR なら、主に次を見る。

```txt
composer.json
composer.lock
```

その内容を `vendor/` に反映するために実行する。

```bash
composer install
```

npm の PR なら、主に次を見る。

```txt
package.json
package-lock.json
```

その内容を `node_modules/` に反映するために実行する。

```bash
npm install
```

再現性を強く見たい場合は `npm ci` も選択肢になる。

```bash
npm ci
```

ここで使うのは `update` ではなく `install`。
`update` は依存バージョンを動かす操作で、Dependabot PR の確認側が何も考えずに実行するものではない。

PR をいきなり `main` に入れない。

確認用のブランチを作り、そこに PR の内容を取り込んで開発環境で確認する。

流れ。

```txt
main から確認用ブランチを作る
Dependabot PR の内容を確認用ブランチに取り込む
composer install / npm install を実行する
build する
開発環境で動作確認する
問題なければ main にマージする
```

例。

```bash
git switch main
git pull
git switch -c verify/dependabot-update
```

この確認用ブランチで PR の内容を取り込み、動作確認する。

確認が通ったら、`main` にマージする。

見ること。

```txt
どのパッケージが上がったか
メジャーバージョンが上がっていないか
lockfile がどう変わったか
ビルドが通るか
ログインできるか
```

確認コマンド。

```bash
composer install
composer check-platform-reqs
npm install
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
```

PR を見てすぐマージしない。
更新内容と動作を確認してからマージする。

---

## 3-16 npm audit を見る

`npm audit` は、npm パッケージに既知の脆弱性がないか確認するコマンド。

`audit` は「オーディット」と読む。
監査、確認という意味。

プロジェクトの依存関係を見て、npm の脆弱性データベースと照合する。

対象になる主なファイル。

```txt
package.json
package-lock.json
```

見ているもの。

```txt
直接入れているパッケージ
そのパッケージが依存しているパッケージ
脆弱性の深刻度
修正可能かどうか
```

今回 `npm install` のあとに次の表示が出た。

```txt
6 vulnerabilities (3 moderate, 3 high)
```

`npm install` のあとに出るのは、npm が依存パッケージを入れたあとに、既知の脆弱性情報も確認しているため。

つまり、インストール結果の最後に簡易的な audit 結果が表示される。

詳細を見たい場合に `npm audit` を実行する。

`npm audit` は単体でも実行できる。

```bash
npm audit
```

`npm install` は依存パッケージのインストール後に、簡易的な audit 結果を表示する。
詳細を確認したい場合は、あらためて `npm audit` を実行する。

読み方。

```txt
6 vulnerabilities
脆弱性として検出された項目が 6 件ある

3 moderate
中程度の深刻度が 3 件ある

3 high
高い深刻度が 3 件ある
```

これは、必ずしも「直接使っているパッケージが 6 個危険」という意味ではない。

依存パッケージのさらに先の依存で検出されることもある。

まず詳細を見る。

詳細を見る。

```bash
npm audit
```

自動修正を試すコマンド。

```bash
npm audit fix
```

`npm audit fix` は、脆弱性を解消できる範囲で `package-lock.json` や `node_modules` を更新する。

基本的には、今の `package.json` のバージョン範囲内で直せる更新を試す。

例。

```txt
"axios": "^1.6.0"
```

この場合、`1.x` の範囲で修正版があれば、そこへ更新されることがある。

`npm audit fix` で変わる可能性があるもの。

```txt
package-lock.json
node_modules/
```

`package.json` は、通常は大きく変えない。

ただし、解消できない脆弱性が残ることもある。

その場合に案内されることがあるのが次のコマンド。

```bash
npm audit fix --force
```

`--force` は、メジャーバージョンアップを含む変更になることがある。
そのため、画面やビルドが壊れる可能性がある。

ただし、すぐに実行して終わりではない。

見ること。

```txt
どのパッケージか
直接入れているパッケージか
間接依存か
メジャーバージョンが変わるか
build が通るか
画面が壊れないか
```

`npm audit fix --force` はメジャーアップデートを含むことがあるため、慎重に扱う。

---

## 3-17 composer audit を見る

Composer 側の脆弱性確認。

```bash
composer audit
```

見ること。

```txt
対象パッケージ
影響するバージョン
修正可能なバージョン
Laravel 本体か
FileMaker 関連か
```

Composer 側も、更新したら確認する。

```bash
composer install
composer check-platform-reqs
php artisan optimize:clear
```

---

## 3-18 npm パッケージのリスク

npm パッケージは、便利だが攻撃対象にもなる。

例。

```txt
乗っ取られたパッケージ
悪意ある postinstall
依存先のさらに依存先の脆弱性
急なメジャーアップデート
```

今回 `.npmrc` にはこれを入れている。

```txt
min-release-age=7
```

新しく公開された直後のパッケージをすぐ入れないための設定。

これで完全に安全になるわけではない。
ただし、公開直後の危険な更新を少し避けやすくなる。

---

## 3-19 失敗したときの戻し方

アップデート後に壊れたら、まず状態を確認する。

```bash
git status
git log --oneline -5
```

直前の commit に戻す判断をする場合。

```bash
git switch main
git pull
```

依存を戻す場合は lockfile を基準にする。

```bash
composer install
npm install
```

ビルドを作り直す。

```bash
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
```

nginx 設定を変えた場合は、必ず確認してから反映する。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 3-20 アップデート時の最短チェックリスト

```bash
git status
git pull
composer install
composer check-platform-reqs
npm install
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
test -f public/hot && cat public/hot || echo "public/hot なし"
sudo nginx -t
sudo systemctl reload nginx
```

ブラウザで確認する。

```txt
http://192.168.139.209/login
```

見ること。

```txt
localhost:5173 を読みに行っていない
localhost:8000 に POST していない
public/build/manifest.json がある
ログインできる
```

---

## 3-21 この章のポイント

- アップデートは `git pull` だけでは終わらない。
- PHP 側は Composer を確認する。
- JavaScript 側は npm と build を確認する。
- Ziggy は環境ごとに再生成する。
- `public/hot` が残ると Vite dev server を見に行く。
- `.env` は公開確認用にする。
- Dependabot PR は確認してからマージする。
- audit は警告を読むところから始める。
- 壊れたときに戻せるように、更新前の状態を確認しておく。
