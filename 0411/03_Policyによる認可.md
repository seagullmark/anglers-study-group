# 第3章：Policy による認可

この章では、  
**釣行機能の認可が、どこで判定され、どこで画面に反映されているか**を整理する。

ポイントは次の3つ。

- この実装の中心は `Gate` ではなく `Policy`
- 判定は Controller の `$this->authorize()` から通している
- `Gate::authorize()` との関係も理解しておく

---

## 3-1 まず設計ルールを見る

### 参照資料（実装ルール）

- `anglers/docs/laravel-filemaker-implementation-rules.md`

### この資料で示している方針

- 認可は Laravel の `Gate / Policy` を使う
- モデル単位の認可は `Policy` を優先する
- Controller や Vue に独自の認可ロジックを散らさない

### ここで押さえること（Gate と Policy）

- 教材の言葉としては `Gate / Policy`
- 実コードでは `Policy` を中心に組んでいる
- つまり「まず Policy を読む」が正しい入口になる

---

## 3-2 `Gate::authorize()` と `$this->authorize()` の関係

### Laravel で見かける 2 つの書き方

```php
Gate::authorize('update', $trip);
```

```php
$this->authorize('update', $trip);
```

### この 2 つの意味

- どちらも Laravel の認可判定
- どちらも `Policy` を通して可否を判断する
- 認可に失敗したときは 403 として扱う

### 違い

- `Gate::authorize(...)`  
  `Gate` ファサードを直接使う書き方
- `$this->authorize(...)`  
  Controller で使う書き方

### このプロジェクトではどうなっているか

- `Gate::authorize(...)` は実コードに出てこない
- Controller では `$this->authorize(...)` を使っている
- つまり、`Gate` の考え方は使っているが、表面の書き方は Controller 向けにしている

### ここで押さえること

- `Gate` がコードに見えなくても、認可の考え方が消えているわけではない
- この実装では `$this->authorize(...)` が `Gate` の入口になっている
- Controller 外で認可したいときは `Gate::authorize(...)` という書き方もある

> 今回の教材では、  
> `Gate` を概念として押さえつつ、実装は `$this->authorize()` と `Policy` で読む

---

## 3-3 実際の認可ルールは `FishingTripPolicy`

### 対象ファイル（Policy）

- `anglers/app/Policies/FishingTripPolicy.php`

```php
class FishingTripPolicy
{
    public function viewAny(User $user): bool
    {
        return ! empty($user->id);
    }

    public function view(User $user, FishingTrip $fishingTrip): bool
    {
        return ! empty($user->id);
    }

    public function create(User $user): bool
    {
        return ! empty($user->id);
    }

    public function update(User $user, FishingTrip $fishingTrip): bool
    {
        return (string) $fishingTrip->user_id === (string) $user->id;
    }

    public function delete(User $user, FishingTrip $fishingTrip): bool
    {
        return (string) $fishingTrip->user_id === (string) $user->id;
    }
}
```

### このコードの役割（authorize）

- 一覧表示、閲覧、作成、更新、削除の可否を決める
- 更新と削除は所有者だけに絞る
- 釣行本体の `user_id` を基準にする

### この教材での判断ルール

- 一覧: ログイン済みなら見られる
- 詳細: ログイン済みなら見られる
- 作成: ログイン済みならできる
- 更新: 本人の釣行だけできる
- 削除: 本人の釣行だけできる

> 「誰のデータか」の判定は  
> `FishingTrip` の `user_id` を中心に読む

---

## 3-4 `viewAny` と `view` の違い

### `viewAny`

```php
public function viewAny(User $user): bool
{
    return ! empty($user->id);
}
```

これは、  
**一覧を見てよいか**  
を判定する。

まだ特定の 1 件は決まっていない。

### `view`

```php
public function view(User $user, FishingTrip $fishingTrip): bool
{
    return ! empty($user->id);
}
```

これは、  
**この 1 件を見てよいか**  
を判定する。

対象レコードが決まっている。

### このプロジェクトでの対応

一覧画面:

```php
$this->authorize('viewAny', FishingTrip::class);
```

詳細画面:

```php
$this->authorize('view', $trip);
```

### 違いを一言でいうと

- `viewAny`  
  一覧を見てよいか
- `view`  
  この 1 件を見てよいか

### 今回の `FishingTripPolicy` では

- `viewAny` も `view` も、今は「ログインしていれば見てよい」
- ただし役割は別
- 一覧の権限と、1 件ごとの権限を分けて設計できる

> Laravel は `viewAny` と `view` を分けることで、  
> 一覧と詳細の認可を別々に組み立てられる

---

## 3-5 Policy は CRUD リソースと相性がよい

Laravel の `Policy` は、  
CRUD リソースの画面や処理と対応づけやすい。

たとえば `FishingTrip` では、次のように読める。

- `index` ⇔ `viewAny`
- `show` ⇔ `view`
- `create` / `store` ⇔ `create`
- `edit` / `update` ⇔ `update`
- `destroy` ⇔ `delete`

### この対応のよさ

- Controller の各アクションで何を判定するかが分かりやすい
- 認可ルールを `Policy` にまとめやすい
- CRUD の流れと認可の流れを揃えて考えやすい

