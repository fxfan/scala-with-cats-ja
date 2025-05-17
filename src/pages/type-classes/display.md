<!--

## Exercise: Display Library {#sec:type-classes:display}

Scala provides a `toString` method
to let us convert any value to a `String`.
This method comes with a few disadvantages:

1. It is implemented for *every* type in the language.
   There are situations where we don't want to be able to view data.
   For example, we may want to ensure we don't log sensitive information,
   such as passwords,
   in plain text.

2. We can't customize `toString` for types we don't control.

Let's define a `Display` type class to work around these problems:

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

<div class="solution">
These steps define the three main components of our type class.
First we define `Display`---the type class itself:

```scala mdoc:silent:reset-object
trait Display[A] {
  def display(value: A): String
}
```

Then we define some default instances of `Display`
and package them in the `Display` companion object:

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
</div>

### Using the Library {#sec:type-classes:cat}

The code above forms a general purpose printing library
that we can use in multiple applications.
Let's define an "application" now that uses the library.

First we'll define a data type to represent a well-known type of furry animal:

```scala
final case class Cat(name: String, age: Int, color: String)
```

Next we'll create an implementation of `Display` for `Cat`
that returns content in the following format:

```ruby
NAME is a AGE year-old COLOR cat.
```

Finally, use the type class on the console or in a short demo app:
create a `Cat` and print it to the console:

```scala
// Define a cat:
val cat = Cat(/* ... */)

// Print the cat!
```

<div class="solution">
This is a standard use of the type class pattern.
First we define custom data type for our application:

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

Then we define type class instances for the types we care about.
These either go into the companion object of `Cat`
or a separate object to act as a namespace:

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

<div class="solution">
First we define `DisplaySyntax` with the extension methods we want.

```scala mdoc:silent
object DisplaySyntax {
  extension [A](value: A)(using p: Display[A]) {
    def display: String = p.display(value)
    def print: Unit = Display.print(value)
  }
}
```

Now we can show everything working by calling `print` on a `Cat`.

```scala mdoc
import DisplaySyntax.*

Cat("Garfield", 41, "ginger and black").print
```

We get a compile error if we haven't defined an instance of `Display`
for the relevant type:

```scala mdoc:fail
import java.util.Date
new Date().print
```
</div>


```scala mdoc:reset:silent
```
--->

## 演習: 表示ライブラリ {#sec:type-classes:display}

Scala は任意の値を `String` オブジェクトに変換するメソッド `toString` を提供している。このメソッドにはいくつか不便な点がある。

1. `toString` は言語内の*すべての*型に対して実装されているが、データを表示可能にしたくない場合もある。 たとえば、パスワードのような機密情報はログに記録されないようにしたいかもしれない
2. 自分たちのコントール外にある型に対して `toString` をカスタマイズすることはできない

これらの問題を解決するため、以下の詳細に従って `Display` 型クラスを定義せよ。

 1. 単一のメソッド `display` をもつ型クラス `Display[A]` を定義する。`display` は型 `A` の値を受け取り、`String` を返すメソッドとする
 2. `Display` コンパニオンオブジェクト上に、`String` および `Int` 用の ` Display` インスタンスを作成する
 3. `Display` コンパニオンオブジェクト上に、次のふたつのジェネリックなインターフェースメソッドを作成する
    - `display` メソッド。型 `A` の値と、それに対応する `Display` インスタンスを受け取る。対応する `Display` を使って `A` を `String` に変換する
    - `print` メソッド。`display` と同じパラメータを受け取り `Unit` を返す。`display` から得られる `A` の表示用文字列を `println` でコンソールに出力する

<div class="solution">
以下のステップは、今回の型クラスに関連した三つのコンポーネントを定義する。最初は、型クラスそのものである `Display` である。

```scala mdoc:silent:reset-object
trait Display[A] {
  def display(value: A): String
}
```

続いて、`Display` にいくつかデフォルトのインスタンスを定義する。これらは `Display` のコンパニオンオブジェクトに配置する。

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

最後に、`Display` コンパニオンオブジェクトを拡張し、型クラスの利用窓口となる基本的なインターフェースを提供する。

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

`print` メソッドのパラメータになっている `Display` インスタンスが無名であることに注目しよう。この書き方は Scala3 から導入された。このインスタンスは `display` メソッドに受け渡されるだけなので、名前がなくても問題ない。
</div>

### 表示ライブラリの利用 {#sec:type-classes:cat}

上記のコードは、さまざまなアプリケーションで使用できる汎用的な表示ライブラリを形成している。次の手順に従い、このライブラリを利用するアプリケーションを定義せよ。

まず、みんなおなじみのモフモフした動物を表すデータ型を定義する。

```scala
final case class Cat(name: String, age: Int, color: String)
```

次に `Cat` 用の `Display` 実装を作成する。この実装はデータを以下のフォーマットで返す。

```ruby
NAME is a AGE year-old COLOR cat.
```

最後に、コンソールか簡単なデモアプリでこの型クラスを使う。`Cat` オブジェクトを作成し、これをコンソールに表示せよ。

```scala
// 猫オブジェクトを作成
val cat = Cat(/* ... */)

// ここで猫を表示！
```

<div class="solution">
これは型クラスパターンの標準的な使い方である。まずはこのアプリケーション用にデータ型を定義する。

```scala mdoc:silent
final case class Cat(name: String, age: Int, color: String)
```

そして、そのデータ型のための型クラスインスタンスを定義する。定義場所は `Cat` のコンパニオンオブジェクトか、もしくは名前空間の役割をもった別のオブジェクトである。

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

最後に、使いたい型クラスインスタンスをスコープにもちこみ、インターフェースオブジェクトもしくはインターフェース構文を用いて、型クラスを利用する。型クラスインスタンスをコンパニオンオブジェクトに定義したのであれば Scala は自動的にそれらをスコープに含めるが、そうでない場合は `import` を用いてアクセスする。

```scala mdoc:silent
val cat = Cat("Garfield", 41, "ginger and black")
```
```scala mdoc
Display.print(cat)
```
</div>


### 表示ライブラリの構文をもっと便利にする

以下の手順に従って拡張メソッドを追加し、この表示ライブラリをもっと簡単に扱えるようにせよ。

 1. `DisplaySyntax` オブジェクトを作成する
 2. `display` と `print` を拡張メソッドとして `DisplaySyntax` 上に定義する
 3. その拡張メソッドを使って、前の演習で作成した `Cat` オブジェクトを表示する

<div class="solution">
まず `DisplaySyntax` と必要な拡張メソッドを定義する。

```scala mdoc:silent
object DisplaySyntax {
  extension [A](value: A)(using p: Display[A]) {
    def display: String = p.display(value)
    def print: Unit = Display.print(value)
  }
}
```

これで、`Cat` オブジェクトに対して `print` を呼び出せば、その猫に関する全情報を表示できる。

```scala mdoc
import DisplaySyntax.*

Cat("Garfield", 41, "ginger and black").print
```

`Display` インスタンスが定義されていない型に対して拡張メソッドを呼び出そうとすると、コンパイルエラーになる。

```scala mdoc:fail
import java.util.Date
new Date().print
```
</div>
