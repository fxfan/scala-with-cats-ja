<!--

## Either

Let's look at another useful monad:
the `Either` type from the Scala standard library.
In Scala 2.11 and earlier,
many people didn't consider `Either` a monad
because it didn't have `map` and `flatMap` methods.
In Scala 2.12, however, `Either` became *right biased*.

### Left and Right Bias

In Scala 2.11, `Either` had no default
`map` or `flatMap` method.
This made the Scala 2.11 version of `Either`
inconvenient to use in for comprehensions.
We had to insert calls to `.right`
in every generator clause:

```scala mdoc:silent:reset-object
val either1: Either[String, Int] = Right(10)
val either2: Either[String, Int] = Right(32)
```

```scala
for {
  a <- either1.right
  b <- either2.right
} yield a + b
```

In Scala 2.12, `Either` was redesigned.
The modern `Either` makes the decision
that the right side represents the success case
and thus supports `map` and `flatMap` directly.
This makes for comprehensions much more pleasant:

```scala mdoc
for {
  a <- either1
  b <- either2
} yield a + b
```

Cats back-ports this behaviour to Scala 2.11
via the `cats.syntax.either` import,
allowing us to use right-biased `Either`
in all supported versions of Scala.
In Scala 2.12+ we can either omit this import
or leave it in place without breaking anything:

```scala mdoc:silent
import cats.syntax.either.* // for map and flatMap

for {
  a <- either1
  b <- either2
} yield a + b
```

### Creating Instances

In addition to creating instances of `Left` and `Right` directly,
we can also import the `asLeft` and `asRight` extension methods
from [`cats.syntax.either`][cats.syntax.either]:

```scala mdoc:silent
import cats.syntax.either.* // for asRight
```

```scala mdoc
val a = 3.asRight[String]
val b = 4.asRight[String]

for {
  x <- a
  y <- b
} yield x*x + y*y
```

These "smart constructors" have
advantages over `Left.apply` and `Right.apply`
because they return results of type `Either`
instead of `Left` and `Right`.
This helps avoid type inference problems
caused by over-narrowing,
like the issue in the example below:

```scala mdoc:fail
def countPositive(nums: List[Int]) =
  nums.foldLeft(Right(0)) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

This code fails to compile for two reasons:

1. the compiler infers the type of the accumulator
   as `Right` instead of `Either`;
2. we didn't specify type parameters for `Right.apply`
   so the compiler infers the left parameter as `Nothing`.

Switching to `asRight` avoids both of these problems.
`asRight` has a return type of `Either`,
and allows us to completely specify the type
with only one type parameter:

```scala mdoc:silent
def countPositive(nums: List[Int]) =
  nums.foldLeft(0.asRight[String]) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

```scala mdoc
countPositive(List(1, 2, 3))
countPositive(List(1, -2, 3))
```

`cats.syntax.either` adds
some useful extension methods
to the `Either` companion object.
The `catchOnly` and `catchNonFatal` methods
are great for capturing `Exceptions`
as instances of `Either`:

```scala mdoc
Either.catchOnly[NumberFormatException]("foo".toInt)
Either.catchNonFatal(sys.error("Badness"))
```

There are also methods for creating an `Either`
from other data types:

```scala mdoc
Either.fromTry(scala.util.Try("foo".toInt))
Either.fromOption[String, Int](None, "Badness")
```

### Transforming Eithers

`cats.syntax.either` also adds
some useful methods for instances of `Either`.

Users of Scala 2.11 or 2.12 
can use `orElse` and `getOrElse` to extract
values from the right side or return a default:

```scala mdoc:silent
import cats.syntax.either.*
```

```scala mdoc
"Error".asLeft[Int].getOrElse(0)
"Error".asLeft[Int].orElse(2.asRight[String])
```

The `ensure` method allows us
to check whether the right-hand value
satisfies a predicate:

```scala mdoc
-1.asRight[String].ensure("Must be non-negative!")(_ > 0)
```

The `recover` and `recoverWith` methods
provide similar error handling to their namesakes on `Future`:

