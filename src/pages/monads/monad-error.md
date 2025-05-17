<!--

## Aside: Error Handling and MonadError

Cats provides an additional type class called `MonadError`
that abstracts over `Either`-like data types
that are used for error handling.
`MonadError` provides extra operations
for raising and handling errors.

<div class="callout callout-info">
#### This Section is Optional! {-}

You won't need to use `MonadError`
unless you need to abstract over error handling monads.
For example, you can use `MonadError`
to abstract over `Future` and `Try`,
or over `Either` and `EitherT`
(which we will meet in Chapter [@sec:monad-transformers]).

If you don't need this kind of abstraction right now,
feel free to skip onwards to Section [@sec:monads:eval].
</div>

### The MonadError Type Class

Here is a simplified version
of the definition of `MonadError`:

```scala
package cats

trait MonadError[F[_], E] extends Monad[F] {
  // Lift an error into the `F` context:
  def raiseError[A](e: E): F[A]

  // Handle an error, potentially recovering from it:
  def handleErrorWith[A](fa: F[A])(f: E => F[A]): F[A]
  
  // Handle all errors, recovering from them:
  def handleError[A](fa: F[A])(f: E => A): F[A]

  // Test an instance of `F`,
  // failing if the predicate is not satisfied:
  def ensure[A](fa: F[A])(e: E)(f: A => Boolean): F[A]
}
```

`MonadError` is defined in terms of two type parameters:

- `F` is the type of the monad;
- `E` is the type of error contained within `F`.

To demonstrate how these parameters fit together,
here's an example where we
instantiate the type class for `Either`:

```scala mdoc:silent
import cats.MonadError

type ErrorOr[A] = Either[String, A]

val monadError = MonadError[ErrorOr, String]
```

<div class="callout callout-warning">
*ApplicativeError*

In reality, `MonadError` extends another type class
called `ApplicativeError`.
However, we won't encounter `Applicatives`
until Chapter [@sec:applicatives].
The semantics are the same for each type class
so we can ignore this detail for now.
</div>

### Raising and Handling Errors

The two most important methods of `MonadError`
are `raiseError` and `handleErrorWith`.
`raiseError` is like the `pure` method for `Monad`
except that it creates an instance representing a failure:

```scala mdoc
val success = monadError.pure(42)
val failure = monadError.raiseError("Badness")
```

`handleErrorWith` is the complement of `raiseError`.
It allows us to consume an error and (possibly)
turn it into a success,
similar to the `recover` method of `Future`:

```scala mdoc
monadError.handleErrorWith(failure) {
  case "Badness" =>
    monadError.pure("It's ok")

  case _ =>
    monadError.raiseError("It's not ok")
}
```

If we know we can handle all possible errors 
we can use `handleWith`.

```scala mdoc
monadError.handleError(failure) {
  case "Badness" => 42

  case _ => -1
}
```

There is another useful method called `ensure`
that implements `filter`-like behaviour.
We test the value of a successful monad with a predicate
and specify an error to raise if the predicate returns `false`:

```scala mdoc
monadError.ensure(success)("Number too low!")(_ > 1000)
```

Cats provides syntax for `raiseError` and `handleErrorWith`
via [`cats.syntax.applicativeError`][cats.syntax.applicativeError]
and `ensure` via [`cats.syntax.monadError`][cats.syntax.monadError]:

```scala mdoc:invisible:reset
import cats.MonadError

type ErrorOr[A] = Either[String, A]
```
```scala mdoc:silent
import cats.syntax.applicative.*      // for pure
import cats.syntax.applicativeError.* // for raiseError etc
import cats.syntax.monadError.*       // for ensure
```

```scala mdoc
val success = 42.pure[ErrorOr]
val failure = "Badness".raiseError[ErrorOr, Int]
failure.handleErrorWith{
  case "Badness" =>
    256.pure

  case _ =>
    ("It's not ok").raiseError
}
success.ensure("Number to low!")(_ > 1000)
```

