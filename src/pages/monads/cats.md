## Catsにおけるモナド Monads in Cats

It's time to give monads our standard Cats treatment.
As usual we'll look at the type class, instances, and syntax.

ここからは、Catsの標準的な方法でモナドを扱う。いつものように、型クラス本体とインスタンスと構文について見ていこう。

### モナド型クラス The Monad Type Class {#monad-type-class}

The monad type class is [`cats.Monad`][cats.Monad].
`Monad` extends two other type classes:
`FlatMap`, which provides the `flatMap` method,
and `Applicative`, which provides `pure`.
`Applicative` also extends `Functor`,
which gives every `Monad` a `map` method
as we saw in the exercise above.
We'll discuss `Applicatives` in Chapter [@sec:applicatives].

モナド型クラスは [`cats.Monad`][cats.Monad] として提供されている。`Monad` は二つの型クラスを拡張している。そのうちのひとつ `FlatMap` は `flatMap` メソッドを提供し、もうひとつの `Applicative` は `pure` を提供する。`Applicative` はさらに `Functor` を拡張しており、これによってすべての `Monad` が `map` メソッドをもつ。これは前述の演習で見た通りである。`Applicative` については[@sec:applicatives]章で説明する。

Here are some examples using `pure` and `flatMap`, and `map` directly:

以下は、型クラス自体がもつ `pure`、`flatMap`、および `map` を直接使用した例である。

```scala mdoc:silent
import cats.Monad
```

```scala mdoc
val opt1 = Monad[Option].pure(3)
val opt2 = Monad[Option].flatMap(opt1)(a => Some(a + 2))
val opt3 = Monad[Option].map(opt2)(a => 100 * a)

val list1 = Monad[List].pure(3)
val list2 = Monad[List].
  flatMap(List(1, 2, 3))(a => List(a, a*10))
val list3 = Monad[List].map(list2)(a => a + 123)
```

`Monad` provides many other methods,
including all of the methods from `Functor`.
See the [scaladoc][cats.Monad] for more information.

`Monad` は、`Functor` から継承されたメソッドも含めて、他にも多くのメソッドを提供している。詳しくは [scaladoc][cats.Monad] を参照してほしい。

### デフォルトのインスタンス Default Instances

