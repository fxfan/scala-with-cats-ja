<!--

### Traverse in Cats

Our `listTraverse` and `listSequence` methods
work with any type of `Applicative`,
but they only work with one type of sequence: `List`.
We can generalise over different sequence types using a type class,
which brings us to Cats' `Traverse`.
Here's the abbreviated definition:

```scala
package cats

trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B]
      (inputs: F[A])(func: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, B]
      (inputs: F[G[B]]): G[F[B]] =
    traverse(inputs)(identity)
}
```

Cats provides instances of `Traverse`
for `List`, `Vector`, `Stream`, `Option`, `Either`,
and a variety of other types.
We can summon instances as usual using `Traverse.apply`
and use the `traverse` and `sequence` methods
as described in the previous section:

```scala mdoc:invisible
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)

def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60)
```

```scala mdoc:silent
import cats.Traverse
import cats.instances.future._ // for Applicative
import cats.instances.list._   // for Traverse

val totalUptime: Future[List[Int]] =
  Traverse[List].traverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(totalUptime, 1.second)
```

```scala mdoc:silent
val numbers = List(Future(1), Future(2), Future(3))

val numbers2: Future[List[Int]] =
  Traverse[List].sequence(numbers)
```

```scala mdoc
Await.result(numbers2, 1.second)
```

There are also syntax versions of the methods,
imported via [`cats.syntax.traverse`][cats.syntax.traverse]:

```scala mdoc:silent
import cats.syntax.traverse._ // for sequence and traverse
```

```scala mdoc
val numbers3 = hostnames.traverse(getUptime)
val numbers4 = numbers.sequence

Await.result(numbers3, 1.second)
Await.result(numbers4, 1.second)
```

As you can see, this is much more compact and readable
than the `foldLeft` code we started with earlier this chapter!


```scala mdoc:reset:silent
```
--->

### Cats における `Traverse`

前節で見た `listTraverse` と `listSequence` メソッドは、任意の `Applicative` に対応しているが、シーケンスの型としては `List` にしか対応していない。型クラスを使えば、異なるシーケンス型を一般化することができる。ここで登場するのが Cats の `Traverse` である。簡略化した定義を以下に示す。

```scala
package cats

trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B]
      (inputs: F[A])(func: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, B]
      (inputs: F[G[B]]): G[F[B]] =
    traverse(inputs)(identity)
}
```

Cats では、`List`、`Vector`、`Stream`、`Option`、`Either` など、さまざまな型に対する `Traverse` インスタンスが提供されている。通常どおり `Traverse.apply` を使ってインスタンスを取得し、前節で説明したように `traverse` と `sequence` メソッドを使用することができる。

```scala mdoc:invisible
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)

def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60)
```

```scala mdoc:silent
import cats.Traverse
import cats.instances.future._ // for Applicative
import cats.instances.list._   // for Traverse

val totalUptime: Future[List[Int]] =
  Traverse[List].traverse(hostnames)(getUptime)
```

```scala mdoc
Await.result(totalUptime, 1.second)
```

```scala mdoc:silent
val numbers = List(Future(1), Future(2), Future(3))

val numbers2: Future[List[Int]] =
  Traverse[List].sequence(numbers)
```

```scala mdoc
Await.result(numbers2, 1.second)
```

メソッドの代わりに構文を利用することもできる。構文は [`cats.syntax.traverse`][cats.syntax.traverse] からインポートできる。

```scala mdoc:silent
import cats.syntax.traverse._ // sequence と traverse
```

```scala mdoc
val numbers3 = hostnames.traverse(getUptime)
val numbers4 = numbers.sequence

Await.result(numbers3, 1.second)
Await.result(numbers4, 1.second)
```

見てのとおり、この節のスタート地点であった `foldLeft` を用いたコードと比べると、はるかにコンパクトで読みやすい。
