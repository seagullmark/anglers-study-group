# 第2章：OrbStack Ubuntu で Laravel を実行する

## 目的

OrbStack の Ubuntu に、Laravel を実行するための環境を作る。

公開ディレクトリはここにする。

```txt
/var/www/anglers/public
```

---

## 2-1 Ubuntu / nginx / PHP-FPM を準備する

### 1. OrbStack の Ubuntu に入った

```bash
orb
```

Ubuntu のバージョンを確認した。

```bash
cat /etc/os-release
```

確認した環境。

```txt
Ubuntu 25.10 (Questing Quokka)
```

### 2. apt のパッケージ一覧を更新した

```bash
sudo apt update
```

### 3. nginx をインストールした

```bash
sudo apt install nginx
```

nginx のバージョンを確認した。

```bash
nginx -v
```

確認したバージョン。

```txt
nginx version: nginx/1.28.0 (Ubuntu)
```

nginx が起動していることを確認した。

```bash
systemctl status nginx
```

確認した状態。

```txt
Active: active (running)
```

HTTP 応答を確認した。

```bash
curl -I http://localhost
```

確認した応答。

```txt
HTTP/1.1 200 OK
```

### 4. PHP と PHP-FPM をインストールした

```bash
sudo apt install php php-cli php-fpm
```

PHP のバージョンを確認した。

```bash
php -v
```

確認したバージョン。

```txt
PHP 8.4.11 (cli)
```

PHP-FPM が起動していることを確認した。

```bash
systemctl status php8.4-fpm
```

確認した状態。

```txt
Active: active (running)
```

PHP-FPM の socket を確認した。

```bash
ls -la /run/php
```

確認した socket。

```txt
/run/php/php8.4-fpm.sock
```

### 5. nginx から PHP-FPM に渡す設定を書いた

編集したファイル。

```txt
/etc/nginx/sites-available/default
```

PHP を実行するための設定。

```nginx
location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
}
```

nginx の設定を確認した。

```bash
sudo nginx -t
```

確認した結果。

```txt
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

nginx に設定を反映した。

```bash
sudo systemctl reload nginx
```

### 6. Laravel 用の公開ディレクトリを作った

```bash
sudo mkdir -p /var/www/anglers/public
```

作業ユーザーで扱えるように所有者を変更した。

```bash
sudo chown -R $USER:$USER /var/www/anglers
```

作成した場所を確認した。

```bash
ls -la /var/www/anglers
```

公開ディレクトリ。

```txt
/var/www/anglers/public
```

### 7. nginx の公開先を Laravel の public にした

編集したファイル。

```txt
/etc/nginx/sites-available/default
```

Laravel 公式の Deployment にある nginx 設定をベースにする。

```txt
https://laravel.com/docs/13.x/deployment#nginx
```

この環境では `root` と `fastcgi_pass` を合わせる。

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /var/www/anglers/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

nginx の設定を確認した。

```bash
sudo nginx -t
```

nginx に設定を反映した。

```bash
sudo systemctl reload nginx
```

### 8. 公開先が切り替わったことを確認した

```bash
curl http://localhost
```

確認した表示。

```html
<h1>anglers public</h1>
```

---

## 2-2 Laravel 実行前の状態

- nginx は入っている。
- PHP 8.4 は入っている。
- PHP-FPM は動いている。
- nginx は PHP-FPM に PHP 実行を渡せる。
- nginx の公開先は `/var/www/anglers/public`。
- `/var/www/anglers/public` への HTTP 到達は確認済み。

---

## 2-3 Laravel 本体を配置して実行できる状態にする

### 1. PHP 拡張を確認する

Laravel は PHP 本体だけでは動かない。
必要な PHP 拡張が入っているか確認する。

```bash
php -m
```

確認結果。

```txt
calendar
Core
ctype
date
exif
FFI
fileinfo
filter
ftp
gettext
hash
iconv
json
libxml
openssl
pcntl
pcre
PDO
Phar
posix
random
readline
Reflection
session
shmop
sockets
sodium
SPL
standard
sysvmsg
sysvsem
sysvshm
tokenizer
Zend OPcache
zlib
```

Laravel で確認するもの。

```txt
ctype
curl
dom
fileinfo
filter
mbstring
openssl
pcre
pdo
session
tokenizer
xml
```

今回の確認結果。

```txt
入っている:
ctype
fileinfo
filter
openssl
pcre
pdo
session
tokenizer