There are other useful variants of these methods.
See the source of [`cats.MonadError`][cats.MonadError]
and [`cats.ApplicativeError`][cats.ApplicativeError]
for more information.

### Instances of MonadError

Cats provides instances of `MonadError`
for numerous data types including
`Either`, `Future`, and `Try`.
The instance for `Either` is customisable to any error type,
whereas the instances for `Future` and `Try`
always represent errors as `Throwables`:

```scala mdoc:silent
import scala.util.Try

val exn: Throwable =
  new RuntimeException("It's all gone wrong")
```

```scala mdoc
exn.raiseError[Try, Int]
```

### Exercise: Abstracting

Implement a method `validateAdult` with the following signature

```scala
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  ???
```

When passed an `age` greater than or equal to 18 it should return that value as a success. Otherwise it should return a error represented as an `IllegalArgumentException`.

```scala mdoc:invisible
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  if(age >= 18) age.pure[F]
  else new IllegalArgumentException("Age must be greater than or equal to 18").raiseError[F, Int]
```

Here are some examples of use.

```scala mdoc
validateAdult[Try](18)
validateAdult[Try](8)
type ExceptionOr[A] = Either[Throwable, A]
validateAdult[ExceptionOr](-1)
```

<div class="solution">
We can solve this using `pure` and `raiseError`. Note the use of type parameters to these methods, to aid type inference.

```scala mdoc:invisible:reset-object
import cats.MonadError
import cats.syntax.all.*
```
```scala mdoc:silent
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  if(age >= 18) age.pure[F]
  else new IllegalArgumentException("Age must be greater than or equal to 18").raiseError[F, Int]
```
</div>


```scala mdoc:reset:silent
```
--->

## 補足: エラーハンドリングと `MonadError`

Cats は `MonadError` と呼ばれる型クラスを付加的に提供している。これは、エラーハンドリングに使用される `Either` のようなデータ型を抽象化するためのものである。`MonadError` は、エラーを発生させたり処理したりするための追加の操作を提供する。

<div class="callout callout-info">
#### この節は任意である {-}

複数種類のエラーハンドリング用モナドを抽象化したいのでないかぎり、`MonadError` を使用する必要はない。`MonadError` は、たとえば `Future` と `Try` あるいは `Either` と `EitherT`（[@sec:monad-transformers]章に登場する）などを抽象化するときに使用する。

さしあたってそのような抽象化が必要ないのであれば、この節をスキップして[@sec:monads:eval]節に進んでも構わない。
</div>

### `MonadError` 型クラス

以下に `MonadError` の定義を簡略化したものを示す。

```scala
package cats

trait MonadError[F[_], E] extends Monad[F] {
  // エラーを `F` のコンテキストへとリフトする
  def raiseError[A](e: E): F[A]

  // エラーをハンドリングする。エラーはリカバリ可能かもしれないしそうでないかもしれない
  def handleErrorWith[A](fa: F[A])(f: E => F[A]): F[A]
  
  // エラーをハンドリングし、必ずリカバリする
  def handleError[A](fa: F[A])(f: E => A): F[A]

  // `F` のインスタンスを検証し、条件を満たさなかった場合に失敗させる
  def ensure[A](fa: F[A])(e: E)(f: A => Boolean): F[A]
}
```

`MonadError` は次のふたつの型パラメータに基づいて定義される。

- `F` はモナドの型
- `E` は `F` 内に含まれるエラーの型

これらの型パラメータがどのように指定されるのかを示す。ここでは `Either` に対してこの型クラスをインスタンス化する例を見てみよう。

```scala mdoc:silent
import cats.MonadError

type ErrorOr[A] = Either[String, A]

val monadError = MonadError[ErrorOr, String]
```

<div class="callout callout-warning">
*`ApplicativeError`*

実際には `MonadError` は `ApplicativeError` という別の型クラスを拡張している。だが、`Applicative` は[@sec:applicatives]章まで登場しない。このふたつの型クラスは意味的に同じなので、今のところ詳細は無視しても問題ない。
</div>

### エラーの発生とハンドリング

