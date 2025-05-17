<!--

## Traverse {#sec:traverse}

`foldLeft` and `foldRight` are flexible iteration methods
but they require us to do a lot of work
to define accumulators and combinator functions.
The `Traverse` type class is a higher level tool
that leverages `Applicatives` to provide
a more convenient, more lawful, pattern for iteration.

### Traversing with Futures

We can demonstrate `Traverse` using
the `Future.traverse` and `Future.sequence` methods in the Scala standard library.
These methods provide `Future`-specific implementations of the traverse pattern.
As an example, suppose we have a list of server hostnames
and a method to poll a host for its uptime:

```scala mdoc:silent
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)

def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60) // just for demonstration
```

Now, suppose we want to poll all of the hosts and collect all of their uptimes.
We can't simply `map` over `hostnames`
because the result---a `List[Future[Int]]`---would
contain more than one `Future`.
We need to reduce the results to a single `Future`
to get something we can block on.
Let's start by doing this manually using a fold:

```scala mdoc:silent
val allUptimes: Future[List[Int]] =
  hostnames.foldLeft(Future(List.empty[Int])) {
    (accum, host) =>
      val uptime = getUptime(host)
      for {
        accum  <- accum
        uptime <- uptime
      } yield accum :+ uptime
  }
```

```scala mdoc
Await.result(allUptimes, 1.second)
```

Intuitively, we iterate over `hostnames`, call `func` for each item,
and combine the results into a list.
This sounds simple, but the code is fairly unwieldy
because of the need to create and combine `Futures` at every iteration.
We can improve on things greatly using `Future.traverse`,
which is tailor-made for this pattern:

```scala mdoc:invisible:reset-object
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)
def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60) // just for demonstration
```
```scala mdoc:silent
val allUptimes: Future[List[Int]] =
  Future.traverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(allUptimes, 1.second)
```

This is much clearer and more concise---let's see how it works.
If we ignore distractions like `CanBuildFrom` and `ExecutionContext`,
the implementation of `Future.traverse` in the standard library looks like this:

```scala
def traverse[A, B](values: List[A])
    (func: A => Future[B]): Future[List[B]] =
  values.foldLeft(Future(List.empty[B])) { (accum, host) =>
    val item = func(host)
    for {
      accum <- accum
      item  <- item
    } yield accum :+ item
  }
```

This is essentially the same as our example code above.
`Future.traverse` is abstracting away the pain of folding
and defining accumulators and combination functions.
It gives us a clean high-level interface to do what we want:

- start with a `List[A]`;
- provide a function `A => Future[B]`;
- end up with a `Future[List[B]]`.

The standard library also provides another method, `Future.sequence`,
that assumes we're starting with a `List[Future[B]]`
and don't need to provide an identity function:

```scala
object Future {
  def sequence[B](futures: List[Future[B]]): Future[List[B]] =
    traverse(futures)(identity)

  // etc...
}
```

In this case the intuitive understanding is even simpler:

- start with a `List[Future[A]]`;
- end up with a `Future[List[A]]`.

`Future.traverse` and `Future.sequence`
solve a very specific problem:
they allow us to iterate over a sequence of `Futures`
and accumulate a result.
The simplified examples above only work with `Lists`,
but the real `Future.traverse` and `Future.sequence`
work with any standard Scala collection.

Cats' `Traverse` type class generalises these patterns
to work with any type of `Applicative`:
`Future`, `Option`, `Validated`, and so on.
We'll approach `Traverse` in the next sections in two steps:
first we'll generalise over the `Applicative`,
then we'll generalise over the sequence type.
We'll end up with an extremely valuable tool that trivialises
many operations involving sequences and other data types.

### Traversing with Applicatives

If we squint, we'll see that we can rewrite `traverse`
in terms of an `Applicative`.
Our accumulator from the example above:

```scala mdoc:silent
Future(List.empty[Int])
```

is equivalent to `Applicative.pure`:

```scala mdoc:silent
import cats.Applicative
import cats.instances.future._   // for Applicative
import cats.syntax.applicative._ // for pure

List.empty[Int].pure[Future]
```

Our combinator, which used to be this:

```scala mdoc:silent
def oldCombine(
  accum : Future[List[Int]],
  host  : String
): Future[List[Int]] = {
  val uptime = getUptime(host)
  for {
    accum  <- accum
    uptime <- uptime
  } yield accum :+ uptime
}
```

is now equivalent to `Semigroupal.combine`:

```scala mdoc:silent
import cats.syntax.apply._ // for mapN

// Combining accumulator and hostname using an Applicative:
def newCombine(accum: Future[List[Int]],
      host: String): Future[List[Int]] =
  (accum, getUptime(host)).mapN(_ :+ _)
```

By substituting these snippets back into the definition of `traverse`
we can generalise it to to work with any `Applicative`:

