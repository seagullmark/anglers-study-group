# Application 設定の解説（Laravel Authentication × FileMaker）

このファイルは **Laravel 11 で導入された新しい Application 設定（bootstrap/app.php）** で、Laravel 12 でも同じ方式が継続しています。\
ここでは、ルーティング・ミドルウェア・例外処理など、 **アプリ全体の振る舞い**を定義しています。

本レッスンでは特に、

- 認証前 / 認証後のリダイレクト先
- Inertia 用ミドルウェアの登録

がポイントになります。

---

## 対象コード

```php
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use App\Http\Middleware\HandleInertiaRequests;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->web(append: [
            // Inertia の標準ミドルウェア
            HandleInertiaRequests::class,
        ]);

        $middleware->redirectGuestsTo(fn () => route('login'));
        $middleware->redirectUsersTo(fn () => route('index'));
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })->create();
```

---

## 1. Application::configure()

```php
return Application::configure(basePath: dirname(__DIR__))
```

- アプリケーションの起点を定義します
- Laravel 11 から、ここが **設定の集約ポイント**になりました（Laravel 12 でも継続）。

---

## 2. ルーティングの登録

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

- `web`：Web リクエスト用ルート
- `commands`：Artisan コマンド用ルート
- `health`：ヘルスチェック用エンドポイント

認証とは直接関係ありませんが、 **アプリの構成を把握するために重要**です。

---

## 3. Web ミドルウェアへの Inertia 追加

```php
$middleware->web(append: [
    HandleInertiaRequests::class,
]);
```

- Inertia を使うための標準ミドルウェアを追加しています
- これにより、
  - 共有 props
  - フラッシュメッセージ
  - CSRF 情報

などが自動的にフロントへ渡されます。

---

## 4. 認証状態によるリダイレクト先（重要）

```php
$middleware->redirectGuestsTo(fn () => route('login'));
$middleware->redirectUsersTo(fn () => route('index'));
```

ここがこのファイルの **最大のポイント**です。

### redirectGuestsTo

- `auth` ミドルウェアに引っかかった **未ログインユーザー**は
- 自動的に `login` ルートへリダイレクト

### redirectUsersTo

- `guest` ミドルウェアに引っかかった **ログイン済みユーザー**は
- 自動的に `index` ルートへリダイレクト

これにより、

- ルーティング側では `guest` / `auth` を付けるだけ
- 実際の遷移先は Application 設定で一元管理

という構成になります。

---

## 5. なぜここで設定するのか

Laravel 11 から、

> **認証状態による挙動は Application レベル（bootstrap/app.php）で定義する**

という設計に変わりました（Laravel 12 でも継続）。

その結果、

- コントローラーに分岐を書かない
- ルート定義をシンプルに保てる
- 認証フローの全体像が把握しやすい

というメリットがあります。

---

## 6. FileMaker を意識しない理由

この設定ファイルでも、

- User の保存先
- 認証の実装方法

は一切登場しません。

それでも問題なく動作するのは、

> **Laravel の Authentication が「認証状態」を完全に抽象化している**

からです。

---

## 7. まとめ

この Application 設定のポイントは次の通りです。

1. Inertia 用ミドルウェアを Web スタックに追加
2. 未ログイン / ログイン済みのリダイレクト先を一元管理
3. 認証フローを Laravel 標準に完全委任

その結果、

- User は FileMaker
- 認証は Laravel
- 画面遷移はシンプル

という、教材として分かりやすい構成になります。
