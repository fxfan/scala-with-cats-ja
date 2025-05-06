<!--

## Defining Custom Monads

We can define a `Monad` for a custom type
by providing implementations of three methods:
`flatMap`, `pure`, and
a method we haven't seen yet called `tailRecM`.
Here is an implementation of `Monad` for `Option` as an example:

```scala mdoc:silent:reset-object
import cats.Monad
import scala.annotation.tailrec

val optionMonad = new Monad[Option] {
  def flatMap[A, B](opt: Option[A])
      (fn: A => Option[B]): Option[B] =
    opt.flatMap(fn)

  def pure[A](opt: A): Option[A] =
    Some(opt)

  @tailrec
  def tailRecM[A, B](a: A)(fn: A => Option[Either[A, B]]): Option[B] = {
    fn(a) match {
      case None           => None
      case Some(Left(a1)) => tailRecM(a1)(fn)
      case Some(Right(b)) => Some(b)
    }
  }
}
```

The `tailRecM` method is an optimisation used in Cats to limit
the amount of stack space consumed by nested calls to `flatMap`.
The technique comes from a [2015 paper][link-phil-freeman-tailrecm]
by PureScript creator Phil Freeman.
The method should recursively call itself
until the result of `fn` returns a `Right`.

To motivate its use let's use the following example:
Suppose we want to write a method that calls a
function until the function indicates it should stop.
The function will return a monad instance because,
as we know,
monads represent sequencing
and many monads have some notion of stopping.

We can write this method in terms of `flatMap`.

```scala mdoc:silent
import cats.syntax.flatMap._ // For flatMap

def retry[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  f(start).flatMap{ a =>
    retry(a)(f)
  }
```

Unfortunately it is not stack-safe.
It works for small input.

```scala mdoc
import cats.instances.option._

retry(100)(a => if(a == 0) None else Some(a - 1))
```

but if we try large input we get a `StackOverflowError`.

```scala
retry(100000)(a => if(a == 0) None else Some(a - 1))
// KABLOOIE!!!!
```

We can instead rewrite this method
using `tailRecM`.

```scala mdoc:silent
import cats.syntax.functor._ // for map

def retryTailRecM[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  Monad[F].tailRecM(start){ a =>
    f(a).map(a2 => Left(a2))
  }
```

Now it runs successfully
no matter how many time we recurse.

```scala mdoc
retryTailRecM(100000)(a => if(a == 0) None else Some(a - 1))
```

It's important to note
that we have to explicitly call `tailRecM`.
There isn't a code transformation
that will convert non-tail recursive
code into tail recursive code
that uses `tailRecM`.
However there are several utilities
provided by the `Monad` type class
that makes these kinds of methods easier to write.
For example, we can rewrite `retry`
in terms of `iterateWhileM`
and we don't have to explicitly call `tailRecM`.

```scala mdoc:silent
import cats.syntax.monad._ // for iterateWhileM

def retryM[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  start.iterateWhileM(f)(a => true)
```

```scala mdoc
retryM(100000)(a => if(a == 0) None else Some(a - 1))
```

We'll see more methods that use `tailRecM` in Section [@sec:foldable].

All of the built-in monads in Cats have
tail-recursive implementations of `tailRecM`,
although writing one for custom monads
can be a challenge... as we shall see.

### Exercise: Branching out Further with Monads

Let's write a `Monad` for our `Tree` data type from last chapter.
Here's the type again:

```scala mdoc:silent
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A])
  extends Tree[A]

final case class Leaf[A](value: A) extends Tree[A]

def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
  Branch(left, right)

def leaf[A](value: A): Tree[A] =
  Leaf(value)
```

Verify that the code works on instances of `Branch` and `Leaf`,
and that the `Monad` provides `Functor`-like behaviour for free.

Also verify that having a `Monad` in scope allows us to use for comprehensions,
despite the fact that we haven't directly implemented `flatMap` or `map` on `Tree`.

Don't feel you have to make `tailRecM` tail-recursive.
Doing so is quite difficult.
We've included both tail-recursive
and non-tail-recursive implementations
in the solutions so you can check your work.

<div class="solution">
The code for `flatMap` is similar to the code for `map`.
Again, we recurse down the structure
and use the results from `func` to build a new `Tree`.

The code for `tailRecM` is fairly complex
regardless of whether we make it tail-recursive or not.

