# モノイドと半群 Monoids and Semigroups {#sec:monoids}

In this section we explore our first type classes, **monoid** and **semigroup**.
These allow us to add or combine values.
There are instances for `Ints`, `Strings`, `Lists`, `Options`, and many more.
Let's start by looking at a few simple types and operations
to see what common principles we can extract.

この節では、**モノイド（monoid）**および**半群（semigroup）**という型クラスについて見ていく。これらの型クラスは、値同士を加算したり結合したりするために用いられる。`Int`、`String`、`List`、`Option` など、さまざまな型にインスタンスが用意されている。まずは、いくつかの簡単な型と操作について確認し、そこからどのような共通原則を抽出できるか見てみよう。

<!-- 
TODO: Cats導入の章が加筆されて、先にShowとEqが紹介されており、「最初の」ではなくなってしまった件
-->

#### 整数の加算 Integer addition

Addition of `Ints` is a binary operation that is *closed*,
meaning that adding two `Ints` always produces another `Int`:

`Int` の加算は、*閉じた*二項演算である。閉じているとは、二つの `Int` を足した結果も常に `Int` になるということを意味する。

```scala mdoc
2 + 1
```

There is also the *identity* element `0` with the property
that `a + 0 == 0 + a == a` for any `Int` `a`:

また、`0` という*単位元（identity element）*が存在する。単位元である `0` は、任意の `Int` 値 `a` に対して `a + 0 == 0 + a == a` となる性質をもっている。

```scala mdoc
2 + 0

0 + 2
```

There are also other properties of addition.
For instance, it doesn't matter in what order we add elements
because we always get the same result.
This is a property known as *associativity*:

加算には他の性質もある。たとえば、どのような順序で値を足し合わせても、常に同じ結果が得られる。この性質は*結合律（associativity）*として知られる。

```scala mdoc
(1 + 2) + 3

1 + (2 + 3)
```


#### 整数の乗算 Integer multiplication

The same properties for addition also apply for multiplication,
provided we use `1` as the identity instead of `0`:

単位元として `0` ではなく `1` を選べば、乗算にも加算と同じ性質が当てはまる。

```scala mdoc
1 * 3

3 * 1
```

Multiplication, like addition, is associative:

乗算も加算と同様に結合律を満たす。

```scala mdoc
(1 * 2) * 3

1 * (2 * 3)
```


#### String and sequence concatenation

We can also add `Strings`,
using string concatenation as our binary operator:

二項演算として文字列の連結を考えれば、`String` 同士も演算可能である。

```scala mdoc
"One" ++ "two"
```

and the empty string as the identity:

空文字が単位元として使える。

```scala mdoc
"" ++ "Hello"

"Hello" ++ ""
```

Once again, concatenation is associative:

文字列の連結もまた結合律を満たす。

```scala mdoc
("One" ++ "Two") ++ "Three"

"One" ++ ("Two" ++ "Three")
```

Note that we used `++` above instead of the more usual `+`
to suggest a parallel with sequences.
We can do the same with other types of sequence,
using concatenation as the binary operator
and the empty sequence as our identity.

ここでは、シーケンス（順序付きコレクション、つまり `Seq`）との類似性を示すために、通常の `+` ではなく `++` を使っている。連結処理を二項演算とし、空のシーケンスを単位元とすれば、他のシーケンス型についても `String` と同じことが言える。

<!--
TODO: シーケンス・順序付きコレクション・Seq。カッコじゃなく訳註にするか？
-->

## モノイドの定義 Definition of a Monoid

We've seen a number of "addition" scenarios above
each with an associative binary addition
and an identity element.
It will be no surprise to learn that this is a monoid.
Formally, a monoid for a type `A` is:

ここまで、結合律を満たす二項演算と単位元をもった加法をいくつか見てきた。そういうものがモノイドであると言われても驚きはないだろう。形式的には、型 `A` についてのモノイドは以下の二つから構成される。

- `(A, A) => A` という型をもつ操作 `combine`
- `A` 型の要素 `empty`

<!-- TODO: "addition" senarios の訳 -->

This definition translates nicely into Scala code.
Here is a simplified version of the definition from Cats:

この定義はScalaコードにうまく翻訳できる。以下は、Catsにおける定義を簡略化したものである。

