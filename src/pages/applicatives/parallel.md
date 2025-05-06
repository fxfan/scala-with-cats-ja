<!--

## Parallel

In the previous section we saw that
when call `product` on a type that
has a `Monad` instance
we get sequential semantics.
This makes sense from the point-of-view
of keeping consistency with 
implementations of `product` in terms of `flatMap` and `map`.
However it's not always what we want.
The `Parallel` type class, and its associated syntax,
allows us to access alternate semantics
for certain monads.

We've seen how the `product` method on `Either`
stops at the first error.

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.either._ // for Semigroupal

type ErrorOr[A] = Either[Vector[String], A]
val error1: ErrorOr[Int] = Left(Vector("Error 1"))
val error2: ErrorOr[Int] = Left(Vector("Error 2"))
```

```scala mdoc
Semigroupal[ErrorOr].product(error1, error2)
```

We can also write this
using `tupled`
as a short-cut.

```scala mdoc:silent
import cats.syntax.apply._ // for tupled
import cats.instances.vector._ // for Semigroup on Vector
```
```scala mdoc
(error1, error2).tupled
```

To collect all the errors
we simply replace `tupled` with its "parallel" version
called `parTupled`.

```scala mdoc:silent
import cats.syntax.parallel._ // for parTupled
```
```scala mdoc
(error1, error2).parTupled
```

Notice that both errors are returned! 
This behaviour is not special to using `Vector` as the error type.
Any type that has a `Semigroup` instance will work.
For example, here we use `List` instead.

```scala mdoc:silent
import cats.instances.list._ // for Semigroup on List

type ErrorOrList[A] = Either[List[String], A]
val errStr1: ErrorOrList[Int] = Left(List("error 1"))
val errStr2: ErrorOrList[Int] = Left(List("error 2"))
```
```scala mdoc

(errStr1, errStr2).parTupled
```

There are many syntax methods provided by `Parallel`
for methods on `Semigroupal` and related types,
but the most commonly used is `parMapN`.
Here's an example of `parMapN` 
in an error handling situation.

```scala mdoc:silent
val success1: ErrorOr[Int] = Right(1)
val success2: ErrorOr[Int] = Right(2)
val addTwo = (x: Int, y: Int) => x + y
```
```scala mdoc
(error1, error2).parMapN(addTwo)
(success1, success2).parMapN(addTwo)
```

Let's dig into how `Parallel` works.
The definition below is the core of `Parallel`.

```scala
trait Parallel[M[_]] {
  type F[_]
  
  def applicative: Applicative[F]
  def monad: Monad[M]
  def parallel: ~>[M, F]
}
```

This tells us if there is a `Parallel` instance for some type constructor `M` then:

- there must be a `Monad` instance for `M`;
- there is a related type constructor `F` that has an `Applicative` instance; and
- we can convert `M` to `F`.

We haven't seen `~>` before. 
It's a type alias for [`FunctionK`][cats.arrow.FunctionK] 
and is what performs the conversion from `M` to `F`. 
A normal function `A => B` converts values of type `A` to values of type `B`. 
Remember that `M` and `F` are not types; they are type constructors. 
A `FunctionK` `M ~> F` is a function from a value with type `M[A]` to a value with type `F[A]`. 
Let's see a quick example 
by defining a `FunctionK` that converts an `Option` to a `List`.

```scala mdoc:silent
import cats.arrow.FunctionK

object optionToList extends FunctionK[Option, List] {
  def apply[A](fa: Option[A]): List[A] =
    fa match {
      case None    => List.empty[A]
      case Some(a) => List(a)
    }
}
```
```scala mdoc
optionToList(Some(1))
optionToList(None)
```

As the type parameter `A` is generic a `FunctionK` cannot inspect
any values contained with the type constructor `M`.
The conversion must be performed
purely in terms of the structure of the type constructors `M` and `F`.
We can see in `optionToList` above
this is indeed the case.

So in summary,
`Parallel` allows us to take a type that has a monad instance
and convert it to some related type 
that instead has an applicative (or semigroupal) instance.
This related type will have some useful alternate semantics.
We've seen the case above where the related applicative for `Either`
allows for accumulation of errors
instead of fail-fast semantics.

Now we've seen `Parallel`
it's time to finally learn about `Applicative`.


#### Exercise: Parallel List

Does `List` have a `Parallel` instance? If so, what does the `Parallel` instance do?

<div class="solution">
`List` does have a `Parallel` instance, 
and it zips the `List`
insted of creating the cartesian product.

We can see by writing a little bit of code.

```scala mdoc:silent
import cats.instances.list._
```
```scala mdoc
(List(1, 2), List(3, 4)).tupled
(List(1, 2), List(3, 4)).parTupled
```
</div>


```scala mdoc:reset:silent
```
--->

## `Parallel`

前節では、`Monad` インスタンスをもつ型に対して `product` を呼び出すと、逐次的なセマンティクスが得られることを見た。これは `flatMap` と `map` を用いた `product` の実装と一貫性を保つという観点からは理にかなっている。しかし、この動作が常に望ましいわけではない。`Parallel` 型クラスとその関連構文を使用すれば、特定のモナドについて、別のセマンティクスにアクセスできるようになる。

`Either` における `product` メソッドが最初のエラーで処理を停止することはすでに見た。

```scala mdoc:silent
import cats.Semigroupal
import cats.instances.either._ // Semigroupal

