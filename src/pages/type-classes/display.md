## 演習: Display Library {#sec:type-classes:display}

Scala provides a `toString` method
to let us convert any value to a `String`.
This method comes with a few disadvantages:

Scalaは任意の値を `String` オブジェクトに変換するメソッド `toString` を提供している。

1. It is implemented for *every* type in the language.
   There are situations where we don't want to be able to view data.
   For example, we may want to ensure we don't log sensitive information,
   such as passwords,
   in plain text.

2. We can't customize `toString` for types we don't control.

-----

1. `toString` は言語内の*すべての*型に対して実装されているが、データを表示できないようにしたい状況もある。 たとえば、パスワードのような機密情報は、平文のままログに記録されることがないようにしたいだろう
2. 我々のコントール外にある型に対して `toString` をカスタマイズすることはできない

Let's define a `Display` type class to work around these problems:

これらの問題を解決するため、以下の詳細に従って `Display` 型クラスを定義せよ。

 1. Define a type class `Display[A]` containing a single method `display`.
    `display` should accept a value of type `A` and return a `String`.

 2. Create instances of `Display` for `String` and `Int` 
    on the `Display` companion object.

 3. On the `Display` companion object create two generic interface methods:

    - `display` accepts a value of type `A`
    and a `Display` of the corresponding type.
    It uses the relevant `Display` to convert the `A` to a `String`.

    - `print` accepts the same parameters as `display` and returns `Unit`.
    It prints the displayed `A` value to the console using `println`.

-----

 1. 単一のメソッド `display` を含む型クラス `Display[A]` を定義する。`display` は型 `A` の値を受け取り、`String` を返すメソッドとする

 2. `Display` コンパニオンオブジェクト上に、`String` および `Int` 型の ` Display` インスタンスを作成する

 3. `Display` コンパニオンオブジェクト上に、次の二つのジェネリックなインターフェースメソッドを作成する

    - `display` : 型 `A` の値と、それに対応する `Display` インスタンスを受け取る。対応する `Display` を使って、`A` を `String` に変換する

    - `print` : `display` と同じパラメータを受け取り、`Unit` を返す。`display` から得られる `A` の表示用文字列を、`println` を使ってコンソールに出力する

<div class="solution">
These steps define the three main components of our type class.
First we define `Display`---the type class itself:

以下のステップによって、今回の型クラスに関連した三つのコンポーネントを定義する。最初は、型クラスそのものである `Display` である。

```scala mdoc:silent:reset-object
trait Display[A] {
  def display(value: A): String
}
```

Then we define some default instances of `Display`
and package them in the `Display` companion object:

続いて、`Display` にいくつかデフォルトのインスタンスを定義する。それらは `Display` のコンパニオンオブジェクトに配置する。

```scala mdoc:silent
object Display {
  given stringDisplay: Display[String] with {
    def display(input: String) = input
  }

  given intDisplay: Display[Int] with {
    def display(input: Int) = input.toString
  }
}
```

Finally we extend the `Display` companion object to provide a basic interface:

最後に、`Display` コンパニオンオブジェクトを拡張して、型クラスの利用窓口となる基本的なインターフェースを提供する。

```scala mdoc:invisible:reset-object
trait Display[A] {
  def display(value: A): String
}
```
```scala mdoc:silent
object Display {
  given stringDisplay: Display[String] with {
    def display(input: String) = input
  }

  given intDisplay: Display[Int] with {
    def display(input: Int) = input.toString
  }

  def display[A](input: A)(using p: Display[A]): String =
    p.display(input)

  def print[A](input: A)(using Display[A]): Unit =
    println(display(input))
}
```

Notice that the `Display` instance on `print` is anonymous.
This is allowed in Scala 3, and works because we only pass it to `display`.

`print` メソッドのパラメータになっている `Display` インスタンスが無名であることに注目しよう。このインスタンスは `display` メソッドに自動的に受け渡されるだけなので、名前がなくても問題ない。この書き方はScala3から導入された。
</div>

### Using the Library {#sec:type-classes:cat}

The code above forms a general purpose printing library
that we can use in multiple applications.
Let's define an "application" now that uses the library.

上記のコードは、さまざまなアプリケーションで使用できる汎用的な出力ライブラリを形成している。次の手順に従い、このライブラリを利用するアプリケーションを定義せよ。

First we'll define a data type to represent a well-known type of furry animal:

まず、みんなおなじみのモフモフした動物を表すデータ型を定義する。

```scala
final case class Cat(name: String, age: Int, color: String)
```

