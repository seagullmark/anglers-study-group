# 第4章：`modId` で行う楽観的排他

この章では、  
**FileMaker の `modId` を使って、更新競合をどう扱っているか**を整理する。

ポイントは次の3つ。

- 編集画面表示時の `modId` をフォームに持つ
- 更新時は `withModId()` で FileMaker に渡す
- 競合時は `mod_id` エラーとして画面に返す

---

## 4-1 先にルール文書を読む

### 参照資料（実装ルール）

- `anglers/docs/laravel-filemaker-implementation-rules.md`

### この資料で示している方針

- 楽観的排他は FileMaker の `modId` を使う
- 画面表示時に持っていた `modId` を更新時に返す
- `withModId($request->mod_id)->save()` で保存する
- FileMaker の `306` を競合として扱う

### ここで押さえること（mod_id 送信）

- 独自の version field を増やしていない
- `updated_at` 比較を主役にしていない
- FileMaker 側の版管理に合わせている

---

## 4-2 編集画面表示時に `mod_id` を持たせる

### 対象ファイル（Controller 更新）

- `anglers/app/Http/Controllers/FishingTripController.php`

```php
private function tripFormData(FishingTrip $trip): array
{
    return [
        'id' => (string) $trip->id,
        'mod_id' => (string) $trip->getModId(),
        'trip_date' => $trip->trip_date_label,
        'start_at' => $trip->start_at_input,
        'end_at' => $trip->end_at_input,
        'river_name' => $trip->river_name,
        'point_name' => $trip->point_name,
        'tackle_name' => $trip->tackle_name,
        'memo' => $trip->memo,
        'photos' => ...
    ];
}
```

### このコードの役割（tripFormData）

- 取得済みレコードの `modId` を `mod_id` として Vue に渡す
- 編集画面で保持すべき版情報を作る
- 入力項目と一緒に返す

### ここで押さえること

- `mod_id` は編集画面の重要な値
- ただの補助情報ではない
- 更新時に比較するための基準になる

---

## 4-3 Vue 側は hidden と `useForm()` で保持する

### 対象ファイル（フォーム画面）

- `anglers/resources/js/Pages/FishingTrips/Form.vue`

```vue
<div
  v-if="form.errors.mod_id"
  class="rounded-2xl border border-amber-200 bg-amber-50 px-4 py-3 text-sm text-amber-800"
>
  {{ form.errors.mod_id }}
</div>

<form class="space-y-8" @submit.prevent="submit">
  <input v-if="isEdit" v-model="form.mod_id" type="hidden" />
```

```js
const buildFormState = () => ({
  mod_id: props.trip.mod_id,
  trip_date: props.trip.trip_date ?? '',
  start_at: props.trip.start_at ?? '',
  end_at: props.trip.end_at ?? '',
  river_name: props.trip.river_name ?? '',
  point_name: props.trip.point_name ?? '',
  tackle_name: props.trip.tackle_name ?? '',
  memo: props.trip.memo ?? '',
  photos: [],
  remove_photo_ids: []
})
```

### このコードの役割（Vue 側）

- 編集時だけ hidden の `mod_id` をフォームに持つ
- `useForm()` の状態として `mod_id` を保持する
- 競合エラーは `form.errors.mod_id` で表示する

### ここで見えること

- `mod_id` は画面から返す前提で持っている
- 競合エラーも通常のバリデーション表示に乗せている
- 特別な別画面に飛ばしていない

---

## 4-4 更新時は `withModId()` を使う

### 対象ファイル（Controller）

- `anglers/app/Http/Controllers/FishingTripController.php`

```php
try {
    $trip->withModId((string) $request->string('mod_id'))->save();
} catch (FileMakerDataApiException $e) {
    if ((int) $e->getCode() === 306) {
        throw ValidationException::withMessages([
            'mod_id' => __('Another update was saved first. Reload the page and try again.'),
        ]);
    }

    throw $e;
}
```

### このコードの役割

