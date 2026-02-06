# コマンド集

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

```bash
./vendor/bin/sail artisan ziggy:generate resources/js/ziggy.js 
```