> つまり `Policy` は、  
> Laravel の resource controller と相性のよい認可の置き場所といえる

---

## 3-6 Controller は必ず `authorize()` を通す

### 対象ファイル（Controller）

- `anglers/app/Http/Controllers/FishingTripController.php`

```php
$this->authorize('viewAny', FishingTrip::class);
$this->authorize('create', FishingTrip::class);
$this->authorize('view', $trip);
$this->authorize('update', $trip);
$this->authorize('delete', $trip);
```

### 実際に使われている場所

- `index()` で `viewAny`
- `create()` と `store()` で `create`
- `show()` で `view`
- `edit()` と `update()` で `update`
- `destroy()` で `delete`

### このコードの役割

- 画面表示前に認可を通す
- 保存前にも認可を通す
- URL を直接叩かれてもサーバ側で止める

### ここで押さえること（Controller 認可）

- 認可はフロント側では完結しない
- Controller 側で毎回確認している
- 保存処理の前に必ず通す形が大事

---

## 3-7 画面側の `can_edit` は最終判定ではない

### 対象コード（一覧・詳細用データ）

```php
$canEdit = (string) $trip->user_id === $currentUserId;
```

```php
'can_edit' => $canEdit,
```

### 対象ファイル

- `anglers/app/Http/Controllers/FishingTripController.php`
- `anglers/resources/js/Pages/FishingTrips/Index.vue`
- `anglers/resources/js/Pages/FishingTrips/Show.vue`

### 画面側での使い方

```vue
<span v-if="trip.can_edit">Yours</span>
```

```vue
<Link
  v-if="trip.can_edit"
  :href="route('fishing-trips.edit', trip.id)"
>
  Edit Trip
</Link>
```

### このコードの意味

- 画面表示を分かりやすくするための補助情報
- 編集ボタンを出すかどうかに使う
- ただし、これ自体が最終認可ではない

### ここで押さえること（can_edit）

- `can_edit` は UI の補助
- 本当の判定は `$this->authorize()` と `Policy`
- Vue にだけ判定を置かない

> ボタンを隠すことと、  
> 操作を許可することは別で考える

### FileMaker の文脈で見ると

- レイアウト上でボタンを隠す
- 条件付きで編集できないように見せる

こうした対応は FileMaker でもよく行う。

ただし、これは **見せ方の制御** であって、  
本来の最終制御とは分けて考える必要がある。

本来は、

- FileMaker ならアクセス権設定
- Laravel なら `Policy` / `Gate`
- Vue は表示の補助

という分け方の方が安全。

特に複数人で開発すると、

- ある画面では隠した
- 別画面では隠し忘れた
- URL や別経路の制御が抜けた

という漏れが起きやすい。

> FileMaker でも Laravel でも、  
> 「見せないこと」と「許可しないこと」は分けて考えるのが基本になる

---

## 3-8 画像表示も親の Policy を基準にする

### 対象ファイル（画像配信）

- `anglers/app/Http/Controllers/ContainerController.php`

```php
public function showFishingTripPhoto(Request $request, string $fishing_trip_photo)
{
    $photo = FishingTripPhoto::where('id', '==', $fishing_trip_photo)
        ->firstOrFail();

    $trip = FishingTrip::where('id', '==', $photo->fishing_trip_id)
        ->firstOrFail();

    $this->authorize('view', $trip);

    return $this->proxyImage(
        $this->resolveContainerPath($photo, ['image']),
    );
}
```

### このコードの役割（画像配信）

- 画像レコードを直接返さない
- 親の `FishingTrip` を取得する
- 親の `view` 判定を通してから画像を返す

### この形にする理由

- 画像だけ別ルールにしない
- 釣行本体の認可と画像表示の認可を揃える
- 子画像の公開範囲を親に合わせられる

---

## 3-9 テストでも認可ルールを確認している

### 対象ファイル（テスト）

- `anglers/tests/Unit/Policies/FishingTripPolicyTest.php`

```php
$this->assertTrue($policy->view($user, $trip));
$this->assertTrue($policy->update($user, $trip));
$this->assertTrue($policy->delete($user, $trip));
```

```php
$this->assertTrue($policy->view($user, $trip));
$this->assertFalse($policy->update($user, $trip));
$this->assertFalse($policy->delete($user, $trip));
```

### このテストで見ていること

- 所有者は閲覧・更新・削除できる
- 非所有者は閲覧できるが、更新・削除はできない
- `Policy` のルールを単体で確認している

---

## 第3章のまとめ

> この実装の認可は、  
> **`Policy` を中心にし、Controller の `authorize()` から毎回通す**形で組まれている。

- 教材上の言葉は `Gate / Policy` だが、実装の中心は `Policy`
- `Gate::authorize()` と `$this->authorize()` はどちらも Laravel の認可
- このプロジェクトでは Controller なので `$this->authorize()` を使っている
- `FishingTripPolicy` が所有者判定を持つ
- Controller は各アクションで `authorize()` を呼ぶ
- 画面側の `can_edit` は補助であり、最終判定ではない
- 画像表示も親の `FishingTrip` の `view` 判定を基準にしている
