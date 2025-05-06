<!--

# Monad Transformers {#sec:monad-transformers}

Monads are [like burritos][link-monads-burritos],
which means that once you acquire a taste,
you'll find yourself returning to them again and again.
This is not without issues.
As burritos can bloat the waist,
monads can bloat the code base through nested for-comprehensions.

Imagine we are interacting with a database.
We want to look up a user record.
The user may or may not be present, so we return an `Option[User]`.
Our communication with the database could fail for many reasons
(network issues, authentication problems, and so on),
so this result is wrapped up in an `Either`,
giving us a final result of `Either[Error, Option[User]]`.

To use this value we must nest `flatMap` calls
(or equivalently, for-comprehensions):

```scala mdoc:invisible:reset-object
type Error = String

final case class User(id: Long, name: String)

def lookupUser(id: Long): Either[Error, Option[User]] = ???
```

```scala mdoc:silent
def lookupUserName(id: Long): Either[Error, Option[String]] =
  for {
    optUser <- lookupUser(id)
  } yield {
    for { user <- optUser } yield user.name
  }
```

This quickly becomes very tedious.

## Exercise: Composing Monads

A question arises.
Given two arbitrary monads,
can we combine them in some way to make a single monad?
That is, do monads *compose*?
We can try to write the code but we soon hit problems:

```scala mdoc:silent
import cats.syntax.applicative._ // for pure
```

```scala
// Hypothetical example. This won't actually compile:
def compose[M1[_]: Monad, M2[_]: Monad] = {
  type Composed[A] = M1[M2[A]]

  new Monad[Composed] {
    def pure[A](a: A): Composed[A] =
      a.pure[M2].pure[M1]

    def flatMap[A, B](fa: Composed[A])
        (f: A => Composed[B]): Composed[B] =
      // Problem! How do we write flatMap?
      ???
  }
}
```

It is impossible to write a general definition of `flatMap`
without knowing something about `M1` or `M2`.
However, if we *do* know something about one or other monad,
we can typically complete this code.
For example, if we fix `M2` above to be `Option`,
a definition of `flatMap` comes to light:

```scala
def flatMap[A, B](fa: Composed[A])
    (f: A => Composed[B]): Composed[B] =
  fa.flatMap(_.fold[Composed[B]](None.pure[M1])(f))
```

Notice that the definition above makes use of `None`---an
`Option`-specific concept that
doesn't appear in the general `Monad` interface.
We need this extra detail to combine `Option` with other monads.
Similarly, there are things about other monads
that help us write composed `flatMap` methods for them.
This is the idea behind monad transformers:
Cats defines transformers for a variety of monads,
each providing the extra knowledge we need
to compose that monad with others.
Let's look at some examples.

## A Transformative Example

Cats provides transformers for many monads,
each named with a `T` suffix:
`EitherT` composes `Either` with other monads,
`OptionT` composes `Option`, and so on.

Here's an example that uses `OptionT`
to compose `List` and `Option`.
We can use `OptionT[List, A]`,
aliased to `ListOption[A]` for convenience,
to transform a `List[Option[A]]` into a single monad:

```scala mdoc:silent
import cats.data.OptionT

type ListOption[A] = OptionT[List, A]
```

Note how we build `ListOption` from the inside out:
we pass `List`, the type of the outer monad,
as a parameter to `OptionT`,
the transformer for the inner monad.

We can create instances of `ListOption`
using the `OptionT` constructor,
or more conveniently using `pure`:

```scala mdoc:silent
import cats.instances.list._     // for Monad
import cats.syntax.applicative._ // for pure
```

```scala mdoc
val result1: ListOption[Int] = OptionT(List(Option(10)))

val result2: ListOption[Int] = 32.pure[ListOption]
```

The `map` and `flatMap` methods
combine the corresponding methods of `List` and `Option`
into single operations:

```scala mdoc
result1.flatMap { (x: Int) =>
  result2.map { (y: Int) =>
    x + y
  }
}
```

This is the basis of all monad transformers.
The combined `map` and `flatMap` methods
allow us to use both component monads
without having to recursively unpack
and repack values at each stage in the computation.
Now let's look at the API in more depth.

<div class="callout callout-warning">
*Complexity of Imports*

The imports in the code samples above
hint at how everything bolts together.

We import [`cats.syntax.applicative`][cats.syntax.applicative]
to get the `pure` syntax.
`pure` requires an implicit parameter of type `Applicative[ListOption]`.
We haven't met `Applicatives` yet,
but all `Monads` are also `Applicatives`
so we can ignore that difference for now.

In order to generate our `Applicative[ListOption]`
we need instances of `Applicative` for `List` and `OptionT`.
`OptionT` is a Cats data type so its instance
is provided by its companion object.
The instance for `List` comes from
[`cats.instances.list`][cats.instances.list].

Notice we're not importing
[`cats.syntax.functor`][cats.syntax.functor] or
[`cats.syntax.flatMap`][cats.syntax.flatMap].
This is because `OptionT` is a concrete data type
with its own explicit `map` and `flatMap` methods.
It wouldn't cause problems if we imported the syntax---the
compiler would ignore it in favour of the explicit methods.

