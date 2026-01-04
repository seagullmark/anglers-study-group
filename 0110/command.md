# コマンド集

## Artisan Tinker

```bash
# Artisan Tinker を起動
./vendor/bin/sail artisan tinker
```

```php
# パスワードのハッシュ化
Hash::make('laravel');
```

## Artisan コマンド

```bash
# コントローラーを作成
./vendor/bin/sail artisan make:controller AuthController

# Ziggy のルート定義を生成（JavaScriptでLaravelのルートを使用するため）
./vendor/bin/sail artisan ziggy:generate resources/js/ziggy.js
```
