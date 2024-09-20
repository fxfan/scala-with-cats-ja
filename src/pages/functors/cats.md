## Catsにおけるファンクター Functors in Cats

Let's look at the implementation of functors in Cats.
We'll examine the same aspects we did for monoids:
the *type class*, the *instances*, and the *syntax*.

Catsにおけるファンクターの実装を見ていこう。調べるのは、モノイドのときと同じ3つの側面、*型クラス*、*インスタンス*、*構文*である。

### ファンクターの型クラスとインスタンス The Functor Type Class and Instances

The functor type class is [`cats.Functor`][cats.Functor].
We obtain instances using the standard `Functor.apply`
method on the companion object.
As usual, default instances are found on companion objects
and do not have to be explicity imported:

ファンクター型クラスは [`cats.Functor`][cats.Functor] として定義されている。インスタンスは、Catsの標準的な設計に従い、コンパニオンオブジェクト上の `Functor.apply` メソッドを使って取得する。通常通り、デフォルトのインスタンスはコンパニオンオブジェクト上に置かれており、明示的にインポートする必要はない。

```scala mdoc:silent:reset-object
import cats.*
import cats.syntax.all.*
```

```scala mdoc
val list1 = List(1, 2, 3)
val list2 = Functor[List].map(list1)(_ * 2)

val option1 = Option(123)
val option2 = Functor[Option].map(option1)(_.toString)
```

`Functor` provides a method called `lift`,
which converts a function of type `A => B`
to one that operates over a functor and has type `F[A] => F[B]`:

`Functor` は `lift` というメソッドを提供している。これは、`A => B` 型の関数を、ファンクター内の値を操作する `F[A] => F[B]` 型の関数に変換する。

```scala mdoc
val func = (x: Int) => x + 1

val liftedFunc = Functor[Option].lift(func)

liftedFunc(Option(1))
```

The `as` method is the other method you are likely to use.
It replaces with value inside the `Functor` with the given value.

もうひとつよく使うメソッドとして `as` がある。これは、ファンクター内の値を指定された値に置き換えるメソッドである。

```scala mdoc
Functor[List].as(list1, "As")
```

### ファンクターの構文 Functor Syntax

The main method provided by the syntax for `Functor` is `map`.
It's difficult to demonstrate this with `Options` and `Lists`
as they have their own built-in `map` methods
and the Scala compiler will always prefer
a built-in method over an extension method.
We'll work around this with two examples.

`Functor` の構文によって提供される主なメソッドは `map` だが、これを `Option` や `List` を使って実演するのは難しい。それらのクラスには元々 `map` メソッドが組み込まれており、そしてScalaコンパイラは常に拡張メソッドよりも組み込みのメソッドを優先するからである。以下では、このような問題をもたない例を二つ挙げて説明する。

