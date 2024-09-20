## 例題: Eq

We will finish off this chapter by looking at another useful type class:
[`cats.Eq`][cats.kernel.Eq].
`Eq` is designed to support *type-safe equality*
and address annoyances using Scala's built-in `==` operator.

この章の最後に、もうひとつの便利な型クラスである[`cats.Eq`][cats.kernel.Eq]を見ていく。`Eq` は*型安全な等価性*をサポートし、Scala組み込みの `==` 演算子の使いづらさを解消するように設計されている。

Almost every Scala developer has written code like this before:

ほとんどのScala開発者は、以下のようなコードを書いたことがあるだろう。


```scala
List(1, 2, 3).map(Option(_)).filter(item => item == 1)
// warning: Option[Int] and Int are unrelated: they will most likely never compare equal
// res: List[Option[Int]] = List()

```

Ok, many of you won't have made such a simple mistake as this,
but the principle is sound.
The predicate in the `filter` clause always returns `false`
because it is comparing an `Int` to an `Option[Int]`.

もちろん、このような単純なミスを犯すことは少ないだろうが、ここで言いたいことの要点は伝わるだろう。`filter` 句の述語は、`Int` と `Option[Int]` を比較しているため、常に `false` を返す。

This is programmer error---we
should have compared `item` to `Some(1)` instead of `1`.
However, it's not technically a type error because
`==` works for any pair of objects, no matter what types we compare.
`Eq` is designed to add some type safety to equality checks
and work around this problem.

これはプログラマのミスで、`item` は `1` ではなく `Some(1)` と比較されるべきだった。しかし、 `==` はどのような型のオブジェクト同士にも使えるので、形式的には型エラーではない。`Eq` は、型安全な等価性チェックを提供し、この問題を解決する。

### 平等、自由、友愛 Equality, Liberty, and Fraternity

We can use `Eq` to define type-safe equality
between instances of any given type:

`Eq` を使えば、任意の型のインスタンス間で型安全な等価性を定義できる。

```scala
package cats

trait Eq[A] {
  def eqv(a: A, b: A): Boolean
  // other concrete methods based on eqv...
  // ...eqvに基づくその他の具象メソッド
}
```

The interface syntax, defined in [`cats.syntax.eq`][cats.syntax.eq],
provides two methods for performing equality checks
provided there is an instance `Eq[A]` in scope:

 - `===` compares two objects for equality;
 - `=!=` compares two objects for inequality.

[`cats.syntax.eq`][cats.syntax.eq]で定義されているインターフェース構文では、スコープ内に `Eq[A]` インスタンスが存在する場合、等価性チェックを行うための二つのメソッドが提供される。

- `===` は二つのオブジェクトが等しいかどうかを比較する。
- `=!=` は二つのオブジェクトが等しくないかどうかを比較する。

### `Int` の比較 Comparing Ints

Let's look at a few examples. First we import the type class:

いくつか例を見てみよう。とりあえず型クラスをインポートしておく。

```scala mdoc:silent:reset-object
import cats.*
```

Now let's grab an instance for `Int`:

次に `Int` 用のインスタンスを手に入れよう。

```scala mdoc:silent
val eqInt = Eq[Int]
```

We can use `eqInt` directly to test for equality:

`eqInt` 直接用いて等価性を調べることができる。

```scala mdoc
eqInt.eqv(123, 123)
eqInt.eqv(123, 234)
```

Unlike Scala's `==` method,
if we try to compare objects of different types using `eqv`
we get a compile error:

Scalaの `==` メソッドと異なり、異なる型のオブジェクトを `eqv` で比較しようとするとコンパイルエラーと成る。」

```scala mdoc:fail
eqInt.eqv(123, "234")
```

We can also import the interface syntax in [`cats.syntax.eq`][cats.syntax.eq]
to use the `===` and `=!=` methods:

[`cats.syntax.eq`][cats.syntax.eq]からインターフェース構文をインポートし `===` および `=!=` を使うこともできる。

```scala mdoc:silent
import cats.syntax.all.* // === と =!= をインポートする
```