```scala mdoc:silent

def listTraverse[F[_]: Applicative, A, B]
      (list: List[A])(func: A => F[B]): F[List[B]] =
  list.foldLeft(List.empty[B].pure[F]) { (accum, item) =>
    (accum, func(item)).mapN(_ :+ _)
  }

def listSequence[F[_]: Applicative, B]
      (list: List[F[B]]): F[List[B]] =
  listTraverse(list)(identity)
```

We can use `listTraverse` to re-implement our uptime example:

```scala mdoc:silent
val totalUptime = listTraverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(totalUptime, 1.second)
```

or we can use it with other `Applicative` data types
as shown in the following exercises.

#### Exercise: Traversing with Vectors

What is the result of the following?

```scala mdoc:silent
import cats.instances.vector._ // for Applicative

listSequence(List(Vector(1, 2), Vector(3, 4)))
```

<div class="solution">
The argument is of type `List[Vector[Int]]`,
so we're using the `Applicative` for `Vector`
and the return type is going to be `Vector[List[Int]]`.

`Vector` is a monad,
so its semigroupal `combine` function is based on `flatMap`.
We'll end up with a `Vector` of `Lists`
of all the possible combinations of `List(1, 2)` and `List(3, 4)`:

```scala mdoc
listSequence(List(Vector(1, 2), Vector(3, 4)))
```
</div>

What about a list of three parameters?

```scala mdoc:silent
listSequence(List(Vector(1, 2), Vector(3, 4), Vector(5, 6)))
```

<div class="solution">
With three items in the input list, we end up with combinations of three `Ints`:
one from the first item, one from the second, and one from the third:

```scala mdoc
listSequence(List(Vector(1, 2), Vector(3, 4), Vector(5, 6)))
```
</div>

#### Exercise: Traversing with Options

Here's an example that uses `Options`:

```scala mdoc:silent
import cats.instances.option._ // for Applicative

def process(inputs: List[Int]) =
  listTraverse(inputs)(n => if(n % 2 == 0) Some(n) else None)
```

What is the return type of this method? What does it produce for the following inputs?

```scala mdoc:silent
process(List(2, 4, 6))
process(List(1, 2, 3))
```

<div class="solution">
The arguments to `listTraverse` are of types `List[Int]` and `Int => Option[Int]`,
so the return type is `Option[List[Int]]`.
Again, `Option` is a monad,
so the semigroupal `combine` function follows from `flatMap`.
The semantics are therefore fail-fast error handling:
if all inputs are even, we get a list of outputs.
Otherwise we get `None`:

```scala mdoc
process(List(2, 4, 6))
process(List(1, 2, 3))
```
</div>

#### Exercise: Traversing with Validated

Finally, here is an example that uses `Validated`:

```scala mdoc:invisible:reset
import cats.Applicative
import cats.syntax.applicative._ // for pure
import cats.syntax.apply._ // for mapN
def listTraverse[F[_]: Applicative, A, B]
      (list: List[A])(func: A => F[B]): F[List[B]] =
  list.foldLeft(List.empty[B].pure[F]) { (accum, item) =>
    (accum, func(item)).mapN(_ :+ _)
  }
```
```scala mdoc:silent
import cats.data.Validated
import cats.instances.list._ // for Monoid

type ErrorsOr[A] = Validated[List[String], A]

def process(inputs: List[Int]): ErrorsOr[List[Int]] =
  listTraverse(inputs) { n =>
    if(n % 2 == 0) {
      Validated.valid(n)
    } else {
      Validated.invalid(List(s"$n is not even"))
    }
  }
```

What does this method produce for the following inputs?

```scala mdoc:silent
process(List(2, 4, 6))
process(List(1, 2, 3))
```

<div class="solution">
The return type here is `ErrorsOr[List[Int]]`,
which expands to `Validated[List[String], List[Int]]`.
The semantics for semigroupal `combine` on validated are accumulating error handling,
so the result is either a list of even `Ints`,
or a list of errors detailing which `Ints` failed the test:

```scala mdoc
process(List(2, 4, 6))
process(List(1, 2, 3))
```
</div>


```scala mdoc:reset:silent
```
--->

## `Traverse` {#sec:traverse}

`foldLeft` と `foldRight` は繰り返し処理の柔軟な手段だが、蓄積変数や結合関数を定義するための手間が多い。`Traverse` 型クラスは、`Applicative` を活用することで、より便利で一貫性のある繰り返しパターンを提供する高水準のツールである。

### `Future` のトラバース

Scala 標準ライブラリの `Future.traverse` と `Future.sequence` メソッドを実例として用い `Traverse` を解説していこう。これらのメソッドは、`Future` に特化した Traverse パターンの実装を提供している。たとえば、サーバのホスト名のリストと、あるホストの稼働時間を取得するメソッドがあるとする。