Remember that we're subjecting ourselves to these shenanigans
because we're stubbornly refusing to use the universal Cats import,
[`cats.implicits`][cats.implicits].
If we did use that import,
all of the instances and syntax we needed would be in scope
and everything would just work.
</div>

## Monad Transformers in Cats

Each monad transformer is a data type,
defined in [`cats.data`][cats.data],
that allows us to *wrap* stacks of monads
to produce new monads.
We use the monads we've built via the `Monad` type class.
The main concepts we have to cover
to understand monad transformers are:

- the available transformer classes;
- how to build stacks of monads using transformers;
- how to construct instances of a monad stack; and
- how to pull apart a stack to access the wrapped monads.

### The Monad Transformer Classes

By convention, in Cats a monad `Foo`
will have a transformer class called `FooT`.
In fact, many monads in Cats are defined
by combining a monad transformer with the `Id` monad.
Concretely, some of the available instances are:

- [`cats.data.OptionT`][cats.data.OptionT] for `Option`;
- [`cats.data.EitherT`][cats.data.EitherT] for `Either`;
- [`cats.data.ReaderT`][cats.data.ReaderT] for `Reader`;
- [`cats.data.WriterT`][cats.data.WriterT] for `Writer`;
- [`cats.data.StateT`][cats.data.StateT] for `State`;
- [`cats.data.IdT`][cats.data.IdT] for the [`Id`][cats.Id] monad.

<div class="callout callout-info">
*Kleisli Arrows*

In Section [@sec:monads:reader]
we mentioned that the `Reader` monad was a specialisation
of a more general concept called a "kleisli arrow",
represented in Cats as
[`cats.data.Kleisli`][cats.data.Kleisli].

We can now reveal that `Kleisli` and `ReaderT`
are, in fact, the same thing!
`ReaderT` is actually a type alias for `Kleisli`.
Hence, we were creating `Readers` last chapter
and seeing `Kleislis` on the console.
</div>

### Building Monad Stacks

All of these monad transformers follow the same convention.
The transformer itself represents the *inner* monad in a stack,
while the first type parameter specifies the outer monad.
The remaining type parameters are the types
we've used to form the corresponding monads.

For example, our `ListOption` type above
is an alias for `OptionT[List, A]`
but the result is effectively a `List[Option[A]]`.
In other words, we build monad stacks from the inside out:

```scala mdoc:invisible:reset
import cats.data.OptionT
import cats.syntax.applicative._ // for pure
```
```scala mdoc:silent
type ListOption[A] = OptionT[List, A]
```

Many monads and all transformers have at least two type parameters,
so we often have to define type aliases for intermediate stages.

For example, suppose we want to wrap `Either` around `Option`.
`Option` is the innermost type
so we want to use the `OptionT` monad transformer.
We need to use `Either` as the first type parameter.
However, `Either` itself has two type parameters
and monads only have one.
We need a type alias
to convert the type constructor to the correct shape:

```scala mdoc:silent
// Alias Either to a type constructor with one parameter:
type ErrorOr[A] = Either[String, A]

// Build our final monad stack using OptionT:
type ErrorOrOption[A] = OptionT[ErrorOr, A]
```

`ErrorOrOption` is a monad, just like `ListOption`.
We can use `pure`, `map`, and `flatMap` as usual
to create and transform instances:

```scala mdoc:silent
import cats.instances.either._ // for Monad
```

```scala mdoc
val a = 10.pure[ErrorOrOption]
val b = 32.pure[ErrorOrOption]

val c = a.flatMap(x => b.map(y => x + y))
```

Things become even more confusing
when we want to stack three or more monads.

For example, let's create a `Future` of an `Either` of `Option`.
Once again we build this from the inside out
with an `OptionT` of an `EitherT` of `Future`.
However, we can't define this in one line
because `EitherT` has three type parameters:

```scala
case class EitherT[F[_], E, A](stack: F[Either[E, A]]) {
  // etc...
}
```

The three type parameters are as follows:

- `F[_]` is the outer monad in the stack (`Either` is the inner);
- `E` is the error type for the `Either`;
- `A` is the result type for the `Either`.

This time we create an alias for `EitherT` that
fixes `Future` and `Error` and allows `A` to vary:

```scala mdoc:silent
import scala.concurrent.Future
import cats.data.{EitherT, OptionT}

type FutureEither[A] = EitherT[Future, String, A]

type FutureEitherOption[A] = OptionT[FutureEither, A]
```

Our mammoth stack now composes three monads
and our `map` and `flatMap` methods
cut through three layers of abstraction:

```scala mdoc:silent
import cats.instances.future._ // for Monad
import scala.concurrent.Await
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
```

```scala mdoc:silent
val futureEitherOr: FutureEitherOption[Int] =
  for {
    a <- 10.pure[FutureEitherOption]
    b <- 32.pure[FutureEitherOption]
  } yield a + b
```

<div class="callout callout-warning">
*Kind Projector*

If you frequently find yourself
defining multiple type aliases when building monad stacks,
you may want to try
the [Kind Projector][link-kind-projector] compiler plugin.
Kind Projector enhances Scala's type syntax
to make it easier to define partially applied type constructors.
For example:

