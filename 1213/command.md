# コマンド集

## Laravel Sail

### コンテナの起動・停止

```bash
# コンテナを停止
./vendor/bin/sail down

# コンテナをバックグラウンドで起動
./vendor/bin/sail up -d
```

### コンテナ内での作業

```bash
# コンテナのシェルに入る
./vendor/bin/sail shell
```

### Artisan コマンド

```bash
# FileMaker用のモデルを作成
./vendor/bin/sail artisan make:model Angler --filemaker
```

### Tinker での使用例

```bash
# Artisan Tinker を起動
./vendor/bin/sail artisan tinker
```

```php
App\Models\Angler::all()
```

## Docker ネットワーク

### ブリッジネットワークの作成

```bash
# ブリッジネットワークを作成
docker network create anglers-network
```

### ネットワークの確認

```bash
# ネットワークに参加しているコンテナなどを確認
docker network inspect anglers-network
```

### コンテナをネットワークに接続

```bash
# コンテナを anglers-network に接続
docker network connect anglers-network コンテナID
```

## 環境変数

`compose.yaml` の `laravel.test` サービスの `environment` セクションに追記：

```yaml
environment:
  LANG: C.UTF-8
  LC_ALL: C.UTF-8
```
