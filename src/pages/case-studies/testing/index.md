<!--

# Case Study: Testing Asynchronous Code {#sec:case-studies:testing}

We'll start with a straightforward case study:
how to simplify unit tests for asynchronous code
by making them synchronous.

Let's return to the example
from Chapter [@sec:foldable-traverse]
where we're measuring the uptime on a set of servers.
We'll flesh out the code into a more complete structure.
There will be two components.
The first is an `UptimeClient`
that polls remote servers for their uptime:

```scala mdoc:silent
import scala.concurrent.Future

trait UptimeClient {
  def getUptime(hostname: String): Future[Int]
}
```

We'll also have an `UptimeService` that maintains a list of servers
and allows the user to poll them for their total uptime:

```scala mdoc:silent
import cats.instances.future._ // for Applicative
import cats.instances.list._   // for Traverse
import cats.syntax.traverse._  // for traverse
import scala.concurrent.ExecutionContext.Implicits.global

class UptimeService(client: UptimeClient) {
  def getTotalUptime(hostnames: List[String]): Future[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

We've modelled `UptimeClient` as a trait
because we're going to want to stub it out in unit tests.
For example, we can write a test client
that allows us to provide dummy data
rather than calling out to actual servers:

```scala mdoc:silent
class TestUptimeClient(hosts: Map[String, Int]) extends UptimeClient {
  def getUptime(hostname: String): Future[Int] =
    Future.successful(hosts.getOrElse(hostname, 0))
}
```

Now, suppose we're writing unit tests for `UptimeService`.
We want to test its ability to sum values,
regardless of where it is getting them from.
Here's an example:

```scala mdoc:fail
def testTotalUptime() = {
  val hosts    = Map("host1" -> 10, "host2" -> 6)
  val client   = new TestUptimeClient(hosts)
  val service  = new UptimeService(client)
  val actual   = service.getTotalUptime(hosts.keys.toList)
  val expected = hosts.values.sum
  assert(actual == expected)
}
```

The code doesn't compile
because we've made a classic error[^warnings].
We forgot that our application code is asynchronous.
Our `actual` result is of type `Future[Int]`
and our `expected` result is of type `Int`.
We can't compare them directly!

[^warnings]: Technically this is a *warning* not an error.
It has been promoted to an error in our case
because we're using the `-Xfatal-warnings` flag on `scalac`.

There are a couple of ways to solve this problem.
We could alter our test code
to accommodate the asynchronousness.
However, there is another alternative.
Let's make our service code synchronous
so our test works without modification!

## Abstracting over Type Constructors

We need to implement two versions of `UptimeClient`:
an asynchronous one for use in production
and a synchronous one for use in our unit tests:

```scala
trait RealUptimeClient extends UptimeClient {
  def getUptime(hostname: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient {
  def getUptime(hostname: String): Int
}
```

The question is: what result type should we give
to the abstract method in `UptimeClient`?
We need to abstract over `Future[Int]` and `Int`:

```scala
trait UptimeClient {
  def getUptime(hostname: String): ???
}
```

At first this may seem difficult.
We want to retain the `Int` part from each type
but "throw away" the `Future` part in the test code.
Fortunately, Cats provides a solution
in terms of the *identity type*, `Id`,
that we discussed way back in Section [@sec:monads:identity].
`Id` allows us to "wrap" types in a type constructor
without changing their meaning:

```scala
package cats

type Id[A] = A
```

`Id` allows us to abstract over the return types in `UptimeClient`.
Implement this now:

- write a trait definition for `UptimeClient`
  that accepts a type constructor `F[_]` as a parameter;

- extend it with two traits,
  `RealUptimeClient` and `TestUptimeClient`,
  that bind `F` to `Future` and `Id` respectively;

- write out the method signature for `getUptime`
  in each case to verify that it compiles.

<div class="solution">
Here's the implementation:

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
```
```scala mdoc:silent
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient[Id] {
  def getUptime(hostname: String): Id[Int]
}
```

Note that, because `Id[A]` is just a simple alias for `A`,
we don't need to refer to the type in `TestUptimeClient`
as `Id[Int]`---we can simply write `Int` instead:

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}
```
```scala mdoc:silent
trait TestUptimeClient extends UptimeClient[Id] {
  def getUptime(hostname: String): Int
}
```

Of course, technically speaking
we don't need to redeclare `getUptime`
in `RealUptimeClient` or `TestUptimeClient`.
However, writing everything out
helps illustrate the technique.
</div>

You should now be able to flesh your definition of `TestUptimeClient`
out into a full class based on a `Map[String, Int]` as before.

<div class="solution">
The final code is similar to
our original implementation of `TestUptimeClient`,
except we no longer need
the call to `Future.successful`:

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}
```
```scala mdoc:silent
class TestUptimeClient(hosts: Map[String, Int])
  extends UptimeClient[Id] {
  def getUptime(hostname: String): Int =
    hosts.getOrElse(hostname, 0)
}
```
</div>

## Abstracting over Monads

Let's turn our attention to `UptimeService`.
We need to rewrite it to abstract over
the two types of `UptimeClient`.
We'll do this in two stages:
first we'll rewrite the class and method signatures,
then the method bodies.
Starting with the method signatures:

- comment out the body of `getTotalUptime`
  (replace it with `???` to make everything compile);

- add a type parameter `F[_]` to `UptimeService`
  and pass it on to `UptimeClient`.

<div class="solution">
The code should look like this:

```scala
class UptimeService[F[_]](client: UptimeClient[F]) {
  def getTotalUptime(hostnames: List[String]): F[Int] =
    ???
    // hostnames.traverse(client.getUptime).map(_.sum)
}
```
</div>

Now uncomment the body of `getTotalUptime`.
You should get a compilation error similar to the following:

```scala
// <console>:28: error: could not find implicit value for
//               evidence parameter of type cats.Applicative[F]
//            hostnames.traverse(client.getUptime).map(_.sum)
//                              ^
```

The problem here is that `traverse` only works
on sequences of values that have an `Applicative`.
In our original code we were traversing a `List[Future[Int]]`.
There is an applicative for `Future` so that was fine.
In this version we are traversing a `List[F[Int]]`.
We need to *prove* to the compiler that `F` has an `Applicative`.
Do this by adding an implicit constructor parameter
to `UptimeService`.

<div class="solution">
We can write this as an implicit parameter:

```scala mdoc:invisible:reset-object
import cats.syntax.traverse._  // for traverse
import cats.instances.list._

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}
```
```scala mdoc:silent
import cats.Applicative
import cats.syntax.functor._ // for map

class UptimeService[F[_]](client: UptimeClient[F])
    (implicit a: Applicative[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

or more tersely as a context bound:

```scala mdoc:reset-object:invisible
import cats.Applicative
import cats.syntax.functor._
import cats.syntax.traverse._
import cats.instances.list._

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}
```
```scala mdoc:silent
class UptimeService[F[_]: Applicative]
    (client: UptimeClient[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

Note that we need to import `cats.syntax.functor`
as well as `cats.Applicative`.
This is because we're switching from using
`future.map` to the Cats' generic extension method
that requires an implicit `Functor` parameter.
</div>

Finally, let's turn our attention to our unit tests.
Our test code now works
as intended without any modification.
We create an instance of `TestUptimeClient`
and wrap it in an `UptimeService`.
This effectively binds `F` to `Id`,
allowing the rest of the code to operate
synchronously without worrying about monads or applicatives:

```scala mdoc:invisible:reset-object
import cats.{Id, Applicative}
import cats.instances.list._  // for Traverse
import cats.syntax.functor._  // for map
import cats.syntax.traverse._ // for traverse
import scala.concurrent.Future

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future]

class TestUptimeClient(hosts: Map[String, Int])
    extends UptimeClient[Id] {
  def getUptime(hostname: String): Int =
    hosts.getOrElse(hostname, 0)
  }

class UptimeService[F[_]: Applicative]
    (client: UptimeClient[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```
```scala mdoc:silent
def testTotalUptime() = {
  val hosts    = Map("host1" -> 10, "host2" -> 6)
  val client   = new TestUptimeClient(hosts)
  val service  = new UptimeService(client)
  val actual   = service.getTotalUptime(hosts.keys.toList)
  val expected = hosts.values.sum
  assert(actual == expected)
}

testTotalUptime()
```

## Summary

This case study provides an example of how Cats can help us
abstract over different computational scenarios.
We used the `Applicative` type class
to abstract over asynchronous and synchronous code.
Leaning on a functional abstraction allows us
to specify the sequence of computations we want to perform
without worrying about the details of the implementation.

Back in Figure [@fig:applicatives:hierarchy],
we showed a "stack" of computational type classes
that are meant for exactly this kind of abstraction.
Type classes like `Functor`, `Applicative`, `Monad`,
and `Traverse` provide abstract implementations
of patterns such as mapping, zipping, sequencing, and iteration.
The mathematical laws on those types ensure
that they work together with a consistent set of semantics.

We used `Applicative` in this case study because
it was the least powerful type class that did what we needed.
If we had required `flatMap`,
we could have swapped out `Applicative` for `Monad`.
If we had needed to abstract over different sequence types,
we could have used `Traverse`.
There are also type classes like `ApplicativeError`
and `MonadError` that help model failures
as well as successful computations.

Let's move on now to a more complex case study
where type classes will help us produce something more interesting:
a map-reduce-style framework for parallel processing.


```scala mdoc:reset:silent
```
--->

# ケーススタディ: 非同期処理のテスト {#sec:case-studies:testing}

簡単なケーススタディから始めよう。非同期なコードを同期化することで単体テストをシンプルにする方法について考える。

[@sec:foldable-traverse]章で示した、サーバの稼働時間を計測する例に戻ろう。このコードに肉付けを行い、より完全な形へと近づける。ここではふたつのコンポーネントを作成する。ひとつは、リモートサーバから稼働時間を取得する `UptimeClient` である。

```scala mdoc:silent
import scala.concurrent.Future

trait UptimeClient {
  def getUptime(hostname: String): Future[Int]
}
```

もうひとつは `UptimeService` で、サーバのリストを管理し、利用者がそれらの総稼働時間を取得できるようにする。

```scala mdoc:silent
import cats.instances.future._ // Applicative
import cats.instances.list._   // Traverse
import cats.syntax.traverse._  // traverse
import scala.concurrent.ExecutionContext.Implicits.global

class UptimeService(client: UptimeClient) {
  def getTotalUptime(hostnames: List[String]): Future[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

`UptimeClient` をトレイトとしてモデリングしたのは、単体テストでスタブ化するためである。たとえば、次に示すようにダミーデータを提供するテストクライアントを作成することができる。

```scala mdoc:silent
class TestUptimeClient(hosts: Map[String, Int]) extends UptimeClient {
  def getUptime(hostname: String): Future[Int] =
    Future.successful(hosts.getOrElse(hostname, 0))
}
```

ここで、`UptimeService` の単体テストを書くことを考える。このサービスが値をどのように取得しているかには触れず、それを合計する能力をテストしたい。以下に例を示す。

```scala mdoc:fail
def testTotalUptime() = {
  val hosts    = Map("host1" -> 10, "host2" -> 6)
  val client   = new TestUptimeClient(hosts)
  val service  = new UptimeService(client)
  val actual   = service.getTotalUptime(hosts.keys.toList)
  val expected = hosts.values.sum
  assert(actual == expected)
}
```

だが、このアプリケーションが非同期で実行されることを考慮していなかったというありがちなミス[^warnings]のせいで、このコードはコンパイルできない。`actual` は `Future[Int]` 型で、`expected` は `Int` 型である。これらを直接比較することはできない。

[^warnings]: 厳密には、これは*警告*であってエラーではない。ここでは `scalac` に `-Xfatal-warnings` フラグを設定していることによりエラー扱いされている。

これを解決する方法はいくつかある。テストコードを非同期処理に対応させることもできるが、別の選択肢もある。サービスクラスのコードを同期的にして先ほどのテストを変更することなく動作させてみよう。

## 型コンストラクタの抽象化

`UptimeClient` には、本番で使う非同期的なものと単体テストで使う同期的なもの、二種類の実装が必要となる。

```scala
trait RealUptimeClient extends UptimeClient {
  def getUptime(hostname: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient {
  def getUptime(hostname: String): Int
}
```

問題は `UptimeClient` における抽象メソッドの戻り値型を何にするかである。`Future[Int]` と `Int` を抽象化する必要がある。

```scala
trait UptimeClient {
  def getUptime(hostname: String): ???
}
```

両方の型がもつ `Int` の部分を保持しつつ、テストコードでは `Future` の部分を取り除きたいということである。一見これは難しく思われるかもしれないが、幸いなことに Cats は `Id` 型という解決策を提供している。`Id` については[@sec:monads:identity]節で取り上げた。これを使えば、型をその意味を変えることなく型コンストラクタでラップすることができる。

```scala
package cats

type Id[A] = A
```

`Id` を使えば `UptimeClient` の戻り値型を抽象化できる。これを以下のとおり実装せよ。

- 型コンストラクタ `F[_]` をパラメータとして受け取るように `UptimeClient` トレイトを定義する
- `UptimeClient` を拡張したトレイト `RealUptimeClient` と `TestUptimeClient` を定義し、`F` をそれぞれ `Future` と `Id` に束縛する
- それぞれの `getUptime` のメソッドシグネチャを書き出し、それらがコンパイルされることを確認する

<div class="solution">

以下に実装例を示す。

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
```
```scala mdoc:silent
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient[Id] {
  def getUptime(hostname: String): Id[Int]
}
```

`Id[A]` は単なる `A` のエイリアスにすぎない。そのため、`TestUptimeClient` において型を `Id[Int]` とする必要はなく、単に `Int` と書くことができる。

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}
```
```scala mdoc:silent
trait TestUptimeClient extends UptimeClient[Id] {
  def getUptime(hostname: String): Int
}
```

もちろん、厳密に言えば `RealUptimeClient` や `TestUptimeClient` で `getUptime` を再定義する必要はないが、これらを書き出すことは技法の詳細を明らかにする助けとなるだろう。
</div>

これで、`TestUptimeClient` の定義を、以前と同じように `Map[String, Int]` を用いた完全なクラスへと肉付けすることができるはずである。これを実装せよ。

<div class="solution">

最終的なコードは元々の `TestUptimeClient` 実装と似たものとなる。ただし、`Future.successful` 呼び出しはもう必要ない。

```scala mdoc:reset-object:invisible
import scala.concurrent.Future
import cats.Id

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  def getUptime(hostname: String): Future[Int]
}
```
```scala mdoc:silent
class TestUptimeClient(hosts: Map[String, Int])
  extends UptimeClient[Id] {
  def getUptime(hostname: String): Int =
    hosts.getOrElse(hostname, 0)
}
```
</div>

## モナドの抽象化

`UptimeService` に目を向けよう。二種類の `UptimeClient` を抽象化できるようにこれを書き直す必要がある。この作業をふたつのステップに分けて行う。まず、クラスとメソッドのシグネチャを書き換え、それからメソッドの本体を修正する。以下のとおりメソッドシグネチャを書き換えよ。

- `getTotalUptime` の本体をコメントアウトする（`???` に置き換えてコンパイルできるようにしておく）
- `UptimeService` に型パラメータ `F[_]` を追加し、それを `UptimeClient` に渡す

<div class="solution">

コードは以下のようになるはずである。

```scala
class UptimeService[F[_]](client: UptimeClient[F]) {
  def getTotalUptime(hostnames: List[String]): F[Int] =
    ???
    // hostnames.traverse(client.getUptime).map(_.sum)
}
```
</div>

続いて、`getTotalUptime` のボディ部のコメントアウトをもとに戻す。次のようなコンパイルエラーが出力されるはずである。

```scala
// <console>:28: error: could not find implicit value for
//               evidence parameter of type cats.Applicative[F]
//            hostnames.traverse(client.getUptime).map(_.sum)
//                              ^
```

問題は、`traverse` が `Applicative` インスタンスをもつ値のシーケンスに対してしか使えないということである。元のコードでは `List[Future[Int]]` をトラバースしていた。`Future` には `Applicative` インスタンスが存在するので問題はなかった。しかし、今回のケースではトラバース対象が `List[F[Int]]` であるため、`F` が `Applicative` をもっていることをコンパイラに*証明*する必要がある。`UptimeService` のコンストラクタに暗黙のパラメータを追加し、これを実現せよ。

<div class="solution">

これは暗黙パラメータを用いて以下のように書くことができる。

```scala mdoc:invisible:reset-object
import cats.syntax.traverse._  // traverse
import cats.instances.list._

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}
```
```scala mdoc:silent
import cats.Applicative
import cats.syntax.functor._ // map

class UptimeService[F[_]](client: UptimeClient[F])
    (implicit a: Applicative[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

もしくはコンテキスト境界を使えばもっと簡潔に記述できる。

```scala mdoc:reset-object:invisible
import cats.Applicative
import cats.syntax.functor._
import cats.syntax.traverse._
import cats.instances.list._

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}
```
```scala mdoc:silent
class UptimeService[F[_]: Applicative]
    (client: UptimeClient[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

`cats.Applicative` だけでなく `cats.syntax.functor` もインポートする必要がある点に注意してほしい。`Future` の `map` メソッドの代わりに Cats が提供する拡張メソッドの `map` を使っているが、これが `Functor` 型の暗黙パラメータを必要とするからである。
</div>

最後に単体テストに目を向けよう。これまでに加えた修正によって、テストコードは何も変更しなくても意図したとおりに動作する。`TestUptimeClient` のインスタンスを作成し、それを `UptimeService` にラップすることで、`F` が `Id` にバインドされ、残りのコードはモナドやアプリカティブのことを気にせず同期的に動作できるようになる。

```scala mdoc:invisible:reset-object
import cats.{Id, Applicative}
import cats.instances.list._  // Traverse
import cats.syntax.functor._  // map
import cats.syntax.traverse._ // traverse
import scala.concurrent.Future

trait UptimeClient[F[_]] {
  def getUptime(hostname: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future]

class TestUptimeClient(hosts: Map[String, Int])
    extends UptimeClient[Id] {
  def getUptime(hostname: String): Int =
    hosts.getOrElse(hostname, 0)
  }

class UptimeService[F[_]: Applicative]
    (client: UptimeClient[F]) {

  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```
```scala mdoc:silent
def testTotalUptime() = {
  val hosts    = Map("host1" -> 10, "host2" -> 6)
  val client   = new TestUptimeClient(hosts)
  val service  = new UptimeService(client)
  val actual   = service.getTotalUptime(hosts.keys.toList)
  val expected = hosts.values.sum
  assert(actual == expected)
}

testTotalUptime()
```

## まとめ

このケーススタディで示したのは、Cats を用いて異なる計算シナリオを抽象化する例である。非同期コードと同期コードを抽象化するために `Applicative` 型クラスを使用した。関数型の抽象化を用いることで、実装の詳細を気にすることなく、実行したい一連の計算を記述することができる。

図[@fig:applicatives:hierarchy]では、まさにこの種の抽象化のために設計された計算型クラスのスタックを図示している。`Functor`、`Applicative`、`Monad`、`Traverse` といった型クラスは、マッピング、結合、順次実行、反復などのパターンの抽象的な実装を提供する。これらの型は、その数学的な法則によって、一貫したセマンティクスに基づく挙動を保証されている。

このケーススタディでは `Applicative` を使用した。この型クラスが、今回必要とする最低限の能力をもつものだったからである。もし `flatMap` が必要だったなら `Applicative` の代わりに `Monad` を使うこともできたし、異なるシーケンス型の抽象化を求められていたなら `Traverse` を使うこともできた。また、計算の成功だけでなく、失敗をモデリングする `ApplicativeError` や `MonadError` のような型クラスも存在する。

次はもっと複雑なケーススタディに進もう。型クラスを活用して、並列処理のための MapReduce スタイルのフレームワークを作るという興味深い事例を取り上げる。
