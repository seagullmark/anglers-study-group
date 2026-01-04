# Index 画面の解説（認証後画面と認証状態の利用）

このファイルは、**ログイン後に表示される画面**です。

ここでは、

- 認証済みユーザーだけがアクセスできる
- フロント側で「認証状態」をどう扱うか

を最小構成で示しています。

---

## 対象コード

```vue
<template>
  <div>
    <h1>Index</h1>
    <p>You are logged in.</p>
    <p>User ID: {{ userId }}</p>
    <Link :href="route('logout')" method="post" as="button">Log out</Link>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { Link, usePage } from '@inertiajs/vue3'

const page = usePage()
const userId = computed(() => page.props.auth?.user?.id ?? 'unknown')
</script>
```

---

## 1. この画面が表示される条件

この Index 画面は、

- `routes/web.php` で `auth` ミドルウェアが付与されている

ため、**ログイン済みユーザーのみ**がアクセスできます。

未ログインの場合は、

- Application 設定により `/login` へリダイレクト

されます。

---

## 2. usePage：サーバから渡された props を受け取る

```js
const page = usePage()
```

- `usePage()` は Inertia が提供するフック
- サーバ側から渡された props にアクセスできます

ここで使っている `page.props` は、

- `HandleInertiaRequests` ミドルウェアで定義された共有 props

です。

---

## 3. 認証状態の取得（auth.user）

```js
const userId = computed(() => page.props.auth?.user?.id ?? 'unknown')
```

- `auth.user` が存在すればログイン済み
- 存在しなければ `null`

ここでは教材用に、

- `id` だけを表示

しています。

重要なのは、

> フロント側は「User ID が取れるかどうか」だけを見ている

という点です。

---

## 4. なぜ FileMaker を意識しないのか

この画面では、

- User が FileMaker に保存されているか
- 認証がどう実装されているか

を一切知りません。

`auth.user.id` が取れれば、

> 「ログインしている」

という判断ができます。

---

## 5. ログアウトリンク（Inertia Link）

```vue
<Link :href="route('logout')" method="post" as="button">Log out</Link>
```

- Inertia の `Link` コンポーネントを使用
- POST メソッドで `/logout` を呼び出します
- フルリロードせずにログアウト処理が行われます

---

## 6. 抽象化（設計）の観点

この Index 画面は、

- 認証の実装
- User の保存先

を完全に意識していません。

フロント側が扱っているのは、

> **認証状態の結果（ログインしているかどうか）**

だけです。

これにより、

- フロントは表示ロジックに集中できる
- 認証の変更がフロントに波及しない

という設計になります。

---

## 7. まとめ

- Index 画面は認証後専用の画面
- `usePage().props` から認証状態を取得
- FileMaker の存在を意識せずに動作

この画面も、

> **認証状態が抽象化されている**

ことを示す一例です。