`MonadError` のもっとも重要なメソッドは `raiseError` と `handleErrorWith` である。`raiseError` はモナドにおける `pure` メソッドと似ているが、作成するのは失敗を表すインスタンスである。

```scala mdoc
val success = monadError.pure(42)
val failure = monadError.raiseError("Badness")
```

`handleErrorWith` は `raiseError` を補完するメソッドで、エラーを処理し、場合によっては成功に変換することを可能にする。これは `Future` の `recover` メソッドに似ている。

```scala mdoc
monadError.handleErrorWith(failure) {
  case "Badness" =>
    monadError.pure("It's ok")

  case _ =>
    monadError.raiseError("It's not ok")
}
```

起こり得るエラーをすべてハンドリングできるとわかっているのであれば、`handleError` を利用することができる。

```scala mdoc
monadError.handleError(failure) {
  case "Badness" => 42

  case _ => -1
}
```

また、`filter` のような振る舞いを実装した `ensure` と呼ばれる便利なメソッドもある。成功したモナドの値を述語でテストし、述語が `false` を返した場合に発生させるエラーを指定することができる。

```scala mdoc
monadError.ensure(success)("Number too low!")(_ > 1000)
```

Cats は `raiseError` と `handleErrorWith` に関する構文を [`cats.syntax.applicativeError`][cats.syntax.applicativeError] で、そして `ensure` については [`cats.syntax.monadError`][cats.syntax.monadError] で提供している。

```scala mdoc:invisible:reset
import cats.MonadError

type ErrorOr[A] = Either[String, A]
```
```scala mdoc:silent
import cats.syntax.applicative.*      // pure
import cats.syntax.applicativeError.* // raiseError など
import cats.syntax.monadError.*       // ensure
```

```scala mdoc
val success = 42.pure[ErrorOr]
val failure = "Badness".raiseError[ErrorOr, Int]
failure.handleErrorWith{
  case "Badness" =>
    256.pure

  case _ =>
    ("It's not ok").raiseError
}
success.ensure("Number to low!")(_ > 1000)
```

これらのメソッドには他にも便利なバリエーションがある。詳しくは [`cats.MonadError`][cats.MonadError] と [`cats.ApplicativeError`][cats.ApplicativeError] のソースコードを参照してほしい。

### `MonadError` のインスタンス

Cats は、`Either`、`Future`、`Try` など、さまざまなデータ型に対する `MonadError` のインスタンスを提供している。`Future` と `Try` のインスタンスはエラーを常に `Throwable` で表現するが、`Either` に対するインスタンスはエラー型をカスタマイズ可能である。

```scala mdoc:silent
import scala.util.Try

val exn: Throwable =
  new RuntimeException("It's all gone wrong")
```

```scala mdoc
exn.raiseError[Try, Int]
```

### 演習: 抽象化

次のシグネチャをもつメソッド `validateAdult` を実装せよ。

```scala
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  ???
```

渡された `age` が18歳以上の場合、その値を成功として返し、そうでない場合は `IllegalArgumentException` で表されるエラーを返すこと。

```scala mdoc:invisible
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  if(age >= 18) age.pure[F]
  else new IllegalArgumentException("Age must be greater than or equal to 18").raiseError[F, Int]
```

使い方の例を以下に示す。

```scala mdoc
validateAdult[Try](18)
validateAdult[Try](8)
type ExceptionOr[A] = Either[Throwable, A]
validateAdult[ExceptionOr](-1)
```

<div class="solution">
この課題は `pure` と `raiseError` を使用することで解決できる。型推論を助けるために、これらのメソッドに型パラメータを指定している点に注意してほしい。

```scala mdoc:invisible:reset-object
import cats.MonadError
import cats.syntax.all.*
```
```scala mdoc:silent
def validateAdult[F[_]](age: Int)(implicit me: MonadError[F, Throwable]): F[Int] =
  if(age >= 18) age.pure[F]
  else new IllegalArgumentException("Age must be greater than or equal to 18").raiseError[F, Int]
```
</div>
