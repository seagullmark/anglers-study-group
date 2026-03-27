# 第3章：`useForm` とエラー表示

この章では、  
**Inertia の `useForm` を使って、入力値とバリデーションエラーをどう扱うか**を整理する。

ポイントは次の3つ。

- 入力値は **`useForm` にまとめる**
- エラー表示は **Laravel の結果を Inertia 経由で受ける**
- Vue 側は **`v-model` `v-if` `@submit.prevent` を使って画面を組み立てる**

---

## 3-1 前提と全体像

本資料では、以下を前提とする。

- ログイン画面は Vue で作る
- 送信は Inertia の `form.post()` を使う
- バリデーションは Laravel 側で行う

### 基本フロー（入力とエラー）

1. Vue で `useForm` を作る
2. `v-model` で入力値を `form` に結びつける
3. `form.post()` で Laravel に送信する
4. Laravel 側でバリデーションを行う
5. エラーがあれば Inertia 経由で `form.errors` に入る
6. Vue 側は `v-if` で表示する

> ここで大事なのは、  
> エラー表示のために別の状態管理を増やさないこと

---

## 3-2 `useForm` が入力の入口になる

### 対象ファイル（ログイン画面）

- ページ：`resources/js/Pages/Auth/Login.vue`

```js
const form = useForm({
  email: '',
  password: ''
})
```

### このコードの役割（入力欄）

- `email` と `password` の初期値を定義する
- 入力値を一つの `form` にまとめる
- 送信状態やエラー情報も同じ `form` で扱える形にする

### この形にする理由

- 入力値の置き場が分かりやすい
- 送信処理とエラー表示を同じまとまりで追える
- `ref` を複数並べるより、画面の意図が読みやすい

> `useForm` は単なる入力箱ではなく、  
> 画面送信に必要な情報をまとめて持つための入口になる

---

## 3-3 入力値は `v-model` でつなぐ

### 対象コード（入力欄）

```vue
<input
  id="email"
  v-model="form.email"
  type="email"
  autocomplete="username"
  required
/>
```

```vue
<input
  id="password"
  v-model="form.password"
  type="password"
  autocomplete="current-password"
  required
/>
```

### このコードの役割（エラー表示）

- 入力欄と `form` の値を結びつける
- ユーザーが入力した内容を `form.email` と `form.password` に反映する
- 送信時にそのまま `form.post()` へ流せるようにする

### ポイント

- 入力欄ごとに別の状態管理を作らない
- `form` を見れば、今の入力状態が分かる
- 入力値と送信先の流れが切れにくい

---

## 3-4 Vue の基本構文をこの画面でどう読むか

### 対象コード（フォーム）

```vue
<form class="space-y-5" @submit.prevent="submit">
```

```vue
<input v-model="form.email" />
```

```vue
<p v-if="form.errors.email">
  {{ form.errors.email }}
</p>
```

### `v-model`

- 入力欄の値を `form.email` や `form.password` に結びつける
- ユーザーの入力がそのまま `form` に反映される
- Inertia 側へ送る値の元になる

### `v-if`

- 条件が true のときだけ要素を表示する
- この画面では `form.errors.email` があるときだけエラーメッセージを出す
- 「エラーがなければ表示しない」という書き方を素直に表現できる

### `@submit.prevent="submit"`

- フォーム送信時に `submit` 関数を実行する
- `.prevent` は通常のフォーム送信を止める
- ブラウザの通常遷移ではなく、Inertia の `form.post()` に流すための入口になる

### ここで押さえること（エラーの流れ）

- `v-model` `v-if` `@submit.prevent` 自体は Vue の基本構文
- Inertia 特有なのは、その先で `useForm` と `form.post()` を使って Laravel に流すところ
- つまり「画面の書き方は Vue」「送信の流れは Inertia」という見方をすると整理しやすい

> Inertia を理解するときは、  
> Vue の基本構文と Inertia の送信の流れを混ぜて考えないことが大事

---

## 3-5 エラー表示は `form.errors` を見る

### 対象コード（エラー表示）

```vue
<p v-if="form.errors.email" class="mt-2 text-sm text-rose-600">
  {{ form.errors.email }}
</p>
```

```vue
<p v-if="form.errors.password" class="mt-2 text-sm text-rose-600">
  {{ form.errors.password }}
</p>
```

### このコードの役割

- `email` にエラーがあるときだけメッセージを表示する
- `password` にエラーがあるときだけメッセージを表示する
- エラーの文言をそのまま画面に出す

### Vue 側でやっていること

- エラーがあるかどうかを `v-if` で判定する
- 表示位置と見た目を決める
- 文言の組み立てや判定ロジックを増やしすぎない

> Vue 側は「表示するかどうか」を見るだけにしておくと、  
> 画面が分かりやすくなる

---

## 3-6 エラーはどこから来るのか

### この流れで起きていること

- `form.post()` で Laravel に送信する
- Laravel 側でバリデーションエラーが出る
- エラー情報が Inertia の流れで画面に戻る
- `useForm` が `form.errors` として受け取る

### ここで押さえること

- Vue 側で手動の JSON エラー処理を書いていない
- axios の `catch` でメッセージを組み立てていない
- Laravel と Inertia の標準的な流れをそのまま使っている

### この形の利点

- バリデーションの本体は Laravel 側に置ける
- 画面側はエラー表示の書き方を揃えやすい
- 入力とエラーの流れが一本になる

> 「どこでチェックするか」と「どこで見せるか」を分けると、  
> 構造が崩れにくい

---

## 3-7 この画面で見える Inertia の良さ

### Login.vue の見方

- `form.email` と `form.password` が入力値の本体
- `form.errors.email` と `form.errors.password` がエラー表示の入口
- 同じ `form` の中に入力とエラーがまとまっている

### 設計上のポイント

- 入力値の管理とエラー表示の管理を分断しない
- フロント側に独自ルールを増やしすぎない
- 「送る」「戻る」「表示する」の流れを追いやすくする

---

## 第3章のまとめ

> `useForm` を使うと、  
> **入力値・送信・エラー表示を一つの流れで扱える。**

- `useForm` で入力値の初期状態を定義する
- `v-model` で入力欄と `form` を結びつける
- Laravel 側のバリデーション結果は Inertia 経由で `form.errors` に入る
- Vue 側は `v-if` で必要な場所に表示する
- 画面で余計な状態管理を増やさずに済む