First let's look at mapping over functions.
Scala's `Function1` type doesn't have a `map` method
(it's called `andThen` instead)
so there are no naming conflicts:

まずは関数に対する変換を見てみよう。Scalaの `Function1` 型には `map` メソッドが存在しない（同じ機能をもつメソッドは `andThen` と呼ばれる）ので、ファンクターの構文と名前が衝突することはない。

```scala mdoc:silent
val func1 = (a: Int) => a + 1
val func2 = (a: Int) => a * 2
val func3 = (a: Int) => s"${a}!"
val func4 = func1.map(func2).map(func3)
```

```scala mdoc
func4(123)
```

Let's look at another example.
This time we'll abstract over functors
so we're not working with any particular concrete type.
We can write a method that applies an equation to a number
no matter what functor context it's in:

別の例を見てみよう。今回は、特定の具体的な型に依存しないように、ファンクター全般を抽象化して扱う。数値がどのようなファンクターのコンテキストに包まれていようと、それに対して計算処理を適用するメソッドを記述できる。

```scala mdoc:silent
def doMath[F[_]](start: F[Int])
    (implicit functor: Functor[F]): F[Int] =
  start.map(n => n + 1 * 2)
```

```scala mdoc
doMath(Option(20))
doMath(List(1, 2, 3))
```

To illustrate how this works,
let's take a look at the definition of
the `map` method in `cats.syntax.functor`.
Here's a simplified version of the code:

これがどのように機能するかを示すために、`cats.syntax.functor` 内の `map` メソッドの定義を見てみよう。以下はそのコードを簡略化したものである。

```scala
implicit class FunctorOps[F[_], A](src: F[A]) {
  def map[B](func: A => B)
      (implicit functor: Functor[F]): F[B] =
    functor.map(src)(func)
}
```

The compiler can use this extension method
to insert a `map` method wherever no built-in `map` is available:

ある型に対して、その型自体に `map` メソッドが定義されていなければ、コンパイラは、この拡張メソッドを使って `map` メソッドを挿入できる。

```scala
foo.map(value => value + 1)
```

Assuming `foo` has no built-in `map` method,
the compiler detects the potential error and
wraps the expression in a `FunctorOps` to fix the code:

`foo` それ自体は `map` メソッドをもたないと仮定すると、コンパイラはこの記述がこのままではエラーになることを検出し、それを修正するために式を `FunctorOps` で包み込む。

```scala
new FunctorOps(foo).map(value => value + 1)
```

The `map` method of `FunctorOps` requires
an implicit `Functor` as a parameter.
This means this code will only compile
if we have a `Functor` for `F` in scope.
If we don't, we get a compiler error:

`FunctorOps` の `map` メソッドは暗黙の `Functor` インスタンスをパラメータとして要求する。つまり、`F` のための `Functor` インスタンスがスコープ内に存在すれば、このコードはコンパイルできるが、そうでなければコンパイルエラーとなる。

```scala mdoc:silent
final case class Box[A](value: A)

val box = Box[Int](123)
```

```scala mdoc:fail
box.map(value => value + 1)
```

The `as` method is also available as syntax.

`as` メソッドも構文として利用できる。

```scala mdoc
List(1, 2, 3).as("As")
```

### 独自型のためのインスタンス Instances for Custom Types

We can define a functor simply by defining its map method.
Here's an example of a `Functor` for `Option`,
even though such a thing already exists in [`cats.instances`][cats.instances].
The implementation is trivial---we simply call `Option's` `map` method:

ファンクターを定義するのは簡単で、`map` メソッドを定義するだけでよい。`Option` 用の `Functor` は [`cats.instances`][cats.instances] に既に存在するが、これを例として以下に示す。実装は単純で、`Option` の `map` メソッドを呼び出すだけである。

```scala
implicit val optionFunctor: Functor[Option] =
  new Functor[Option] {
    def map[A, B](value: Option[A])(func: A => B): Option[B] =
      value.map(func)
  }
```

Sometimes we need to inject dependencies into our instances.
For example, if we had to define a custom `Functor` for `Future`
(another hypothetical example---Cats provides one in `cats.instances.future`)
we would need to account for the implicit `ExecutionContext` parameter on `future.map`.
We can't add extra parameters to `functor.map`
so we have to account for the dependency when we create the instance:

インスタンスに依存性を注入しなければならないことが時々ある。たとえば、`Future` のために独自の `Functor` を定義する場合（これも仮の例である。Catsはこれを `cats.instances.future` で提供している）、`Future` の `map` メソッドが必要とする `ExecutionContext` 型の暗黙パラメータについて考慮する必要がある。`Functor` の `map` に追加のパラメータを渡すことはできないので、この依存性はインスタンスを作成するときに解決しなければならない。

```scala mdoc:silent
import scala.concurrent.{Future, ExecutionContext}

implicit def futureFunctor
    (implicit ec: ExecutionContext): Functor[Future] =
  new Functor[Future] {
    def map[A, B](value: Future[A])(func: A => B): Future[B] =
      value.map(func)
  }
```

Whenever we summon a `Functor` for `Future`,
either directly using `Functor.apply`
or indirectly via the `map` extension method,
the compiler will locate `futureFunctor` by implicit resolution
and recursively search for an `ExecutionContext` at the call site.
This is what the expansion might look like:

`Future` 用の `Functor` インスタンスを呼び出すと、コンパイラは暗黙の解決によって、まず  `futureFunctor` 関数を発見する。次に、再帰的な解決により `ExecutionContext` インスタンスを呼び出し元で探す。この振る舞いは、`Functor.apply` を直接使う場合でも、`map` 拡張メソッド経由で間接的に呼び出す場合でも変わらない。コンパイラによるこの展開は以下のようになるだろう。

```scala
// We write this:
// 実際に記述するコード
Functor[Future]

// The compiler expands to this first:
// コンパイラはまずこのように展開する
Functor[Future](futureFunctor)

// And then to this:
// そしてこう
Functor[Future](futureFunctor(executionContext))
```

### 演習: Branching out with Functors

Write a `Functor` for the following binary tree data type.
Verify that the code works as expected on instances of `Branch` and `Leaf`:

以下の二分木データ型に対して `Functor` を作成し、`Branch` と `Leaf` のインスタンスについて期待通りに動作するか確認せよ。

```scala mdoc:silent
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A])
  extends Tree[A]

final case class Leaf[A](value: A) extends Tree[A]
```

<div class="solution">
The semantics are similar to writing a `Functor` for `List`.
We recurse over the data structure, applying the function to every `Leaf` we find.
The functor laws intuitively require us to retain the same structure
with the same pattern of `Branch` and `Leaf` nodes:

セマンティクスは `List` に対して `Functor` を作成する場合と似ている。データ構造を再帰的に処理し、見つかったすべての `Leaf` に関数を適用する。ファンクター則により、`Branch` と `Leaf` の構造はそのまま保たれる必要がある。

```scala mdoc:silent
implicit val treeFunctor: Functor[Tree] =
  new Functor[Tree] {
    def map[A, B](tree: Tree[A])(func: A => B): Tree[B] =
      tree match {
        case Branch(left, right) =>
          Branch(map(left)(func), map(right)(func))
        case Leaf(value) =>
          Leaf(func(value))
      }
  }
```

Let's use our `Functor` to transform some `Trees`:

では、この `Functor` を用いて `Tree` を変換してみよう。

```scala mdoc:fail
Branch(Leaf(10), Leaf(20)).map(_ * 2)
```

Oops! This falls foul of
the same invariance problem we discussed in Section [@sec:variance].
The compiler can find a `Functor` instance for `Tree` but not for `Branch` or `Leaf`.
Let's add some smart constructors to compensate:

だがこのコードは [@sec:variance] で議論した非変性の問題に引っかかる。コンパイラは `Tree` に対しては `Functor` インスタンスを見つけることができるが、`Branch` や `Leaf` に対しては見つけられない。この問題を補うためにスマートコンストラクタを追加しよう。

<!--
TODO: 非変性の問題に引っかかる、の部分をもうすこし具体性のある文にしたい
-->

```scala mdoc:silent
object Tree {
  def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
    Branch(left, right)

  def leaf[A](value: A): Tree[A] =
    Leaf(value)
}
```

Now we can use our `Functor` properly:

これで、この `Functor` は正常に利用可能となる。

```scala mdoc
Tree.leaf(100).map(_ * 2)

Tree.branch(Tree.leaf(10), Tree.leaf(20)).map(_ * 2)
```
</div>