```scala mdoc
import cats.instances.option._ // for Monad

123.pure[EitherT[Option, String, _]]
```

Kind Projector can't simplify all type declarations down to a single line,
but it can reduce the number of intermediate type definitions
needed to keep our code readable.
</div>

### Constructing and Unpacking Instances

As we saw above, we can create transformed monad stacks
using the relevant monad transformer's `apply` method
or the usual `pure` syntax[^eithert-monad-error]:

```scala mdoc
// Create using apply:
val errorStack1 = OptionT[ErrorOr, Int](Right(Some(10)))

// Create using pure:
val errorStack2 = 32.pure[ErrorOrOption]
```

[^eithert-monad-error]: Cats provides an instance
of `MonadError` for `EitherT`,
allowing us to create instances
using `raiseError` as well as `pure`.

Once we've finished with a monad transformer stack,
we can unpack it using its `value` method.
This returns the untransformed stack.
We can then manipulate the individual monads in the usual way:

```scala mdoc
// Extracting the untransformed monad stack:
errorStack1.value

// Mapping over the Either in the stack:
errorStack2.value.map(_.getOrElse(-1))
```

Each call to `value` unpacks a single monad transformer.
We may need more than one call to completely unpack a large stack.
For example, to `Await` the `FutureEitherOption` stack above,
we need to call `value` twice:

```scala mdoc
futureEitherOr

val intermediate = futureEitherOr.value

val stack = intermediate.value

Await.result(stack, 1.second)
```

### Default Instances

Many monads in Cats are defined
using the corresponding transformer and the `Id` monad.
This is reassuring as it confirms
that the APIs for monads and transformers are identical.
`Reader`, `Writer`, and `State`
are all defined in this way:

```scala
type Reader[E, A] = ReaderT[Id, E, A] // = Kleisli[Id, E, A]
type Writer[W, A] = WriterT[Id, W, A]
type State[S, A]  = StateT[Id, S, A]
```

In other cases monad transformers
are defined separately to their corresponding monads.
In these cases, the methods of the transformer tend
to mirror the methods on the monad.
For example, `OptionT` defines `getOrElse`,
and `EitherT` defines `fold`, `bimap`, `swap`,
and other useful methods.

### Usage Patterns

Widespread use of monad transformers is sometimes difficult
because they fuse monads together in predefined ways.
Without careful thought,
we can end up having to unpack and repack monads
in different configurations
to operate on them in different contexts.

We can cope with this in multiple ways.
One approach involves creating a single "super stack"
and sticking to it throughout our code base.
This works if the code is simple and largely uniform in nature.
For example, in a web application,
we could decide that all request handlers are asynchronous
and all can fail with the same set of HTTP error codes.
We could design a custom ADT representing the errors
and use a fusion `Future` and `Either` everywhere in our code:

```scala mdoc:invisible:reset-object
import cats.data.EitherT
import cats.instances.list._
import scala.concurrent.Future
```
```scala mdoc:silent
sealed abstract class HttpError
final case class NotFound(item: String) extends HttpError
final case class BadRequest(msg: String) extends HttpError
// etc...

type FutureEither[A] = EitherT[Future, HttpError, A]
```

The "super stack" approach starts to fail in larger,
more heterogeneous code bases
where different stacks make sense in different contexts.
Another design pattern that makes more sense in these contexts
uses monad transformers as local "glue code".
We expose untransformed stacks at module boundaries,
transform them to operate on them locally,
and untransform them before passing them on.
This allows each module of code to make its own decisions
about which transformers to use:

```scala mdoc:silent
import cats.data.Writer

type Logged[A] = Writer[List[String], A]

// Methods generally return untransformed stacks:
def parseNumber(str: String): Logged[Option[Int]] =
  util.Try(str.toInt).toOption match {
    case Some(num) => Writer(List(s"Read $str"), Some(num))
    case None      => Writer(List(s"Failed on $str"), None)
  }

// Consumers use monad transformers locally to simplify composition:
def addAll(a: String, b: String, c: String): Logged[Option[Int]] = {
  import cats.data.OptionT

  val result = for {
    a <- OptionT(parseNumber(a))
    b <- OptionT(parseNumber(b))
    c <- OptionT(parseNumber(c))
  } yield a + b + c

  result.value
}
```

```scala mdoc
// This approach doesn't force OptionT on other users' code:
val result1 = addAll("1", "2", "3")
val result2 = addAll("1", "a", "3")
```

Unfortunately, there aren't one-size-fits-all
approaches to working with monad transformers.
The best approach for you may depend on a lot of factors:
the size and experience of your team,
the complexity of your code base, and so on.
You may need to experiment and gather feedback from colleagues
to determine whether monad transformers are a good fit.

## Exercise: Monads: Transform and Roll Out

The Autobots, well-known robots in disguise,
frequently send messages during battle
requesting the power levels of their team mates.
This helps them coordinate strategies
and launch devastating attacks.
The message sending method looks like this:

```scala
def getPowerLevel(autobot: String): Response[Int] =
  ???
```

Transmissions take time in Earth's viscous atmosphere,
and messages are occasionally lost
due to satellite malfunction or sabotage by pesky Decepticons[^transformers].
`Responses` are therefore represented as a stack of monads:

