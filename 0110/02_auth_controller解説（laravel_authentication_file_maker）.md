# AuthController の解説（Laravel Authentication × FileMaker）

このレッスンでは、ログイン処理を **Laravel の標準 Authentication** に委ねます。\
コントローラー側では「入力検証 → Auth::attempt → セッション再生成」だけを行い、\
ユーザーの保存先（FileMaker）を意識しない構成にしています。

---

## 対象コード

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function create()
    {
        return inertia('Auth/Login');
    }

    public function store(Request $request)
    {
        $request->validate(
            [
                'email' => 'required|string|email',
                'password' => 'required|string'
            ]
        );

        $credentials = $request->only('email', 'password');
        $credentials['email'] = '==' . str_replace('@', '\@', $credentials['email']);

        if (!Auth::attempt($credentials, true)) {
            throw ValidationException::withMessages([
                'email' => __('Authentication failed')
            ]);
        }

        $request->session()->regenerate();

        return redirect()->intended(route('index'));
    }

    public function destroy(Request $request)
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect()->route('login');
    }
}
```

---

## 1. create()：ログイン画面を返す

```php
public function create()
{
    return inertia('Auth/Login');
}
```

- Inertia のログイン画面コンポーネントを返します
- ここは **認証ロジックと無関係**です

---

## 2. store()：ログイン処理（重要）

### 2-1. 入力検証

```php
$request->validate([
    'email' => 'required|string|email',
    'password' => 'required|string'
]);
```

- `email` が email 形式であることを保証
- `password` が存在することを保証

---

### 2-2. 資格情報（credentials）を作る

```php
$credentials = $request->only('email', 'password');
$credentials['email'] = '==' . str_replace('@', '\@', $credentials['email']);
```

- `Auth::attempt()` に渡す配列を作ります
- `email` の `@` を `\@` に置換し、`'=='` を前置して **FileMaker で完全一致検索**させるための調整です（※運用ルールに合わせる）
  - ここは「このプロジェクト固有の小さな調整」です
  - 認証フロー自体は Laravel 標準のままです

---

### 2-3. Auth::attempt（Laravel 標準認証）

```php
if (!Auth::attempt($credentials, true)) { // true is for remember me
    throw ValidationException::withMessages([
        'email' => __('Authentication failed')
    ]);
}
```

- `Auth::attempt($credentials, true)` は **標準のログイン処理**です
- `true` を渡すと **remember me（ログイン保持）**を有効にします

ここで重要なのは次の点です。

> コントローラーは **User の保存先（FileMaker）を知りません**。
>
> `Auth::attempt` は Auth Provider を通じて User を取得し、 `password` を検証し、セッションを開始します。

※ 本レッスンでは `User` が FileMaker に紐づいていますが、\
`Auth::attempt()` の呼び出し方は一切変わりません。

---

### 2-4. セッション再生成（セキュリティ）

```php
$request->session()->regenerate();
```

- ログイン直後にセッションIDを再生成します
- セッション固定化（Session Fixation）対策
- Laravel 標準の推奨パターン

---

### 2-5. intended でリダイレクト

```php
return redirect()->intended(route('index'));
```

- 直前にアクセスしようとしていたページがあればそこへ戻す
- なければ `route('index')` へ

---

## 3. destroy()：ログアウト処理

```php
Auth::logout();

$request->session()->invalidate();
$request->session()->regenerateToken();

return redirect()->route('login');
```

- `Auth::logout()`：認証状態を解除
- `invalidate()`：セッションを破棄
- `regenerateToken()`：CSRF トークンを再生成

ログアウト後はログイン画面へ戻します。

---

## 4. まとめ

この AuthController のポイントは次の3つです。

1. **認証処理は Auth::attempt() に完全委任**
2. コントローラーは **FileMaker を意識しない**（認証の入口だけ）
3. ログイン後に `session()->regenerate()` を行い **セキュリティも標準通り**

そのため、User の保存先が FileMaker でも、\
Laravel のデフォルト Authentication をそのまま教材として示せます。
