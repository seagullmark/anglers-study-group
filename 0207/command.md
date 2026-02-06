# コマンド集

## 依存ライブラリのインストール

```bash
./vendor/bin/sail composer require intervention/image
```

- 画像のリサイズ・トリミング等を行うためのライブラリ（Intervention Image）をインストールする
- 本資料では GD ドライバを使用（ImageManager::gd()）

---

## モデル作成（FileMaker 用）

### UserPhoto モデル作成

```bash
./vendor/bin/sail artisan make:model UserPhoto --filemaker
```

- FileMaker の `user_photos` テーブルを Laravel から扱うためのモデル
- テーブルを作成するコマンドではない

---

## Controller 作成

### ContainerController（画像表示）

```bash
./vendor/bin/sail artisan make:controller ContainerController
```

---

### UserPhotoController（アップロード）

```bash
./vendor/bin/sail artisan make:controller UserPhotoController --resource
```

---

## Request 作成（バリデーション・正規化）

### ContainerRequest

```bash
./vendor/bin/sail artisan make:request ContainerRequest
```

---

## ルーティング確認

### 登録されているルート一覧を確認

```bash
./vendor/bin/sail artisan route:list
```

---

## フロントエンド用ルート生成（Ziggy）

```bash
./vendor/bin/sail artisan ziggy:generate resources/js/ziggy.js
```

- Laravel の named route を JavaScript から利用するためのファイルを生成
- Inertia / Vue 側で `route('user-photos.store')` のように使えるようにする
- ルート変更時は **再生成が必要**
