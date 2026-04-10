# 第1章：Vue 側の入口とレイアウト

この章では、  
**Laravel × Vue（Inertia）構成のうち、Vue 側の入口とレイアウトの考え方**を整理する。

ポイントは次の3つ。

- Inertia のページ読込は **`app.js` が入口になる**
- レイアウトの標準は **共通で決めて、例外だけページ側で切り替える**
- 画面送信は **axios ではなく Inertia の流れで行う**

---

## 1-1 前提と全体像

本資料では、以下を前提とする。

- フロントエンドは Vue
- Laravel と Vue の接続に Inertia を利用
- 画面単位のファイルは `resources/js/Pages` 配下に置く

### 基本フロー（Vue 側）

1. Laravel から Inertia でページ名が渡る
2. `app.js` が対応する Vue ファイルを読み込む
3. レイアウト未指定なら `MainLayout` を適用する
4. ページ側で必要なら個別レイアウトに切り替える
5. フォーム送信は `useForm` から Laravel のルートへ流す

> Vue のページは単独で動くのではなく、  
> Inertia の入口を通る前提で組み立てる

---

## 1-2 app.js が入口になる

### 対象ファイル（Inertia 入口）

- 入口ファイル：`resources/js/app.js`

```js
createInertiaApp({
  resolve: (name) => {
    const page = resolvePageComponent(
      `./Pages/${name}.vue`,
      import.meta.glob('./Pages/**/**/*.vue')
    )
    page.then((module) => {
      module.default.layout = module.default.layout || MainLayout
    })

    return page
  },
```

### このコードの役割（app.js）

- Inertia が渡してきたページ名から Vue ファイルを探す
- 読み込んだページに `layout` 指定がなければ `MainLayout` を使う
- 各ページで毎回レイアウトを書かなくてもよいようにする

### この形にする理由

- 共通レイアウトの適用ルールを一箇所に置ける
- ページごとの差分が「例外だけ」になる
- レイアウト方針がページごとに崩れにくい

> 標準を `app.js` に置くと、  
> 「どの画面がどの枠で表示されるか」が追いやすい

---

## 1-3 SingleLayout は最小レイアウト

### 対象ファイル（レイアウト）

- レイアウト：`resources/js/Layouts/SingleLayout.vue`

```vue
<template>
    <slot>Default Layout</slot>
</template>
```

### このコードの役割（Login.vue）

- 子ページの中身だけをそのまま表示する
- 共通ヘッダーやナビを付けない画面に使う
- レイアウトの差し替え先として機能する

### 向いている画面

- ログイン画面
- パスワード再設定画面
- 独立した単体フォーム画面

> 「レイアウトなし」に見えても、  
> 最小の枠として切り出しておくと意図が明確になる

---

## 1-4 Login.vue の作り

### 対象ファイル（ログイン画面）

- ページ：`resources/js/Pages/Auth/Login.vue`

```vue
<script setup>
import { Head, useForm } from '@inertiajs/vue3'
import SingleLayout from '/resources/js/Layouts/SingleLayout.vue'
defineOptions({ layout: SingleLayout })
import { useZiggyRoute } from '@/composables/useZiggyRoute'

const route = useZiggyRoute()

const form = useForm({
  email: '',
  password: ''
})

const submit = () => {
  form.post(route('login.store'), {
    onFinish: () => form.reset('password')
  })
}
</script>
```

### このコードの役割

- `Head` でページタイトルを設定する
- `useForm` で入力値・エラー・送信中状態をまとめて扱う
- `defineOptions({ layout: SingleLayout })` で、この画面だけ個別レイアウトに切り替える
- `form.post()` で Laravel 側の named route に送信する

### 画面側で見えていること

- 入力欄は `v-model` で `form.email` と `form.password` に結びつく
- エラー表示は `form.errors.email` のように直接受ける
- 送信中は `form.processing` でボタン制御できる

### ポイント

- URL を直書きせず `route('login.store')` を使う
- ページは `SingleLayout` を宣言するだけで見た目の枠を切り替えられる
- パスワードだけ `onFinish` でリセットし、不要な値を残さない

> 画面は「入力して送る」に集中し、  
> 通信状態やエラー表示は Inertia の仕組みに乗せる

---

## 1-5 Inertia の流れで送る

### 対象コード（送信）

```js
form.post(route('login.store'), {
  onFinish: () => form.reset('password')
})
```

### この書き方で押さえること

- Vue から直接 API を組み立てて叩いていない
- Inertia のフォーム送信として Laravel に渡している
- バリデーション結果や送信状態を画面側で扱いやすい

### ここで避けたいズレ

- Inertia を使っているのに axios で別の送信経路を作る
- サーバから受けた値を手作業で JSON 整形して渡す
- 画面ごとに通信の書き方が変わる

> 動くことより先に、  
> どの流れで送る画面なのかを揃えることが大事

---

## 1-6 JSON 文字列にして渡さない

### ここで強く押さえること

Inertia では、
Laravel から渡した props は
内部的にシリアライズされて Vue に渡る。

これは Inertia の通常動作であり、
問題ではない。

問題になるのは、
開発側が自分で次のような流れを作ること。

- `json_encode()` する
- 文字列として渡す
- Vue 側で `JSON.parse()` する

### こういうコードはズレる

```php
return inertia('FishingTrips/Form', [
    'trip' => json_encode($trip),
]);
```

```js
const trip = JSON.parse(props.trip)
```

### なぜズレるのか

- Inertia の props の流れを崩す
- Vue 側で余計な変換が必要になる
- データの形が分かりにくくなる
- 「Laravel から何を渡しているか」が曖昧になる

### この教材での方針

- Collection や配列は、そのまま Inertia に渡してよい
- 必要なら `map()` で画面用の形に整える
- ただし JSON 文字列を手で組み立てて渡さない

> Inertia では、  
> **JSON 化そのものが問題ではない。**
>
> **自分で JSON 文字列を作って渡し、Vue 側でまた戻すのがズレ。**

---

## 第1章のまとめ

> Vue 側で最初に整えるべきなのは、  
> **入口・レイアウト・送信方法の3点を揃えること。**

- `app.js` が Inertia ページ読込の入口になる
- 標準レイアウトは共通化し、例外ページだけ個別指定する
- `SingleLayout` は単体画面用の最小レイアウトとして使える
- `Login.vue` は `useForm` と `form.post()` で Inertia の流れに沿って送信する
- Vue は見た目と入力操作に集中し、通信の流れは統一しておく
- JSON 文字列を手で組み立てて渡す形は取らない