```scala mdoc
type Response[A] = Future[Either[String, A]]
```

[^transformers]: It is a well known fact
that Autobot neural nets are implemented in Scala.
Decepticon brains are, of course, dynamically typed.

Optimus Prime is getting tired of
the nested for comprehensions in his neural matrix.
Help him by rewriting `Response` using a monad transformer.

<div class="solution">
This is a relatively simple combination.
We want `Future` on the outside
and `Either` on the inside,
so we build from the inside out
using an `EitherT` of `Future`:

```scala mdoc:silent:reset-object
import cats.data.EitherT
import scala.concurrent.Future

type Response[A] = EitherT[Future, String, A]
```
</div>

Now test the code by implementing `getPowerLevel`
to retrieve data from a set of imaginary allies.
Here's the data we'll use:

```scala mdoc:silent
val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)
```

If an Autobot isn't in the `powerLevels` map,
return an error message reporting
that they were unreachable.
Include the `name` in the message for good effect.

<div class="solution">
```scala mdoc:silent:reset
import cats.data.EitherT
import scala.concurrent.Future
val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)
```
```scala mdoc:silent
import cats.instances.future._ // for Monad
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
</div>

Two autobots can perform a special move
if their combined power level is greater than 15.
Write a second method, `canSpecialMove`,
that accepts the names of two allies
and checks whether a special move is possible.
If either ally is unavailable,
fail with an appropriate error message:

```scala mdoc:silent
def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  ???
```

<div class="solution">
We request the power level from each ally
and use `map` and `flatMap` to combine the results:

```scala mdoc:invisible:reset-object
import cats.implicits._
import cats.data._
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
```scala mdoc:silent
def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  for {
    power1 <- getPowerLevel(ally1)
    power2 <- getPowerLevel(ally2)
  } yield (power1 + power2) > 15
```
</div>

Finally, write a method `tacticalReport` that
takes two ally names and prints a message
saying whether they can perform a special move:

```scala mdoc:silent
def tacticalReport(ally1: String, ally2: String): String =
  ???
```

<div class="solution">
We use the `value` method to unpack the monad stack
and `Await` and `fold` to unpack the `Future` and `Either`:

```scala mdoc:invisible:reset
import cats.implicits._
import cats.data._
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
```scala mdoc:silent
import scala.concurrent.Await
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  for {
    power1 <- getPowerLevel(ally1)
    power2 <- getPowerLevel(ally2)
  } yield (power1 + power2) > 15

def tacticalReport(ally1: String, ally2: String): String = {
  val stack = canSpecialMove(ally1, ally2).value

  Await.result(stack, 1.second) match {
    case Left(msg) =>
      s"Comms error: $msg"
    case Right(true)  =>
      s"$ally1 and $ally2 are ready to roll out!"
    case Right(false) =>
      s"$ally1 and $ally2 need a recharge."
  }
}
```
</div>

You should be able to use `report` as follows:

```scala mdoc
tacticalReport("Jazz", "Bumblebee")
tacticalReport("Bumblebee", "Hot Rod")
tacticalReport("Jazz", "Ironhide")
```


```scala mdoc:reset:silent
```
--->

# モナド変換子 {#sec:monad-transformers}

モナドは[ブリトーのようなもの][link-monads-burritos]だとよく言われる。一度その味を覚えれば、何度も手を伸ばしてしまうということである。しかし、これは問題がないわけではない。ブリトーが腰回りを太らせるように、モナドも入れ子になった for 内包表記によってコードベースを太らせてしまうことがある。

データベースとやり取りをする場面を想像してみよう。ユーザのレコードを取得したいが、そのユーザが存在するかどうかはわからない。そのため、結果は `Option[User]` として返すことになる。さらに、データベースとの通信はネットワークの問題や認証のトラブルなど、さまざまな理由で失敗する可能性があるため、その結果は `Either` に包まれる。最終的に得られる結果は `Either[Error, Option[User]]` という形になる。

この値を使うには `flatMap` 呼び出しを入れ子にするか、それと同じことを for 内包表記で行わなければならない。

```scala mdoc:invisible:reset-object
type Error = String

final case class User(id: Long, name: String)

def lookupUser(id: Long): Either[Error, Option[User]] = ???
```

```scala mdoc:silent
def lookupUserName(id: Long): Either[Error, Option[String]] =
  for {
    optUser <- lookupUser(id)
  } yield {
    for { user <- optUser } yield user.name
  }
```

これはすぐに厄介な問題へと発展する。

## 演習: モナドの合成

ここでひとつの疑問が生じる。任意のふたつのモナドが与えられたとき、それらを何らかの方法で組み合わせてひとつのモナドにできるだろうか。つまり、モナドは*合成*できるだろうか。コードを書いてみるとすぐに問題に直面する。

```scala mdoc:silent
import cats.syntax.applicative._ // pure
```

```scala
// 説明用。実際にはコンパイルされない
def compose[M1[_]: Monad, M2[_]: Monad] = {
  type Composed[A] = M1[M2[A]]

  new Monad[Composed] {
    def pure[A](a: A): Composed[A] =
      a.pure[M2].pure[M1]

    def flatMap[A, B](fa: Composed[A])
        (f: A => Composed[B]): Composed[B] =
      // 問題発生！ flatMap をどう書けばいいのだろうか
      ???
  }
}
```

`M1` や `M2` のことを何も知らない状態で `flatMap` の一般的な定義を書くのは不可能である。しかし、どちらか一方のモナドについて知識があれば、コードを完成させることができる場合が多い。たとえば、上記の `M2` を `Option` に固定すれば、`flatMap` の定義は明らかとなる。

```scala
def flatMap[A, B](fa: Composed[A])
    (f: A => Composed[B]): Composed[B] =
  fa.flatMap(_.fold[Composed[B]](None.pure[M1])(f))