```scala mdoc:silent
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

In addition to providing the `combine` and `empty` operations,
monoids must formally obey several *laws*.
For all values `x`, `y`, and `z`, in `A`,
`combine` must be associative and
`empty` must be an identity element:

モノイドは、`combine` と `empty` という操作を提供するだけでなく、いくつかの*法則*を満たす必要がある。`A` 型の任意の値 `x`, `y`, `z` について、`combine` は結合律を満たし、`empty` は単位元でなければならない。

```scala mdoc:silent
def associativeLaw[A](x: A, y: A, z: A)
      (using m: Monoid[A]): Boolean = {
  m.combine(x, m.combine(y, z)) ==
    m.combine(m.combine(x, y), z)
}

def identityLaw[A](x: A)
      (using m: Monoid[A]): Boolean = {
  (m.combine(x, m.empty) == x) &&
    (m.combine(m.empty, x) == x)
}
```

Integer subtraction, for example,
is not a monoid because subtraction is not associative:

たとえば、整数の減算は結合律を満たさないので、モノイドではない。

```scala mdoc
(1 - 2) - 3

1 - (2 - 3)
```

In practice we only need to think about laws
when we are writing our own `Monoid` instances.
Unlawful instances are dangerous because
they can yield unpredictable results
when used with the rest of Cats' machinery.
Most of the time we can rely on the instances provided by Cats
and assume the library authors know what they're doing.

実際には、`Monoid` インスタンスを自作する場合にだけ、法則について考慮すればよい。法則を満たさないインスタンスは危険である。Catsの他の機能と組み合わせたときに予測不能な結果を生じる可能性がある。大抵の場合、Catsが提供するインスタンスは信頼できるし、ライブラリの作者は適切にそれらを実装してくれていると考えてよい。

## 半群の定義 Definition of a Semigroup

A semigroup is just the `combine` part of a monoid,
without the `empty` part.
While many semigroups are also monoids,
there are some data types for which we cannot define an `empty` element.
For example, we have just seen that
sequence concatenation and integer addition are monoids.
However, if we restrict ourselves
to non-empty sequences and positive integers,
we are no longer able to define a sensible `empty` element.
Cats has a [`NonEmptyList`][cats.data.NonEmptyList] data type
that has an implementation of `Semigroup` but no implementation of `Monoid`.

半群は、モノイドから `empty` 部を除いた、単なる `combine` 部分のことである。多くの半群はモノイドでもあるが、`empty` を定義できないデータ型も存在する。たとえば、シーケンスの結合や整数の加算がモノイドであることは見てきたが、非空のシーケンスや正の整数に限定すると、妥当な `empty` を定義することはできなくなる。Catsには、[`NonEmptyList`][cats.data.NonEmptyList] というデータ型がある。この型には `Semigroup` の実装はあるが、`Monoid` の実装はない。

A more accurate (though still simplified)
definition of Cats' [`Monoid`][cats.Monoid] is:

Catsにおける[`Monoid`][cats.Monoid]のより正確な定義は以下のとおりである。ただし、これもやはりまだ簡略化されている。

```scala mdoc:silent:reset-object
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```

We'll see this kind of inheritance often when discussing type classes.
It provides modularity and allows us to re-use behaviour.
If we define a `Monoid` for a type `A`, we get a `Semigroup` for free.
Similarly, if a method requires a parameter of type `Semigroup[B]`,
we can pass a `Monoid[B]` instead.

型クラスについて議論する際には、このような継承の形をよく目にするだろう。この設計により、モジュール性が生まれ、振る舞いを再利用できるようになる。型 `A` に対して `Monoid` を定義すれば、自動的に `Semigroup` が得られる。同様に、`Semigroup[B]` 型のパラメータを要求するメソッドに対して `Monoid[B]` を代わりに渡すこともできる。

#### 演習: モノイドの真実 The Truth About Monoids 

We've seen a few examples of monoids but there are plenty more to be found.
Consider `Boolean`. How many monoids can you define for this type?
For each monoid, define the `combine` and `empty` operations
and convince yourself that the monoid laws hold.
Use the following definitions as a starting point:

モノイドの例をいくつか見てきたが、まだモノイドとして認識されていないものが他にも数多く存在する。

`Boolean` 型について、いくつのモノイドを定義できるか考察せよ。また、それらの各モノイドに `combine` と `empty` を定義し、モノイドの法則が成り立つことを確認せよ。なお、出発点として以下の定義を利用すること。

```scala mdoc:reset:silent
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