If we follow the types,
the non-tail-recursive solution falls out:

```scala mdoc:silent
import cats.Monad

implicit val treeMonad: Monad[Tree] = new Monad[Tree] {
  def pure[A](value: A): Tree[A] =
    Leaf(value)

  def flatMap[A, B](tree: Tree[A])
      (func: A => Tree[B]): Tree[B] =
    tree match {
      case Branch(l, r) =>
        Branch(flatMap(l)(func), flatMap(r)(func))
      case Leaf(value)  =>
        func(value)
    }

  def tailRecM[A, B](a: A)(func: A => Tree[Either[A, B]]): Tree[B] = {
    flatMap(func(a)) {
      case Left(value) =>
        tailRecM(value)(func)
      case Right(value) =>
        Leaf(value)
    }
  }
}
```

The solution above is perfectly fine for this exercise.
Its only downside is that Cats cannot make guarantees about stack safety.

The tail-recursive solution is much harder to write.
We adapted this solution from
[this Stack Overflow post][link-so-tree-tailrecm] by Nazarii Bardiuk.
It involves an explicit depth first traversal of the tree,
maintaining an `open` list of nodes to visit
and a `closed` list of nodes to use to reconstruct the tree:

```scala mdoc:invisible:reset-object
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A])
  extends Tree[A]

final case class Leaf[A](value: A) extends Tree[A]

def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
  Branch(left, right)

def leaf[A](value: A): Tree[A] =
  Leaf(value)
```
```scala mdoc:silent
import cats.Monad
import scala.annotation.tailrec

implicit val treeMonad: Monad[Tree] = new Monad[Tree] {
  def pure[A](value: A): Tree[A] =
    Leaf(value)

  def flatMap[A, B](tree: Tree[A])
      (func: A => Tree[B]): Tree[B] =
    tree match {
      case Branch(l, r) =>
        Branch(flatMap(l)(func), flatMap(r)(func))
      case Leaf(value)  =>
        func(value)
    }

  def tailRecM[A, B](arg: A)
      (func: A => Tree[Either[A, B]]): Tree[B] = {
    @tailrec
    def loop(
          open: List[Tree[Either[A, B]]],
          closed: List[Option[Tree[B]]]): List[Tree[B]] =
      open match {
        case Branch(l, r) :: next =>
          loop(l :: r :: next, None :: closed)

        case Leaf(Left(value)) :: next =>
          loop(func(value) :: next, closed)

        case Leaf(Right(value)) :: next =>
          loop(next, Some(pure(value)) :: closed)

        case Nil =>
          closed.foldLeft(Nil: List[Tree[B]]) { (acc, maybeTree) =>
            maybeTree.map(_ :: acc).getOrElse {
              acc match {
                case left :: right :: tail => branch(left, right) :: tail
              }
            }
          }
      }

    loop(List(func(arg)), Nil).head
  }
}
```

Regardless of which version of `tailRecM` we define,
we can use our `Monad` to `flatMap` and `map` on `Trees`:

```scala mdoc:silent
import cats.syntax.functor._ // for map
import cats.syntax.flatMap._ // for flatMap
```

```scala mdoc
branch(leaf(100), leaf(200)).
  flatMap(x => branch(leaf(x - 1), leaf(x + 1)))
```

We can also transform `Trees` using for comprehensions:

```scala mdoc
for {
  a <- branch(leaf(100), leaf(200))
  b <- branch(leaf(a - 10), leaf(a + 10))
  c <- branch(leaf(b - 1), leaf(b + 1))
} yield c
```

The monad for `Option` provides fail-fast semantics.
The monad for `List` provides concatenation semantics.
What are the semantics of `flatMap` for a binary tree?
Every node in the tree has the potential to be replaced with a whole subtree,
producing a kind of "growing" or "feathering" behaviour,
reminiscent of list concatenation along two axes.
</div>


```scala mdoc:reset:silent
```
--->

## 独自のモナドを定義する

独自の型に対して `Monad` を定義するには、`flatMap` と `pure`、そしてこれまでは取り上げてこなかった `tailRecM` という三つのメソッドを実装すればよい。例として `Option` に対する `Monad` 実装を以下に示す。

