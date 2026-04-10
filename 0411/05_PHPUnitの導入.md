# 第5章：PHPUnit の導入

この章では、  
**今回の教材で出てくる PHPUnit が何で、どういう位置づけのものか**を整理する。

ポイントは次の3つ。

- PHPUnit は PHP のテストフレームワーク
- テストは自動で生えるものではなく、自分でコードを書く
- 今回の教材では認可や Request の確認にも使える

---

## 5-1 PHPUnit とは何か

PHPUnit は、  
**PHP でテストを書くための標準的なテストフレームワーク**。

Laravel でも長く使われていて、

- モデル
- Request
- Policy
- Controller
- サービス

などの動作確認に使える。

### このプロジェクトでの位置づけ

- テストフレームワークは PHPUnit
- 設定ファイルは `anglers/phpunit.xml`
- テストファイルは `anglers/tests/` 配下に置く

---

## 5-2 テストは自分で書くコード

ここは大事。

PHPUnit は、
**何かを自動で保証してくれる魔法の仕組みではない。**

自分で

- 何を確認したいか決める
- その条件を作る
- 期待する結果を書く

という形で、**テスト用のコードを自分で書く。**

### つまりどういうことか

- アプリ本体のコードを書く
- その動きを確かめるためのコードも書く

この 2 本立てになる。

> テストは「後から勝手に付いてくるもの」ではなく、  
> 確認したいことを自分でコードにする作業になる

---

## 5-3 どんな形で書くのか

### 例：`FishingTripPolicyTest.php`

- `anglers/tests/Unit/Policies/FishingTripPolicyTest.php`

```php
public function test_owner_can_view_update_and_delete_trip(): void
{
    $user = new User();
    $user->id = 'user-1';

    $trip = new FishingTrip();
    $trip->user_id = 'user-1';

    $policy = new FishingTripPolicy();

    $this->assertTrue($policy->view($user, $trip));
    $this->assertTrue($policy->update($user, $trip));
    $this->assertTrue($policy->delete($user, $trip));
}
```

### このテストでやっていること

1. ユーザーを用意する
2. 釣行データを用意する
3. `FishingTripPolicy` を呼ぶ
4. 結果が `true` になることを確認する

### ここで押さえること

- テストのために `User` や `FishingTrip` を自分で用意している
- `assertTrue()` で期待結果を書いている
- これが「テスト用に自分でコードを書く」という意味

---

## 5-4 Request のテストも自分で書く

### 例：`FishingTripRequestTest.php`

- `anglers/tests/Unit/Requests/FishingTripRequestTest.php`

```php
public function test_update_request_requires_mod_id_and_remove_photo_ids(): void
{
    $rules = (new UpdateFishingTripRequest())->rules();

    $this->assertArrayHasKey('mod_id', $rules);
    $this->assertArrayHasKey('remove_photo_ids', $rules);
    $this->assertArrayHasKey('remove_photo_ids.*', $rules);
}
```

### このテストで見ていること

- `UpdateFishingTripRequest` に `mod_id` があるか
- `remove_photo_ids` のルールがあるか
- Request の定義が崩れていないか

### ポイント

- ブラウザを開かなくても確認できる
- `FormRequest` のルールをコードとして見張れる
- 今回の教材のように、設計ルールを確認するのに向いている

---

## 5-5 どうやって作るのか

Laravel では、テストファイルの雛形をコマンドで作れる。

### 例

```bash
./vendor/bin/sail artisan make:test FishingTripPolicyTest --unit
./vendor/bin/sail artisan make:test FishingTripRequestTest --unit
```

### 生成されたあとにやること

- テストメソッド名を書く
- 必要なデータを用意する
- `assertTrue()` や `assertFalse()` などで確認を書く

つまり、
**雛形はコマンドで作れるが、中身は自分で書く。**

---

## 5-6 どうやって実行するのか

### PHPUnit を直接実行

```bash
cd anglers
./vendor/bin/phpunit
```

### 特定ファイルだけ実行

```bash
cd anglers
./vendor/bin/phpunit tests/Unit/Policies/FishingTripPolicyTest.php
```

### Laravel 経由で実行

```bash
cd anglers
php artisan test
```

### Sail を使うなら

```bash
./vendor/bin/sail test
```

### 今回の確認で実行した例

```bash
./vendor/bin/phpunit \
  tests/Unit/Policies/FishingTripPolicyTest.php \
  tests/Unit/Policies/UserPhotoPolicyTest.php \
  tests/Unit/Requests/FishingTripRequestTest.php
```

---

## 5-7 今回の教材で PHPUnit をどう見るか

今回のテーマは

- CRUD
- 認可処理
- 楽観的排他

なので、PHPUnit は次の確認に向いている。

- `Policy` の判定が合っているか
- `Request` に必要なルールがあるか
- 競合判定の前提になる `mod_id` が必須になっているか

### 逆に押さえること

- PHPUnit は UI の見た目確認そのものではない
- 画面のデザイン確認より、ロジック確認に向いている
- Laravel 側のルールを小さく確認するのに強い

---

## 5-8 テストコードを AI に書かせてよいか

結論から言うと、  
**書かせてよい。**

ただし、そのまま信用しないことが大事。

### AI と相性がよい理由

テストコードは、

- 条件が明確
- パターンが決まっている
- 繰り返しの形が多い

という理由で、AI と相性がよい。

特に今回のような

- `Policy` の owner / non-owner 判定
- `Request` のルール確認
- `mod_id` の必須チェック

などは、AI に叩き台を作らせやすい。

### 注意点

- 通るだけの薄いテストになりやすい
- 今の実装をなぞるだけで、仕様確認になっていないことがある
- 失敗ケースや境界条件が抜けやすい
- 何を守るテストなのかが曖昧になりやすい

### 現実的な使い方

- 何を確認したいかは人が決める
- テストコードの叩き台は AI に書かせてもよい
- 最後に、そのテストが何を保証しているかを人が確認する

> AI はテストを書く補助にはなるが、  
> テストの意味を決める役割までは代わらない

---

## 第5章のまとめ

> PHPUnit は、  
> **Laravel の動きを確認するために自分で書くテストコードの土台**になる。

- PHPUnit は PHP の標準的なテストフレームワーク
- テストは自分でコードを書く
- `Policy` や `Request` の確認にも使える
- 雛形は `make:test` で作れるが、中身は自分で書く
- 今回の教材では、認可と入力ルールの確認に特に相性がよい
- AI に叩き台を書かせてもよいが、何を守るテストかは人が確認する
