## Type Class Composition {#sec:type-classes:composition}

```scala mdoc:invisible:reset-object
// JSONのシンプルな抽象構文木を定義する
sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json
object Json {
  def toJson[A](value: A)(using w: JsonWriter[A]): Json =
    w.write(value)
}

// このトレイトが「JSONへのシリアライズ」機能を表す
trait JsonWriter[A] {
  def write(value: A): Json
}
```

So far we've seen type classes as a way to get the compiler to pass values to methods.
This is nice but it does seem like we've introduced a lot of new concepts for a small gain.
The real power of type classes lies in
the compiler's ability to combine given instances
to construct new given instances.
This is known as **type class composition**.

ここまで、コンパイラにメソッド呼び出しのコンテキストパラメータを供給させる手段としての型クラスを見てきた。これは便利だが、小さな利点のためにたくさんの新しい概念を導入しているようにも見える。型クラスの真の強みは、コンパイラが複数のgivenインスタンスを組み合わせて新しいgivenインスタンスを構築できる点にある。これは**型クラスの合成（type class composition）**として知られている。

Type class composition works by a feature of given instances we have not yet seen:
given instances can themselves have context parameters.
However, before we go into this
let's see a motivational example.

型クラスの合成は、まだ説明していないgivenインスタンスの機能を利用することで実現される。実は、givenインスタンスはそれ自体がコンテキストパラメータを持つことができる。しかし、その詳細に入る前に、この仕組みの必要性が分かる例を見てみよう。

Consider defining a `JsonWriter` for `Option`.
We would need a `JsonWriter[Option[A]]`
for every `A` we care about in our application.
We could try to brute force the problem by creating
a library of given instances:

`Option` 用の `JsonWriter` を定義することを考える。アプリケーションで必要となるすべての型 `A` に対して、`JsonWriter[Option[A]]` を用意する必要がある。givenインスタンスを集めたライブラリを作成するなど、力ずくでの問題解決を試みることもできるだろう。

```scala
given optionIntWriter: JsonWriter[Option[Int]] =
  ???

given optionPersonWriter: JsonWriter[Option[Person]] =
  ???

// などなど……
```

However, this approach clearly doesn't scale.
We end up requiring two given instances
for every type `A` in our application:
one for `A` and one for `Option[A]`.

だが、このアプローチがスケールしないのは明らかである。アプリケーション内の一つひとつの型について、その型を `A` とすれば、`A` 用のgivenインスタンスと `Option[A]` 用のgivenインスタンスの二つが必要になってしまう。

Fortunately, we can abstract the code for handling `Option[A]`
into a common constructor based on the instance for `A`:

幸いなことに、`Option[A]` をJSONシリアライズするコードは、`A` 用の型クラスインスタンスに基づいた共通のコンストラクタとして抽象化することができる。

- if the option is `Some(aValue)`,
  write `aValue` using the writer for `A`;

- if the option is `None`, return `JsNull`.

- `Some(aValue)` に対しては、型 `A` 用の `JsonWriter` を使って `aValue` を書き出す
- `None` に対しては、`JsNull` を返す

Here is the same code written out using a parameterized given instance:

次に示すのは、上記のアイデアをパラメータ化されたgivenインスタンスを用いてコード化したものである。

```scala mdoc:silent
given optionWriter[A](using writer: JsonWriter[A]): JsonWriter[Option[A]] =
  new JsonWriter[Option[A]] {
    def write(option: Option[A]): Json =
      option match {
        case Some(aValue) => writer.write(aValue)
        case None         => JsNull
      }
  }
```

This method constructs a `JsonWriter` for `Option[A]` by
relying on a context parameter to
fill in the `A`-specific functionality.
When the compiler sees an expression like this:

このメソッドは、型 `A` に固有のシリアライズ化ロジックをコンテキストパラメータとして受け取ることによって、`Option[A]` 用の `JsonWriter` を構築する。

```scala mdoc:invisible
given stringWriter: JsonWriter[String] =
  new JsonWriter[String] {
    def write(value: String): Json = JsString(value)
  }
```
```scala mdoc:silent
Json.toJson(Option("A string"))
```

it searches for an given instance `JsonWriter[Option[String]]`.
It finds the given instance for `JsonWriter[Option[A]]`:

上記のような式を見つけると、コンパイラは `JsonWriter[Option[String]]` 型のgivenインスタンスを探し、発見したインスタンスをコンテキストパラメータとしてメソッドに渡す。

```scala mdoc:silent
Json.toJson(Option("A string"))(using optionWriter[String])
```

続いて、`optionWriter` のコンテキストパラメータとして利用するため、再帰的に `JsonWriter[String]` を探す。

```scala mdoc:silent
Json.toJson(Option("A string"))(using optionWriter(using stringWriter))
```

In this way, given instance resolution becomes
a search through the space of possible combinations
of given instance, to find
a combination that creates a type class instance
of the correct overall type.

このようにして、givenインスタンスの解決は、正しい型クラスインスタンスを生成するために、可能な組み合わせの空間を検索するプロセスとなる。

このように、givenインスタンスの解決は、利用可能なgivenインスタンスの組み合わせ空間を探索し、目指す型のための型クラスインスタンスを作成する正しい組み合わせを発見するプロセスとなる。

### Type Class Composition in Scala 2

In Scala 2 we can achieve the same effect with an `implicit` method with `implicit` parameters.
Here's the Scala 2 equivalent of `optionWriter` above.

Scala2では、`implicit` メソッドと `implicit` パラメータを使うことで同じ効果を得ることができる。次に、上述の `optionWriter` と同等のものをScala2で記述したコードを示す。

```scala mdoc:invisible:reset-object
// JSONのシンプルな抽象構文木を定義する
sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json

// このトレイトが「JSONへのシリアライズ」機能を表す
trait JsonWriter[A] {
  def write(value: A): Json
}
```
```scala mdoc:silent
implicit def scala2OptionWriter[A]
    (implicit writer: JsonWriter[A]): JsonWriter[Option[A]] =
  new JsonWriter[Option[A]] {
    def write(option: Option[A]): Json =
      option match {
        case Some(aValue) => writer.write(aValue)
        case None         => JsNull
      }
  }
```

Make sure you make the method's parameter implicit!
If you don't, you'll end up defining an **implicit conversion**.
Implicit conversion is an older programming pattern
that is frowned upon in modern Scala code.
Fortunately, the compiler will warn you should you do this.

メソッドのパラメータをimplicitにすることを忘れてはならない。そうしないと、**暗黙の型変換（implicit conversion）**を定義したことになってしまう。暗黙の型変換は古いプログラミングパターンであり、最近のScalaコードでは推奨されない。幸いなことに、もし間違ってしまっても、コンパイラはメソッドパラメータに `implicit` をつけるよう警告してくれる。