```scala mdoc
"error".asLeft[Int].recover {
  case _: String => -1
}

"error".asLeft[Int].recoverWith {
  case _: String => Right(-1)
}
```

There are `leftMap` and `bimap` methods to complement `map`:

```scala mdoc
"foo".asLeft[Int].leftMap(_.reverse)
6.asRight[String].bimap(_.reverse, _ * 7)
"bar".asLeft[Int].bimap(_.reverse, _ * 7)
```

The `swap` method lets us exchange left for right:

```scala mdoc
123.asRight[String]
123.asRight[String].swap
```

Finally, Cats adds a host of conversion methods:
`toOption`, `toList`, `toTry`, `toValidated`, and so on.

### Error Handling

`Either` is typically used to implement fail-fast error handling.
We sequence computations using `flatMap` as usual.
If one computation fails,
the remaining computations are not run:

```scala mdoc
for {
  a <- 1.asRight[String]
  b <- 0.asRight[String]
  c <- if(b == 0) "DIV0".asLeft[Int]
       else (a / b).asRight[String]
} yield c * 100
```

When using `Either` for error handling,
we need to determine
what type we want to use to represent errors.
We could use `Throwable` for this:

```scala mdoc:silent
type Result[A] = Either[Throwable, A]
```

This gives us similar semantics to `scala.util.Try`.
The problem, however, is that `Throwable`
is an extremely broad type.
We have (almost) no idea about what type of error occurred.

Another approach is to define an algebraic data type
to represent errors that may occur in our program:

```scala mdoc:silent
enum LoginError {
  case UserNotFound(username: String)

  case PasswordIncorrect(username: String)

  case UnexpectedError 
}
```

```scala mdoc:silent
case class User(username: String, password: String)

type LoginResult = Either[LoginError, User]
```

This approach solves the problems we saw with `Throwable`.
It gives us a fixed set of expected error types
and a catch-all for anything else that we didn't expect.
We also get the safety of exhaustivity checking
on any pattern matching we do:

```scala mdoc:silent
import LoginError.*

// Choose error-handling behaviour based on type:
def handleError(error: LoginError): Unit =
  error match {
    case UserNotFound(u) =>
      println(s"User not found: $u")

    case PasswordIncorrect(u) =>
      println(s"Password incorrect: $u")

    case UnexpectedError =>
      println(s"Unexpected error")
  }
```

```scala mdoc
val result1: LoginResult = User("dave", "passw0rd").asRight
val result2: LoginResult = UserNotFound("dave").asLeft

result1.fold(handleError, println)
result2.fold(handleError, println)
```

### Exercise: What is Best?

Is the error handling strategy in the previous examples
well suited for all purposes?
What other features might we want from error handling?

<div class="solution">
This is an open question.
It's also kind of a trick question---the
answer depends on the semantics we're looking for.
Some points to ponder:

- Error recovery is important when processing large jobs.
  We don't want to run a job for a day
  and then find it failed on the last element.

- Error reporting is equally important.
  We need to know what went wrong,
  not just that something went wrong.

- In a number of cases, we want to collect all the errors,
  not just the first one we encountered.
  A typical example is validating a web form.
  It's a far better experience to
  report all errors to the user when they submit a form
  than to report them one at a time.
</div>


```scala mdoc:reset:silent
```
--->

## `Either`

もうひとつ有用なモナドを見てみよう。Scala 標準ライブラリの `Either` 型である。Scala 2.11 以前は、`Either` は `map` や `flatMap` メソッドをもたなかったため、多くの人は `Either` をモナドとは考えていなかった。しかし、Scala 2.12 から `Either`  は*右バイアス（right biased）*となった。

### 左バイアスと右バイアス

Scala 2.11 では `Either` はデフォルトの `map` や `flatMap` メソッドをもっておらず、for 内包表記で使うには不便だった。各ジェネレーター句に `.right` 呼び出しを挿入する必要があったのである。