```

上記の定義が `None` を使用していることに注意してほしい。これは `Option` 固有の概念であり、一般的な `Monad` インターフェースには現れない。`Option` を他のモナドと組み合わせるには、このような詳細情報が必要である。他のモナドについても同様で、それらを組み合わせた `flatMap` メソッドを書くのに役立つ固有の情報が存在する。これがモナド変換子（monad transformer）の背後にあるアイデアである。Cats はさまざまなモナドのための変換子を定義しており、それぞれがそのモナドを他と組み合わせるのに必要な付加知識を提供してくれる。いくつか例を見てみよう。

## 変換の例

Cats は多くのモナド用に変換子を提供しており、その名称にはそれぞれ `T` が接尾辞として付けられている。たとえば、`EitherT` は `Either` を他のモナドと組み合わせ、`OptionT` は `Option` を他と組み合わせる。

以下は `OptionT` を使って `List` と `Option` を組み合わせる例である。簡便のため `ListOption[A]` というエイリアスを使い、`List[Option[A]]` をひとつのモナドとして取り扱う。

```scala mdoc:silent
import cats.data.OptionT

type ListOption[A] = OptionT[List, A]
```

ここで注目してほしいのは、`ListOption` を内側から外側に向かって組み立てている点である。`OptionT` は内側のモナドである `Option` 用の変換子であり、外側のモナドの型である `List` をパラメータとしてそこに渡している。

We can create instances of `ListOption`
using the `OptionT` constructor,
or more conveniently using `pure`:

`ListOption` のインスタンスを作成するには `OptionT` のコンストラクタを使う。もしくは `pure` を使えばもっと便利である。

```scala mdoc:silent
import cats.instances.list._     // Monad
import cats.syntax.applicative._ // pure
```

```scala mdoc
val result1: ListOption[Int] = OptionT(List(Option(10)))