足りない:
curl
dom
mbstring
xml
```

足りない拡張を入れる。

```bash
sudo apt install php-curl php-mbstring php-xml
```

再確認した。

```bash
php -m
```

確認結果。

```txt
curl
dom
mbstring
SimpleXML
xml
xmlreader
xmlwriter
xsl
```

Laravel 実行に必要な基本拡張はそろった。

Composer が使える状態なら、`composer.json` が要求している PHP と拡張を確認する。

```bash
composer check-platform-reqs
```

足りない拡張があれば、表示された `ext-...` を見て追加する。

例。

```bash
sudo apt install php-mbstring php-xml php-curl php-zip
```

### 2. Laravel に必要なファイルを移動する

移動元。

```txt
/Users/seagull_macmini4/dockerenv/anglers/
```

移動先。

```txt
/var/www/anglers/
```

必要なもの。

```txt
app/
bootstrap/
config/
database/
public/
resources/
routes/
storage/
artisan
composer.json
composer.lock
.env
```

コピーしないもの。

```txt
vendor/
node_modules/
```

コピー例。

```bash
rsync -av \
  --exclude='vendor' \
  --exclude='node_modules' \
  /Users/seagull_macmini4/dockerenv/anglers/ \
  /var/www/anglers/
```

コピー後に確認する。

```bash
ls -la /var/www/anglers
ls -la /var/www/anglers/public
```

移動元を確認した。

```bash
ls -la /Users/seagull_macmini4/dockerenv/anglers
```

確認できた主なファイル。

```txt
.env
.env.example
.npmrc
app/
artisan
bootstrap/
composer.json
composer.lock
config/
database/
lang/
package-lock.json
package.json
public/
resources/
routes/
storage/
vendor/
node_modules/
```

`vendor/`、`node_modules/` は移動対象から外す。

`.git/` は移動する。
この Ubuntu 上で `git pull` してから `build` するため。

`rsync` が入っていなかった。

```bash
rsync -av \
  --exclude='vendor' \
  --exclude='node_modules' \
  /Users/seagull_macmini4/dockerenv/anglers/ \
  /var/www/anglers/
```

結果。

```txt
-bash: rsync: command not found
```

`rsync` を入れる。

```bash
sudo apt install rsync
```

`rsync` でコピーした。

```bash
rsync -av \
  --exclude='vendor' \
  --exclude='node_modules' \
  /Users/seagull_macmini4/dockerenv/anglers/ \
  /var/www/anglers/
