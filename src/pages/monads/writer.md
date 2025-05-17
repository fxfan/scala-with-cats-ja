<!--

## The Writer Monad {#writer-monad}

[`cats.data.Writer`][cats.data.Writer]
is a monad that lets us carry a log along with a computation.
We can use it to record messages, errors,
or additional data about a computation,
and extract the log alongside the final result.

One common use for `Writers` is
recording sequences of steps in multi-threaded computations
where standard imperative logging techniques
can result in interleaved messages from different contexts.
With `Writer` the log for the computation is tied to the result,
so we can run concurrent computations without mixing logs.

<div class="callout callout-info">
*Cats Data Types*

`Writer` is the first data type we've seen
from the [`cats.data`][cats.data.package] package.
This package provides instances of various type classes
that produce useful semantics.
Other examples from `cats.data` include
the monad transformers that we will see in the next chapter,
and the [`Validated`][cats.data.Validated] type
we will encounter in Chapter [@sec:applicatives].
</div>

### Creating and Unpacking Writers

A `Writer[W, A]` carries two values:
a *log* of type `W` and a *result* of type `A`.
We can create a `Writer` from values of each type as follows:

```scala mdoc:silent
import cats.data.Writer
import cats.instances.vector._ // for Monoid
```

```scala mdoc
Writer(Vector(
  "It was the best of times",
  "it was the worst of times"
), 1859)
```

Notice that the type reported on the console
is actually `WriterT[Id, Vector[String], Int]`
instead of `Writer[Vector[String], Int]` as we might expect.
In the spirit of code reuse,
Cats implements `Writer` in terms of another type, `WriterT`.
`WriterT` is an example of a new concept called a *monad transformer*,
which we will cover in the next chapter.

Let's try to ignore this detail for now.
`Writer` is a type alias for `WriterT`,
so we can read types like `WriterT[Id, W, A]` as `Writer[W, A]`:

```scala
type Writer[W, A] = WriterT[Id, W, A]
```

For convenience, Cats provides a way of creating `Writers`
specifying only the log or the result.
If we only have a result we can use the standard `pure` syntax.
To do this we must have a `Monoid[W]` in scope
so Cats knows how to produce an empty log:

```scala mdoc:silent
import cats.instances.vector._   // for Monoid
import cats.syntax.applicative._ // for pure

type Logged[A] = Writer[Vector[String], A]
```

```scala mdoc
123.pure[Logged]
```

If we have a log and no result
we can create a `Writer[Unit]` using the `tell` syntax
from [`cats.syntax.writer`][cats.syntax.writer]:

```scala mdoc:silent
import cats.syntax.writer._ // for tell
```

```scala mdoc
Vector("msg1", "msg2", "msg3").tell
```

If we have both a result and a log,
we can either use `Writer.apply`
or we can use the `writer` syntax
from [`cats.syntax.writer`][cats.syntax.writer]:

```scala mdoc:silent
import cats.syntax.writer._ // for writer
```

```scala mdoc
val a = Writer(Vector("msg1", "msg2", "msg3"), 123)
val b = 123.writer(Vector("msg1", "msg2", "msg3"))
```

We can extract the result and log from a `Writer`
using the `value` and `written` methods respectively:

```scala mdoc
val aResult: Int =
  a.value
val aLog: Vector[String] =
  a.written
```

We can extract both values at the same time using the `run` method:

```scala mdoc
val (log, result) = b.run
```

### Composing and Transforming Writers

The log in a `Writer` is preserved when we `map` or `flatMap` over it.
`flatMap` appends the logs from the source `Writer`
and the result of the user's sequencing function.
For this reason it's good practice to use a log type
that has an efficient append and concatenate operations,
such as a `Vector`:

```scala mdoc
val writer1 = for {
  a <- 10.pure[Logged]
  _ <- Vector("a", "b", "c").tell
  b <- 32.writer(Vector("x", "y", "z"))
} yield a + b

writer1.run
```

In addition to transforming the result with `map` and `flatMap`,
we can transform the log in a `Writer` with the `mapWritten` method:

```scala mdoc
val writer2 = writer1.mapWritten(_.map(_.toUpperCase))

writer2.run
```

We can transform both log and result simultaneously using `bimap` or `mapBoth`.
`bimap` takes two function parameters, one for the log and one for the result.
`mapBoth` takes a single function that accepts two parameters:

