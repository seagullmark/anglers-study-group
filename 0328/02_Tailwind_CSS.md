# 第2章：Tailwind CSS の入口と画面反映

この章では、  
**Vue 画面で Tailwind CSS がどのように読み込まれ、見た目に反映されるか**を整理する。

ポイントは次の3つ。

- `app.js` で **CSS を読み込むところが入口**になる
- `app.css` で **Tailwind を有効化し、監視対象を決める**
- `Login.vue` では **クラス名を直接書いて画面を組み立てる**

---

## 2-1 前提と全体像

本資料では、以下を前提とする。

- フロントエンドは Vue
- スタイルは Tailwind CSS を利用
- 画面単位の装飾は Vue テンプレート内の class で指定する

### 基本フロー（Tailwind CSS）

1. `resources/js/app.js` で `app.css` を読み込む
2. `resources/css/app.css` で Tailwind CSS を有効化する
3. `@source` でクラス検出対象のファイルを指定する
4. Vue ファイルに書かれた class がビルド対象になる
5. 生成された CSS が画面に反映される

> Tailwind CSS は「魔法の見た目設定」ではなく、  
> どこで読み込み、どのファイルを対象にするかを揃えて初めて機能する

---

## 2-2 app.js で CSS を読み込む

### 対象ファイル（入口）

- 入口ファイル：`resources/js/app.js`

```js
import '../css/app.css'
```

### このコードの役割（app.css）

- フロントエンド全体で使う CSS を読み込む
- Tailwind CSS をアプリに反映する入口になる
- 各ページで個別に CSS を import しなくてよい形にする

### ここで押さえること

- Tailwind CSS は Vue ファイルだけあっても効かない
- まず `app.css` が読み込まれていることが前提になる

> 見た目の出発点は Vue ではなく、  
> まず `app.js` で CSS を読み込むところから始まる

---

## 2-3 app.css で Tailwind CSS を有効化する

### 対象ファイル（CSS 設定）

- CSS 定義：`resources/css/app.css`

```css
@import 'tailwindcss';

@source '../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php';
@source '../../storage/framework/views/*.php';
@source '../**/*.blade.php';
@source '../**/*.js';
@source '../**/*.vue';

@theme {
    --font-sans: 'Instrument Sans', ui-sans-serif, system-ui, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji',
        'Segoe UI Symbol', 'Noto Color Emoji';
}
```

### このコードの役割（@theme）

- `@import 'tailwindcss';` で Tailwind CSS を読み込む
- `@source` でクラス検出対象のファイルを指定する
- `@theme` でフォントなど共通テーマを定義する

### `@source` の意味

- Blade に書かれた class も対象にする
- JS に書かれた class も対象にする
- Vue に書かれた class も対象にする
- Laravel のページネーション用ビューも対象にする

### この形にする理由

- 使っている class だけを生成対象にできる
- Vue と Blade が混在していても一つの方針で拾える
- 「class を書いたのに効かない」という事故を減らせる

> Tailwind CSS は class 名を見て CSS を組み立てるため、  
> どのファイルを見に行くかの指定が重要になる

---

## 2-4 `@theme` は共通ルールの置き場

### 対象コード（テーマ）

```css
@theme {
    --font-sans: 'Instrument Sans', ui-sans-serif, system-ui, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji',
        'Segoe UI Symbol', 'Noto Color Emoji';
}
```

### このコードの役割（Login.vue 例）

- アプリ全体で使うフォントの基準を定義する
- Tailwind CSS 側の `font-sans` に対応する共通値を決める
- 各画面で同じ見た目の基準を使えるようにする

### ポイント（Login.vue）

- 画面ごとにフォント指定をばらけさせない
- デザインの基準を CSS 側に寄せる
- Vue 側は部品や class の組み合わせに集中できる

---

## 2-5 Login.vue では class を直接組み立てる

### 対象ファイル（ログイン画面）

- ページ：`resources/js/Pages/Auth/Login.vue`

```vue
<Head title="Login" />

<div class="min-h-screen bg-slate-100 px-4 py-12">
  <div class="mx-auto flex min-h-[calc(100vh-6rem)] max-w-md items-center justify-center">
    <div class="w-full rounded-2xl border border-slate-200 bg-white p-8 shadow-sm">
      <div class="mb-8 text-center">
        <p class="text-sm font-medium tracking-wide text-sky-600">Anglers</p>
        <h1 class="mt-2 text-3xl font-semibold tracking-tight text-slate-900">Log in</h1>
        <p class="mt-2 text-sm text-slate-500">Enter your credentials to continue.</p>
      </div>
```

### このコードの役割（状態切り替え）

- レイアウト、余白、色、文字サイズを class で直接指定する
- 小さなスタイル定義を別 CSS に逃がさず、その場で意図を読める形にする
- コンポーネント単位で見た目を組み立てる

### ここで見えている Tailwind CSS

- `min-h-screen`  
  画面の高さいっぱいまで広げる
- `bg-slate-100`  
  背景色を設定する
- `max-w-md`  
  横幅の上限を決める
- `rounded-2xl`  
  角丸を付ける
- `shadow-sm`  
  軽い影を付ける
- `text-sky-600`  
  テキスト色を設定する

> 「この箱をどう見せたいか」を  
> class の並びでそのまま表現している

---

## 2-6 状態によって class を切り替える

### 対象コード（入力欄）

```vue
:class="{
  'border-rose-400 focus:border-rose-500 focus:ring-rose-100': form.errors.email
}"
```

### このコードの役割

- バリデーションエラー時だけ追加の class を当てる
- 入力欄の状態変化を見た目に反映する
- エラー表示と入力欄の見た目を揃える

### ポイント（状態切り替え）

- 通常時と異常時の見た目を同じ場所で読める
- 条件に応じた class 切り替えが Vue と相性がよい
- スタイルの分岐もテンプレート内で追いやすい

> Tailwind CSS は固定の見た目だけでなく、  
> 状態変化も class の切り替えで扱いやすい

---

## 2-7 この画面で伝えたいこと

### Login.vue の見方

- `Head` はページタイトルの設定
- 外側の `div` は画面全体の余白と背景
- 中央の `div` は配置の調整
- 内側の `div` はカード本体
- `input` と `button` は状態に応じた見た目を持つ

### 設計上のポイント

- ページ構造と見た目の意図が近い場所にある
- CSS ファイルに大量の独自クラスを増やしていない
- 見た目の調整単位が小さいので修正箇所を追いやすい

---

## 第2章のまとめ

> Tailwind CSS を使うときは、  
> **入口・検出対象・画面内の class 記述**の3点を揃えることが基本になる。

- `app.js` で `app.css` を読み込む
- `app.css` で Tailwind CSS を有効化し、`@source` で対象ファイルを指定する
- `@theme` で共通の見た目ルールを置ける
- `Login.vue` では class を直接書いて画面を組み立てる
- エラー時の見た目も `:class` で自然に切り替えられる