```scala mdoc:silent:reset-object
import cats.Monad
import scala.annotation.tailrec

val optionMonad = new Monad[Option] {
  def flatMap[A, B](opt: Option[A])
      (fn: A => Option[B]): Option[B] =
    opt.flatMap(fn)

  def pure[A](opt: A): Option[A] =
    Some(opt)

  @tailrec
  def tailRecM[A, B](a: A)(fn: A => Option[Either[A, B]]): Option[B] = {
    fn(a) match {
      case None           => None
      case Some(Left(a1)) => tailRecM(a1)(fn)
      case Some(Right(b)) => Some(b)
    }
  }
}
```

`tailRecM` メソッドは、`flatMap` のネストされた呼び出しによって消費されるスタック領域を制限するために Cats で使われる最適化技法である。このテクニックは PureScript の作成者 Phil Freeman による[2015年の論文][link-phil-freeman-tailrecm]に由来している。このメソッドは、`fn` の結果が `Right` を返すまで再帰的に自身を呼び出す必要がある。

これを利用することの必要性について、例を挙げて説明しよう。ある関数を、それが停止を指示するまで繰り返し呼び出すメソッドを書きたいとする。この関数はモナドのインスタンスを返す。なぜなら、モナドは計算の連なりを表すし、また多くのモナドには停止の概念が含まれているからである。

`flatMap` を用いてこのようなメソッドを記述できる。

```scala mdoc:silent
import cats.syntax.flatMap._ // flatMap

def retry[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  f(start).flatMap{ a =>
    retry(a)(f)
  }
```

残念なことに、これはスタックセーフではない。このコードは入力が小さいときは正しく動作する。

```scala mdoc
import cats.instances.option._

retry(100)(a => if(a == 0) None else Some(a - 1))
```

だが、大きな値を入力すると `StackOverflowError` が発生する。

```scala
retry(100000)(a => if(a == 0) None else Some(a - 1))
// KABLOOIE!!!!
```

このメソッドは `tailRecM` を用いて書き直すことができる。

```scala mdoc:silent
import cats.syntax.functor._ // map

def retryTailRecM[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  Monad[F].tailRecM(start){ a =>
    f(a).map(a2 => Left(a2))
  }
```

これで、このメソッドはどれだけ多く再帰を繰り返しても正常に実行される。

```scala mdoc
retryTailRecM(100000)(a => if(a == 0) None else Some(a - 1))
```

`tailRecM` を明示的に呼び出す必要があるということに留意してほしい。非末尾再帰のコードを `tailRecM` を使った末尾再帰のコードへと自動的に変換するような仕組みは存在しない。ただし、`Monad` 型クラスは、このようなメソッドをより簡単に書けるようにするいくつかのユーティリティを提供している。たとえば、`retry` は `iterateWhileM` を使って書き直すことができ、その場合は `tailRecM` を直接呼び出さなくてもよい。

```scala mdoc:silent
import cats.syntax.monad._ // iterateWhileM

def retryM[F[_]: Monad, A](start: A)(f: A => F[A]): F[A] =
  start.iterateWhileM(f)(a => true)
```

```scala mdoc
retryM(100000)(a => if(a == 0) None else Some(a - 1))
```

[@sec:foldable]節では `tailRecM` を利用している他のメソッドについても見ていく。

Cats に組み込まれているモナドにはすべて `tailRecM` の末尾再帰の実装が備わっている。しかし、これから見ていくように、独自モナド用にこれを実装するのは難しい場合がある。

### 演習: モナドで広がる枝分かれ

前章で登場した `Tree` データ型に対して `Monad` を実装せよ。以下に `Tree` 型の定義を再掲する。

```scala mdoc:silent
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A])
  extends Tree[A]

final case class Leaf[A](value: A) extends Tree[A]

def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
  Branch(left, right)

def leaf[A](value: A): Tree[A] =
  Leaf(value)
```

作成したコードが `Branch` および `Leaf` のインスタンスに対して動作することを確認せよ。また、この `Monad` が `Functor` 的な振る舞いを自動的に提供してくれることを確認せよ。

また、`Tree` 自身が `flatMap` や `map` を直接実装していなくても、`Monad` がスコープにあれば for 内包表記が使えることも確認せよ。

`tailRecM` を末尾再帰にしなければならないと気負う必要はない。これを末尾再帰にするのは非常に難しい。解答には末尾再帰の実装とそうでない実装の両方を載せているので、自身の答えと照らし合わせてほしい。

