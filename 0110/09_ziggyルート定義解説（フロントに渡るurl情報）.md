# Ziggy ルート定義の解説（フロントに渡る URL 情報）

このファイルは、Laravel 側で定義された **ルート情報を JSON としてフロントへ渡す** ためのものです。

Ziggy を使うことで、

- Laravel のルート定義
- ルート名と URL の対応関係

を **フロントエンドでも安全に再利用**できます。

---

## 対象コード

```js
const Ziggy = {
  "url": "http://localhost:8000",
  "port": 8000,
  "defaults": {},
  "routes": {
    "login": {"uri": "login", "methods": ["GET", "HEAD"]},
    "login.store": {"uri": "login", "methods": ["POST"]},
    "index": {"uri": "/", "methods": ["GET", "HEAD"]},
    "logout": {"uri": "logout", "methods": ["POST"]},
    "storage.local": {
      "uri": "storage/{path}",
      "methods": ["GET", "HEAD"],
      "wheres": {"path": ".*"},
      "parameters": ["path"]
    }
  }
};

if (typeof window !== 'undefined' && typeof window.Ziggy !== 'undefined') {
  Object.assign(Ziggy.routes, window.Ziggy.routes);
}

export { Ziggy };
```

---

## 1. このファイルの役割

このファイルは、

> **Laravel の routes 定義を、フロントで参照可能な形に変換したもの**

です。

- URL を直接書かない
- ルート名だけで画面遷移・通信を行う

という設計を支えています。

---

## 2. routes オブジェクト

```js
"routes": {
  "login": {"uri": "login"},
  "login.store": {"uri": "login"},
  "index": {"uri": "/"},
  "logout": {"uri": "logout"}
}
```

- キー：Laravel の **ルート名**
- 値：そのルートが指す **URI と HTTP メソッド**

ここにより、フロント側は

> 「この名前のルートは、どの URL か」

を自前で知る必要がなくなります。

---

## 3. window.Ziggy とのマージ

```js
if (typeof window !== 'undefined' && typeof window.Ziggy !== 'undefined') {
  Object.assign(Ziggy.routes, window.Ziggy.routes);
}
```

- サーバ側で埋め込まれた `window.Ziggy` が存在する場合
- その内容を現在の定義にマージします

これにより、

- ビルド時
- 実行時

どちらでも同じ `Ziggy` オブジェクトを使えます。

---

## 4. useZiggyRoute との関係

このファイル単体では、

- ルート定義を **保持しているだけ**

です。

```js
useRoute(Ziggy)
```

と組み合わせることで初めて、

```js
route('login.store')
```

のような呼び出しが可能になります。

---

## 5. なぜ URL を直接書かないのか

この構成の最大のメリットは、

> **URL 構造を変更しても、フロントのコードを直さなくてよい**

という点です。

- `/login` → `/auth/login`
- `/logout` → `/auth/logout`

のような変更も、

- Laravel の routes 定義だけ変更
- フロントはそのまま

で済みます。

---

## 6. 抽象化（設計）の観点

ここでも抽象化が行われています。

- フロントは URL を知らない
- フロントはルート定義ファイルを知らない

知っているのは、

> **この名前のルートが存在する**

という事実だけです。

これは、

- 認証の抽象化
- データ保存先の抽象化

と同じ考え方です。

---

## 7. FileMaker 開発への持ち帰り

FileMaker でも同じことが言えます。

- レイアウト名を直書きしない
- スクリプト名（入口）だけを呼ぶ
- 実体は後から変えられる

こうすることで、

> **変更に強い設計**

になります。

---

## 8. まとめ

- Ziggy は Laravel のルート定義をフロントへ渡す仕組み
- フロントは URL ではなくルート名だけを扱う
- URL 変更の影響範囲を最小化できる

このファイルも、

> **設計のヒントが詰まった重要なピース**

です。
