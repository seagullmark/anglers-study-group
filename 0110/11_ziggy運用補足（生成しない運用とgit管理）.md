# Ziggy 運用補足（生成しない運用と Git 管理）

この補足は、教材の Ziggy 周り（`useZiggyRoute` と `ziggy.js`）に対して、

- 本番で `ziggy:generate` が必要か？
- `ziggy.js` を Git 管理してよいか？

を整理したものです。

---

## 1. 結論（この教材の前提）

- **本番で `php artisan ziggy:generate` は不要**
- `ziggy.js` は **Git 管理して OK**

この教材は「実行時注入型（Laravel から `window.Ziggy` を受け取る）」の運用を前提にしています。

---

## 2. Ziggy の運用は 2 パターンある

### A) 生成型（ビルド時固定）

```bash
php artisan ziggy:generate resources/js/ziggy.js
```

- ルート定義をビルド時に JS ファイルへ書き出す
- 環境（URL/port/route）差分でファイルが汚れやすい
- **生成物**として扱うのが自然（Git 管理しない判断も多い）

### B) 実行時注入型（Laravel から受け取る）

- Blade などで `@routes` を使い、実行時に `window.Ziggy` が供給される
- フロントは常に「その環境の routes」を参照できる
- ビルド時にルート定義を固定しない

**本教材は B（実行時注入型）です。**

---

## 3. この教材の `ziggy.js` がやっていること

教材の `ziggy.js` には次のような処理があります。

```js
if (typeof window !== 'undefined' && typeof window.Ziggy !== 'undefined') {
  Object.assign(Ziggy.routes, window.Ziggy.routes);
}
```

意味はこれです。

- ビルド時に用意した最小限の定義（このファイル）
- 実行時に Laravel が提供する最新のルート定義（`window.Ziggy`）

をマージして使う。

> ルートの正（ソース・オブ・トゥルース）は **実行時の Laravel** 側に寄せる

という設計です。

---

## 4. だから `ziggy:generate` をしなくてよい

- `@routes` により実行時に `window.Ziggy` が供給される
- フロントはそれを優先して参照できる

このため、**本番で `ziggy:generate` を回す必要がありません**。

---

## 5. Git 管理の判断

この教材（実行時注入型）の前提では、`ziggy.js` は

- ルート定義を固定する生成物

ではなく、

- Ziggy を使うための **薄いアダプタ／定義ファイル**

として扱えます。

そのため、**Git 管理して OK** です。

※ 逆に A（生成型）で運用する場合は環境差分が出やすいので、
生成先 `ziggy.js` を Git 管理しない（.gitignore）方針も検討対象になります。
