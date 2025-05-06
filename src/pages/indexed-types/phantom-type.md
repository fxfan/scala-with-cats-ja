<!--

## Phantom Types {#sec:indexed-types:phantom}

Phantom types are a basic building block of indexed types, so we'll start with an example of them. A phantom type is simply a type parameter that doesn't correspond to any value. In the example below, the type parameter `A` is a phantom type, because there is no value of type `A`, while `B` is not because there is a value of that type.

```scala mdoc:silent
final case class PhantomExample[A, B](value: B)
```

Phantom types are used to shift constraints to compile time.
A simple example involves units of measurement. 
Most of the world has standardized on SI units, such as metres and litres. 
However, other measuring systems, such as Imperial units, remain in use some countries or in some niches within countries that otherwise use metric.
Differences between different measurement systems can cause problems.
A dramatic example is the [loss of the Mars orbiter][mars], caused by two software components using incompatible measurements (one using metric, and one using US customary measurements.)

With phantom types we can annotate measurements with their units, which in turn can prevent us ever using incompatible units.
Let's work with just length, which is sufficient to show the idea.
We'll start by defining a length type with a phantom type recording the unit, 
and a method that allows us to add together lengths.

```scala mdoc:silent
final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
```

We'll need to define a few unit types to use this, and some `Lengths` using these units.

```scala mdoc:silent
trait Metres
trait Feet

val threeMetres = Length[Metres](3)
val threeFeetAndRising = Length[Feet](3)
```

Now we can add `Lengths` together if they have the same unit.

```scala mdoc
threeMetres + threeMetres
```

However if we try to add `Lengths` with different units the code will not compile.

```scala mdoc:fail
threeMetres + threeFeetAndRising
```

There is one big problem with phantom types on their own: 
there is no way to use the information stored in the phantom type
in further processing.
For example, force times length gives torque (with the SI unit of newton-metres).
However we cannot define a `*` method on `Length` that can only be called if the `Unit` is `Metre` using just the tool of phantom types.
Similarly, we cannot define, say, a `toString` method that uses the `Unit` type to appropriately print the result.
Solving these problems leads us to indexed codata, so let's now look at that.


[mars]: https://en.wikipedia.org/wiki/Mars_Climate_Orbiter#Cause_of_failure


```scala mdoc:reset:silent
```
--->

## ファントム型 {#sec:indexed-types:phantom}

ファントム型はインデックス付き型の基本的な構成要素である。まずはその例を示すことから始めよう。ファントム型とは単に、値と対応しない型パラメータのことを指す。以下の例では、型パラメータ `A` はその型の値が存在しないためファントム型である。一方、`B` は値の型として使われているのでファントム型ではない。

```scala mdoc:silent
final case class PhantomExample[A, B](value: B)
```

ファントム型は、制約をコンパイル時に移すために用いられる。

単位系の例を考えてみよう。ほとんどの国では、メートルやリットルといった SI 単位を標準として用いるが、いくつかの国や一部の分野では、依然としてヤード・ポンド法などの異なる単位系が使用されている。単位系の違いは時に問題を引き起こすことがある。劇的な例としては[火星探査機の損失][mars]が挙げられる。これは、ふたつのソフトウェアコンポーネントが異なる単位を使用していたことが原因だった（ひとつはメートル法、もうひとつは米国慣用単位を用いていた）。

ファントム型を使うことで、値に対して単位を注釈として追加でき、互換性のない単位の誤使用を防ぐことができる。ここでは、例として長さについてだけ考えてみよう。どのようなアイデアなのかを示すにはそれで十分である。まず、単位をファントム型として記録する長さの型を定義し、長さ同士を加算するためのメソッドを定義する。

```scala mdoc:silent
final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
```

次に、このデータ型を使用するのに必要な単位型をいくつか定義し、それらの単位を用いた `Length` インスタンスを作成する。

```scala mdoc:silent
trait Metres
trait Feet

val threeMetres = Length[Metres](3)
val threeFeetAndRising = Length[Feet](3)
```

`Length` 同士は、同じ単位をもっていれば加算できる。

```scala mdoc
threeMetres + threeMetres
```

だが、異なる単位をもつ `Length` 同士を加算しようとすると、コードはコンパイルされない。

```scala mdoc:fail
threeMetres + threeFeetAndRising
```

ファントム型自体には大きな問題がひとつある。ファントム型に格納された情報は、後の処理で利用することができない。たとえば、力と長さを掛け合わせるとトルク（ SI 単位ではニュートン・メートル）になるが、ファントム型だけでは、`Unit` が `Metre` の場合にのみ呼び出せる `*` メソッドを `Length` に定義することができない。同様に、`Unit` 型に応じた適切な結果を表示する `toString` メソッドを定義することもできない。これらの問題を解決するためにはインデックス付き余データが必要になる。次にそれを見ていこう。


[mars]: https://en.wikipedia.org/wiki/Mars_Climate_Orbiter#Cause_of_failure