```scala mdoc
val writer3 = writer1.bimap(
  log => log.map(_.toUpperCase),
  res => res * 100
)

writer3.run

val writer4 = writer1.mapBoth { (log, res) =>
  val log2 = log.map(_ + "!")
  val res2 = res * 1000
  (log2, res2)
}

writer4.run
```

Finally, we can clear the log with the `reset` method
and swap log and result with the `swap` method:

```scala mdoc
val writer5 = writer1.reset

writer5.run

val writer6 = writer1.swap

writer6.run
```

### Exercise: Show Your Working

`Writers` are useful for logging operations in multi-threaded environments.
Let's confirm this by computing (and logging) some factorials.

The `factorial` function below computes a factorial
and prints out the intermediate steps as it runs.
The `slowly` helper function ensures this takes a while to run,
even on the very small examples below:

```scala mdoc:silent
def slowly[A](body: => A) =
  try body finally Thread.sleep(100)

def factorial(n: Int): Int = {
  val ans = slowly(if(n == 0) 1 else n * factorial(n - 1))
  println(s"fact $n $ans")
  ans
}
```

Here's the output---a sequence of monotonically increasing values:

```scala mdoc
factorial(5)
```

If we start several factorials in parallel,
the log messages can become interleaved on standard out.
This makes it difficult to see
which messages come from which computation:

```scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent.duration._

Await.result(Future.sequence(Vector(
  Future(factorial(5)),
  Future(factorial(5))
)), 5.seconds)
// fact 0 1
// fact 0 1
// fact 1 1
// fact 1 1
// fact 2 2
// fact 2 2
// fact 3 6
// fact 3 6
// fact 4 24
// fact 4 24
// fact 5 120
// fact 5 120
// res: scala.collection.immutable.Vector[Int] =
//   Vector(120, 120)
```

<!--
HACK: tut isn't capturing stdout from the threads above,
so i gone done hacked it.
--

Rewrite `factorial` so it captures
the log messages in a `Writer`.
Demonstrate that this allows us to
reliably separate the logs
for concurrent computations.

<div class="solution">
We'll start by defining a type alias for `Writer`
so we can use it with `pure` syntax:

```scala mdoc:silent:reset-object
import cats.data.Writer
import cats.instances.vector._
import cats.syntax.applicative._ // for pure

type Logged[A] = Writer[Vector[String], A]
```

```scala mdoc
42.pure[Logged]
```

We'll import the `tell` syntax as well:

```scala mdoc:silent
import cats.syntax.writer._ // for tell
```

```scala mdoc
Vector("Message").tell
```

Finally, we'll import
the `Semigroup` instance for `Vector`.
We need this to `map` and `flatMap` over `Logged`:

```scala mdoc:silent
import cats.instances.vector._ // for Monoid
```

```scala mdoc
41.pure[Logged].map(_ + 1)
```

With these in scope, the definition of `factorial` becomes:

```scala mdoc:invisible
def slowly[A](body: => A) =
  try body finally Thread.sleep(10)
```
```scala mdoc:silent
def factorial(n: Int): Logged[Int] =
  for {
    ans <- if(n == 0) {
             1.pure[Logged]
           } else {
             slowly(factorial(n - 1).map(_ * n))
           }
    _   <- Vector(s"fact $n $ans").tell
  } yield ans
```

When we call `factorial`,
we now have to `run` the return value
to extract the log and our factorial:

```scala mdoc
val (log, res) = factorial(5).run
```

We can run several `factorials` in parallel as follows,
capturing their logs independently
without fear of interleaving:

```scala
Await.result(Future.sequence(Vector(
  Future(factorial(5)),
  Future(factorial(5))
)).map(_.map(_.written)), 5.seconds)
// res: scala.collection.immutable.Vector[cats.Id[Vector[String]]] = 
//   Vector(
//     Vector(fact 0 1, fact 1 1, fact 2 2, fact 3 6, fact 4 24, fact 5 120), 
//     Vector(fact 0 1, fact 1 1, fact 2 2, fact 3 6, fact 4 24, fact 5 120)
//   )
```

