## 型クラスの構造

Let's now look at how type classes are implemented.
There are three important components to a type class:
the type class itself, which defines an interface,
type class instances, which implement the type class for particular types,
and the methods that use type classes.
The table below shows the language features that correspond to each component.

ここからは、型クラスがどのように実装されるかを見ていこう。型クラスには三つの重要な構成要素がある。インターフェースを定義する型クラスそのもの、特定の型に対して型クラスを実装する型クラスインスタンス、そして型クラスを使用するメソッドである。以下の表は、それぞれの構成要素に対応する言語機能を示している。

+---------------------+------------------+
| 型クラスの概念         | 言語機能          |
+=====================+==================+
| 型クラス              | trait            |
+---------------------+------------------+
| 型クラスインスタンス    | given instance   |
+---------------------+------------------+
| 型クラスの使用         | using clause     |
+---------------------+------------------+

Let's see how this works in detail.

これがどのように機能するか詳しく見ていこう。

### 型クラス

A type class is an interface or API
that represents some functionality we want implemented.
In Scala a type class is represented by a trait with at least one type parameter.
For example, we can represent generic "serialize to JSON" behaviour
as follows:

型クラスは、実装したい何らかの機能を表現するインターフェースやAPIである。Scalaにおいて、型クラスは少なくともひとつの型パラメータをもつトレイトとして表現される。たとえば、汎用的な「JSONへのシリアライズ」機能を次のように表現できる。

```scala mdoc:silent:reset-object
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

`JsonWriter` is our type class in this example,
with `Json` and its subtypes providing supporting code.
When we come to implement instances of `JsonWriter`,
the type parameter `A` will be the concrete type of data we are writing.

この例では、`JsonWriter` が型クラスであり、`Json` とそのサブタイプは、型クラスの使い方を見ていくために補助的に必要となるコードを提供している。`JsonWriter` のインスタンスを実装するにあたっては、型パラメータ `A` がシリアライズしたいデータの具体的な型となる。

### 型クラスインスタンス

The instances of a type class
provide implementations of the type class for specific types we care about,
which can include types from the Scala standard library
and types from our domain model.

型クラスのインスタンスは、関心のある特定の型に対して型クラスの実装を提供する。ここでいう型には、Scala標準ライブラリの型と独自のドメインモデルで定義された型の両方が含まれる。

In Scala we create type class instances by defining
given instances implementing the type class.

Scalaでは、型クラスを実装したgivenインスタンスを定義することによって、型クラスインスタンスを作成する。

```scala mdoc:silent
object JsonWriterInstances {
  given stringWriter: JsonWriter[String] =
    new JsonWriter[String] {
      def write(value: String): Json =
        JsString(value)
    }
  
  final case class Person(name: String, email: String)
  
  given JsonWriter[Person] with
    def write(value: Person): Json =
      JsObject(Map(
        "name" -> JsString(value.name),
        "email" -> JsString(value.email)
      ))
  
