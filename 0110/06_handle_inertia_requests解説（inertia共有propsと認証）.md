# HandleInertiaRequests の解説（Inertia 共有 props と認証）

このミドルウェアは、Inertia のリクエストに対して

- 初回ロードで使う root view
- アセットのバージョン
- フロントへ常に渡す共有データ（shared props）

を定義します。

今回の教材では特に、

> **認証状態（auth.user）をフロントへ共有する**

部分がポイントです。

---

## 対象コード

```php
namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    protected $rootView = 'app';

    public function version(Request $request): ?string
    {
        return parent::version($request);
    }

    public function share(Request $request): array
    {
        return [
            ...parent::share($request),
            'auth' => [
                'user' => $request->user()
                    ? ['id' => $request->user()->id]
                    : null,
            ],
        ];
    }
}
```

---

## 1. rootView（初回ロード用テンプレート）

```php
protected $rootView = 'app';
```

- 初回アクセス時に使う Blade テンプレート名
- `resources/views/app.blade.php` が root になります
- Inertia のエントリーポイント

---

## 2. version（アセットのバージョン）

```php
public function version(Request $request): ?string
{
    return parent::version($request);
}
```

- アセット（JS/CSS）のバージョニングを扱うための仕組み
- 今回は親クラスの実装をそのまま利用
- （教材としては深入り不要）

---

## 3. share（共有 props：ここが重要）

```php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => [
            'user' => $request->user()
                ? ['id' => $request->user()->id]
                : null,
        ],
    ];
}
```

### 3-1. parent::share を維持する

```php
...parent::share($request),
```

- Inertia が標準で共有する props を壊さないために必要
- ここを消すと、Inertia 側の標準データが欠ける可能性があります

---

### 3-2. auth.user を常に渡す

```php
'auth' => [
    'user' => $request->user()
        ? ['id' => $request->user()->id]
        : null,
],
```

- `$request->user()` は、ログイン済みなら User を返します
- 未ログインなら `null`

つまりフロント側は、

- `props.auth.user` が `null` かどうか

だけを見れば、

> 「ログインしているかどうか」

を判断できます。

---

## 4. なぜ id だけ渡しているのか

この実装では、

- `id` だけを共有
- email / name などは共有しない

ようにしています。

目的はシンプルです。

- 教材として最小
- 個人情報をフロントへ過剰に渡さない
- 認証状態の説明に集中できる

必要になったら、ここに `name` などを追加できます。

---

## 5. 抽象化（設計）の観点

このミドルウェアでも、

- User が FileMaker かどうか
- 認証がどう実装されているか

は関係ありません。

`$request->user()` が返ってくるかどうか、

> **「認証状態」だけ**をフロントへ共有

しているだけです。

これも「認証状態が抽象化されている」例です。

---

## 6. まとめ

- HandleInertiaRequests は Inertia の共有 props を定義する
- `$request->user()` で認証状態を取り出せる
- フロントは `auth.user` が `null` かどうかだけを見ればよい

その結果、

> フロント側も「User の保存先」を意識せずに
> 認証状態だけを扱える

という構成になります。