```scala mdoc
123 === 123
123 =!= 234
```

Again, comparing values of different types causes a compiler error:

こちらも、異なる型をもつ値を比較するとコンパイルエラーになる。

```scala mdoc:fail
123 === "123"
```

### `Option` の比較 {#sec:type-classes:comparing-options}

Now for a more interesting example---`Option[Int]`.

ここで少し興味深い例を見てみよう。`Option[Int]` についてである。

```scala mdoc:fail
Some(1) === None
```

We have received an error here because the types don't quite match up.
We have `Eq` instances in scope for `Int` and `Option[Int]`
but the values we are comparing are of type `Some[Int]`.
To fix the issue we have to re-type the arguments as `Option[Int]`:

このコードは、型が完全には一致していないため、エラーとなる。`Int` と `Option[Int]` の `Eq` インスタンスはスコープ内にあるが、比較している値は `Some[Int]` 型である。この問題を解決するには、引数を `Option[Int]` 型に明示的にキャストする必要がある。

```scala mdoc
(Some(1) : Option[Int]) === (None : Option[Int])
```

We can do this in a friendlier fashion using
the `Option.apply` and `Option.empty` methods from the standard library:

標準ライブラリの `Option.apply` と `Option.empty` メソッドを使えば、もっとわかりやすく書ける。

```scala mdoc
Option(1) === Option.empty[Int]
```

or using special syntax from [`cats.syntax.option`][cats.syntax.option]:

あるいは、[`cats.syntax.option`][cats.syntax.option]の特別なシンタックスを使って、以下のように書くこともできる。

```scala mdoc
1.some === none[Int]
1.some =!= none[Int]
```


### 独自型の比較 Comparing Custom Types

We can define our own instances of `Eq` using the `Eq.instance` method,
which accepts a function of type `(A, A) => Boolean` and returns an `Eq[A]`:

`Eq.instance` メソッドを使って、独自の `Eq` インスタンスを定義することができる。`Eq.instance` は `(A, A) => Boolean` 型の関数を受け取って `Eq[A]` を返す。

```scala mdoc:silent
import java.util.Date

given dateEq: Eq[Date] =
  Eq.instance[Date] { (date1, date2) =>
    date1.getTime === date2.getTime
  }
```

```scala mdoc:silent
val x = new Date() // 現在日時
val y = new Date() // 現在より一瞬後の日時
```

```scala mdoc
x === x
x === y
```


#### 演習: 平等、自由、友愛

Implement an instance of `Eq` for our running `Cat` example:

`Cat` の例に対して `Eq` インスタンスを実装せよ。

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

Use this to compare the following pairs of objects for equality and inequality:

また、それを用いて、以下のオブジェクトのペアについて等価性と非等価性を確認せよ。

```scala mdoc:silent
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]
```

<div class="solution">
First we need our Cats imports.
In this exercise we'll be using the `Eq` type class
and the `Eq` interface syntax,
so we start by importing that.

まずはCatsのインポートを行う。この演習では `Eq` 型クラスと `Eq` のインターフェース構文を使用するので、それらをインポートから始める。

```scala mdoc:silent:reset-object
import cats.*
import cats.syntax.all.* 
```

Our `Cat` class is the same as ever:

`Cat` クラスはこれまでどおりである。

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

We bring the `Eq` instances for `Int` and `String`
into scope for the implementation of `Eq[Cat]`:

`Eq[Cat]` の実装に必要となる `Int`と `String` の `Eq` インスタンスをスコープに入れる。

```scala mdoc:silent
given catEqual: Eq[Cat] =
  Eq.instance[Cat] { (cat1, cat2) =>
    (cat1.name  === cat2.name ) &&
    (cat1.age   === cat2.age  ) &&
    (cat1.color === cat2.color)
  }
```

Finally, we test things out in a sample application:

サンプルアプリケーションでテストすれば完了である。

```scala mdoc
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 32, "orange and black")

cat1 === cat2
cat1 =!= cat2

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]

optionCat1 === optionCat2
optionCat1 =!= optionCat2
```
</div>
