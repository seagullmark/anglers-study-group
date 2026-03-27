# 第4章：`Link` と画面遷移

この章では、  
**Inertia の `Link` を使って、画面遷移をどう書くか**を整理する。

ポイントは次の3つ。

- 画面遷移は **`a` タグではなく `Link` を使う**
- 遷移先は **named route で表現する**
- Vue 側は **「どこへ移動するか」を分かりやすく書く**

---

## 4-1 前提と全体像

本資料では、以下を前提とする。

- 画面は Vue + Inertia で構成する
- Laravel 側に named route がある
- Vue 側では Ziggy 経由で route を使える

### 基本フロー（画面遷移）

1. 画面に遷移用の `Link` を置く
2. `:href="route(...)"` で遷移先を指定する
3. ユーザーがクリックする
4. Inertia が遷移を処理する
5. 対応するページコンポーネントが表示される

> Inertia を使う画面では、  
> 単なる HTML リンクではなく「Inertia の遷移」として書くことが基本になる

---

## 4-2 Index.vue の画面構成

### 対象ファイル（ホーム画面）

- ページ：`resources/js/Pages/Index/Index.vue`

```vue
<template>
  <Head title="Home" />

  <div class="space-y-8">
    <div>
      <p class="text-sm font-medium text-sky-600">Home</p>
      <h1 class="mt-2 text-3xl font-semibold tracking-tight text-slate-900">User Settings</h1>
      <p class="mt-3 max-w-2xl text-sm text-slate-600">
        Choose a page to manage your account settings.
      </p>
    </div>

    <div class="grid gap-6 md:grid-cols-2">
      <Link
        :href="route('user.profile-photo')"
        class="block rounded-2xl border border-slate-200 bg-slate-50 p-6 transition hover:border-sky-300 hover:bg-sky-50"
      >
        <p class="text-sm font-medium text-sky-600">User</p>
        <h2 class="mt-2 text-xl font-semibold text-slate-900">Profile Photo</h2>
        <p class="mt-3 text-sm text-slate-600">
          Upload and replace the image used for your profile.
        </p>
      </Link>
    </div>
  </div>
</template>
```

### このコードの役割（Link）

- ホーム画面としてメニューを表示する
- 各設定画面への導線をカード形式で置く
- クリック可能な領域を `Link` でまとめる

### この画面で見えていること

- `Head` はページタイトルの設定
- 上部はページの説明
- 下部は設定画面への遷移カード
- カード全体がクリック対象になっている

---

## 4-3 `Link` は Inertia の画面遷移用

### 対象コード（遷移）

```vue
<Link
  :href="route('user.profile-photo')"
  class="block rounded-2xl border border-slate-200 bg-slate-50 p-6 transition hover:border-sky-300 hover:bg-sky-50"
>
```

### このコードの役割（route指定）

- クリック時の遷移先を指定する
- Inertia の画面遷移として動かす
- 見た目と遷移の意味を一つの要素にまとめる

### `Link` を使う理由

- Inertia の流れに沿った画面遷移になる
- Vue / Laravel のつながりを崩しにくい
- 「この要素は別ページへ移動する」という意図が明確になる

### `a` タグとの見方の違い

- `a` タグは通常のリンク
- `Link` は Inertia の画面遷移用コンポーネント
- Inertia を使う画面では、遷移を `Link` に寄せると構造が揃う

> 「クリックできる箱」ではなく、  
> 「Inertia で別ページに移動する箱」として読む

---

## 4-4 `:href="route(...)"` の意味

### 対象コード（ルート指定）

```vue
:href="route('user.profile-photo')"
```

### このコードの役割（カード）

- Laravel 側の named route を使って URL を組み立てる
- URL の直書きを避ける
- ルート名で画面遷移先を表現する

### ポイント

- Vue 側は URL 文字列よりルート名を見る
- Laravel 側でルート変更があっても追いやすい
- `settings/profile-photo` のような文字列を各所に散らさずに済む

### ここで見えてくること

- 遷移先の管理は Laravel 側のルート定義が基準
- Vue 側はそのルートを呼び出す形にする
- 画面遷移の出発点と管理元が分かれる

---

## 4-5 `Link` の中にカード全体を入れる意味

### 対象コード（カード）

```vue
<Link ... class="block rounded-2xl border border-slate-200 bg-slate-50 p-6 transition hover:border-sky-300 hover:bg-sky-50">
  <p class="text-sm font-medium text-sky-600">User</p>
  <h2 class="mt-2 text-xl font-semibold text-slate-900">Profile Photo</h2>
  <p class="mt-3 text-sm text-slate-600">
    Upload and replace the image used for your profile.
  </p>
</Link>
```

### このコードの役割

- カード全体をクリック可能にする
- 遷移の分かりやすさを上げる
- 見た目のまとまりと操作のまとまりを一致させる

### 画面としての利点

- クリック範囲が広くなる
- 「どこを押せばよいか」が分かりやすい
- メニュー画面として読みやすい

---

## 4-6 この画面で押さえたいこと

### `Link` の見方

- `Link` は Vue の部品だが、中身は Inertia の画面遷移を担う
- `:href="route(...)"` で Laravel 側のルートとつながる
- クリック後は Inertia の流れで次のページが表示される

### 設計上のポイント

- 画面遷移の書き方を揃える
- URL を直書きしない
- 遷移先をルート名で表現する

---

## 第4章のまとめ

> Inertia で画面遷移を書くときは、  
> **`Link` と named route を組み合わせる**のが基本になる。

- `Link` は Inertia の画面遷移用コンポーネント
- `:href="route('...')"` で Laravel 側のルート名とつなぐ
- URL を直書きせず、ルート名で遷移先を表現する
- カード全体を `Link` にすると、見た目と操作が一致しやすい