val result2: ListOption[Int] = 32.pure[ListOption]
```

`map` や `flatMap` メソッドは、`List` と `Option` がもつ同名のメソッド同士を組み合わせてひとつの操作にする。

```scala mdoc
result1.flatMap { (x: Int) =>
  result2.map { (y: Int) =>
    x + y
  }
}
```

これがすべてのモナド変換子の基礎である。組み合わされた `map` や `flatMap` メソッドのおかげで、計算の各段階で値を再帰的に取り出して包み直すようなことをしなくても、両方のモナドを扱うことができる。次に API をさらに詳しく見ていこう。

<div class="callout callout-warning">
*インポートの複雑さについて*

上記コードサンプルのインポート文は、すべての要素がどのように結びつけられているかを示している。

まず、`pure` 構文を得るために [`cats.syntax.applicative`][cats.syntax.applicative] をインポートしている。`pure` は `Applicative[ListOption]` 型の暗黙パラメータを必要とする。まだ `Applicative` については学んでいないが、すべての `Monad` は `Applicative` でもあるため、今はその違いを無視してかまわない。

`Applicative[ListOption]` を生成するには、`List` と `OptionT` それぞれの `Applicative` インスタンスが必要である。`OptionT` は Cats 独自のデータ型なので、そのインスタンスはコンパニオンオブジェクトによって提供される。`List` 用のインスタンスは [`cats.instances.list`][cats.instances.list] に置かれている。

[`cats.syntax.functor`][cats.syntax.functor] や [`cats.syntax.flatMap`][cats.syntax.flatMap] をインポートしていないことにも注目してほしい。これは、`OptionT` が具体的なデータ型であり、独自の `map` や `flatMap` メソッドをもっているためである。これらの構文をインポートしても問題は生じないが、コンパイラは明示的なメソッドを優先するので、インポートされた構文は無視される。

今回こういった複雑さに直面しているのは、[`cats.implicits`][cats.implicits] による包括的なインポートを意図的に避けているためである。このインポートを使用すれば、必要なすべてのインスタンスや構文がスコープに入り、すべてが簡単に動作する。
</div>

## Cats におけるモナド変換子

モナド変換子はいずれもデータ型であり、[`cats.data`][cats.data] に定義されている。モナド変換子は、モナドの積み重ねをラップして新しいモナドを作り出すことができる。ここで使用されるモナドは Cats の `Monad` 型クラスを通じて構築されたものである。モナド変換子を理解するためにカバーすべき主なポイントは以下のとおりである。

- どのような変換子クラスが提供されているのか
- 変換子を使用してモナドの積み重ねを構築する方法
- モナドスタックのインスタンスを作成する方法
- 積み重ねたモナドを分解し、ラップされたモナドにアクセスする方法

### モナド変換子クラス

Cats では慣例的にモナド `Foo` に対して `FooT` という名前の変換子クラスが用意されている。実際、Cats の多くのモナドは、モナド変換子と `Id` モナドを組み合わせて定義されている。以下に、利用可能なインスタンスをいくつか具体的に挙げてみよう。

- `Option` 用の [`cats.data.OptionT`][cats.data.OptionT]
- `Either` 用の [`cats.data.EitherT`][cats.data.EitherT]
- `Reader` 用の [`cats.data.ReaderT`][cats.data.ReaderT]
- `Writer` 用の [`cats.data.WriterT`][cats.data.WriterT]
- `State` 用の [`cats.data.StateT`][cats.data.StateT]
- [`Id`][cats.Id] モナド用の [`cats.data.IdT`][cats.data.IdT]

<div class="callout callout-info">
*クライスリ射*

[@sec:monads:reader]節で、`Reader` モナドが「クライスリ射」というさらに一般的な概念を特殊化したものであると述べた。クライスリ射は Cats では [`cats.data.Kleisli`][cats.data.Kleisli] として表されている。

ようやく、`Kleisli` と `ReaderT` が実は同じものであると明かすことができる。`ReaderT` は実際には `Kleisli` の型エイリアスとして定義されている。そのため、以前の章で `Reader` を作成したときに、コンソールには `Kleisli` と表示されていたのである。
</div>

### モナドスタックの構築

これらのモナド変換子はすべて同じ慣例に従っている。変換子自体はスタック内の*内側*にあるモナドを表し、最初の型パラメータが外側のモナドを指定する。残りの型パラメータは、対応するモナドを形成するために使用される型である。

たとえば、先ほどの `ListOption` 型は `OptionT[List, A]` のエイリアスだが、これは実質的に `List[Option[A]]` と同じである。言い換えると、モナドスタックは内側から外側に向かって構築される。

```scala mdoc:invisible:reset
import cats.data.OptionT
import cats.syntax.applicative._ // pure
```
```scala mdoc:silent
type ListOption[A] = OptionT[List, A]
```

多くのモナドやすべての変換子はふたつ以上の型パラメータをもっているため、中間段階に対して型エイリアスを定義しなければならない場合がよくある。

たとえば、`Option` を `Either` で包みたいとする。`Option` がもっとも内側の型なので、`OptionT` モナド変換子を使うことになる。ここで `Either` を最初の型パラメータとしたいが、`Either` 自体にはふたつの型パラメータがある一方で、モナドはひとつしか型パラメータをもたない。この場合、型コンストラクタを必要な形状に合わせるため型エイリアスが必要となる。

```scala mdoc:silent
// 型コンストラクタのパラメータがひとつになるように Either にエイリアスを定義
type ErrorOr[A] = Either[String, A]

// OptionT を使って最終的なモナドスタックを構築
type ErrorOrOption[A] = OptionT[ErrorOr, A]
```

`ErrorOrOption` は、`ListOption` がそうであるように、モナドである。通常どおり `pure` や `map` や `flatMap` を用いてインスタンスの作成や変換を行うことができる。

```scala mdoc:silent
import cats.instances.either._ // Monad
```

```scala mdoc
val a = 10.pure[ErrorOrOption]
val b = 32.pure[ErrorOrOption]

val c = a.flatMap(x => b.map(y => x + y))
```

三つ以上のモナドを積み重ねたい場合はさらにややこしくなる。

たとえば、`Option` の `Either` を包んだ `Future` を作りたいとする。ここでも内側から外側へと組み立てを行う。`OptionT` を `EitherT` で包み、それをさらに `Future` 包む。しかし、`EitherT` は型パラメータを三つもっているため、これを一行で定義することはできない。

```scala
case class EitherT[F[_], E, A](stack: F[Either[E, A]]) {
  // etc...
}
```

ここで使われる三つの型パラメータは以下のとおり。

- `F[_]` はスタックにおける外側のモナド（ここでは `Either` は内側）
- `E` は `Either` のエラー型
- `A` は `Either` の結果型

ここでは、`Future` と `Error` を固定し、`A` を可変とする `EitherT` のエイリアスを作成する。

```scala mdoc:silent
import scala.concurrent.Future
import cats.data.{EitherT, OptionT}

type FutureEither[A] = EitherT[Future, String, A]

type FutureEitherOption[A] = OptionT[FutureEither, A]
```

Our mammoth stack now composes three monads
and our `map` and `flatMap` methods
cut through three layers of abstraction:

これで、この巨大なスタックは三つのモナドを合成したものとなり、`map` や `flatMap` は三層の抽象化を突き抜けて動作する。

```scala mdoc:silent
import cats.instances.future._ // Monad
import scala.concurrent.Await
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
```

```scala mdoc:silent
val futureEitherOr: FutureEitherOption[Int] =
  for {
    a <- 10.pure[FutureEitherOption]
    b <- 32.pure[FutureEitherOption]
  } yield a + b