```scala mdoc:silent:reset-object
val either1: Either[String, Int] = Right(10)
val either2: Either[String, Int] = Right(32)
```

```scala
for {
  a <- either1.right
  b <- either2.right
} yield a + b
```

Scala 2.12 で `Either` は再設計された。現在の `Either` では、右側が成功ケースを表すという決定がなされ、それに伴い `map` と `flatMap` が直接サポートされている。これにより、for 内包表記における `Either` の利用は非常に快適になった。

```scala mdoc
for {
  a <- either1
  b <- either2
} yield a + b
```

Cats はこの動作を `cats.syntax.either` のインポートを通じて Scala 2.11 にバックポートし、サポートしているすべての Scala バージョンで右バイアスの `Either` を使用できるようにしている。Scala 2.12 以降では、このインポートを省略することもできるし、残しておいても不都合が生じることはない。

```scala mdoc:silent
import cats.syntax.either.* // map と flatMap

for {
  a <- either1
  b <- either2
} yield a + b
```

### インスタンスの作成

直接 `Left` と `Right` のインスタンスを作成する他に、[`cats.syntax.either`][cats.syntax.either] から拡張メソッドの `asLeft` と `asRight` をインポートして使用することもできる。

```scala mdoc:silent
import cats.syntax.either.* // asRight
```

```scala mdoc
val a = 3.asRight[String]
val b = 4.asRight[String]

for {
  x <- a
  y <- b
} yield x*x + y*y
```

これらのスマートコンストラクタには `Left.apply` や `Right.apply` にない利点がある。それは、これらが `Left` や `Right` 型ではなく、`Either` 型の結果を返すことである。これにより、過度に型を狭めることによって生じる型推論の問題を回避できる。どのような問題か、以下に例を挙げよう。

```scala mdoc:fail
def countPositive(nums: List[Int]) =
  nums.foldLeft(Right(0)) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

このコードはふたつの理由によりコンパイルに失敗する。

1. コンパイラが蓄積変数の型を `Either` ではなく `Right` と推論してしまう
2. `Right.apply` の型パラメータを指定していないため、コンパイラが左側のパラメータを `Nothing` と推論してしまう

`asRight` に切り替えることで、これらの問題を回避できる。`asRight` は `Either` を戻り値型とし、型パラメータひとつだけで型を完全に指定することができる。

```scala mdoc:silent
def countPositive(nums: List[Int]) =
  nums.foldLeft(0.asRight[String]) { (accumulator, num) =>
    if(num > 0) {
      accumulator.map(_ + 1)
    } else {
      Left("Negative. Stopping!")
    }
  }
```

```scala mdoc
countPositive(List(1, 2, 3))
countPositive(List(1, -2, 3))
```

`cats.syntax.either` は、`Either` コンパニオンオブジェクトに便利な拡張メソッドをいくつか追加する。`catchOnly` と `catchNonFatal` は、例外を `Either` インスタンスとして捕捉するために非常に役立つメソッドである。

```scala mdoc
Either.catchOnly[NumberFormatException]("foo".toInt)
Either.catchNonFatal(sys.error("Badness"))
```

他のデータ型から `Either` インスタンスを作成するメソッドもある。

```scala mdoc
Either.fromTry(scala.util.Try("foo".toInt))
Either.fromOption[String, Int](None, "Badness")
```

### `Either` の変換

`cats.syntax.either` は `Either` のインスタンスに対しても便利なメソッドをいくつか追加する。

Scala 2.11 または 2.12 の利用者は、右側の値を取り出すかもしくはデフォルト値を返す、という処理に `orElse` と `getOrElse` を使用することができる。

```scala mdoc:silent
import cats.syntax.either.*
```

```scala mdoc
"Error".asLeft[Int].getOrElse(0)
"Error".asLeft[Int].orElse(2.asRight[String])
```

`ensure` メソッドを使えば、右側の値がある条件を満たすかどうかを確認することができる。

```scala mdoc
-1.asRight[String].ensure("Must be non-negative!")(_ > 0)
```

`recover` と `recoverWith` メソッドは、`Future` における同名のメソッドと同様のエラーハンドリングを提供する。

```scala mdoc
"error".asLeft[Int].recover {
  case _: String => -1
}

