<!--

## What Type Classes Are

We've have now seen the mechanics of type classes: they are a specific arrangement of trait, given instances, and using clauses. This is a very craft-level explanation. Let's now raise the level of the explanation with three different views of type classes.

The first view goes back Chapter [@sec:codata], where we looked at codata. The type class itself---the trait---is an example of codata with the usual advantages of codata (we can easily add implementations) and disadvantages (we cannot easily change the interface). Given instances and using clauses add the ability to chose the codata implementation based on the type of the context parameter and the instances in the given scope, and to compose instances from smaller components.

Raising the level of abstraction again, we can say that type classes allow us to implement functionality (the type class instance) separately from the type to which it applies, so that the implementation only needs to be defined at the point of the use---the call site---not at the point of declaration.

Raising the level again, we can say type classes allow us to implement **ad-hoc polymorphism**. I find it easiest to understand ad-hoc polymorphism in contrast to **parametric polymorphism**. Parametric polymorphism is what we get with type parameters, also known as generic types. It allows us to treat all types in a uniform way. For example, the following function calculates the length of any list of an arbitrary type `A`.

```scala mdoc:silent
def length[A](list: List[A]): Int =
  list match {
    case Nil => 0
    case x :: xs => 1 + length(xs)
  }
```

We can implement `length` because we don't require any particular functionality from the values of type `A` that make up the elements of the list. We don't call any methods on them, and indeed we cannot call any methods on them because we don't know what concrete type `A` will be at the point where `length` is defined[^abstraction].

Ad-hoc polymorphism allows us to call methods on values with a generic type. The methods we can call are exactly those defined by the type class. For example, we can use the `Numeric` type class from the standard library to write a method that adds together elements of any type that implements that type class.

```scala mdoc:silent
import scala.math.Numeric

def add[A](x: A, y: A)(using n: Numeric[A]): A = {
  n.plus(x, y)
}
```

So parametric polymorphism can be understood as meaning any type, while ad-hoc polymorphism means any type *that also implements this functionality*. In ad-hoc polymorphism there doesn't have to be any particular type relationship between the concrete types that implement the functionality of interest. This is in contast to object-oriented style polymorphism (i.e. codata) where all concrete types must be subtypes of the type that defines the functionality of interest.

[^abstraction]: Parametric polymorphism represents an abstraction boundary. At the point of definition we don't know the concrete types that `A` will take; the concrete types are only known at the point of use. (Once again we see the distinction between definition site and call site.) This abstraction boundary allows a kind of reasoning known as *free theorems* [@10.1145/99370.99404]. For example, if we see a function with type `A => A` we know it must be the identity function. This is the only possible function with this type. Unfortunately the JVM allows us to break the abstraction boundary introduced by parametric polymorphism. We can call `equals`, `hashCode`, and a few other methods on all values, and we can inspect runtime tags that reflect some type information at run-time.


```scala mdoc:reset:silent
```
--->

## 型クラスとは何か

ここまで型クラスの仕組みを見てきた。型クラスはトレイト、given インスタンス、using 句を組みわせることによって成り立っているわけだが、これは技法レベルの説明である。ここからは抽象度を上げ、型クラスを三つの異なる視点から考察したい。

最初の視点は、余データについて取り上げた[@sec:codata]章に戻る。トレイトとして定義される型クラス本体は余データの例であり、余データが通常そうであるように、実装を簡単に追加できるという利点と、インターフェースを簡単に変更できないという欠点をもっている。given インスタンスと using 句があることで、コンテキストパラメータの型と given スコープ内のインスタンスに基づいて余データの実装を選択することが可能となる。複数の小さなパーツからインスタンスを合成することもできるようになる。

もうひとつ抽象度を上げれば、型クラスとは、適用対象の型から切り離して機能（型クラスインスタンス）を実装できるようにする仕組みであると言える。機能の実装は、対象となる型の宣言時ではなく、使用時つまり呼び出される場所で定義されていればよい。

さらに抽象度を上げよう。型クラスとは**アドホック多相（ad-hoc polymorphism）**を実現するものであると言うことができる。アドホック多相を理解するには、**パラメトリック多相（parametric polymorphism）**と対比するのがもっとも簡単だろう。パラメトリック多相は、型パラメータ（ジェネリック型とも呼ばれる）によって実現されるもので、すべての型を一様に扱うことを可能にする。たとえば、次の関数は、任意の型 `A` のリストの長さを計算する。

```scala mdoc:silent
def length[A](list: List[A]): Int =
  list match {
    case Nil => 0
    case x :: xs => 1 + length(xs)
  }
```

`length` を実装できるのは、リストの要素である型 `A` の値に対して特別な機能を必要としないからである。`A` 型の値に対してメソッドを呼び出すことはないし、そもそも、`length` が定義される時点では、`A` が具体的にどの型になるかがわからないため、メソッドを呼び出すこともできない[^abstraction]。

アドホック多相により、ジェネリックな型の値に対してメソッドを呼び出すことが可能になる。呼び出せるメソッドは、型クラスによって定義されたものに限られる。たとえば、標準ライブラリの `Numeric` 型クラスを使えば、その型クラスを実装する任意の型の値同士を足し合わせるメソッドを作成することができる。

```scala mdoc:silent
import scala.math.Numeric

def add[A](x: A, y: A)(using n: Numeric[A]): A = {
  n.plus(x, y)
}
```

したがって、パラメトリック多相が「任意の型」に対する多相性である一方、アドホック多相は「ある特定の機能を実装している任意の型」に対するものをいう。アドホック多相では、ある特定の機能を実装している具体型同士が特別な関係性をもっている必要はない。これに対し、オブジェクト指向スタイルの多相性（つまり、余データ）では、具体型はすべて、その機能を定義している型の部分型でなくてはならない。

[^abstraction]: パラメトリック多相は抽象化の境界を表しており、型の具体的な詳細へのアクセスを制約している。定義の時点では、`A` が具体的にどの型をとるかはわからず、具体的な型は、使用時にのみ判明する。ここでも定義場所と呼び出し場所の違いが表れているのは興味深いが、それはさておき、この抽象化の境界によって、*自由定理（free theorems）*[@10.1145/99370.99404]と呼ばれる推論の一種が可能になる。たとえば、`A => A` という型をもつ関数があれば、それは恒等関数であることがわかる。この型をもつ可能性のある関数はそれだけである。ただし、JVM ではパラメトリック多相によって導入された抽象化の境界を破ることができる。すべての値に対して `equals` や `hashCode` などのメソッドを呼び出せる上、実行時の型情報が反映されたランタイムタグを調べることができてしまう。