```

`.git/` はコピー対象に含めた。

コピー先を確認した。

```bash
ls -la /var/www/anglers
ls -la /var/www/anglers/public
```

必要なファイルがコピーされていることを確認した。

### 3. Composer を確認する

Laravel プロジェクトに移動した。

```bash
cd /var/www/anglers
```

Composer を確認した。

```bash
composer -V
```

結果。

```txt
-bash: composer: command not found
```

Composer を入れる。

```bash
sudo apt install composer
```

Composer をインストールした。

Composer の要求を確認した。

```bash
cd /var/www/anglers
composer check-platform-reqs
```

結果。

```txt
No vendor dir present, checking platform requirements from the lock file
composer-runtime-api 2.2.2 success
ext-ctype            *      success
ext-dom              20031129 success
ext-fileinfo         8.4.11 success
ext-filter           8.4.11 success
ext-hash             8.4.11 success
ext-iconv            8.4.11 success
ext-json             8.4.11 success
ext-libxml           8.4.11 success
ext-mbstring         *      success
ext-openssl          8.4.11 success
ext-pcre             8.4.11 success
ext-phar             8.4.11 success
ext-session          8.4.11 success
ext-tokenizer        8.4.11 success
ext-xml              8.4.11 success
ext-xmlwriter        8.4.11 success
lib-pcre             10.46  success
php                  8.4.11 success
```

PHP と拡張の条件は満たしている。

### 4. Composer で vendor を作る

```bash
composer install
```

`vendor/` が作成された。

### 5. `.env` の APP_KEY を確認する

```bash
grep -n "^APP_KEY=" .env
```

結果。

```txt
3:APP_KEY=base64:...
```

`APP_KEY` は設定済み。

### 6. Laravel のキャッシュをクリアする

```bash
php artisan optimize:clear
```

Laravel のキャッシュをクリアした。

### 7. Laravel が書き込むディレクトリを確認する

```bash
ls -ld storage bootstrap/cache
```

結果。

```txt
drwxr-xr-x 1 seagull_macmini4 seagull_macmini4 20 May  1 18:15 bootstrap/cache
drwxr-xr-x 1 seagull_macmini4 seagull_macmini4 50 Apr 10 17:03 storage
```

`storage/` と `bootstrap/cache/` は Laravel が書き込む場所。

所有者を `www-data` に変更した。

```bash
sudo chown -R www-data:www-data storage bootstrap/cache
```

再確認した。

```bash
ls -ld storage bootstrap/cache
```

結果。

```txt
drwxr-xr-x 1 www-data www-data 20 May  1 18:15 bootstrap/cache
drwxr-xr-x 1 www-data www-data 50 Apr 10 17:03 storage
```

### 8. storage のシンボリックリンクを作る

Laravel で `storage/app/public` のファイルを Web から見せる場合は、`public/storage` のシンボリックリンクを作る。

```bash
php artisan storage:link
```

この時点で、`storage/` と `bootstrap/cache/` を `www-data` 所有にしていたため、シェルユーザーで `php artisan` を実行すると書き込めなかった。

対応として、作業ユーザーと `www-data` の両方が書き込めるようにする。

```bash
sudo chown -R $USER:www-data storage bootstrap/cache
sudo chmod -R ug+rw storage bootstrap/cache
```

確認する。

```bash
ls -la public | grep storage
```

作られるリンク。

```txt
public/storage -> ../storage/app/public
```

`storage:link` を実行し、`public/storage` のリンクを作成した。

---

## 2-4 Node/npm と Vite build を行う

### 1. Node/npm を入れる

Node.js と npm を確認した。

```bash
node -v
npm -v
```

どちらも入っていなかった。

Node.js と npm を入れる。

```bash
sudo apt install nodejs npm
```

インストール後に確認する。

```bash
node -v
npm -v
```

確認したバージョン。

```txt
v20.19.4
9.2.0
```

### 2. `npm install`

```bash
npm install
```

結果。

```txt
added 121 packages, and audited 122 packages in 2s
31 packages are looking for funding
6 vulnerabilities (3 moderate, 3 high)
```

### 3. `npm run build`

```bash
npm run build
```

ビルドした。

ビルド成果物を確認した。

```bash
ls -la public/build
```

結果。

```txt
assets/
manifest.json
```

### 4. Laravel に HTTP で到達するか確認する

ブラウザで確認した。

```txt
http://192.168.139.209/login
```

`/login` に飛ばされたが、404 になった。

この場合は、nginx の `location /` が Laravel の `index.php` に渡しているか確認する。

確認した。

```bash
grep -n "root\|try_files\|fastcgi_pass" /etc/nginx/sites-available/default
```

結果。

```txt
41:     root /var/www/anglers/public;
51:             try_files $uri $uri/ =404;
57:                fastcgi_pass unix:/run/php/php8.4-fpm.sock;
```

`try_files` が `=404` になっていた。
この状態だと `/login` のような Laravel のルートが nginx で 404 になる。

Laravel に渡すため、次に修正する。

```nginx
try_files $uri $uri/ /index.php?$query_string;
```

あわせて Laravel 公式の nginx 設定に寄せる。

```txt
https://laravel.com/docs/13.x/deployment#nginx
```

```nginx
location ~ ^/index\.php(/|$) {
    fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    include fastcgi_params;
    fastcgi_hide_header X-Powered-By;
}
```

`.env` の `APP_URL` も確認した。

```env
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000
APP_PORT=8000
```

今回はビルド後の公開確認なので、`local` のままにしない。
本番ではなく検証用なら `staging` として扱う。

```env
APP_ENV=staging
APP_DEBUG=false
```

今回は nginx の 80 番で確認しているため、`APP_URL` も実際にブラウザで開く URL に合わせる。

```env
APP_URL=http://192.168.139.209
```

変更後は Laravel のキャッシュをクリアする。

```bash
php artisan optimize:clear
```

ブラウザで `/login` を開いたところ、Vite の開発サーバーを読みに行っていた。

```txt
GET http://localhost:5173/@vite/client net::ERR_CONNECTION_RESET
GET http://localhost:5173/resources/js/app.js net::ERR_CONNECTION_RESET
```

原因は `public/hot` が残っていたこと。
`public/hot` があると、Laravel はビルド済みの `public/build` ではなく Vite dev server を見に行く。

確認する。

```bash
ls -la public/hot
cat public/hot
```

ビルド済みのファイルを使うため、`public/hot` を削除する。

```bash
rm public/hot
```

削除後に Laravel のキャッシュをクリアする。

```bash
php artisan optimize:clear
```

ログイン送信時に CORS エラーが出た。

```txt
Access to XMLHttpRequest at 'http://localhost:8000/login' from origin 'http://192.168.139.209' has been blocked by CORS policy
POST http://localhost:8000/login net::ERR_FAILED
AxiosError: Network Error
```

画面は `http://192.168.139.209` で開いている。
しかし JavaScript は `http://localhost:8000/login` に送信していた。