  // etc...
}
```

In this example we define two type class instances of `JsonWriter`, one for `String` and one for `Person`.
The definition for `String` uses the syntax we saw in the previous section.
The definition for `Person` uses two bits of syntax that are new in Scala 3.
Firstly, writing `given JsonWriter[Person]` creates an anonymous given instance. 
We declare just the type and don't need to name the instance.
This is fine because we don't usually need to refer to given instances by name.
The second bit of syntax is the use of `with` to implement a trait directly without having to 
write out `new JsonWriter[Person]` and so on.

この例では、`JsonWriter` の型クラスインスタンスを二つ定義している。ひとつは `String` 用で、もうひとつは `Person` 用である。`String` に対する定義では、前節で見たとおりの構文を使用し、`Person` に対する定義では、Scala3で新たに導入された二つの構文を使用している。まず、`given JsonWriter[Person]` と書くことで、無名のgivenインスタンスが作成される。型だけを宣言すればよく、インスタンスに名前を付ける必要はない。通常、givenインスタンスを名前で参照する必要はないので、これで問題ない。二つ目の構文は、`new JsonWriter[Person]` などと書き出す代わりに `with` を使用してトレイトを直接実装していることである。

In a real implementation we'd usually want to define the instances on a companion object: the instance for `String` on the `JsonWriter` companion object (because we cannot define it on the `String` companion object) and the instance for `Person` on the `Person` companion object. 
I haven't done this here because I would need to redeclare `JsonWriter`, as a type and it's companion object must be declared at the same time.

実際の実装では、コンパニオンオブジェクト上にインスタンスを定義することが一般的に望ましい。たとえば、`String` 用のインスタンスは `JsonWriter` のコンパニオンオブジェクトに（`String` のコンパニオンオブジェクトには定義できないため）、`Person` 用のインスタンスは `Person` のコンパニオンオブジェクトに定義するのである。しかし、ここではそれを行っていない。型とそのコンパニオンオブジェクトは同時に宣言する必要があり、`JsonWriter` を再定義しなければならなくなるからである。

<!--
TODO: 「しかし、ここではそれを行っていない」以降の文章が理解できない
-->

### 型クラスの利用 Type Class Use

A type class use is any functionality 
that requires a type class instance to work.
In Scala this means any method 
that accepts instances of the type class as part of a using clause.

型クラスインスタンスを必要とする機能はすべて、型クラスを利用していると言うことができる。Scalaにおいては、using句の一部として型クラスのインスタンスを受け取るメソッドがこれに該当する。

We're going to look at two patterns of type class usage, 
which we call **interface objects** and **interface syntax**.
You'll find these in Cats and other libraries.

これから、型クラスの二つの利用パターンである**インターフェースオブジェクト**と**インターフェース構文**を紹介する。これらはCatsやその他のライブラリで見ることができる。

#### インターフェースオブジェクト Interface Object

The simplest way of creating an interface that uses a type class
is to place methods in a singleton object:

型クラスを利用するインターフェースを定義するもっとも簡単な方法は、シングルトンオブジェクトの中にそれらのメソッドを配置するというものである。

<!--
TODO: 訳注するか？
ここでいうインターフェースとは、型クラスを利用する窓口となるようなusing句付きのメソッドを指している
-->

```scala mdoc:silent
object Json {
  def toJson[A](value: A)(using w: JsonWriter[A]): Json =
    w.write(value)
}
```

To use this object, we import any type class instances we care about
and call the relevant method:

利用するときは、使いたい型クラスインスタンスをインポートし、該当メソッドを呼び出す。

```scala mdoc:silent
import JsonWriterInstances.{*, given}
```

```scala mdoc
Json.toJson(Person("Dave", "dave@example.com"))
```

The compiler spots that we've called the `toJson` method
without providing the given instances.
It tries to fix this by searching for given instances
of the relevant types and inserting them at the call site.

コンパイラは、この `toJson` メソッド呼び出しにおいてコンテキストパラメータが与えられていないことを検出すると、関連する型のgivenインスタンスを探し、それを呼び出し箇所に挿入することで、問題を解決しようとする。

#### インターフェース構文 Interface Syntax

We can alternatively use **extension methods** to
extend existing types with interface methods.
This is sometimes referred to as as **syntax** for the type class, 
which is the term used by Cats.
Scala 2 has an equivalent to extension methods known as **implicit classes**.

**拡張メソッド**を使用することで、型クラスの利用インターフェースとなるメソッドを既存の型に追加することもできる[^pimping]。Catsでは、これは型クラスの**構文（syntax）**と呼ばれる。Scala2では、拡張メソッドに相当するものとして**暗黙クラス（implicit classes）**が存在する。

[]: You may occasionally see extension methods
referred to as "type enrichment" or "pimping".
These are older terms that we don't use anymore.

[^pimping]: 拡張メソッドは「型の強化（type enrichment）」や「ピンピング（pimping）」と呼ばれることもあるが、これらは古い用語であり、現在は使用しない。

Here's an example defining an extension method to add a `toJson` method to
any type for which we have a `JsonWriter` instance.

次に示すのは、拡張メソッドを定義することによって、`JsonWriter` インスタンスが提供されている任意の型に `toJson` メソッドを追加している例である。

```scala mdoc:silent
object JsonSyntax {
  extension [A](value: A) {
    def toJson(using w: JsonWriter[A]): Json =
      w.write(value)
  }
}
```

We use interface syntax by importing it
alongside the instances for the types we need:

これを必要な型クラスインスタンスと一緒にインポートすることでインターフェース構文を利用できる。

```scala mdoc:silent
import JsonWriterInstances.given
import JsonSyntax.*
```

```scala mdoc
Person("Dave", "dave@example.com").toJson
```

<div class="callout callout-info">
#### トレイトの拡張メソッド Extension Methods on Traits {-}

In Scala 3 we can define extension methods directly on a type class trait.
Since we're defining `toJson` as just calling `write` on `JsonWriter`,
we could instead define `toJson` directly on `JsonWriter` and avoid creating an separate extension method.

Scala3では、型クラスのトレイト上に直接拡張メソッドを定義することができる。`toJson` は、単純に `JsonWriter` の `write` メソッドを呼び出す形で定義されているが、そうする代わりに `toJson` を直接 `JsonWriter` 上に定義することで、別途拡張メソッドを作成する必要がなくなる。

```scala mdoc:invisible:reset-object
// JSONのシンプルな抽象構文木を定義する
sealed trait Json
final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json
```

```scala mdoc:silent
trait JsonWriter[A] {
  extension (value: A) def toJson: Json
}