```

<div class="callout callout-warning">
*Kind Projector*

モナドスタックを構築する際に、頻繁に複数の型エイリアスを定義しているなら、[Kind Projector][link-kind-projector] コンパイラプラグインを試してみるといいかもしれない。Kind Projector は Scala の型構文を拡張し、部分適用された型コンストラクタをより簡単に定義できるようにしてくれる。たとえば、次のように使うことができる。

```scala mdoc
import cats.instances.option._ // Monad

123.pure[EitherT[Option, String, _]]
```

Kind Projector がすべての型宣言を一行に簡素化できるわけではないが、必要な中間型定義の数を減らしコードを読みやすく保つことができる。
</div>

### インスタンスの構築と展開

すでに見たように、モナド変換子の `apply` メソッドやおなじみの `pure` 構文[^eithert-monad-error]を使うことで、ひとつのモナドとして扱えるように変換されたモナドスタックを作成することができる。

```scala mdoc
// apply を使ってインスタンス作成
val errorStack1 = OptionT[ErrorOr, Int](Right(Some(10)))

// pure を使ってインスタンス作成
val errorStack2 = 32.pure[ErrorOrOption]
```

[^eithert-monad-error]: Cats は `EitherT` 用の `MonadError` インスタンスを提供しているので、`raiseError` を使っても `pure` と同じようにインスタンスを作成できる。

モナド変換子スタックを用いた計算が終わった後は、`value` メソッドを使ってスタックを展開することができる。これにより、変換されていないスタックが返され、通常どおりに個別のモナドを操作できる。

```scala mdoc
// 変換されていないモナドスタックの抽出
errorStack1.value

// スタック内の Either に対する map 操作
errorStack2.value.map(_.getOrElse(-1))
```

`value` の呼び出しごとにモナド変換子がひとつ展開される。大きなスタックを完全に展開するには複数回の呼び出しが必要になることもある。たとえば、前出の `FutureEitherOption` スタックを `Await` するには、`value` を二回呼び出す必要がある。

```scala mdoc
futureEitherOr

val intermediate = futureEitherOr.value

val stack = intermediate.value

Await.result(stack, 1.second)
```

### デフォルトインスタンス

Cats が提供する多くのモナドは、対応する変換子と `Id` モナドを用いて定義されている。この事実は、モナドと変換子の API が同一であることを裏付けており、ユーザに安心感を与えてくれる。`Reader`、`Writer`、そして `State` は、いずれもこの方法で定義されている。

```scala
type Reader[E, A] = ReaderT[Id, E, A] // = Kleisli[Id, E, A]
type Writer[W, A] = WriterT[Id, W, A]
type State[S, A]  = StateT[Id, S, A]
```

一方で、対応するモナドとは別々にモナド変換子が定義されることもある。このような場合、変換子のメソッドは、モナドのメソッドを模倣する傾向がある。たとえば、`OptionT` には `getOrElse` が定義されているし、`EitherT` には `fold`、`bimap`、`swap` などが定義されている。

### 利用パターン

変換子はあらかじめ定義された方法でモナドを融合するため、さまざまな場所で広範囲にモナド変換子を利用するのは、時に難しいことがある。考えなしに使用すると、モナド変換子を異なる文脈で扱う際に、一旦モナドスタックを展開して別の構成で再構築する必要が生じることもある。

対処方法はいくつかある。ひとつは、単一の「スーパー・スタック」を作成し、それをコードベース全体で一貫して使用するというアプローチである。この方法は、コードが単純で大部分が均一な性質をもっている場合にうまく機能する。たとえばウェブアプリケーションであれば、リクエストハンドラはすべて非同期であり、失敗時には同じ体系のHTTPエラーコードを返す、と決めてしまうことができる。この場合、エラーを表現する代数的データ型を設計し、`Future` と `Either` を融合させたものをコード全体で使用すればよい。

```scala mdoc:invisible:reset-object
import cats.data.EitherT
import cats.instances.list._
import scala.concurrent.Future
```
```scala mdoc:silent
sealed abstract class HttpError
final case class NotFound(item: String) extends HttpError
final case class BadRequest(msg: String) extends HttpError
// etc...

type FutureEither[A] = EitherT[Future, HttpError, A]
```

この手法は、コードベースが大規模で、部分ごとの技術的特性の違いが大きい場合にはうまく機能しなくなる。そのような場面ではコンテキストごとに適しているスタックが異なる。そういったコンテキストに適しているもうひとつのデザインパターンが、モナド変換子を局所的な「接着コード」として使うアプローチである。モジュールの境界では変換されていないスタックを公開し、モジュール内部での操作のためにそれらを一時的に変換し、処理が終わったら再び変換を解除して次に渡す。この方法であれば、各モジュールはどのモナド変換子を使用するかを独自に決定できるようになる。

```scala mdoc:silent
import cats.data.Writer

type Logged[A] = Writer[List[String], A]

// メソッドは変換されていないスタックを返す
def parseNumber(str: String): Logged[Option[Int]] =
  util.Try(str.toInt).toOption match {
    case Some(num) => Writer(List(s"Read $str"), Some(num))
    case None      => Writer(List(s"Failed on $str"), None)
  }

