<!--

## Semigroupal Applied to Different Types

`Semigroupal` doesn't always provide the behaviour we expect,
particularly for types that also have instances of `Monad`.
We have seen the behaviour of the `Semigroupal` for `Option`.
Let's look at some examples for other types.

**Future**

The semantics for `Future`
provide parallel as opposed to sequential execution:

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.future._ // for Semigroupal
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val futurePair = Semigroupal[Future].
  product(Future("Hello"), Future(123))
```

```scala mdoc
Await.result(futurePair, 1.second)
```

The two `Futures` start executing the moment we create them,
so they are already calculating results
by the time we call `product`.
We can use apply syntax to zip fixed numbers of `Futures`:

```scala mdoc:silent
import cats.syntax.apply._ // for mapN

case class Cat(
  name: String,
  yearOfBirth: Int,
  favoriteFoods: List[String]
)

val futureCat = (
  Future("Garfield"),
  Future(1978),
  Future(List("Lasagne"))
).mapN(Cat.apply)
```

```scala mdoc
Await.result(futureCat, 1.second)
```

**List**

Combining `Lists` with `Semigroupal`
produces some potentially unexpected results.
We might expect code like the following to *zip* the lists,
but we actually get the cartesian product of their elements:

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.list._ // for Semigroupal
```

```scala mdoc
Semigroupal[List].product(List(1, 2), List(3, 4))
```

This is perhaps surprising.
Zipping lists tends to be a more common operation.
We'll see why we get this behaviour in a moment.

**Either**

We opened this chapter with a discussion of
fail-fast versus accumulating error-handling.
We might expect `product` applied to `Either`
to accumulate errors instead of fail fast.
Again, perhaps surprisingly,
we find that `product` implements
the same fail-fast behaviour as `flatMap`:

```scala mdoc:silent
import cats.instances.either._ // for Semigroupal

type ErrorOr[A] = Either[Vector[String], A]
```

```scala mdoc
Semigroupal[ErrorOr].product(
  Left(Vector("Error 1")),
  Left(Vector("Error 2"))
)
```

In this example `product` sees the first failure and stops,
even though it is possible to examine the second parameter
and see that it is also a failure.

### Semigroupal Applied to Monads

The reason for the surprising results
for `List` and `Either` is that they are both monads.
If we have a monad we can implement `product` as follows.

```scala mdoc:silent
import cats.Monad
import cats.syntax.functor._ // for map
import cats.syntax.flatMap._ // for flatmap

def product[F[_]: Monad, A, B](fa: F[A], fb: F[B]): F[(A,B)] =
  fa.flatMap(a => 
    fb.map(b =>
      (a, b)
    )
  )
```

It would be very strange
if we had different semantics
for `product` depending
on how we implemented it.
To ensure consistent semantics,
Cats' `Monad` (which extends `Semigroupal`)
provides a standard definition of `product`
in terms of `map` and `flatMap`
as we showed above.

Even our results for `Future` are a trick of the light.
`flatMap` provides sequential ordering,
so `product` provides the same.
The parallel execution we observe
occurs because our constituent `Futures`
start running before we call `product`.
This is equivalent to the classic
create-then-flatMap pattern:

```scala mdoc:silent
val a = Future("Future 1")
val b = Future("Future 2")

for {
  x <- a
  y <- b
} yield (x, y)
```

So why bother with `Semigroupal` at all?
The answer is that we can create useful data types that
have instances of `Semigroupal` (and `Applicative`) but not `Monad`.
This frees us to implement `product` in different ways.
We'll examine this further in a moment
when we look at an alternative data type for error handling.

#### Exercise: The Product of Lists

Why does `product` for `List`
produce the Cartesian product?
We saw an example above.
Here it is again.

```scala mdoc
Semigroupal[List].product(List(1, 2), List(3, 4))
```

We can also write this in terms of `tupled`.

```scala mdoc
(List(1, 2), List(3, 4)).tupled
```

<div class="solution">
This exercise is checking that you understood
the definition of `product` in terms of
`flatMap` and `map`.

```scala mdoc:invisible:reset-object
import cats.Monad
```
```scala mdoc:silent
import cats.syntax.functor._ // for map
import cats.syntax.flatMap._ // for flatMap

def product[F[_]: Monad, A, B](x: F[A], y: F[B]): F[(A, B)] =
  x.flatMap(a => y.map(b => (a, b)))
```

This code is equivalent to a for comprehension:

```scala mdoc:invisible:reset-object
import cats.Monad
import cats.syntax.flatMap._ // for flatMap
import cats.syntax.functor._ // for map
```
```scala mdoc:silent
def product[F[_]: Monad, A, B](x: F[A], y: F[B]): F[(A, B)] =
  for {
    a <- x
    b <- y
  } yield (a, b)
```

The semantics of `flatMap` are what give rise
to the behaviour for `List` and `Either`:

```scala mdoc:silent
import cats.instances.list._ // for Semigroupal
```

```scala mdoc
product(List(1, 2), List(3, 4))
```
</div>


```scala mdoc:reset:silent
```
--->

## さまざまな型に対する `Semigroupal`

`Semigroupal` はいつも期待どおりの動作をしてくれるわけではない。特に、型に対して `Monad` インスタンスも同時に定義されている場合、挙動は期待どおりにはならない。すでに `Option` に対する `Semigroupal` の振る舞いを見てきたが、ここで他の型についての例も見てみよう。

