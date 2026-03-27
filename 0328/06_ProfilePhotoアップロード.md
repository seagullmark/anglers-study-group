# 第6章：Profile Photo 画面の流れ

この章では、  
**プロフィール画像アップロード画面で、表示切り替え・成功メッセージ・ファイル送信がどうつながるか**を整理する。

ポイントは次の3つ。

- `v-if` と `v-else` で **画像あり / 画像なし** を切り替える
- Laravel 側の `with('success', ...)` を **Inertia 経由で画面に戻す**
- `submit` では **ファイル送信と送信後の後始末** を行う

---

## 6-1 前提と全体像

本資料では、以下を前提とする。

- ページは `resources/js/Pages/User/ProfilePhoto.vue`
- アップロード完了後は Laravel 側で同じ画面へ戻す
- 成功メッセージはフラッシュデータとして渡す

### 基本フロー（プロフィール画像アップロード）

1. 現在のユーザー情報を画面に表示する
2. 画像があるかどうかで表示を切り替える
3. ファイルを選択する
4. `submit` で `user-photos.store` に送信する
5. Laravel 側で保存後、`with('success', ...)` を付けて戻す
6. Vue 側で `successMessage` を表示する

> この画面では、  
> 「今の状態を見せる」「送る」「結果を見せる」が一つにつながっている

---

## 6-2 ProfilePhoto.vue の全体像

### 対象ファイル（プロフィール画像画面）

- ページ：`resources/js/Pages/User/ProfilePhoto.vue`

```vue
<template>
  <Head title="Profile Photo" />

  <div class="space-y-8">
    <div class="flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between">
      ...
    </div>

    <div
      v-if="successMessage"
      class="rounded-2xl border border-emerald-200 bg-emerald-50 px-4 py-3 text-sm text-emerald-700"
    >
      {{ successMessage }}
    </div>

    <div class="grid gap-6 lg:grid-cols-[280px_minmax(0,1fr)]">
      ...
    </div>
  </div>
</template>
```

### この画面で見えているもの

- 上部にページ説明と操作リンクがある
- 中央に成功メッセージ表示エリアがある
- 左側に現在のユーザー表示がある
- 右側に画像アップロードフォームがある

### この構成の意味

- 画面を見たときに「現在値」と「更新操作」が分かれる
- アップロード結果も同じ画面で確認できる
- ユーザーにとって流れが追いやすい

---

## 6-3 `v-if` と `v-else` で表示を切り替える

### 対象コード（現在の画像表示）

```vue
<div
  v-if="userThumbnail"
  class="h-32 w-32 overflow-hidden rounded-full border border-slate-200 bg-white"
>
  <img
    :src="userThumbnail"
    alt="Current profile photo"
    class="h-full w-full object-cover"
  />
</div>

<div
  v-else
  class="flex h-32 w-32 items-center justify-center rounded-full border border-dashed border-slate-300 bg-white text-3xl font-semibold text-slate-400"
>
  {{ userInitial }}
</div>
```

### `v-if`

- 条件が true のときだけ要素を表示する
- この画面では `userThumbnail` があるときだけ画像を表示する

### `v-else`

- 直前の `v-if` が false のときに表示する
- この画面では画像がないときに頭文字を表示する
- 「画像あり」と「画像なし」の分岐を並べて読める

### ここで押さえること（successMessage）

- `v-if` と `v-else` は Vue の基本構文
- Inertia 特有ではない
- ただし Inertia で受け取ったデータを使って画面を切り替える場面でよく使う

> この書き方にすると、  
> 「画像があれば画像、なければ代替表示」という意図がそのまま読める

---

## 6-4 `successMessage` はどこから来るのか

### 対象コード（Vue 側）

```js
const successMessage = computed(() => page.props.flash?.success ?? null)
```

```vue
<div
  v-if="successMessage"
  class="rounded-2xl border border-emerald-200 bg-emerald-50 px-4 py-3 text-sm text-emerald-700"
>
  {{ successMessage }}
</div>
```

### 対象コード（Laravel 側）

```php
return redirect()->back()
    ->with('success', __('Photo uploaded.'));
```

### この流れで起きていること