```scala mdoc:silent
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)

def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60) // デモ用の適当な計算
```

ここで、すべてのホストに問い合わせを行い、その稼働時間を収集したいとする。しかし、`hostnames` に対して `map` を行うだけでは、結果は複数の `Future` を含んだ `List[Future[Int]]` になってしまう。処理の完了を待機するには、すべての結果をひとつの `Future` にまとめる必要がある。まずは、畳み込みを使ってこれを自力で実装するところから始めてみよう。

```scala mdoc:silent
val allUptimes: Future[List[Int]] =
  hostnames.foldLeft(Future(List.empty[Int])) {
    (accum, host) =>
      val uptime = getUptime(host)
      for {
        accum  <- accum
        uptime <- uptime
      } yield accum :+ uptime
  }
```

```scala mdoc
Await.result(allUptimes, 1.second)
```

一見すると、この処理は `hostnames` に対して繰り返しを行い、各要素に対して `func` を呼び出し、その結果をリストに追加しているだけである。これは単純なようだが、繰り返しの度に `Future` を生成して結合する必要があるため、コードはかなり煩雑になる。`Future.traverse` はこのようなパターンに特化しており、これを使うことでコードは大幅に改善される。

```scala mdoc:invisible:reset-object
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)
def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60) // デモ用の適当な計算
```
```scala mdoc:silent
val allUptimes: Future[List[Int]] =
  Future.traverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(allUptimes, 1.second)
```

このほうがはるかに明快である。これがどのように実現されているか見てみよう。`CanBuildFrom` や `ExecutionContext` など説明に必要のない部分を無視すれば、標準ライブラリにおける `Future.traverse` の実装は以下のようになる。

```scala
def traverse[A, B](values: List[A])
    (func: A => Future[B]): Future[List[B]] =
  values.foldLeft(Future(List.empty[B])) { (accum, host) =>
    val item = func(host)
    for {
      accum <- accum
      item  <- item
    } yield accum :+ item
  }
```

これは先ほどの例で書いたコードとほぼ同じである。`Future.traverse` は、畳み込みとそれに必要な蓄積変数や結合関数を定義する煩わしさを抽象化によって取り除き、次のような処理を行うためのシンプルで高レベルなインターフェースを提供してくれる。

- `List[A]` からスタートし
- `A => Future[B]` 型の関数を与えれば
- `Future[List[B]]` が得られる

標準ライブラリには `Future.sequence` というメソッドも用意されている。こちらは、最初に `List[Future[B]]` が与えられることを前提としている。恒等関数を変換関数として `traverse` するのと同じことだが、利用者が恒等関数を提供する必要はない。

```scala
object Future {
  def sequence[B](futures: List[Future[B]]): Future[List[B]] =
    traverse(futures)(identity)

  // etc...
}
```

こちらのほうがずっとシンプルである。

- `List[Future[A]]` からスタートし
- `Future[List[A]]` が得られる

`Future.traverse` と `Future.sequence` は、極めて具体的な問題を解決するメソッドである。これらを使うと、`Future` のシーケンスを反復し結果をまとめることができる。先述の簡略化された例では `List` しか受け付けてくれないが、実際の `Future.traverse` と `Future.sequence` は、任意の標準的な Scala コレクションで動作する。

Cats の `Traverse` 型クラスは、これらのパターンを一般化し、`Future`、`Option`、`Validated` など、あらゆる種類の `Applicative` で機能するようにしている。次節では `Traverse` を二段階に分けて説明する。まず `Applicative` について一般化を行い、次にシーケンスの型について一般化する。最終的に、シーケンスと他のデータ型を用いる多くの操作を極めてシンプルにしてくれる、非常に価値のあるツールを手に入れることになる。

### アプリカティブのトラバース

注意深く考えれば、`traverse` を `Applicative` を用いて書き換えられることに気づくだろう。上記の例における蓄積変数の初期値は以下のとおりだった。

```scala mdoc:silent
Future(List.empty[Int])
```

これは以下のように `Applicative.pure` するのと同じである。

```scala mdoc:silent
import cats.Applicative
import cats.instances.future._   // Applicative
import cats.syntax.applicative._ // pure

List.empty[Int].pure[Future]
```

そして、以下のような内容であった結合関数は、

```scala mdoc:silent
def oldCombine(
  accum : Future[List[Int]],
  host  : String
): Future[List[Int]] = {
  val uptime = getUptime(host)
  for {
    accum  <- accum
    uptime <- uptime
  } yield accum :+ uptime
}
```

`Semigroupal.combine` と同じである。

```scala mdoc:silent
import cats.syntax.apply._ // mapN

// アプリカティブを用いて蓄積変数とホスト名を結合する
def newCombine(accum: Future[List[Int]],
      host: String): Future[List[Int]] =
  (accum, getUptime(host)).mapN(_ :+ _)
```

