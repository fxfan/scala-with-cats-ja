<!--

## Monoids in Cats

Now we've seen what monoids are,
let's look at their implementation in Cats.
Once again we'll look at the three main aspects of the implementation:
the *type class*, the *instances*, and the *interface*.

### The Monoid Type Class

The monoid type class is `cats.kernel.Monoid`,
which is aliased as [`cats.Monoid`][cats.kernel.Monoid].
`Monoid` extends `cats.kernel.Semigroup`,
which is aliased as [`cats.Semigroup`][cats.kernel.Semigroup].
When using Cats we normally import type classes
from the [`cats`][cats.package] package:

```scala mdoc:silent
import cats.Monoid
import cats.Semigroup
```

or just

```scala mdoc:reset:silent
import cats.*
```

<div class="callout callout-info">
#### Cats Kernel? {-}

Cats Kernel is a subproject of Cats
providing a small set of typeclasses
for libraries that don't require the full Cats toolbox.
While these core type classes are technically
defined in the [`cats.kernel`][cats.kernel.package] package,
they are all aliased to the [`cats`][cats.package] package
so we rarely need to be aware of the distinction.

The Cats Kernel type classes covered in this book are
[`Eq`][cats.kernel.Eq],
[`Semigroup`][cats.kernel.Semigroup],
and [`Monoid`][cats.kernel.Monoid].
All the other type classes we cover
are part of the main Cats project and
are defined directly in the [`cats`][cats.package] package.
</div>


### Monoid Instances {#sec:monoid-instances}

`Monoid` follows the standard Cats pattern for the user interface:
the companion object has an `apply` method
that returns the type class instance for a particular type.
For example, if we want the monoid instance for `String`,
and we have the correct given instances in scope,
we can write the following:

```scala mdoc:silent
import cats.Monoid
```

```scala mdoc
Monoid[String].combine("Hi ", "there")
Monoid[String].empty
```

which is equivalent to:

```scala mdoc
Monoid.apply[String].combine("Hi ", "there")
Monoid.apply[String].empty
```

As we know, `Monoid` extends `Semigroup`.
If we don't need `empty` we can equivalently write:

```scala mdoc:silent
import cats.Semigroup
```

```scala mdoc
Semigroup[String].combine("Hi ", "there")
```

The standard type class instances for `Monoid`
are all found on the appropriate companion objects,
and so are automatically in the given scope with no further imports required.


### Monoid Syntax {#sec:monoid-syntax}

Cats provides syntax for the `combine` method
in the form of the `|+|` operator.
Because `combine` technically comes from `Semigroup`,
we access the syntax by importing from [`cats.syntax.semigroup`][cats.syntax.semigroup]:

```scala mdoc:silent
import cats.syntax.semigroup.* // for |+|
```

```scala mdoc
val stringResult = "Hi " |+| "there" |+| Monoid[String].empty

val intResult = 1 |+| 2 |+| Monoid[Int].empty
```

As always,
unless there is compelling reason not, we recommend importing all the syntax with

```scala mdoc:silent
import cats.syntax.all.*
```


#### Exercise: Adding All The Things

The cutting edge *SuperAdder v3.5a-32* is the world's first choice for adding together numbers.
The main function in the program has signature `def add(items: List[Int]): Int`.
In a tragic accident this code is deleted! Rewrite the method and save the day!

<div class="solution">
We can write the addition as a `foldLeft` using `0` and the `+` operator:

```scala mdoc:silent
def add(items: List[Int]): Int =
  items.foldLeft(0)(_ + _)
```

We can alternatively write the fold using `Monoids`,
although there's not a compelling use case for this yet:

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.syntax.all.*

def add(items: List[Int]): Int =
  items.foldLeft(Monoid[Int].empty)(_ |+| _)
```
</div>

Well done! SuperAdder's market share continues to grow,
and now there is demand for additional functionality.
People now want to add `List[Option[Int]]`.
Change `add` so this is possible.
The SuperAdder code base is of the highest quality,
so make sure there is no code duplication!

<div class="solution">
Now there is a use case for `Monoids`.
We need a single method that adds `Ints` and instances of `Option[Int]`.
We can write this as a generic method that accepts an implicit `Monoid` as a parameter:

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.syntax.all.*

def add[A](items: List[A])(using monoid: Monoid[A]): A =
  items.foldLeft(monoid.empty)(_ |+| _)
```

We can optionally use Scala's *context bound* syntax to write the same code in a shorter way:

```scala mdoc:invisible:reset-object
import cats.Monoid
import cats.syntax.all.* 
```
```scala mdoc:silent
def add[A: Monoid](items: List[A]): A =
  items.foldLeft(Monoid[A].empty)(_ |+| _)
```

We can use this code to add values of type `Int` and `Option[Int]` as requested:

```scala mdoc
add(List(1, 2, 3))
```

```scala mdoc
add(List(Some(1), None, Some(2), None, Some(3)))
```

Note that if we try to add a list consisting entirely of `Some` values,
we get a compile error:

```scala mdoc:fail
add(List(Some(1), Some(2), Some(3)))
```

This happens because the inferred type of the list is `List[Some[Int]]`,
while Cats will only generate a `Monoid` for `Option[Int]`.
We'll see how to get around this in a moment.
</div>

SuperAdder is entering the POS (point-of-sale, not the other POS) market.
Now we want to add up `Orders`:

```scala mdoc:silent
case class Order(totalCost: Double, quantity: Double)
```

We need to release this code really soon so we can't make any modifications to `add`.
Make it so!

<div class="solution">
Easy---we simply define a monoid instance for `Order`!

```scala mdoc:silent
given monoid: Monoid[Order] with {
  def combine(o1: Order, o2: Order) =
    Order(
      o1.totalCost + o2.totalCost,
      o1.quantity + o2.quantity
    )

  def empty = Order(0, 0)
}
```
</div>


```scala mdoc:reset:silent
```
--->

## Cats におけるモノイド

ここまで、モノイドとは何かを見てきた。次に、Cats におけるモノイドの実装を見ていこう。ここでも、実装の三つの主要な側面である*型クラス*、*インスタンス*、そして*利用インターフェース*に注目していく。

### モノイド型クラス

The monoid type class is `cats.kernel.Monoid`,
which is aliased as [`cats.Monoid`][cats.kernel.Monoid].
`Monoid` extends `cats.kernel.Semigroup`,
which is aliased as [`cats.Semigroup`][cats.kernel.Semigroup].
When using Cats we normally import type classes
from the [`cats`][cats.package] package:

モノイドを表す型クラスは `cats.kernel.Monoid` である。これには [`cats.Monoid`][cats.kernel.Monoid] としてエイリアスが定義されている。`Monoid` は `cats.kernel.Semigroup`（[`cats.Semigroup`][cats.kernel.Semigroup] というエイリアスをもつ）を拡張している。Cats を使用するときは、通常 [`cats`][cats.package] パッケージから型クラスをインポートする。

```scala mdoc:silent
import cats.Monoid
import cats.Semigroup
```

あるいは単に次のようにインポートしてもよい。

```scala mdoc:reset:silent
import cats.*
```

<div class="callout callout-info">
#### Cats Kernelとは {-}

Cats Kernel は Cats のサブプロジェクトで、Cats 全体の広範な機能群を必要としないライブラリ向けに、一部の型クラスだけを提供している。Cats のコアであるこれらの型クラスは形式上は [`cats.kernel`][cats.kernel.package] パッケージに定義されているが、すべて [`cats`][cats.package] パッケージにエイリアスされているため、通常は両者の違いを意識する必要はない。

本書で扱う Cats Kernel の型クラスは、[`Eq`][cats.kernel.Eq]、[`Semigroup`][cats.kernel.Semigroup]、および [`Monoid`][cats.kernel.Monoid] である。これ以外に本書で扱う型クラスはすべて Cats のメインとなるプロジェクトに属しており、[`cats`][cats.package] パッケージに直接定義されている。
</div>


### `Monoid` インスタンス {#sec:monoid-instances}

`Monoid` のユーザインターフェースは Cats の標準的なパターンに従っており、コンパニオンオブジェクトには、特定の型のための型クラスインスタンスを返す `apply` メソッドが用意されている。たとえば、`String` に対するモノイドインスタンスを取得したい場合、スコープ内に適切な given インスタンスがあれば、以下のように記述できる。

```scala mdoc:silent
import cats.Monoid
```

```scala mdoc
Monoid[String].combine("Hi ", "there")
Monoid[String].empty
```

これは以下のように書くのと同義である。

```scala mdoc
Monoid.apply[String].combine("Hi ", "there")
Monoid.apply[String].empty
```

すでに学んだとおり、`Monoid` は `Semigroup` を拡張している。`empty` を必要としない場合は、次のように書き換えることもできる。

```scala mdoc:silent
import cats.Semigroup
```

```scala mdoc
Semigroup[String].combine("Hi ", "there")
```