"error".asLeft[Int].recoverWith {
  case _: String => Right(-1)
}
```

`map` を補完する `leftMap` や `bimap` メソッドも用意されている。

```scala mdoc
"foo".asLeft[Int].leftMap(_.reverse)
6.asRight[String].bimap(_.reverse, _ * 7)
"bar".asLeft[Int].bimap(_.reverse, _ * 7)
```

`swap` メソッドは左右を入れ替えてくれる。

```scala mdoc
123.asRight[String]
123.asRight[String].swap
```

また、Cats は `toOption`、`toList`、`toTry`、`toValidated` などさまざまな変換メソッドを追加してくれる。

### エラーハンドリング

`Either` は通常、フェイルファストなエラーハンドリングを実装するために使用される。計算はいつもどおり `flatMap` を使って順に連結する。そのどこかで計算が失敗した場合、それ以降の計算は実行されない。

```scala mdoc
for {
  a <- 1.asRight[String]
  b <- 0.asRight[String]
  c <- if(b == 0) "DIV0".asLeft[Int]
       else (a / b).asRight[String]
} yield c * 100
```

`Either` を使ったエラーハンドリングでは、どの型を使用してエラーを表現するのかを決める必要がある。`Throwable` を使うこともできる。

```scala mdoc:silent
type Result[A] = Either[Throwable, A]
```

これは `scala.util.Try` を使うのとほぼ同じ意味合いをもつ。問題は、`Throwable` が非常に広範な型であり、どのような種類のエラーが発生したのかほとんどわからないことである。

別のアプローチとして、代数的データ型を定義し、それによってプログラムで発生しうるエラーを表す方法がある。

```scala mdoc:silent
enum LoginError {
  case UserNotFound(username: String)

  case PasswordIncorrect(username: String)

  case UnexpectedError 
}
```

```scala mdoc:silent
case class User(username: String, password: String)

type LoginResult = Either[LoginError, User]
```

このアプローチなら `Throwable` が抱えていた問題を解決できる。想定されるエラーの種類を固定したセットとして定義し、それ以外の予期しないエラーについてもすべて捕捉することができる。また、パターンマッチングの際には網羅性チェックによる安全性を得ることができる。

```scala mdoc:silent
import LoginError.*

// 型に応じてエラーのハンドリング内容を決定する
def handleError(error: LoginError): Unit =
  error match {
    case UserNotFound(u) =>
      println(s"User not found: $u")

    case PasswordIncorrect(u) =>
      println(s"Password incorrect: $u")

    case UnexpectedError =>
      println(s"Unexpected error")
  }
```

```scala mdoc
val result1: LoginResult = User("dave", "passw0rd").asRight
val result2: LoginResult = UserNotFound("dave").asLeft

result1.fold(handleError, println)
result2.fold(handleError, println)
```

### 演習: 最適なのはどれか

前の例で示したエラーハンドリング戦略は、どのような目的にも適しているだろうか。エラーハンドリングには他にどのような機能が必要か考察せよ。

<div class="solution">
これはオープンクエスチョンである。また、一種のひっかけ問題でもある。答えは達成したい内容によって異なるからだ。以下に、よく考えるべきポイントをいくつか挙げよう。

- 大規模なジョブを処理する際には、エラーからのリカバリが重要である。一日かけてジョブを実行して、最後の段階になって失敗したことが判明する、ということは避けたい。

- エラーのレポートも同様に重要である。何かがうまくいかなかった、ではなく、具体的に何が失敗したのかを知る必要がある。

- 最初に遭遇したエラーだけでなく、すべてのエラーを収集したいケースがよくある。典型的な例は Web フォームの検証である。ユーザがフォームを送信したとき、すべてのエラーを一度に報告する方が、ひとつずつ順番に報告するよりもはるかによいユーザ体験となる。
</div>