これらのコード片を用いて `traverse` を定義しなおせば、任意の `Applicative` に対応できるよう一般化できる。

```scala mdoc:silent

def listTraverse[F[_]: Applicative, A, B]
      (list: List[A])(func: A => F[B]): F[List[B]] =
  list.foldLeft(List.empty[B].pure[F]) { (accum, item) =>
    (accum, func(item)).mapN(_ :+ _)
  }

def listSequence[F[_]: Applicative, B]
      (list: List[F[B]]): F[List[B]] =
  listTraverse(list)(identity)
```

`listTraverse` を使えば、前述の稼働時間取得の例は以下のように実装しなおすことができる。

```scala mdoc:silent
val totalUptime = listTraverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(totalUptime, 1.second)
```

この関数を他の `Applicative` データ型に対して用いることもできる。以降の演習ではそれを実際に見ていく。

#### 演習: `Vector` のトラバース

次のコードの実行結果はどうなるか考えよ。

```scala mdoc:silent
import cats.instances.vector._ // Applicative

listSequence(List(Vector(1, 2), Vector(3, 4)))
```

<div class="solution">

引数の型が `List[Vector[Int]]` であることから、`Vector` に対する `Applicative` 型クラスインスタンスが使用され、返り値の型は `Vector[List[Int]]` となる。

`Vector` はモナドであるため、その Semigroupal の `combine` 関数は `flatMap` に基づく。結果として、`List(1, 2)` と `List(3, 4)` のすべての組み合わせを表した `List` の `Vector` が得られる。

```scala mdoc
listSequence(List(Vector(1, 2), Vector(3, 4)))
```
</div>

三つの要素をもつリストについても、その結果がどうなるか考えよ。

```scala mdoc:silent
listSequence(List(Vector(1, 2), Vector(3, 4), Vector(5, 6)))
```

<div class="solution">

入力リストに三つの要素がある場合、最初の要素からひとつ、次の要素からひとつ、そして最後の要素からひとつずつ選んだ三つの `Int` の組み合わせが得られる。

```scala mdoc
listSequence(List(Vector(1, 2), Vector(3, 4), Vector(5, 6)))
```
</div>

#### 演習: `Option` のトラバース

以下は `Option` を使った例である。

```scala mdoc:silent
import cats.instances.option._ // Applicative

def process(inputs: List[Int]) =
  listTraverse(inputs)(n => if(n % 2 == 0) Some(n) else None)
```

このメソッドの戻り値型は何か。また以下の入力についてどのような結果が得られるだろうか。

```scala mdoc:silent
process(List(2, 4, 6))
process(List(1, 2, 3))
```

<div class="solution">

`listTraverse` の引数の型が `List[Int]` と `Int => Option[Int]` であることから、返り値の型は `Option[List[Int]]` となる。ここでも `Option` はモナドであるため、Semigroupal の `combine` 関数は `flatMap` に基づく。そのセマンティクスは、フェイルファストなエラーハンドリングである。すべての入力が偶数であれば出力としてリストが得られ、そうでない場合は `None` になる。

```scala mdoc
process(List(2, 4, 6))
process(List(1, 2, 3))
```
</div>

#### 演習: `Validated` のトラバース

最後に `Validated` を用いた例を見ておこう。

```scala mdoc:invisible:reset
import cats.Applicative
import cats.syntax.applicative._ // pure
import cats.syntax.apply._ // mapN
def listTraverse[F[_]: Applicative, A, B]
      (list: List[A])(func: A => F[B]): F[List[B]] =
  list.foldLeft(List.empty[B].pure[F]) { (accum, item) =>
    (accum, func(item)).mapN(_ :+ _)
  }
```
```scala mdoc:silent
import cats.data.Validated
import cats.instances.list._ // Monoid

type ErrorsOr[A] = Validated[List[String], A]

def process(inputs: List[Int]): ErrorsOr[List[Int]] =
  listTraverse(inputs) { n =>
    if(n % 2 == 0) {
      Validated.valid(n)
    } else {
      Validated.invalid(List(s"$n is not even"))
    }
  }
```

このメソッドは下記の入力に対してどのような結果を返すだろうか。

```scala mdoc:silent
process(List(2, 4, 6))
process(List(1, 2, 3))
```

<div class="solution">

返り値の型である `ErrorsOr[List[Int]]` は `Validated[List[String], List[Int]]` に展開される。`Validated` における Semigroupal の `combine` は、エラーハンドリングにおいてエラーを蓄積するセマンティクスをもつ。したがって、結果は偶数の `Int` のリスト、またはどの整数がテストに失敗したかを示すエラーのリストになる。

```scala mdoc
process(List(2, 4, 6))
process(List(1, 2, 3))
```
</div>