これは CORS を許可して直す問題ではなく、送信先 URL が古い設定を見ている問題。

確認するもの。

```bash
grep -n "APP_URL\|VITE\|SESSION_DOMAIN\|SANCTUM_STATEFUL_DOMAINS" .env
```

`APP_URL` や `VITE_...` が `localhost:8000` のままなら直す。

```env
APP_URL=http://192.168.139.209
```

`.env` を直したあと、Laravel のキャッシュを消す。

```bash
php artisan optimize:clear
```

Vite のビルド済み JS に `localhost:8000` が残っている場合は、再ビルドする。

```bash
npm run build
```

さらに確認したところ、Ziggy の生成ファイルに古い URL が入っていた。

```txt
resources/js/ziggy.js
```

中身に次の値が残っていた。

```js
url: "http://localhost:8000"
port: 8000
```

フロント側の `route('login.store')` は Ziggy の情報から URL を作る。
そのため、Ziggy が古いままだと、ログイン送信先が `http://localhost:8000/login` になる。

対応手順。

```bash
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
```

再生成後に確認する。

```bash
grep -n "url\\|port" resources/js/ziggy.js
```

今回の結論。
`resources/js/ziggy.js` は環境の URL を含むため、git 管理しない方がよい。

理由。

- `APP_URL` の値が生成ファイルに入る
- ローカルでは `localhost:8000` になる
- ステージングでは `192.168.139.209` になる
- 環境ごとに値が変わる
- 古い Ziggy を commit すると、フロントの送信先が別環境に向く

方針。

```txt
resources/js/ziggy.js は build 前に生成する
resources/js/ziggy.js は git 管理しない
```

`.gitignore` に追加する。

```txt
/resources/js/ziggy.js
```

Ziggy を再生成し、再ビルドしたあとに動作した。

```bash
php artisan optimize:clear
php artisan ziggy:generate resources/js/ziggy.js
npm run build
```

確認した URL。

```txt
http://192.168.139.209/login
```

ここで Laravel + nginx + PHP-FPM + Vite build + Ziggy の組み合わせで動作確認できた。