<div class="solution">
`flatMap` のコードは `map` と似ている。`map` と同じように構造を辿りながら再帰し、各 `Leaf` に `func` を適用した結果を使って新しい `Tree` を構築する。

`tailRecM` のコードは、末尾再帰にするかどうかにかかわらず、かなり複雑である。

型に従って実装を進めると、自然と非末尾再帰の解答にたどり着く。

```scala mdoc:silent
import cats.Monad

implicit val treeMonad: Monad[Tree] = new Monad[Tree] {
  def pure[A](value: A): Tree[A] =
    Leaf(value)

  def flatMap[A, B](tree: Tree[A])
      (func: A => Tree[B]): Tree[B] =
    tree match {
      case Branch(l, r) =>
        Branch(flatMap(l)(func), flatMap(r)(func))
      case Leaf(value)  =>
        func(value)
    }

  def tailRecM[A, B](a: A)(func: A => Tree[Either[A, B]]): Tree[B] = {
    flatMap(func(a)) {
      case Left(value) =>
        tailRecM(value)(func)
      case Right(value) =>
        Leaf(value)
    }
  }
}
```

上記の解答はこの演習においては文句なく正解だが、唯一の問題は Cats がスタックセーフ性を保証できないことである。

末尾再帰の解答を記述するのは、はるかに難しい。以下の解答は、Nazarii Bardiuk 氏による [Stack Overflow への投稿][link-so-tree-tailrecm]から採用したものである。この解法では、ツリーを明示的に深さ優先で探索し、訪問すべきノードのリストである `open` と、ツリーを再構築するために使用するノードのリストである `closed` を管理する。

```scala mdoc:invisible:reset-object
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A])
  extends Tree[A]

final case class Leaf[A](value: A) extends Tree[A]

def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
  Branch(left, right)

def leaf[A](value: A): Tree[A] =
  Leaf(value)
```
```scala mdoc:silent
import cats.Monad
import scala.annotation.tailrec

implicit val treeMonad: Monad[Tree] = new Monad[Tree] {
  def pure[A](value: A): Tree[A] =
    Leaf(value)

  def flatMap[A, B](tree: Tree[A])
      (func: A => Tree[B]): Tree[B] =
    tree match {
      case Branch(l, r) =>
        Branch(flatMap(l)(func), flatMap(r)(func))
      case Leaf(value)  =>
        func(value)
    }

  def tailRecM[A, B](arg: A)
      (func: A => Tree[Either[A, B]]): Tree[B] = {
    @tailrec
    def loop(
          open: List[Tree[Either[A, B]]],
          closed: List[Option[Tree[B]]]): List[Tree[B]] =
      open match {
        case Branch(l, r) :: next =>
          loop(l :: r :: next, None :: closed)

        case Leaf(Left(value)) :: next =>
          loop(func(value) :: next, closed)

        case Leaf(Right(value)) :: next =>
          loop(next, Some(pure(value)) :: closed)

        case Nil =>
          closed.foldLeft(Nil: List[Tree[B]]) { (acc, maybeTree) =>
            maybeTree.map(_ :: acc).getOrElse {
              acc match {
                case left :: right :: tail => branch(left, right) :: tail
              }
            }
          }
      }

    loop(List(func(arg)), Nil).head
  }
}
```

どちらの `tailRecM` を定義したとしても、この `Monad` を用いれば `Tree` に対して `flatMap` と `map` を呼び出すことが可能である。

```scala mdoc:silent
import cats.syntax.functor._ // map
import cats.syntax.flatMap._ // flatMap
```

```scala mdoc
branch(leaf(100), leaf(200)).
  flatMap(x => branch(leaf(x - 1), leaf(x + 1)))
```

for 内包表記を用いて `Tree` を変換することもできる。

```scala mdoc
for {
  a <- branch(leaf(100), leaf(200))
  b <- branch(leaf(a - 10), leaf(a + 10))
  c <- branch(leaf(b - 1), leaf(b + 1))
} yield c
```

`Option` のモナドはフェイルファスト、そして `List` のモナドは連結というセマンティクスをもっている。では二分木における `flatMap` がもつセマンティクスは何だろうか。ツリー内にあるノードはすべて、サブツリー全体に置き換えられる可能性をもっている。これにより、リストが水平方向だけでなく縦にも結合されるような成長や広がりといった挙動が生み出されるのである。
</div>
