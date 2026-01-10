# ルーティングの解説（Laravel Authentication × FileMaker）

このレッスンでは、Laravel 標準の **guest / auth ミドルウェア**を使って、
ログイン前・ログイン後の画面遷移を明確に分離しています。

ユーザー情報の保存先が FileMaker であっても、
**ルーティング設計は通常の Laravel と一切変わりません**。

---

## 対象コード

```php
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'create'])->name('login');
    Route::post('/login', [AuthController::class, 'store'])->name('login.store');
});

Route::middleware('auth')->group(function () {
    Route::get('/', function () {
        return inertia('Index/Index');
    })->name('index');

    Route::post('/logout', [AuthController::class, 'destroy'])->name('logout');
});
```

### 補足: inertia() と Inertia::render()

inertia() は Inertia::render() のショートカットです。
Laravel / Inertia では helper の inertia() を使うのが一般的です。
Inertia::render を使う場合は `use Inertia\Inertia;` を追加してください。
教材では簡潔さを優先して inertia() を採用します。

---

## 1. guest ミドルウェア（未ログイン専用）

```php
Route::middleware('guest')->group(function () {
    Route::get('/login', [AuthController::class, 'create'])->name('login');
    Route::post('/login', [AuthController::class, 'store'])->name('login.store');
});
```

### 役割

- **未ログインユーザーのみ**アクセス可能
- すでにログインしている場合は、自動的にリダイレクトされます

### このレッスンでの意味

- ログイン済みユーザーが `/login` に戻れないようにする
- 認証状態の分岐を **Laravel に任せる**

---

## 2. auth ミドルウェア（ログイン必須）

```php
Route::middleware('auth')->group(function () {
    Route::get('/', function () {
        return inertia('Index/Index');
    })->name('index');

    Route::post('/logout', [AuthController::class, 'destroy'])->name('logout');
});
```

### 役割（認証必須）

- **ログイン済みユーザーのみ**アクセス可能
- 未ログインの場合は `/login` にリダイレクト

### ポイント

- トップページ（`/`）を認証必須にしている
- ログアウト処理も認証済みユーザーのみが実行

---

## 3. なぜルート分離が重要か

この構成により、

- ログイン前
  - `/login` にだけアクセス可能
- ログイン後
  - `/`（index）や `/logout` にアクセス可能

という **明確な状態遷移**ができます。

> 認証状態による制御を、
> コントローラーやビューではなく **ミドルウェアに任せる**

これが Laravel 標準の考え方です。

---

## 4. FileMaker を意識しない理由

このルーティングでは、

- User がどこに保存されているか
- 認証がどう実装されているか

を一切意識していません。

それでも正しく動作するのは、

> **Laravel の Authentication が「User が誰か」だけを抽象化している**

からです。

---

## 5. まとめ

このルーティング構成のポイントは次の通りです。

1. `guest` / `auth` ミドルウェアで状態を明確に分離
2. 認証制御を Laravel 標準ミドルウェアに完全委任
3. FileMaker を意識しない、通常の Laravel ルーティング

その結果、

- User の保存先が FileMaker
- 認証は Laravel 標準

という構成でも、
安全で分かりやすい画面遷移を実現できます。