<!--
HACK: There is a deadlock in the REPL that prevents the code above from working
(see https://github.com/scala/bug/issues/9076) so i gone done hacked it.
--
</div>


```scala mdoc:reset:silent
```
--->

## `Writer` モナド {#writer-monad}

[`cats.data.Writer`][cats.data.Writer] は、計算にログを付随させることのできるモナドである。これを使えば、計算に関するメッセージやエラー、追加データを記録し、最終的な計算結果とともにそれらのログを取り出すことができる。

`Writer` の一般的な用途のひとつは、マルチスレッド環境での計算におけるステップの記録である。通常の命令型のロギング技術では、異なるコンテキストからのメッセージが混在してしまう可能性があるが、`Writer` を使えば、計算のログは結果に結びつけられるので、ログを混在させることなく並行計算を行うことができる。

<div class="callout callout-info">
*Cats のデータ型*

`Writer` は、[`cats.data`][cats.data.package] パッケージに属する中から紹介する最初のデータ型である。このパッケージは、便利なセマンティクスを生み出すさまざまな型クラスのインスタンスを提供している。`cats.data` に含まれる他の例として、次章で見るモナド変換子や、[@sec:applicatives]章に登場する [`Validated`][cats.data.Validated] 型がある。
</div>

### `Writer` の作成と展開

`Writer[W, A]` はふたつの値をもつ。`W` 型の*ログ*と `A` 型の*結果*である。以下のようにすれば、それぞれの型の値から `Writer` を作成することができる。

```scala mdoc:silent
import cats.data.Writer
import cats.instances.vector._ // Monoid
```

```scala mdoc
Writer(Vector(
  "It was the best of times",
  "it was the worst of times"
), 1859)
```

コンソールに表示される型が想定される `Writer[Vector[String], Int]` ではなく、実際には `WriterT[Id, Vector[String], Int]` であることに注意してほしい。コードの再利用の観点から、Cats は `Writer` を別の型 `WriterT` を使って実装している。`WriterT` は次章で新たに解説する*モナド変換子*という概念の一例である。

そういった詳細は今は無視しよう。`Writer` は `WriterT` の型エイリアスであるため、`WriterT[Id, W, A]` という型を `Writer[W, A]` と読み替えてよい。

```scala
type Writer[W, A] = WriterT[Id, W, A]
```

簡便のため、Cats はログまたは結果のみを指定して `Writer` を作成する方法も提供している。結果だけから作成するのであれば、標準の `pure` 構文を使用できる。この場合、空のログを生成する方法について Cats が知る必要があるので、`Monoid[W]` がスコープ内になければならない。

```scala mdoc:silent
import cats.instances.vector._   // Monoid
import cats.syntax.applicative._ // pure

type Logged[A] = Writer[Vector[String], A]
```

```scala mdoc
123.pure[Logged]
```

手元にログしかない場合は、[`cats.syntax.writer`][cats.syntax.writer] が提供する `tell` 構文を使って `Writer[Unit]` インスタンスを作成することができる。

```scala mdoc:silent
import cats.syntax.writer._ // tell
```

```scala mdoc
Vector("msg1", "msg2", "msg3").tell
```

結果とログがどちらもある場合は、`Writer.apply` か、もしくは　[`cats.syntax.writer`][cats.syntax.writer]　が提供する `writer` 構文を使うことができる。

```scala mdoc:silent
import cats.syntax.writer._ // writer
```

```scala mdoc
val a = Writer(Vector("msg1", "msg2", "msg3"), 123)
val b = 123.writer(Vector("msg1", "msg2", "msg3"))
```

`Writer` の `value` と　`written` メソッドを使えば、それぞれ結果とログを取り出すことができる。

```scala mdoc
val aResult: Int =
  a.value
val aLog: Vector[String] =
  a.written
```

`run` メソッドを使って両方の値を同時に取り出すこともできる。

```scala mdoc
val (log, result) = b.run
```

### `Writer` の合成と変換

`Writer` に格納されているログは、`map` や `flatMap` を行ってもそのまま保持される。`flatMap` は元の `Writer` のログと、指定された変換関数の結果から得られたログを結合する。そのため、`Vector` のように効率的な追加や連結操作ができる型をログに使用することが推奨される。

```scala mdoc
val writer1 = for {
  a <- 10.pure[Logged]
  _ <- Vector("a", "b", "c").tell
  b <- 32.writer(Vector("x", "y", "z"))
} yield a + b

writer1.run
```

`map` や `flatMap` を使って結果を変換するのに加えて、`Writer` 内のログを `mapWritten` メソッドで変換することもできる。

```scala mdoc
val writer2 = writer1.mapWritten(_.map(_.toUpperCase))

writer2.run
```

`bimap` または `mapBoth` を使えばログと結果を同時に変換できる。`bimap` はログ変換用と結果変換用のふたつの関数を引数にとり、`mapBoth` はログと結果のふたつのパラメータを受け取るひとつの関数を引数にとる。

```scala mdoc
val writer3 = writer1.bimap(
  log => log.map(_.toUpperCase),
  res => res * 100
)

writer3.run

val writer4 = writer1.mapBoth { (log, res) =>
  val log2 = log.map(_ + "!")
  val res2 = res * 1000
  (log2, res2)
}

writer4.run
```

最後に、ログを消去することのできる `reset` メソッドと、ログと結果の入れ替えを行う `swap` メソッドを紹介する。

```scala mdoc
val writer5 = writer1.reset

writer5.run

val writer6 = writer1.swap

writer6.run
```

### 演習: Show Your Working

`Writer` はマルチスレッド環境でのロギング処理で役に立つ。いくつかの階乗の計算を並列実行し、そのログを記録することでこれを確認してみよう。

以下に示す `factorial` 関数は階乗を計算し、計算の途中ステップを出力する。この例はとても小さなものだが、実行に時間がかかるように `slowly` というヘルパー関数を用いている。

```scala mdoc:silent
def slowly[A](body: => A) =
  try body finally Thread.sleep(100)

def factorial(n: Int): Int = {
  val ans = slowly(if(n == 0) 1 else n * factorial(n - 1))
  println(s"fact $n $ans")
  ans
}
```

結果は以下のとおりである。単調に増加する一連の値が出力される。

```scala mdoc
factorial(5)
```

階乗計算をいくつか並列して開始すると、標準出力に出力されるログメッセージが交じり合い、どれがどの計算からのメッセージなのか見分けるのが難しくなる。

```scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent.duration._

Await.result(Future.sequence(Vector(
  Future(factorial(5)),
  Future(factorial(5))
)), 5.seconds)
// fact 0 1
// fact 0 1
// fact 1 1
// fact 1 1
// fact 2 2
// fact 2 2
// fact 3 6
// fact 3 6
// fact 4 24
// fact 4 24
// fact 5 120
// fact 5 120
// res: scala.collection.immutable.Vector[Int] =
//   Vector(120, 120)
```

ログメッセージが `Writer` に記録されるよう `factorial` を書き換えよ。それにより、並行計算におけるログを計算ごとに分離できることを示せ。

<div class="solution">
まずは、`Writer` に型エイリアスを定義し、`pure` 構文で扱えるようにするところから始めよう。

```scala mdoc:silent:reset-object
import cats.data.Writer
import cats.instances.vector._
import cats.syntax.applicative._ // pure

type Logged[A] = Writer[Vector[String], A]
```

```scala mdoc
42.pure[Logged]
```

`tell` 構文もインポートしておく。

```scala mdoc:silent
import cats.syntax.writer._ // tell
```

```scala mdoc
Vector("Message").tell
```

最後に `Vector` 用の `Semigroup` インスタンスもインポートしておこう。これは `Logged` に対して `map` や `flatMap` する際に必要となる。

```scala mdoc:silent
import cats.instances.vector._ // Monoid
```

```scala mdoc
41.pure[Logged].map(_ + 1)
```

これらをスコープに置いた上で、`factorial` の定義は以下のようになる。

```scala mdoc:invisible
def slowly[A](body: => A) =
  try body finally Thread.sleep(10)
```
```scala mdoc:silent
def factorial(n: Int): Logged[Int] =
  for {
    ans <- if(n == 0) {
             1.pure[Logged]
           } else {
             slowly(factorial(n - 1).map(_ * n))
           }
    _   <- Vector(s"fact $n $ans").tell
  } yield ans
```

`factorial` を呼び出したら、次はその戻り値に対して `run` を実行し、ログと算出された階乗値を取り出す。

```scala mdoc
val (log, res) = factorial(5).run
```

以下のように、ログが混ざってしまう心配なく複数の `factorial` を並列実行し、計算ごとにログを収集することができる。

```scala
Await.result(Future.sequence(Vector(
  Future(factorial(5)),
  Future(factorial(5))
)).map(_.map(_.written)), 5.seconds)
// res: scala.collection.immutable.Vector[cats.Id[Vector[String]]] = 
//   Vector(
//     Vector(fact 0 1, fact 1 1, fact 2 2, fact 3 6, fact 4 24, fact 5 120), 
//     Vector(fact 0 1, fact 1 1, fact 2 2, fact 3 6, fact 4 24, fact 5 120)
//   )
```
</div>