type ErrorOr[A] = Either[Vector[String], A]
val error1: ErrorOr[Int] = Left(Vector("Error 1"))
val error2: ErrorOr[Int] = Left(Vector("Error 2"))
```

```scala mdoc
Semigroupal[ErrorOr].product(error1, error2)
```

`tupled` を用いた短縮形で書くと以下のようになる。

```scala mdoc:silent
import cats.syntax.apply._ // tupled
import cats.instances.vector._ // Semigroup on Vector
```
```scala mdoc
(error1, error2).tupled
```

この `tupled` を、その並列バージョンである `parTupled` に置き換えるだけで、すべてのエラーを収集することが可能となる。

```scala mdoc:silent
import cats.syntax.parallel._ // parTupled
```
```scala mdoc
(error1, error2).parTupled
```

両方のエラーが返されることに注目してほしい。この振る舞いはエラー型として `Vector` を用いた場合に限らない。`Semigroup` インスタンスをもつ型であれば、どんな型でも動作する。たとえば `List` を用いる例を以下に示す。

```scala mdoc:silent
import cats.instances.list._ // Semigroup on List

type ErrorOrList[A] = Either[List[String], A]
val errStr1: ErrorOrList[Int] = Left(List("error 1"))
val errStr2: ErrorOrList[Int] = Left(List("error 2"))
```
```scala mdoc

(errStr1, errStr2).parTupled
```

`Parallel` は、`Semigroupal` や関連する型のメソッドに対して多くの構文メソッドを提供している。もっともよく使われるのは `parMapN` である。以下に、エラーハンドリングの場面における `parMapN` の例を示す。

```scala mdoc:silent
val success1: ErrorOr[Int] = Right(1)
val success2: ErrorOr[Int] = Right(2)
val addTwo = (x: Int, y: Int) => x + y
```
```scala mdoc
(error1, error2).parMapN(addTwo)
(success1, success2).parMapN(addTwo)
```

`Parallel` の動作原理を掘り下げてみよう。以下が `Parallel` の中核をなす定義である。

```scala
trait Parallel[M[_]] {
  type F[_]
  
  def applicative: Applicative[F]
  def monad: Monad[M]
  def parallel: ~>[M, F]
}
```

これによると、ある型コンストラクタ `M` に対して `Parallel` インスタンスが存在する場合、以下のことが成り立つ。

- `M` に対して `Monad` インスタンスが存在する必要がある
- 関連する型コンストラクタ `F` があり、これには `Applicative` インスタンスが存在する
- `M` を `F` に変換できる

`~>` について本書ではまだ紹介していないが、これは [`FunctionK`][cats.arrow.FunctionK] の型エイリアスで、`M` から `F` への変換を表す。通常の関数 `A => B` は `A` 型の値を `B` 型の値に変換するが、`M` と `F` は型そのものではなく型コンストラクタであることを思い出そう。`M ~> F` という `FunctionK` は、`M[A]` 型の値を `F[A]` 型の値に変換する関数である。簡単な例として、`Option` を `List` に変換する `FunctionK` を定義してみよう。

```scala mdoc:silent
import cats.arrow.FunctionK

object optionToList extends FunctionK[Option, List] {
  def apply[A](fa: Option[A]): List[A] =
    fa match {
      case None    => List.empty[A]
      case Some(a) => List(a)
    }
}
```
```scala mdoc
optionToList(Some(1))
optionToList(None)
```

型パラメータ `A` はジェネリックであるため、`FunctionK` が型コンストラクタ `M` 内に含まれる値を参照することはできない。変換は、純粋に型コンストラクタ `M` と `F` の構造に基づいて行われなければならない。上記の `optionToList` でも、その点を確認することができる。

まとめると、`Parallel` は、`Monad` インスタンスをもつ型を、`Applicative`（または `Semigroupal`）インスタンスをもつ何らかの関連型へと変換可能にしてくれる。変換後の型はモナドとは別の有用なセマンティクスを提供してくれる。先述の例では、`Either` の関連するアプリカティブが、フェイルファストではなく、エラーの蓄積を可能にするケースを見た。

では、`Parallel` を理解したところで、次はいよいよ `Applicative` について学んでいこう。

#### 演習: `List` の `Parallel`

`List` は `Parallel` インスタンスをもつか、もつとしたらその `Parallel` インスタンスは何を行うか、確認せよ。

<div class="solution">
`List` には `Parallel` インスタンスが存在し、デカルト積を作成する代わりにリストを zip する。

簡単なコードを書くことでこれを確認できる。

```scala mdoc:silent
import cats.instances.list._
```
```scala mdoc
(List(1, 2), List(3, 4)).tupled
(List(1, 2), List(3, 4)).parTupled
```
</div>
