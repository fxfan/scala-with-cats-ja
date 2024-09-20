# Using Cats {#sec:cats}

In this Chapter we'll learn how to use the [Cats](https://typelevel.org/cats) library.
Cats provides two main things: type classes and their instances, and some useful data structures.
Our focus will mostly be on the type classes, though we will touch on the data structures where appropriate.

この章では[Cats](https://typelevel.org/cats)ライブラリの使い方について学ぶ。Catsが提供するものは主に二つある。ひとつは型クラスとそのインスタンス、もうひとつはいくつかの便利なデータ構造である。本書では主に型クラスにフォーカスを当てるが、適宜データ構造についても触れる。

## クイックスタート Quick Start

The easiest, and recommended, way to use Cats is to add the following imports:

Catsを使うもっとも簡単で推奨される方法は、以下のインポートを追加することである。


```scala mdoc:silent
import cats.*
import cats.syntax.all.*
```

The first import adds all the type classes 
(and makes their instances available, as they are found in the companion objects.)
The second import adds the syntax helpers,
which makes the type classes easier to work with.
Note we don't need to `import cats.{*, given}` as, at the time of writing, Cats is written in Scala 2 style (using `implicits`) and these are imported by the wildcard import.

最初のインポートは、すべての型クラスを追加する。インスタンスはコンパニオンオブジェクトに定義されており、これで利用可能となる。二つ目のインポートはシンタックスヘルパーを追加する。これにより型クラスがより扱いやすくなる。`import cats.{*, given}` のようなインポートは必要ない。執筆時点では、CatsはScala2スタイル、つまり `implicit` を使って書かれており、それらはワイルドカードによるインポートで取り込まれるからである。

If we want use some of Cats' datastructures, we also need to add

Catsが提供するデータ構造を使用したい場合は、以下も追加する必要がある。

```scala mdoc:silent
import cats.data.*
```


## Catsを使う Using Cats

Let's now see how we work with Cats, 
using [`cats.Show`][cats.Show] as an example.

では、[`cats.Show`][cats.Show]を例に、Catsの使い方を見ていこう。

`Show` is Cats' equivalent of
the `Display` type class we defined in Section [@sec:type-classes:display].
It provides a mechanism for producing
developer-friendly console output without using `toString`.
Here's an abbreviated definition:

`Show` は、[@sec:type-classes]の節で定義した `Display` 型クラスに相当するCatsの型クラスであり、`toString` を使わずに開発者向けのコンソール出力を生成する仕組みを提供する。以下はその定義を簡略化したものである。

```scala
package cats

trait Show[A] {
  def show(value: A): String
}
```

The easiest way to use `Show` is with the wildcard import above.
However, we can also import `Show` directly from the [cats][cats.package] package:

`Show` を使うもっとも簡単な方法は、前述のワイルドカードインポートを利用することだが、`Show` を直接[cats][cats.package]パッケージからインポートすることもできる。

```scala mdoc:silent
import cats.Show
```

The companion object of every Cats type class has an `apply` method
that locates an instance for any type we specify:

Catsの各型クラスのコンパニオンオブジェクトには、指定した型のインスタンスを見つける `apply` メソッドがある。

```scala mdoc:silent
val showInt = Show.apply[Int]
```

Once we have an instance we can call methods on it.

インスタンスを手に入れたら、そのメソッドを呼び出せばよい。

```scala mdoc
showInt.show(42)
```

More common, however, is to use the syntax or extension methods,
which we imported with `import cats.syntax.all.*`.
In the case of `Show`, an extension method `show` is defined.

ただし、`import cats.syntax.all.*` でインポートしたシンタックスや拡張メソッドを使う方が一般的である。`Show` に対しては、拡張メソッド `show` が定義されている。

```scala mdoc
42.show
```

If, for some reason, we wanted just the syntax for `show`,
we could import [`cats.syntax.show`][cats.syntax.show].

何らかの理由で `show` についてのシンタックスだけが必要な場合、[`cats.syntax.show`][cats.syntax.show] をインポートすることもできる。

```scala mdoc:silent
import cats.syntax.show.*
```


### 独自インスタンスの定義 Defining Custom Instances {#defining-custom-instances}

We can define an instance of `Show`
simply by implementing the trait for a given type:

`Show` のインスタンスを定義するには、対象となる型に対してトレイトを実装するだけでよい。

```scala mdoc:silent
import java.util.Date

given dateShow: Show[Date] with 
  def show(date: Date): String =
    s"${date.getTime}ms since the epoch."
```
```scala mdoc
new Date().show
```

However, Cats also provides
a couple of convenient methods to simplify the process.
There are two construction methods on the companion object of `Show`
that we can use to define instances for our own types:

しかし、Catsにはこのプロセスを簡略化するための便利なメソッドもいくつか用意されている。`Show` のコンパニオンオブジェクトには、独自の型に対してインスタンスを定義するための二つの構築メソッドがある。

```scala
object Show {
  // Convert a function to a `Show` instance:
  // 関数を `Show` インスタンスに変換する
  def show[A](f: A => String): Show[A] =
    ???

  // Create a `Show` instance from a `toString` method:
  // `toString` メソッドから `Show` インスタンスを作成する
  def fromToString[A]: Show[A] =
    ???
}
```

These allow us to quickly construct instances
with less ceremony than defining them from scratch:

これらのメソッドを使えば、一から定義するよりも手間をかけずにインスタンスを構築できる。

```scala mdoc:reset:invisible
import cats.Show
import java.util.Date
```
```scala mdoc:silent
given dateShow: Show[Date] =
  Show.show(date => s"${date.getTime}ms since the epoch.")
```

As you can see, the code using construction methods
is much terser than the code without.
Many type classes in Cats provide helper methods like these
for constructing instances, either from scratch
or by transforming existing instances for other types.

見てのとおり、構築メソッドを用いたコードは、そうでないコードよりもかなり簡潔である。Catsの多くの型クラスはこのようなヘルパーメソッドを提供しており、一からもしくは他の型用の既存インスタンスを変換することによるインスタンス構築をサポートしてくれる。

#### 演習: Exercise: Cat Show

Re-implement the `Cat` application from Section [@sec:type-classes:cat]
using `Show` instead of `Display`.

[@sec:type-classes:cat]節の `Cat` アプリケーションを、`Display` の代わりに `Show` を用いて再実装せよ。

<!-- ここから下はコピペの消し忘れ？ -->

Using this data type to represent a well-known type of furry animal:

```scala
final case class Cat(name: String, age: Int, color: String)
```

create an implementation of `Display` for `Cat`
that returns content in the following format:

```ruby
NAME is a AGE year-old COLOR cat.
```

Then use the type class on the console or in a short demo app:
create a `Cat` and print it to the console:

```scala
// Define a cat:
val cat = Cat(/* ... */)

// Print the cat!
```


<div class="solution">
First let's import everything we need from Cats.

まずは必要なものをCatsからインポートしよう。

```scala mdoc:reset-object:silent
import cats.*
import cats.syntax.all.*
```

Our definition of `Cat` remains the same:

`Cat` の定義は次のとおり元のままでよい。

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

In the companion object we replace our `Display` instance with an instance of `Show`
using one of the definition helpers discussed above:

コンパニオンオブジェクトでは、先述のヘルパーメソッドのひとつを使って `Display` インスタンスを `Show` インスタンスに置き換える。

```scala mdoc:silent
given catShow: Show[Cat] = Show.show[Cat] { cat =>
  val name  = cat.name.show
  val age   = cat.age.show
  val color = cat.color.show
  s"$name is a $age year-old $color cat."
}
```

Finally, we use the `Show` interface syntax to print our instance of `Cat`:

最後に、`Show` のインターフェース構文を用いて `Cat` インスタンスを出力する。

```scala mdoc
println(Cat("Garfield", 38, "ginger and black").show)
```
</div>

