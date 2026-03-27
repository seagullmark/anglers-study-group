# 第5章：`usePage()` と `computed()`

この章では、  
**MainLayout で共有データを受け取り、表示用の値に整える流れ**を整理する。

ポイントは次の3つ。

- `usePage()` で **Inertia のページデータを受け取る**
- `computed()` で **表示用の値を整える**
- レイアウト側で **共通表示をまとめる**

---

## 5-1 前提と全体像

本資料では、以下を前提とする。

- レイアウトは `MainLayout.vue`
- 認証ユーザー情報は Inertia の `page.props` に入っている
- ヘッダーでユーザー情報を共通表示したい

### 基本フロー（共有データの表示）

1. Laravel 側から `auth.user` を Inertia に渡す
2. `usePage()` で `page.props` を受け取る
3. `computed()` で表示しやすい値に整える
4. テンプレートで条件付き表示する
5. どのページでも同じヘッダーを使える

> ここで大事なのは、  
> 各ページで毎回ユーザー表示を書かず、レイアウトにまとめること

---

## 5-2 MainLayout.vue の全体像

### 対象ファイル（共通レイアウト）

- レイアウト：`resources/js/Layouts/MainLayout.vue`

```vue
<template>
  <div class="min-h-screen bg-slate-100">
    <header class="border-b border-slate-200 bg-white">
      <div class="mx-auto flex max-w-5xl items-center justify-between gap-4 px-4 py-4 sm:px-6">
        <p class="text-lg font-semibold text-slate-900">Anglers</p>
        <div v-if="userLabel" class="flex items-center gap-3">
          <div
            v-if="userThumbnail"
            class="h-10 w-10 overflow-hidden rounded-full border border-slate-200 bg-slate-100"
          >
            <img :src="userThumbnail" alt="User profile photo" class="h-full w-full object-cover" />
          </div>
          <div
            v-else
            class="flex h-10 w-10 items-center justify-center rounded-full border border-slate-200 bg-slate-100 text-sm font-semibold text-slate-600"
          >
            {{ userInitial }}
          </div>

          <p class="text-sm text-slate-600">
            <span class="font-medium text-slate-900">{{ userLabel }}</span>
          </p>
        </div>
      </div>
    </header>

    <main class="mx-auto max-w-5xl px-4 py-8 sm:px-6">
      <section class="rounded-2xl border border-slate-200 bg-white p-6 shadow-sm sm:p-8">
        <slot />
      </section>
    </main>
  </div>
</template>
```

### このコードの役割（usePage）

- 画面全体の共通枠を作る
- ヘッダーにユーザー情報を表示する
- `<slot />` で各ページの中身を差し込む

### このレイアウトの意味

- どのページでも同じヘッダーを使える
- ページごとの差は `<slot />` の中身に閉じ込められる
- 共通表示を一箇所で管理できる

---

## 5-3 `usePage()` は Inertia から値を受け取る入口

### 対象コード（ページデータ）

```js
import { usePage } from '@inertiajs/vue3'

const page = usePage()
```

### このコードの役割（computed）

- Inertia が持っている現在ページの情報を受け取る
- `page.props` に入った共有データへアクセスする
- 各ページやレイアウトで共通のデータを参照できるようにする

### ここで押さえること

- `usePage()` は Inertia の機能
- Vue 単体のローカル状態ではない
- Laravel 側から渡されたデータを見るための入口になる

> `usePage()` は、  
> 「今のページに Laravel から何が渡ってきているか」を見るための窓口になる

---

## 5-4 `computed()` で表示用の値に整える

### 対象コード（整形）

```js
import { computed } from 'vue'

const user = computed(() => page.props.auth?.user ?? null)
const userThumbnail = computed(() => user.value?.thumbnail ?? null)
const userLabel = computed(() => user.value?.name || user.value?.email || user.value?.id || null)
const userInitial = computed(() => {
  const source = user.value?.name || user.value?.email || 'U'

  return source.charAt(0).toUpperCase()
})
```

### このコードの役割（テンプレート表示）

- `page.props` の値をそのままテンプレートに広げすぎない
- 表示に必要な値だけを取り出す
- テンプレート内の条件分岐を読みやすくする

### それぞれの意味

- `user`  
  認証ユーザーそのものを取り出す
- `userThumbnail`  
  画像 URL があるかどうかを見る
- `userLabel`  
  名前、メール、ID の順で表示用文字列を決める
- `userInitial`  
  画像がないときに出す頭文字を作る

### `computed()` を使う理由

- 表示用の計算をテンプレートの外に出せる
- 同じ計算を繰り返し書かなくてよい
- テンプレートが「何を表示するか」に集中しやすくなる

> データの元は `page.props` でも、  
> 画面に出す前に一段整えておくと読みやすくなる

---

## 5-5 テンプレート側では条件付き表示に集中する

### 対象コード（表示）

```vue
<div v-if="userLabel" class="flex items-center gap-3">
  <div
    v-if="userThumbnail"
    class="h-10 w-10 overflow-hidden rounded-full border border-slate-200 bg-slate-100"
  >
    <img :src="userThumbnail" alt="User profile photo" class="h-full w-full object-cover" />
  </div>
  <div
    v-else
    class="flex h-10 w-10 items-center justify-center rounded-full border border-slate-200 bg-slate-100 text-sm font-semibold text-slate-600"
  >
    {{ userInitial }}
  </div>

  <p class="text-sm text-slate-600">
    <span class="font-medium text-slate-900">{{ userLabel }}</span>
  </p>
</div>
```

### このコードの役割

- ユーザー表示が必要なときだけヘッダーを出す
- サムネイル画像があれば画像を表示する
- なければ頭文字を表示する
- ユーザー名などの表示文字列を出す

### ポイント

- 判定に使う値は `computed()` 側で整えてある
- テンプレートでは表示の分岐だけを読めばよい
- 「画像あり」「画像なし」の見た目が追いやすい

---

## 5-6 このレイアウトで伝えたいこと

### `usePage()` の見方

- Laravel 側から共有されたデータを見る入口
- ページ単体だけでなく、レイアウトでも使える
- 認証情報のような共通データと相性がよい

### `computed()` の見方

- 元データを表示しやすい形に整える
- テンプレートを読みやすくする
- 画面に出す条件や文字列をまとめる

### 設計上のポイント

- 共通表示はレイアウト側に寄せる
- 各ページで同じユーザー表示を繰り返さない
- `page.props` をテンプレートで直接掘り続けない

---

## 第5章のまとめ

> Inertia の共有データは、  
> **`usePage()` で受け取り、`computed()` で整えてから表示する**と分かりやすい。

- `usePage()` は Inertia のページデータを見る入口
- `page.props.auth.user` のような共有情報を取り出せる
- `computed()` で表示用の値を作るとテンプレートが読みやすくなる
- `MainLayout.vue` に共通表示を置くと、各ページの重複を減らせる