- 画面から返ってきた `mod_id` を FileMaker 更新時に渡す
- 競合が起きたら `306` を拾う
- 競合時は `mod_id` エラーとして編集画面に戻す

### `306` をどう読むか（競合エラー）

- 別の更新が先に保存された
- 画面表示時の版と、今の版が一致しない
- そのため上書きせずに止める

> 競合は「保存失敗」ではなく、  
> 「先に別の更新が入った」として扱う

---

## 4-5 削除では事前比較をしている

### 対象コード（削除）

```php
if ((string) $trip->getModId() !== (string) $request->string('mod_id')) {
    throw ValidationException::withMessages([
        'mod_id' => __('Another update was saved first. Reload the page and try again.'),
    ]);
}
```

### このコードの役割（削除）

- 削除前に版の一致を確認する
- 画面表示時と現在の状態が違うなら削除を止める
- 更新と同じ考え方で削除も守る

### ここで押さえること（削除）

- `update` は `withModId()->save()`
- `destroy` は `getModId()` と request の比較
- パッケージの `modId` 対応が保存処理中心なので、削除では明示比較を入れている

---

## 4-6 Request でも `mod_id` を必須にしている

### 対象ファイル（Request）

- `anglers/app/Http/Requests/UpdateFishingTripRequest.php`
- `anglers/app/Http/Requests/DestroyFishingTripRequest.php`

```php
'mod_id' => ['required', 'string'],
```

### このルールの意味

- 更新時に `mod_id` がない状態を許さない
- 削除時も `mod_id` が必要
- 版情報を「任意の補助値」にしない

### テスト

- `anglers/tests/Unit/Requests/FishingTripRequestTest.php`

```php
$this->assertArrayHasKey('mod_id', $rules);
$this->assertSame(['required', 'string'], $rules['mod_id']);
```

---

## 4-7 `submit()` と `deleteForm` で送信の流れを分ける

### 対象ファイル（Vue）

- `anglers/resources/js/Pages/FishingTrips/Form.vue`

```js
const submit = () => {
  const options = {
    forceFormData: true,
    onSuccess: clearSelectedFiles
  }

  if (isEdit.value) {
    form.submit('put', route('fishing-trips.update', props.trip.id), options)

    return
  }

  form.post(route('fishing-trips.store'), options)
}

const deleteForm = useForm({
  mod_id: props.trip.mod_id
})

const destroy = () => {
  if (!window.confirm('Delete this fishing trip?')) {
    return
  }

  deleteForm.submit('delete', route('fishing-trips.destroy', props.trip.id))
}
```

### このコードの役割（送信分岐）

- 更新送信は `form` で行う
- 削除送信は `deleteForm` に分ける
- どちらも `mod_id` を持つ形にしている

### この形にする理由

- 更新用入力と削除用入力を混ぜない
- 削除時も `mod_id` を明示的に返せる
- 送信の目的ごとにフォーム状態を切り分けられる

---

## 4-8 この実装でやっていないこと

### 参照資料での注意点

- `protected $withModId = true;` を既定にしない
- 更新時に再取得した最新 `modId` をそのまま使わない

### ここでの意味（mod_id の扱い）

- 画面表示時に見ていた版を比較したい
- 更新直前に取り直した最新版を使うと、フォーム編集の競合検知にならない
- そのため request の `mod_id` を主役にしている

> この教材の楽観的排他は、  
> 「画面表示時の版」を守るための実装として読む

---

## 第4章のまとめ

> この実装の楽観的排他は、  
> **画面表示時の `mod_id` を返し、それを FileMaker 更新時に比較する**形で組まれている。

- `tripFormData()` で `mod_id` を Vue に渡す
- `Form.vue` が hidden と `useForm()` で保持する
- `update()` は `withModId($request->mod_id)->save()` を使う
- 競合時は FileMaker の `306` を `mod_id` エラーに変えて返す
- `destroy()` でも `mod_id` 比較を入れている