Cats provides instances for all the monads in the standard library
(`Option`, `List`, `Vector` and so on).
Cats also provides a `Monad` for `Future`.
Unlike the methods on the `Future` class itself,
the `pure` and `flatMap` methods on the monad
can't accept implicit `ExecutionContext` parameters
(because the parameters aren't part of the definitions in the `Monad` trait).
To work around this, Cats requires us to have an `ExecutionContext` in scope
when we summon a `Monad` for `Future`:

Catsは、標準ライブラリに存在するすべてのモナド（`Option`、`List`、`Vector` など）に対してインスタンスを提供している。さらに、Catsは `Future` 用の `Monad` インスタンスも提供している。ただし、`Future` クラス自体に定義されたメソッドとは異なり、モナドの `pure` や `flatMap` メソッドは、暗黙的な `ExecutionContext` パラメータを受け取ることができない。`Monad` トレイトの定義にそのようなパラメータが含まれていないためである。この問題を回避するために、Catsでは、`Future` の `Monad` インスタンスを呼び出す際に `ExecutionContext` をスコープに含める必要がある。

```scala mdoc:silent
import scala.concurrent.*
import scala.concurrent.duration.*
```

```scala mdoc:fail
val fm = Monad[Future]
```

Bringing the `ExecutionContext` into scope
fixes the implicit resolution required to summon the instance:

`ExecutionContext` をスコープに含めれば、インスタンスを取得するために必要な暗黙の解決が行われる。

```scala mdoc:silent
import scala.concurrent.ExecutionContext.Implicits.global
```

```scala mdoc
val fm = Monad[Future]
```

The `Monad` instance uses the captured `ExecutionContext`
for subsequent calls to `pure` and `flatMap`:

この `Monad` インスタンスは、暗黙的に渡された `ExecutionContext` を、この後に行われる `pure` や `flatMap` 呼び出しで利用する。

```scala mdoc:silent
val future = fm.flatMap(fm.pure(1))(x => fm.pure(x + 2))
```

```scala mdoc
Await.result(future, 1.second)
```

In addition to the above,
Cats provides a host of new monads that we don't have in the standard library.
We'll familiarise ourselves with some of these in a moment.

上記に加えて、Catsは標準ライブラリには存在しない多くの新しいモナドを提供している。これから、それらのいくつかについて学んでいくことにしよう。

### モナドの構文 Monad Syntax

The syntax for monads comes from three places:

 - [`cats.syntax.flatMap`][cats.syntax.flatMap]
   provides syntax for `flatMap`;
 - [`cats.syntax.functor`][cats.syntax.functor]
   provides syntax for `map`;
 - [`cats.syntax.applicative`][cats.syntax.applicative]
   provides syntax for `pure`.

モナドのための構文は次の三つのパッケージで提供されている。

 - [`cats.syntax.flatMap`][cats.syntax.flatMap]: `flatMap` 用の構文を提供する
 - [`cats.syntax.functor`][cats.syntax.functor]: `map` 用の構文を提供する
 - [`cats.syntax.applicative`][cats.syntax.applicative]: `pure` 用の構文を提供する

In practice it's often easier to import everything in one go
from `cats.syntax.all.**`.
However, we'll use the individual imports here for clarity.

実際には、`cats.syntax.all.*` からすべての構文をまとめてインポートする方が簡単なことが多い。しかし、ここでは明確さを重視し、個別のインポートを使用する。

We can use `pure` to construct instances of a monad.
We'll often need to specify the type parameter to disambiguate the particular instance we want.

モナドインスタンスの構築には `pure` を使うことができる。

```scala mdoc:silent
import cats.syntax.applicative.* // pure
```

```scala mdoc
1.pure[Option]
1.pure[List]
```

It's difficult to demonstrate the `flatMap` and `map` methods
directly on Scala monads like `Option` and `List`,
because they define their own explicit versions of those methods.
Instead we'll write a generic function that
performs a calculation on parameters
that come wrapped in a monad of the user's choice:

`Option` や `List` などScalaが標準で提供しているモナドについて、Catsの構文としての `flatMap` や `map` メソッドを使って見せるのは難しい。それらのメソッドはモナド自体に明示的に定義されているためである。そこで、プログラマが利用したいモナドに包まれたパラメータに対して計算を行う汎用的な関数を作成することにする。




```scala mdoc:silent
import cats.Monad
import cats.syntax.functor.* // map
import cats.syntax.flatMap.* // flatMap

def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  a.flatMap(x => b.map(y => x*x + y*y))
```

```scala mdoc
sumSquare(Option(3), Option(4))
sumSquare(List(1, 2, 3), List(4, 5))
```

We can rewrite this code using for comprehensions.
The compiler will "do the right thing" by
rewriting our comprehension in terms of `flatMap` and `map`
and inserting the correct conversions to use our `Monad`:

このコードはfor内包表記を使って書き直すことができる。コンパイラは、for内包表記を `flatMap` と `map` を用いたコードに変換し、`Monad` を利用するための正しい変換を挿入してくれる。

```scala mdoc:invisible:reset-object
import cats.Monad
import cats.syntax.all.*
```
```scala mdoc:silent
def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  for {
    x <- a
    y <- b
  } yield x*x + y*y
```

```scala mdoc
sumSquare(Option(3), Option(4))
sumSquare(List(1, 2, 3), List(4, 5))
```

That's more or less everything we need to know
about the generalities of monads in Cats.
Now let's take a look at some useful monad instances
that we haven't seen in the Scala standard library.

これで、Catsにおけるモナドの一般的な内容についてはおおむねすべて説明した。次に、Scalaの標準ライブラリでは見たことのない、有用なモナドインスタンスをいくつか見てみよう。