trait Monoid[A] extends Semigroup[A] {
  def empty: A
}

object Monoid {
  def apply[A](implicit monoid: Monoid[A]) =
    monoid
}
```

<div class="solution">
There are at least four monoids for `Boolean`!
First, we have *and* with operator `&&` and identity `true`:

`Boolean` にはすくなくとも4つのモノイドがある。まず、`&&` 演算子で表される*論理積*という二項演算と、単位元 `true` がある。

```scala mdoc:silent
given booleanAndMonoid: Monoid[Boolean] with {
  def combine(a: Boolean, b: Boolean) = a && b
  def empty = true
}
```

Second, we have *or* with operator `||` and identity `false`:

二つ目に、`||` 演算子で表される*論理和*と単位元 `false`。

```scala mdoc:silent
given booleanOrMonoid: Monoid[Boolean] with {
  def combine(a: Boolean, b: Boolean) = a || b
  def empty = false
}
```

Third, we have *exclusive or* with identity `false`:

三つ目に、*排他的論理和*と単位元 `false`。

```scala mdoc:silent
given booleanEitherMonoid: Monoid[Boolean] with {
  def combine(a: Boolean, b: Boolean) =
    (a && !b) || (!a && b)

  def empty = false
}
```

Finally, we have *exclusive nor* (the negation of exclusive or)
with identity `true`:

最後は、*否定排他的論理和*（排他的論理和の否定）と単位元 `true` である。

```scala mdoc:silent
given booleanXnorMonoid: Monoid[Boolean] with {
  def combine(a: Boolean, b: Boolean) =
    (!a || b) && (a || !b)

  def empty = true
}
```

Showing that the identity law holds in each case is straightforward.
Similarly associativity of the `combine` operation
can be shown by enumerating the cases.

それぞれのケースについて単位元の法則が成り立つことは簡単に示せる。同様に、`combine` が結合律を満たすことについても、全パターンを列挙して示すことができる。
</div>


#### 演習: All Set for Monoids

What monoids and semigroups are there for sets?

集合に対してどのようなモノイドと半群があるか、考察せよ。

<div class="solution">
*Set union* forms a monoid along with the empty set:

集合同士の*和集合*をとる演算は、空集合を単位元としてモノイドを形成する。

```scala mdoc:silent
given setUnionMonoid[A]: Monoid[Set[A]] with {
  def combine(a: Set[A], b: Set[A]) = a.union(b)
  def empty = Set.empty[A]
}
```

We need to define `setUnionMonoid` as a method
rather than a value so we can accept the type parameter `A`.
The type parameter allows us to use the same definition
to summon `Monoids` for `Sets` of any type of data:

`setUnionMonoid` は、型パラメータ `A` を受け取れるように、値ではなくメソッドとして定義する必要がある。この型パラメータによって、定義はひとつでも、任意のデータ型の `Set` に対してモノイドを呼び出せるようになる。

```scala mdoc:silent
val intSetMonoid = Monoid[Set[Int]]
val strSetMonoid = Monoid[Set[String]]
```

```scala mdoc
intSetMonoid.combine(Set(1, 2), Set(2, 3))
strSetMonoid.combine(Set("A", "B"), Set("B", "C"))
```

Set intersection forms a semigroup,
but doesn't form a monoid because it has no identity element:

積集合をとる演算は半群を形成するが、単位元がないためモノイドにはならない。

```scala mdoc:silent
given setIntersectionSemigroup[A]: Semigroup[Set[A]] with {
  def combine(a: Set[A], b: Set[A]) =
    a.intersect(b)
}
```

Set complement and set difference are not associative,
so they cannot be considered for either monoids or semigroups.
However, symmetric difference (the union less the intersection)
does form a monoid with the empty set:

補集合と差集合は結合律を満たさないため、モノイドや半群にはなり得ない。しかし、対称差（和集合から積集合を引いたもの）をとる演算は、空集合を単位元としてモノイドを形成する。

```scala mdoc:invisible:reset-object
import cats.Monoid
```
```scala mdoc:silent
given symDiffMonoid[A]: Monoid[Set[A]] with {
  def combine(a: Set[A], b: Set[A]): Set[A] =
    (a.diff(b)).union(b.diff(a))

  def empty: Set[A] = Set.empty
}
```
</div>
