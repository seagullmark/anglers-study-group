# Userモデル差し替えに関する一連の疑問と結論

このドキュメントは、

- FileMaker（Eloquent-FileMaker）を使って
- Laravel の **デフォルト Authentication** を成立させる過程で

生まれた一連の疑問と、その結論をまとめたものです。

技術的な正解だけでなく、
**「なぜそう考えたのか」「なぜその道を選ばなかったのか」**も含めています。

---

## 1. 最初の疑問

> Laravel の User は `Illuminate\Foundation\Auth\User` を使っている。
>
> なら、これを **FMModel ベースに上書き**すればいいだけでは？
>
> なぜこんなに回りくどいことをしているのか？

この疑問は、とても自然です。

---

## 2. いったん思いついた案

```php
use Illuminate\Foundation\Auth\User as Authenticatable;
```

- 標準 User を import して
- 中身の Eloquent を FileMaker に差し替えられないか？

という発想。

過去に ServiceProvider を **差し替え（override）** した経験があれば、
なおさら出てくる考えです。

---

## 3. なぜそれができないのか

### 3-1. メソッドは上書きできるが、クラス構造は上書きできない

PHP では、

- **メソッド（振る舞い）は上書きできる**
- **クラスの継承関係（extends）は上書きできない**

という明確な違いがある。

メソッドは継承や Trait によって後から差し替えられるが、

```php
class User extends Model
```

のような **クラス構造（親クラス）は定義時に確定**し、
ServiceProvider や DI コンテナから差し替える仕組みは存在しない。

---

### 3-2. ServiceProvider の差し替えとは性質が違う

過去に行った「上書き」は、

- ServiceProvider
- コンテナ（binding）

を差し替えるものだった。

`Illuminate\Foundation\Auth\User` は、

- コンテナから解決される部品ではない
- ただのクラス定義

なので、**同じ手法は使えない**。

つまり今回の問題は、

- 「メソッドをどう上書きするか」ではなく
- **「クラスそのもの（extends Model）を差し替えられるか」**という点にある。

vendor を触らない前提では、
`Illuminate\Foundation\Auth\User` の `extends Model` を `extends FMModel` に
置き換える方法は存在しない。

---

## 4. Laravel が用意している正規ルート

Laravel は、User の保存先を差し替えるために、
最初から次の構造を用意している。

- Contract（Interface）
- Trait（実装部品）

---

## 5. Contract と Trait の役割

### Contract（implements）

- 「この User は、こう振る舞えます」という **約束**
- Laravel は **約束だけを見る**

### Trait（use）

- その約束を満たすための **実装**

👉 **約束と実装が分離されている**

---

## 6. 今回採用した構成

```php
class User extends FMModel implements
    AuthenticatableContract,
    AuthorizableContract,
    CanResetPasswordContract,
    MustVerifyEmailContract
{
    use Authenticatable, Authorizable, CanResetPassword, MustVerifyEmail, Notifiable;
}
```

これは、

> **Illuminate\Foundation\Auth\User を FMModel 用に再構成したもの**

であり、

- 回りくどい
- 妥協案

ではなく、

👉 **Laravel が想定している唯一の正攻法**。

---

## 7. 「上書きしたい」気持ちへの整理

### 7-1. 本当にやりたいことは何か

- 標準 User の見た目を使いたい？
- Trait を毎回書きたくない？

であれば、

> **自分用の基底 User クラスを作る**

のが最適。

```php
abstract class FmAuthenticatableUser extends FMModel implements ...
{
    use Authenticatable, Authorizable, ...;
}
```

---

### 7-2. vendor を直接上書きする選択肢

- fork
- composer patch

は技術的には可能だが、

- Laravel アップデートで壊れる
- 依存パッケージとの整合が崩れる

ため、**教材・本番ともに非推奨**。

---

## 8. 結論

- 「標準 User を上書きしたい」という発想は正しい
- しかし Laravel は **その方法を許していない設計**
- 代わりに Contract + Trait という拡張点を用意している

結果として、

> **FMModel 上で標準 User を再構成する**

という現在の実装が、

- 最も安全
- 最も Laravel 的
- 最も教材向き

という結論に至った。

---

## 9. 学び

この一連の試行錯誤から得られたのは、

- Laravel の認証の仕組み
- 抽象化と責務分離の意味
- 他言語を学ぶことで FileMaker の設計が洗練されること

だった。

> 回り道に見えたが、
> 実は **設計の核心に触れる最短ルート**だった。