Next we'll create an implementation of `Display` for `Cat`
that returns content in the following format:

次に、`Cat` 用の `Display` 実装を作成する。この実装はデータを次のフォーマットで返す。

```ruby
NAME is a AGE year-old COLOR cat.
```

Finally, use the type class on the console or in a short demo app:
create a `Cat` and print it to the console:

最後に、コンソールか簡単なデモアプリでこの型クラスを使う。`Cat` オブジェクトを作成し、これをコンソールに出力しよう。

```scala
// 猫オブジェクトを作成
val cat = Cat(/* ... */)

// ここで猫を表示！
```

<div class="solution">
This is a standard use of the type class pattern.
First we define custom data type for our application:

これは型クラスパターンの標準的な使い方である。まずはこのアプリケーション用にデータ型を定義する。

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

Then we define type class instances for the types we care about.
These either go into the companion object of `Cat`
or a separate object to act as a namespace:

そして、そのデータ型のための型クラスインスタンスを定義する。定義場所は、`Cat` のコンパニオンオブジェクトか、名前空間の役割をもった別のオブジェクトのどちらかである。

```scala mdoc:silent
given catDisplay: Display[Cat] = new Display[Cat] {
  def display(cat: Cat) = {
    val name  = Display.display(cat.name)
    val age   = Display.display(cat.age)
    val color = Display.display(cat.color)
    s"$name is a $age year-old $color cat."
  }
}
```

Finally, we use the type class by
bringing the relevant instances into scope
and using interface object/syntax.
If we defined the instances in companion objects
Scala brings them into scope for us automatically.
Otherwise we use an `import` to access them:

最後に、使いたい型クラスインスタンスをスコープに持ち込み、インターフェースオブジェクトもしくはインターフェース構文を用いて、型クラスを利用する。型クラスインスタンスをコンパニオンオブジェクトに定義したのであれば、Scalaは自動的にそれらをスコープに含めるが、そうでない場合は `import` することによってアクセス可能にする。

```scala mdoc:silent
val cat = Cat("Garfield", 41, "ginger and black")
```
```scala mdoc
Display.print(cat)
```
</div>


### Better Syntax

Let's make our printing library easier to use
by adding extension methods for its functionality:

 1. Create an object `DisplaySyntax`.
 
 2. Define `display` and `print` as extension methods on `DisplaySyntax`. 

 3. Use the extension methods to print the example `Cat`
    you created in the previous exercise.

拡張メソッドを追加してこの出力ライブラリをもっと簡単に扱えるようにせよ。手順はつぎのとおり。

 1. `DisplaySyntax` オブジェクトを作成する
 2. `DisplaySyntax` の拡張メソッドとして `display` と `print` を定義する
 3. その拡張メソッドを使って、前の演習で作成した `Cat` オブジェクトを出力する

<div class="solution">
```scala mdoc:reset:invisible
trait Display[A] {
  def display(value: A): String
  def print(value: A): Unit =
    println(display(value))
}
```

First we define `DisplaySyntax` with the extension methods we want.

まず、`DisplaySyntax` と必要な拡張メソッドを定義する。

```scala mdoc:silent
object DisplaySyntax {
  extension [A](value: A)(using p: Display[A]) {
    def display: String = p.display(value)
    def print: Unit = p.print(value)
  }
}
```

Now we can show everything working by calling `print` on a `Cat`.

`Cat` オブジェクトに対して `print` を呼び出すことで、その猫に関する全情報を表示する。

```scala mdoc:reset:invisible
trait Display[A] {
  def display(value: A): String
  def print(value: A): Unit =
    println(display(value))
}
object Display {
  given Display[String] with {
    def display(value: String) = value
  }

  given Display[Int] with {
    def display(value: Int) = value.toString
  }
}

object DisplaySyntax {
  extension [A](value: A)(using p: Display[A]) {
    def display: String = p.display(value)
    def print: Unit = println(value.display)
  }
}
final case class Cat(name: String, age: Int, color: String)
```
```scala mdoc
import DisplaySyntax.*

given Display[Cat] with {
  def display(cat: Cat): String = {
    val name  = cat.name.display
    val age   = cat.age.display
    val color = cat.color.display
    s"$name is a $age year-old $color cat."
  }
}

Cat("Garfield", 41, "ginger and black").print
```

We get a compile error if we haven't defined an instance of `Display`
for the relevant type:

`Display` インスタンスが定義されていない型に対する `print` 呼び出しはエラーとなる。

```scala mdoc:fail
import java.util.Date
new Date().print
```
</div>
