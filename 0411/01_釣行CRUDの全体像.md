# 第1章：釣行 CRUD の全体像

この章では、  
**`anglers/` の釣行機能が、どのファイルで CRUD の流れを組み立てているか**を整理する。

ポイントは次の3つ。

- 入口は `routes/web.php` の `Route::resource()` にある
- 主役のモデルは `FishingTrip` と `FishingTripPhoto` の 2 つ
- 画面は `Index` `Show` `Form` に分かれている

---

## 1-1 先に見るべき資料

今回の整理では、次の 2 つの資料を土台にする。

- `anglers/docs/laravel-filemaker-implementation-rules.md`
- `anglers/docs/laravel-filemaker-fishing-trip-models.md`

### この 2 つで押さえること

- `FishingTrip` と `FishingTripPhoto` を中心に組むこと
- 認可は `Gate / Policy` を使う方針であること
- 楽観的排他は FileMaker の `modId` を使うこと

> 先に設計の前提を置いてから実装を見ると、  
> Controller や Vue のコードが読みやすくなる

---

## 1-2 入口は `routes/web.php`

### 対象ファイル（ルーティング）

- ルーティング定義：`anglers/routes/web.php`

```php
Route::resource('fishing-trips', FishingTripController::class);

Route::get(
    'fishing-trip-photos/{fishing_trip_photo}/image',
    [ContainerController::class, 'showFishingTripPhoto']
)->name('fishing-trip-photos.image');
```

### このコードの役割

- 釣行機能の CRUD ルートをまとめて登録する
- 釣行画像の表示ルートも別で持つ
- すべて `auth` ミドルウェア配下で動かす

### ここで見えること

- 一覧、作成、保存、詳細、編集、更新、削除の入口は `FishingTripController`
- 画像表示だけは `ContainerController`
- 釣行本体と画像配信のルートを分けている

> CRUD の本体と画像表示を分けることで、  
> 更新処理と画像配信の関心が混ざりにくくなる

---

## 1-3 モデルは 2 つで始める

### 参照資料（モデル設計）

- 設計資料：`anglers/docs/laravel-filemaker-fishing-trip-models.md`

### この教材で使うモデル

- `FishingTrip`
- `FishingTripPhoto`

### 関係

- `FishingTrip` が親
- `FishingTripPhoto` が子
- 1 件の釣行に対して画像を複数枚持てる

### 実装ファイル

- モデル：`anglers/app/Models/FishingTrip.php`
- モデル：`anglers/app/Models/FishingTripPhoto.php`

```php
public function photos(): HasMany
{
    return $this->hasMany(FishingTripPhoto::class, 'fishing_trip_id', 'id')
        ->orderBy('sort_order');
}
```

### この形にする理由

- 釣行本体の更新と画像の追加・削除を分けて扱いやすい
- 画像を複数枚にしやすい
- 親モデルの認可を子画像にも広げやすい

---

## 1-4 画面は 3 つに分かれる

### 対象ファイル（Vue ページ）

- 一覧：`anglers/resources/js/Pages/FishingTrips/Index.vue`
- 詳細：`anglers/resources/js/Pages/FishingTrips/Show.vue`
- 作成 / 編集：`anglers/resources/js/Pages/FishingTrips/Form.vue`

### 画面ごとの役割

- `Index.vue`  
  一覧を見る
- `Show.vue`  
  1 件の内容と画像を見る
- `Form.vue`  
  新規作成と編集を共通フォームで扱う

### Controller 側との対応

```php
public function index(Request $request): Response
public function create(): Response
public function show(Request $request, string $fishing_trip): Response
public function edit(string $fishing_trip): Response
```

### この対応で見えること

- 画面単位で Inertia ページを返している
- `create` と `edit` は同じ `Form` を使う
- 一覧、詳細、フォームの分け方が素直

> 最初に画面の分け方を見ておくと、  
> その後の Request や Policy の説明が追いやすい

---

## 1-5 CRUD の流れはこうつながる

### 一覧

1. `GET /fishing-trips`
2. `FishingTripController@index`
3. `FishingTrips/Index.vue`

### 作成

1. `GET /fishing-trips/create`
2. `FishingTripController@create`
3. `FishingTrips/Form.vue`
4. `POST /fishing-trips`
5. `FishingTripController@store`

### 詳細

1. `GET /fishing-trips/{id}`
2. `FishingTripController@show`
3. `FishingTrips/Show.vue`

### 編集・更新・削除

1. `GET /fishing-trips/{id}/edit`
2. `FishingTripController@edit`
3. `FishingTrips/Form.vue`
4. `PUT /fishing-trips/{id}`
5. `FishingTripController@update`
6. `DELETE /fishing-trips/{id}`
7. `FishingTripController@destroy`

---

## 1-6 この章で押さえること

### ファイルの見方

- 入口は `routes/web.php`
- 保存の中心は `FishingTripController`
- 入力チェックは `FormRequest`
- 認可は `Policy`
- 更新競合は `modId`

### 今回の実装の特徴

- Laravel 標準の `resource` ルートに寄せている
- Vue 側は `Index` `Show` `Form` で整理している
- FileMaker 連携でも Laravel の基本的な形を崩していない

---

## 第1章のまとめ

> `anglers/` の釣行機能は、  
> **`Route::resource()` を入口にして、`Model` `Controller` `Vue` を素直につないでいる。**

- CRUD の入口は `routes/web.php`
- 主役のモデルは `FishingTrip` と `FishingTripPhoto`
- 画面は `Index` `Show` `Form` の 3 つに整理されている
- この上に `Request` `Policy` `modId` の仕組みが積み上がる