**Future**

`Future` のセマンティクスは逐次実行の代わりに並列実行を提供する。

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.future._ // Semigroupal
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val futurePair = Semigroupal[Future].
  product(Future("Hello"), Future(123))
```

```scala mdoc
Await.result(futurePair, 1.second)
```

ふたつの `Future` は生成された瞬間から実行を開始し、`product` を呼び出す時点ですでに両方の結果の計算が進行している。apply 構文を使えば、所定の数の `Future` をひとつにまとめることができる。

```scala mdoc:silent
import cats.syntax.apply._ // mapN

case class Cat(
  name: String,
  yearOfBirth: Int,
  favoriteFoods: List[String]
)

val futureCat = (
  Future("Garfield"),
  Future(1978),
  Future(List("Lasagne"))
).mapN(Cat.apply)
```

```scala mdoc
Await.result(futureCat, 1.second)
```

**List**

`Semigroupal` で `List` を結合して得られる結果は予想とは異なるかもしれない。次のコードを見てほしい。`zip` メソッドと同じような結果を期待するかもしれないが、実際には要素同士のデカルト積が得られる。

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.list._ // Semigroupal
```

```scala mdoc
Semigroupal[List].product(List(1, 2), List(3, 4))
```

これはおそらく意外に感じられるだろう。リストにおいては zip 操作のほうが一般的に思えるからである。なぜこのような挙動になるのかについては、すぐ後で説明する。

**Either**

この章の冒頭ではフェイルファストと蓄積型エラーハンドリングについて比較した。`Either` に `product` を適用すると複数のエラーがひとつにまとめられることを期待するかもしれない。しかし、再び驚かされるかもしれないが、`product` は `flatMap` と同じくフェイルファストな振る舞いを実装している。

```scala mdoc:silent
import cats.instances.either._ // Semigroupal

type ErrorOr[A] = Either[Vector[String], A]
```

```scala mdoc
Semigroupal[ErrorOr].product(
  Left(Vector("Error 1")),
  Left(Vector("Error 2"))
)
```

この例では、`product` は最初の失敗を検出した時点で処理を中断する。ふたつ目のパラメータを調べ、そちらも失敗であると確認することは可能なはずだが、それが実行されることはない。

### モナドに対する `Semigroupal`

`List` や `Either` に対して予想外の結果が得られる理由は、これらがどちらもモナドだからである。モナドに対して `product` は次のように実装される。

```scala mdoc:silent
import cats.Monad
import cats.syntax.functor._ // map
import cats.syntax.flatMap._ // flatmap

def product[F[_]: Monad, A, B](fa: F[A], fb: F[B]): F[(A,B)] =
  fa.flatMap(a => 
    fb.map(b =>
      (a, b)
    )
  )
```

実装方法によって `product` のセマンティクスが異なるとしたらおかしな話である。そこで、一貫したセマンティクスを保証するために、Cats の `Monad`（`Semigroupal` を拡張している）は、上記のように `map` と `flatMap` を用いて標準的な `product` の定義を提供している。

`Future` に対する結果も一種の錯覚である。`flatMap` が逐次的な順序付けを提供するのだから、`product` も同じ挙動を示す。並列実行しているように観察されるのは、`product` を呼び出す前に個々の `Future` が実行を開始しているからである。これは、以下に示すようなクラシックな「作成してから `flatMap` する」パターンと同じである。

```scala mdoc:silent
val a = Future("Future 1")
val b = Future("Future 2")

for {
  x <- a
  y <- b
} yield (x, y)
```

では、なぜ `Semigroupal` にこだわる必要があるのだろうか。その答えは、`Semigroupal`（および `Applicative`）のインスタンスをもつが `Monad` インスタンスをもたない有用なデータ型を作成できる点にある。データ型がモナドでなければ `product` を異なる方法で実装するのも自由である。この点については、エラーハンドリングの代替データ型を見ていく際にさらに詳しく説明する。

#### 演習: `List` の `product`

なぜ `List` に対する `product` はデカルト積を生成するのか考察せよ。上述した例を以下に再掲する。

```scala mdoc
Semigroupal[List].product(List(1, 2), List(3, 4))
```

`tupled` を用いて書くこともできる。

```scala mdoc
(List(1, 2), List(3, 4)).tupled
```

<div class="solution">
この演習は `flatMap` と `map` を用いた `product` の定義について理解度を確認するためのものである。

```scala mdoc:invisible:reset-object
import cats.Monad
```
```scala mdoc:silent
import cats.syntax.functor._ // map
import cats.syntax.flatMap._ // flatMap

def product[F[_]: Monad, A, B](x: F[A], y: F[B]): F[(A, B)] =
  x.flatMap(a => y.map(b => (a, b)))
```

これは for 内包表記を用いた以下のコードと等価である。

```scala mdoc:invisible:reset-object
import cats.Monad
import cats.syntax.flatMap._ // flatMap
import cats.syntax.functor._ // map
```
```scala mdoc:silent
def product[F[_]: Monad, A, B](x: F[A], y: F[B]): F[(A, B)] =
  for {
    a <- x
    b <- y
  } yield (a, b)
```

`flatMap` のセマンティクスが、`List` や `Either` に対する `product` の挙動を生じさせる要因である。

```scala mdoc:silent
import cats.instances.list._ // Semigroupal
```

```scala mdoc
product(List(1, 2), List(3, 4))
```
</div>