1. Laravel 側で保存処理が終わる
2. `redirect()->back()` で同じ画面に戻る
3. `with('success', ...)` で成功メッセージを一時的に持たせる
4. Inertia 側でその値が `page.props.flash.success` として見える
5. Vue 側で `successMessage` に取り出す
6. `v-if="successMessage"` で表示する

### `{{ successMessage }}` の意味

- 取り出した成功メッセージ文字列を画面に出す
- 条件判定は `v-if`
- 実際の文言表示は `{{ ... }}` で行う

### ここで押さえること

- 成功メッセージは Vue 側で作っていない
- Laravel 側で決めたメッセージを Inertia 経由で受けている
- 画面側は「あるなら表示する」という形にしている

> これは API レスポンスを手で解析して出す形ではなく、  
> Laravel と Inertia の流れに沿ったメッセージ表示になる

---

## 6-5 ファイル入力は `useForm` で持つ

### 対象コード（フォーム状態）

```js
const form = useForm({
  file: null
})

const fileInput = ref(null)
```

### このコードの役割（フォーム状態）

- 送信するファイルを `form.file` に持つ
- 実際の input 要素は `fileInput` で参照できるようにする
- 送信後に input の中身を空に戻せるようにする

### `ref(null)` を使う理由

- ファイル input は見た目の値と内部状態がある
- `form.reset('file')` だけでは input 要素の表示が残ることがある
- そのため input 要素自体にもアクセスできるようにしている

---

## 6-6 `onFileChange` で選択ファイルを受け取る

### 対象コード（ファイル選択）

```js
const onFileChange = (event) => {
  form.file = event.target.files?.[0] ?? null
}
```

### このコードの役割（ファイル選択）

- ユーザーが選んだファイルを取り出す
- 最初の1件を `form.file` に入れる
- ファイル未選択なら `null` にする

### ポイント

- ファイル input は通常の文字入力とは扱いが違う
- `v-model` ではなく `@change` で受け取る
- 送信対象を明示的に `form.file` に入れている

> ファイル入力は、  
> 文字列入力と同じ感覚で扱わないことが大事

---

## 6-7 `submit` の中で何をしているか

### 対象コード（送信）

```js
const submit = () => {
  form.post(route('user-photos.store'), {
    forceFormData: true,
    onFinish: () => {
      form.reset('file')

      if (fileInput.value) {
        fileInput.value.value = ''
      }
    }
  })
}
```

### このコードの役割（送信）

- `user-photos.store` にファイルを送信する
- `forceFormData: true` でファイル送信の形にする
- 処理完了後にフォーム状態と input 要素を初期化する

### `forceFormData: true`

- 通常の送信ではなく、ファイルを含む送信として扱う
- `multipart/form-data` で送る前提を明確にする
- 画像アップロードでは重要な指定になる

### `onFinish`

- 送信処理が終わったあとに実行される
- `form.reset('file')` で Inertia 側の `file` を空に戻す
- `fileInput.value.value = ''` で input 要素の表示も空に戻す

### ここで押さえること（送信後処理）

- `submit` は単に送るだけではない
- 送信後の後始末まで含めて一つの流れになっている
- 画面を何度も使うフォームでは、この後始末が大事になる

> ファイル送信では、  
> 「送る」だけでなく「送った後に何を空に戻すか」まで考える

---

## 6-8 この画面で伝えたいこと

### Vue 側で見えているもの

- `v-if` / `v-else` で表示を切り替える
- `successMessage` があれば通知を出す
- `useForm` にファイルを持たせて送信する
- 送信完了後は input の見た目もリセットする

### Laravel × Inertia の流れで見えるもの

- 保存完了後は同じ画面に戻る
- 成功メッセージは `with('success', ...)` で渡す
- Vue 側は `page.props.flash.success` を表示する

### 設計上のポイント

- 現在表示と更新操作を分けている
- 成功通知を送信処理とつなげている
- 送信後の後始末までコードに含めている

---

## 第6章のまとめ

> Profile Photo 画面では、  
> **表示切り替え・フラッシュメッセージ・ファイル送信**が一つの流れになっている。

- `v-if` と `v-else` で画像あり / 画像なしを切り替える
- Laravel 側の `with('success', ...)` が Inertia 経由で `page.props.flash.success` に入る
- `{{ successMessage }}` でその文言を画面に出す
- `submit` では `forceFormData: true` でファイル送信を行う
- `onFinish` でフォーム状態と file input の両方を初期化する
