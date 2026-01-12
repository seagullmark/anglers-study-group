# User モデルの解説（Laravel Authentication × FileMaker）

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
> その“最初につまずきやすい場所”を、
> 構造から整理して越えるためのものです。

このレッスンでは、Laravel の **デフォルト Authentication（認証）** を  
FileMaker 上のユーザーデータでそのまま利用します。

そのために、`User` モデルは **通常の Eloquent モデルとは少し違う構成**になっています。

---

## 1. 継承元：FMModel

```php
class User extends FMModel
```

通常の Laravel では、`User` は Eloquent（RDB）を前提としたクラスを継承します。  
本レッスンでは、ユーザー情報を **FileMaker に保存するため**、  
`FMModel` を継承しています。

> 認証の仕組みを変更するためではありません。  
> **保存先を FileMaker にするための差し替え**です。

---

## 2. implements している Contract について

```php
implements
    MustVerifyEmailContract,
    AuthenticatableContract,
    AuthorizableContract,
    CanResetPasswordContract
```

Laravel の認証は、  
「この User が **どんなインターフェースを満たしているか**」だけを見ています。

ここで指定している Contract は、Laravel 標準の User が持っているものと同じです。

| Contract | 役割 |
| ------- | ------ |
| Authenticatable | ログイン（本人確認） |
| Authorizable | 権限判定（Gate / Policy） |
| CanResetPassword | パスワードリセット |
| MustVerifyEmail | メール認証 |

つまり、

> **User の保存先が FileMaker でも、  
> Contract を満たしていれば Laravel の認証はそのまま動く**

ということです。

---

## 3. use している Trait について

```php
use Authenticatable,
    Authorizable,
    CanResetPassword,
    MustVerifyEmail,
    Notifiable;
```

これらは、Laravel 標準 User が内部で使っている Trait です。

今回は `Illuminate\Foundation\Auth\User` を継承できないため、  
**必要な Trait を個別に use しています**。

> Laravel 標準 User を「分解して FMModel に組み直している」だけです。

---

## 4. fillable / hidden

```php
protected $fillable = ['name', 'email', 'password'];
protected $hidden = ['password', 'remember_token'];
```

ここは通常の User モデルと同じです。

- `fillable`：一括代入を許可するフィールド
- `hidden`：JSON 変換時に隠すフィールド

---

## 5. casts（重要）

```php
protected function casts(): array
{
    return [
        'id' => 'string',
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];
}
```

### password => 'hashed'

- パスワードは **Laravel の Hash 機構**をそのまま使用します
- 保存時に自動で bcrypt 化されます
- FileMaker には **ハッシュ化済みの文字列のみ**保存されます

### email_verified_at

- メール認証（MustVerifyEmail）を有効にするための標準フィールドです

---

## 6. まとめ

この User モデルのポイントは次の3つです。

1. **認証ロジックは一切カスタマイズしていない**
2. **Laravel 標準の Authentication をそのまま利用している**
3. **違いは「ユーザーの保存先が FileMaker」であることだけ**

その結果、

- `Auth::attempt()`
- `Auth::user()`
- `middleware('auth')`
- パスワードハッシュ
- remember me
- メール認証

これらが **すべて Laravel 標準のまま動作**します。

---

このレッスンの目的は、

> **認証のような重要で事故りやすい部分は自作せず、  
> フレームワークに任せる**

という設計判断を理解することです。