The standard type class instances for `Monoid`
are all found on the appropriate companion objects,
and so are automatically in the given scope with no further imports required.

`Monoid` の標準的な型クラスインスタンスはすべて、適切なコンパニオンオブジェクトに定義されており、追加のインポートをせずとも、自動的に given スコープに含まれている。

### モノイドの構文 {#sec:monoid-syntax}

Catsは `combine` メソッド用の構文として `|+|` 演算子を提供している。`combine` は正確には `Semigroup` に定義されているメソッドなので、この構文は [`cats.syntax.semigroup`][cats.syntax.semigroup] からインポートして利用する。

```scala mdoc:silent
import cats.syntax.semigroup.* // |+| をインポートする
```

```scala mdoc
val stringResult = "Hi " |+| "there" |+| Monoid[String].empty

val intResult = 1 |+| 2 |+| Monoid[Int].empty
```

As always,
unless there is compelling reason not, we recommend importing all the syntax with

いつもと同様、特別な理由がないかぎりすべてのシンタックスをインポートすることが推奨される。

```scala mdoc:silent
import cats.syntax.all.*
```

#### 演習: すべてを加算する

SuperAdder v3.5a-32 は、数値の加算に関しては世界でも並ぶもののない、最先端ソフトウェアである。このプログラムの中核となるのは `def add(items: List[Int]): Int` というシグネチャをもった関数だが、不幸な事故でこのコードが削除されてしまった。メソッドを書き直して窮地を救え。

<div class="solution">
加算は `0` と `+` 演算子を使った `foldLeft` として書ける。

```scala mdoc:silent
def add(items: List[Int]): Int =
  items.foldLeft(0)(_ + _)
```

現時点では特段の必要性はないものの、`Monoid` を使ってこの畳み込み処理を実装することもできる。

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.syntax.all.*

def add(items: List[Int]): Int =
  items.foldLeft(Monoid[Int].empty)(_ |+| _)
```
</div>

メソッドの復旧は成功したようだ。さて、SuperAdder の市場シェアは成長を続けており、新たな機能が求められている。今回ユーザが求めているのは `List[Option[Int]]` を加算する機能である。この要望に応えるため、`add` の改修を行え。SuperAdder のコードベースは最高品質をもっているため、コードの重複がないように注意すること。

<div class="solution">
ここで `Monoid` を使うべきユースケースが登場する。`Int` の加算と `Option[Int]` の加算を、ひとつのメソッドで行える必要がある。これは、暗黙の `Monoid` インスタンスをパラメータとして受け取るジェネリックメソッドとして記述できる。

```scala mdoc:silent:reset-object
import cats.Monoid
import cats.syntax.all.*

def add[A](items: List[A])(using monoid: Monoid[A]): A =
  items.foldLeft(monoid.empty)(_ |+| _)
```

Scala の*コンテキスト境界（context bound）*を用いて、同じ意味のコードをもっとシンプルに書くこともできる。

```scala mdoc:invisible:reset-object
import cats.Monoid
import cats.syntax.all.* 
```
```scala mdoc:silent
def add[A: Monoid](items: List[A]): A =
  items.foldLeft(Monoid[A].empty)(_ |+| _)
```

このコードを使えば、要望どおりに `Int` の加算と `Option[Int]` の加算を行うことができる。

```scala mdoc
add(List(1, 2, 3))
```

```scala mdoc
add(List(Some(1), None, Some(2), None, Some(3)))
```

`Some` だけで構成されたリストを足し合わせようとするとコンパイルエラーになることに注意してほしい。

```scala mdoc:fail
add(List(Some(1), Some(2), Some(3)))
```

これがエラーになるのは、リストの型が `List[Some[Int]]` であると推論されるのに対して、Cats が `Option[Int]` のための `Monoid` インスタンスしか生成しないからである。後ほど、この問題の回避方法について見ていく。
</div>

SuperAdder は POS 市場に参入しようとしており、下記のような `Order` 型で表される注文を合計する必要がでてきた。

```scala mdoc:silent
case class Order(totalCost: Double, quantity: Double)
```

この機能はすぐにリリースする必要があり、`add` に変更を加える余裕はない。`add` に手を加えずにこの機能を実現せよ。

<div class="solution">
`Order` 用のモノイドインスタンスを定義するだけでよい。

```scala mdoc:silent
given monoid: Monoid[Order] with {
  def combine(o1: Order, o2: Order) =
    Order(
      o1.totalCost + o2.totalCost,
      o1.quantity + o2.quantity
    )

  def empty = Order(0, 0)
}
```
</div>