object JsonWriter {
  given stringWriter: JsonWriter[String] =
    new JsonWriter[String] {
      extension (value: String) 
        def toJson: Json = JsString(value)
    }
  
  // etc...
}
```

We do *not* advocate this approach, because of a limitation in how Scala searches for extension methods.
The following code fails because Scala only looks within the `String` companion object for extension methods,
and consequently does not find the extension method on the instance in the `JsonWriter` companion object.

とはいえ、このアプローチは推奨しない。Scalaにおける拡張メソッドの探索方法には制約があるからである。以下のコードは、Scalaが拡張メソッドを探索する際に `String` のコンパニオンオブジェクト内しか参照せず、`JsonWriter` のコンパニオンオブジェクト内のインスタンスに定義された拡張メソッドが見つからないため、失敗する。

```scala mdoc:fail
"A string".toJson
```

This means that users will have to explicitly import at least the instances for the built-in types (for which we cannot modify the companion objects).

つまり、少なくともコンパニオンオブジェクトを変更できない組み込みの型に関しては、利用者は、明示的にインポートを行わなくてはならない、ということになる。

```scala mdoc
import JsonWriter.given

"A string".toJson
```

For consistency we recommend separating the syntax from the type class instances and always explicitly importing it,
rather than requiring explicit imports for only some extension methods.

一部の拡張メソッドにだけ明示的なインポートが必要になるような書き方よりも、一貫性を重視し、構文（syntax）を型クラスインスタンスから分離して常に明示的にインポートすることを推奨する。
</div>

#### The `summon` Method

The Scala standard library provides
a generic type class interface called `summon`.
Its definition is very simple:

Scala標準ライブラリには `summon` というジェネリックな型クラスのインターフェースが用意されている。その定義は次のとおり、非常にシンプルである。

```scala
def summon[A](using value: A): A =
  value
```

We can use `summon` to summon any value in the given scope.
We provide the type we want and `summon` does the rest:

`summon` を使えば、givenスコープに存在する任意の値を呼び出すことができる。必要なのは、求めている値の型を指定することだけで、あとは `summon` に任せればよい。

```scala mdoc
summon[JsonWriter[String]]
```

Most type classes in Cats provide other means to summon instances.
However, `summon` is a good fallback for debugging purposes.
We can insert a call to `summon` within the general flow of our code
to ensure the compiler can find an instance of a type class
and ensure that there are no ambiguity errors.

Catsのほとんどの型クラスはインスタンスを入手するための方法を他にも提供しているが、`summon` はデバッグ目的で使える便利な代替手段である。コードの途中に `summon` 呼び出しを挿入することで、コンパイラが型クラスのインスタンスを見つけられるかどうか、また曖昧さのエラーがないかを確認することができる。
