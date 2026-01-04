# Login 画面の解説（Inertia + Laravel Authentication）

このファイルは、Inertia + Vue を使った **ログイン画面**です。

ここでは、

- フロント側は「入力と送信」だけを担当
- 認証の成否判断は **すべてサーバ側（Laravel）** に任せる

という設計になっています。

---

## 対象コード

```vue
<template>
  <div>
    <h1>Login</h1>

    <form @submit.prevent="submit">
      <div>
        <label for="email">Email</label>
        <input id="email" v-model="form.email" type="email" autocomplete="username" required />
        <div v-if="form.errors.email">{{ form.errors.email }}</div>
      </div>

      <div>
        <label for="password">Password</label>
        <input
          id="password"
          v-model="form.password"
          type="password"
          autocomplete="current-password"
          required
        />
        <div v-if="form.errors.password">{{ form.errors.password }}</div>
      </div>

      <button type="submit" :disabled="form.processing">Log in</button>
    </form>
  </div>
</template>

<script setup>
import { useForm } from '@inertiajs/vue3'
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

---

## 1. template：入力フォーム

```vue
<form @submit.prevent="submit">
```

- 通常の HTML フォーム
- `@submit.prevent` で **ページ遷移を抑止**
- 送信処理は JavaScript 側に委ねます

---

## 2. v-model：フォーム状態の管理

```vue
<input v-model="form.email" />
<input v-model="form.password" />
```

- 入力値は `form` オブジェクトに自動反映
- Vue 側で state 管理を意識する必要はありません

---

## 3. useForm（Inertia のフォームヘルパ）

```js
const form = useForm({
  email: '',
  password: ''
})
```

`useForm` は Inertia が提供する便利な仕組みで、

- 入力値
- バリデーションエラー
- 送信中状態

をひとまとめで管理します。

---

## 4. submit：ログイン要求を送る

```js
form.post(route('login.store'), {
  onFinish: () => form.reset('password')
})
```

- `login.store` ルートへ POST
- 認証の判定は **サーバ側（AuthController）** が行います
- フロントは「成功したか失敗したか」を意識しません

### onFinish

- 成否に関係なく呼ばれる
- password フィールドだけをリセット

---

## 5. エラー表示

```vue
<div v-if="form.errors.email">{{ form.errors.email }}</div>
```

- サーバ側で投げられた `ValidationException` が自動で反映されます
- フロント側でエラー判定ロジックを書く必要はありません

---

## 6. 処理中状態（processing）

```vue
<button :disabled="form.processing">Log in</button>
```

- リクエスト中は `form.processing === true`
- 二重送信防止

---

## 7. 抽象化（設計）の観点

この Login 画面は、

- User が FileMaker かどうか
- 認証がどう実装されているか

を一切知りません。

フロントが知っているのは、

> **ログインフォームを送信すれば、結果が返ってくる**

という事実だけです。

これにより、

- フロントは UI に集中できる
- 認証ロジックはサーバ側に閉じる

という責務分離が成立します。

---

## 8. まとめ

- Login 画面は「入力と送信」だけを担当
- 認証判定は Laravel に完全委任
- Inertia により、フロントとバックエンドの境界が明確

この構成も、

> **認証状態を抽象化している**

一例です。