// 合成を単純化するため内部的にはモナド変換子を用いる
def addAll(a: String, b: String, c: String): Logged[Option[Int]] = {
  import cats.data.OptionT

  val result = for {
    a <- OptionT(parseNumber(a))
    b <- OptionT(parseNumber(b))
    c <- OptionT(parseNumber(c))
  } yield a + b + c

  result.value
}
```

```scala mdoc
// このアプローチではモジュールのユーザに OptionT を強制することはない
val result1 = addAll("1", "2", "3")
val result2 = addAll("1", "a", "3")
```

残念ながら、モナド変換子の扱いに万能のアプローチは存在しない。チームの規模や経験、コードベースの複雑さなど、さまざまな要因によって、最適なアプローチは異なるだろう。モナド変換子が自分たちに適しているかどうかを判断するためには、試行錯誤し、同僚からのフィードバックを集める必要があるかもしれない。

## 演習: モナド戦士、トランスフォーム、出動！

変形して姿を隠すことで知られるオートボットたちは、戦闘中に仲間のパワーレベルを問い合わせるメッセージを頻繁に送信する。彼らはこの情報を使って戦略を立て、強力な攻撃を仕掛けるのである。メッセージ送信のメソッドは次のようになっている。

```scala
def getPowerLevel(autobot: String): Response[Int] =
  ???
```

地球の粘性の高い大気の中では通信に時間がかかる。衛星の故障ややっかいなディセプティコン[^transformers]による妨害のためにメッセージが失われることもある。そこで、`Response` はモナドのスタックとして表現されている。

```scala mdoc
type Response[A] = Future[Either[String, A]]
```

[^transformers]: オートボットのニューラルネットワークが Scala で実装されているのはよく知られた事実である。一方、ディセプティコンの頭脳はもちろん動的型付けである。

コンボイは自分のニューラルマトリクス内でのネストされた for 内包表記にうんざりしている。モナド変換子を使って `Response` の型定義を書き直し、彼を助けよ。

<div class="solution">
このモナドスタックは比較的シンプルな組み合わせである。`Future` を外側に置き、`Either` を内側に配置したいので、`Future` を型パラメータとする `EitherT` を使って内側から外側に向けて構築する。

```scala mdoc:silent:reset-object
import cats.data.EitherT
import scala.concurrent.Future

type Response[A] = EitherT[Future, String, A]
```
</div>

架空の仲間たちからデータを取得する `getPowerLevel` 関数を実装し、`Response` の定義が適切であることをテストせよ。以下のデータを使用するものとする。

```scala mdoc:silent
val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)
```

オートボットが `powerLevels` のマップに存在しない場合は、アクセスできなかったことを報告するエラーメッセージを返すこと。また、有用性を高めるため、メッセージには `name` を含めること。

<div class="solution">
```scala mdoc:silent:reset
import cats.data.EitherT
import scala.concurrent.Future
val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)
```
```scala mdoc:silent
import cats.instances.future._ // Monad
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
</div>

二体のオートボットは、パワーレベルの合計が15を超えると、必殺技の使用が可能となる。二体の仲間の名前を受け取り、必殺技が使えるかどうかを判定するメソッド `canSpecialMove` を作成せよ。指定した仲間のいずれかが見つからない場合は、適切なエラーメッセージとともに失敗させること。

```scala mdoc:silent
def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  ???
```

<div class="solution">
指定された仲間それぞれにパワーレベルを問い合わせ、得られた結果を `map` と `flatMap` で結合すればよい。

```scala mdoc:invisible:reset-object
import cats.implicits._
import cats.data._
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
```scala mdoc:silent
def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  for {
    power1 <- getPowerLevel(ally1)
    power2 <- getPowerLevel(ally2)
  } yield (power1 + power2) > 15
```
</div>

最後に、二体の仲間の名前を受け取り、彼らに必殺技が使えるかどうかを記したメッセージを出力するメソッド `tacticalReport` を作成せよ。

```scala mdoc:silent
def tacticalReport(ally1: String, ally2: String): String =
  ???
```

<div class="solution">
`value` メソッドを使ってモナドスタックを展開し、さらに `Await` と `fold` で `Future` と `Either` を展開すればよい。

```scala mdoc:invisible:reset
import cats.implicits._
import cats.data._
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

type Response[A] = EitherT[Future, String, A]

val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)

def getPowerLevel(ally: String): Response[Int] = {
  powerLevels.get(ally) match {
    case Some(avg) => EitherT.right(Future(avg))
    case None      => EitherT.left(Future(s"$ally unreachable"))
  }
}
```
```scala mdoc:silent
import scala.concurrent.Await
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  for {
    power1 <- getPowerLevel(ally1)
    power2 <- getPowerLevel(ally2)
  } yield (power1 + power2) > 15

def tacticalReport(ally1: String, ally2: String): String = {
  val stack = canSpecialMove(ally1, ally2).value

  Await.result(stack, 1.second) match {
    case Left(msg) =>
      s"Comms error: $msg"
    case Right(true)  =>
      s"$ally1 and $ally2 are ready to roll out!"
    case Right(false) =>
      s"$ally1 and $ally2 need a recharge."
  }
}
```
</div>

これで、`tacticalReport` は以下のように使えるようになるはずである。

```scala mdoc
tacticalReport("Jazz", "Bumblebee")
tacticalReport("Bumblebee", "Hot Rod")
tacticalReport("Jazz", "Ironhide")
```
