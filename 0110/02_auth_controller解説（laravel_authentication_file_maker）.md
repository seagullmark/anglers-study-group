# AuthController の解説（Laravel Authentication × FileMaker）

> この勉強に参加しているあなたは、FileMaker を使って Web アプリを作ろうとしているという点で、かなり特殊で、そしてラッキーです。
>
> ここを乗り越えられれば、以降の理解は一気に楽になります。
>
> Laravel には Breeze という最小構成の認証スターターがあり、
> Laravel 単体であれば、これを使えばすぐに認証を始められます。
> 多くの教本や解説記事も、ほぼこの構成に沿っています。
>
> ただし、ユーザー情報の保存先が FileMaker の場合、
> Breeze をそのまま使うことはできません。
>
> Eloquent-FileMaker という制約が入った瞬間に、
> 「どこを Laravel に任せ、どこを調整すべきか」を自分で判断する必要が出てきます。
>
> FileMaker を使って Laravel の認証を扱おうとすると、
> 多くの人がまさにこの地点で立ち止まります。
>
> この章は、その“最初につまずきやすい場所”を、
> 構造から整理して越えるためのものです。

## 休憩に読むコラム：Laravel 12 で何が起きているか

当初は意図していなかったのですが、結果としてこの勉強会で扱っている構成は、  
**Laravel 12 で公式に提供されている新しいスターターキットが示す方向性と一致しています。**

私自身も、「いまも Breeze は使われているのだろうか？」と確認したことをきっかけに知ったのですが、Laravel 12 では  
Vue 3 + Inertia v2 + 認証 + Sail（Docker）を前提とした  
モダンな公式スターターがすでに用意されていました。  
しかし、この勉強会で積み上げてきた学びは決して無駄ではありません。  
むしろ、そのスターターが「どのような技術の組み合わせで成り立っているのか」を、  
自分の手で組み、悩み、理解するための確かな土台になっています。

つまりこの構成は、  
「公式スターターを真似た」のではなく、  
**実務上の必然から組み上げた結果、公式が示した標準形と合流していた**、という位置づけになります。

この勉強会を組んでいた時点では、そのようなスターターの存在を知らずに進めていましたが、  
今振り返ると、Laravel が向かおうとしていた方向を  
**自然に先取りしていた構成だった**と言えます。

---

## 本編

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

### TIPS: 認証フィールドを変更する場合

- 他のフィールドに差し替える場合は、`$credentials` のキーを変更して渡します

```php
$credentials = $request->only('account_name', 'password');
```

- `password` は **平文**を受け取り、`password` カラムの **ハッシュと照合**されます
- `password` フィールド名を変更したい場合は、User モデルで `getAuthPasswordName()` をオーバーライドします

```php
public function getAuthPasswordName()
{
    return 'account_password';
}
```

- Eloquent-FileMaker では `fieldMapping` で FileMaker 側のフィールド名を対応付けできます

```php
protected $fieldMapping = [
    'account_password' => 'password',
];
```

この場合、`Auth::attempt()` は `password` のままで運用できます。

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

---

## 5. 番外編：Laravel の実装（参考）

実際の Laravel のソースでは、概ね次の流れになっています。

```php
// vendor/laravel/framework/src/Illuminate/Auth/SessionGuard.php
public function attempt(array $credentials = [], $remember = false)
{
    return $this->timebox->call(function ($timebox) use ($credentials, $remember) {
        $this->fireAttemptEvent($credentials, $remember);

        $this->lastAttempted = $user = $this->provider->retrieveByCredentials($credentials);

        if ($this->hasValidCredentials($user, $credentials)) {
            $this->rehashPasswordIfRequired($user, $credentials);
            $this->login($user, $remember);
            $timebox->returnEarly();
            return true;
        }

        $this->fireFailedEvent($user, $credentials);
        return false;
    }, $this->timeboxDuration);
}
```

```php
// vendor/laravel/framework/src/Illuminate/Auth/EloquentUserProvider.php
public function retrieveByCredentials(array $credentials)
{
    $credentials = array_filter(
        $credentials,
        fn ($key) => ! str_contains($key, 'password'),
        ARRAY_FILTER_USE_KEY
    );

    if (empty($credentials)) {
        return;
    }

    $query = $this->newModelQuery();

    foreach ($credentials as $key => $value) {
        if (is_array($value) || $value instanceof Arrayable) {
            $query->whereIn($key, $value);
        } elseif ($value instanceof Closure) {
            $value($query);
        } else {
            $query->where($key, $value);
        }
    }

    return $query->first();
}

public function validateCredentials(UserContract $user, array $credentials)
{
    if (is_null($plain = $credentials['password'])) {
        return false;
    }

    if (is_null($hashed = $user->getAuthPassword())) {
        return false;
    }

    return $this->hasher->check($plain, $hashed);
}
```

最終的に `Auth::login($user)` が呼ばれています。  
remember を使いたい場合は `Auth::login($user, true)` です。  
時には実際のソースを読むことも必要です。
