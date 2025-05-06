<!--

## Indexed Codata

The basic idea of indexed codata is to prevent methods being called unless certain conditions, encoded in types, are met. More precisely, methods are guarded by type equalities that callers must prove they satisfy to call a method. The contextual abstraction features, `given` instances and `using` clauses, are used to implement this in Scala.

We'll start our exploration of indexed codata with a very simple example.
We are going to define a switch that can only be turned on when it is off, and off when it is on.
Since this is codata, we start with an interface.

```scala mdoc:silent
trait Switch {
  def on: Switch
  def off: Switch
}
```

There are no constraints on this interface as defined; we can turn any switch on, even if it is already on, and vice versa.
The first step to implement such a constraint is to add a type parameter, which will hold the state of the `Switch`.
This type parameter doesn't correspond to any data we store in `Switch`, so it is a phantom type.

```scala mdoc:reset:silent
trait Switch[A] {
  def on: Switch[A]
  def off: Switch[A]
}
```

We are now going to add constraints that say we can only call certain methods when this type parameter corresponds to particular concrete types.
It is in this way that indexed codata goes beyond what phantom types alone can do: we can inspect, at compile-time, the type of a type parameter and make decisions based on this type.

Implementing these constraints has two parts.
The first is defining types to represent on and off. 

```scala mdoc:reset:silent
trait On
trait Off
```

The second step is to add the constraints to the relevant methods on `Switch`.
Here is how we do it.

```scala mdoc:silent
trait Switch[A] {
  def on(using ev: A =:= Off): Switch[On]
  def off(using ev: A =:= On): Switch[Off]
}
```

We can create an implementation to show it really works.

```scala mdoc:silent
final case class SimpleSwitch[A]() extends Switch[A] {
  def on(using ev: A =:= Off): Switch[On] =
    SimpleSwitch()
  def off(using ev: A =:= On): Switch[Off] =
    SimpleSwitch()
}
object SimpleSwitch {
  val on: Switch[On] = SimpleSwitch()
  val off: Switch[Off] = SimpleSwitch()
}
```

Here are some examples of using it correctly

```scala mdoc
SimpleSwitch.on.off
SimpleSwitch.off.on
```

Incorrect uses fail to compile.

```scala mdoc:fail
SimpleSwitch.on.on
```

The constraint is made of two parts: using clauses, which we learned about in Section [@sec:type-classes], and the [`A =:= B`][scala.=:=] construction, which is new. `=:=` represents a type equality. If a given instance `A =:= B` exists, then the type `A` is equal to the type `B`. (Note we can write this with the more familiar prefix notation `=:=[A, B]` if we prefer.) We never create these instances ourselves. Instead the compiler creates them for us. In the `on` method, we are asking the compiler to construct an instance `A =:= Off`, which can only be done if `A` *is* `Off`. This in turn means we can only call the method when the `Switch` is `Off`. This is the core idea of indexed codata: we raise states into types, and restrict method calls to a subset of states.

This is a different use of contextual abstraction to type classes. 
Type classes associate operations with types.
What we're doing here is proving some property of a type with respect to another type.
More precisely we're proving that a type parameter is equal to a particular type.
The given instance only exists when the compiler can prove this is the case.
Hence these given instances are sometimes called **evidence** or **witnesses**.
This different view subsumes type classes, 
as we can think of type classes as evidence that a type implements a certain interface.


#### Exercise: Torque {-}

In Section [@sec:indexed-types:phantom] we saw how we could use phantom types to represent units.
We also ran into a limitation: we had no way to inspect the phantom types and hence make decisions based on them.
Now, with indexed codata, we can do that.

Below if the definition of `Length` we previously used. Your mission is to:

1. implement a type `Force`, parameterized by a phantom type that represents the units of force;
2. implement a type `Torque`, parameterized by a phantom type that represents the units of torque;
3. define types `Newtons` and `NewtonMetres` to represent force in SI units;
4. implement a method `*` on `Force` that accepts a `Length` and returns a `Torque`. It can only be called if the `Force` is in `Newtons` and the `Length` is in `Metres`. In this case the `Torque` is in `NewtonMetres`. (Torque is force times length.)

```scala mdoc:silent
final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
```

<div class="solution">
Defining `Force`, `Torque`, and the unit types is just repeating the pattern we saw in the example code.

```scala mdoc:silent
trait Newtons
trait NewtonMetres

final case class Force[Unit](value: Double)
final case class Torque[Unit](value: Double)
```

To define the `*` method on `Force` we need constraints that `Forces` `Unit` type is `Newtons`, and `Lengths` `Unit` type is `Metres`. These are both type equalities, so we can express them with `=:=`.

```scala mdoc:reset:invisible
trait Metres
trait Feet
trait Newtons
trait NewtonMetres

final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
final case class Torque[Unit](value: Double)
```

```scala mdoc:silent
final case class Force[Unit](value: Double) {
  def *[L](length: Length[L])(using Unit =:= Newtons, L =:= Metres): Torque[NewtonMetres] =
    Torque(this.value * length.value)
}
```
</div>


### API Protocols

An API protocol defines the order in which methods must be called. The protocol in the case of `Switch` is that we can only call `off` after calling `on` and vice versa. This protocol is a simple finite state machine, and illustrated in Figure [@fig:indexed-types:switch]. Many common types have similar protocols. For example, files can only be read once they are opened and cannot be read once they have been closed.

![The switch API protocol](src/pages/indexed-types/switch.pdf){#fig:indexed-types:switch}

Indexed codata allows us to enforce API protocols at compile-time. Often these protocols are finite-state machines. We can represent these protocols with a single type parameter that represents the state, as we did with `Switch`. We can also use multiple type parameters if that makes for a more convenient representation. 

Let's see an example using multiple type parameters. We're going to build an API that represents a very limited subset of [HTML][html], the language the defines web pages. An example of HTML is below.

```html
<!DOCTYPE html>
<html>
  <head><title>Our Amazing Web Page</title></head>
  <body>
    <h1>This Is Our Amazing Web Page</h1>
    <p>Please be in awe of its <strong>amazingness</strong></p>
  </body>
</html>
```

In HTML the content of the page is marked up with tags, like `<h1>`, that give it meaning. 
For example, `<h1>` means a heading at level one, and `<p>` means a paragraph.
An opening tag is closed by a corresponding closing tag, such as `</h1>` for `<h1>` and `</p>` for `<p>`.

There are several rules for valid HTML[^valid-html]. We're going to focus on the following:

1. Within the `html` tag there can only be a `head` and a `body` tag, in that order.
2. Within the `head` tag there must be exactly one `title`, and there can be any other number of allowed tags (of which we're only going to model `link`).
3. Within the `body` there can be any number of allowed tags (of which we are only going to model `h1` and `p`).

We're going to use a Church-encoded representation for HTML, so tags are created by method calls. Figure [@fig:indexed-types:html] shows the finite state machine representation of the API protocol.
I find it easier to read as a regular expression, which we can write down as

$$
\text{head}\ \text{link}^*\ \text{title}\ \text{link}^*\ \text{body}\ (\text{h1}\, |\, \text{p})^*
$$

![The HTML API protocol](src/pages/indexed-types/html.pdf){#fig:indexed-types:html}

As the code is fairly repetitive I will just present all the code and then discuss the important parts.
Here's the implementation.

```scala mdoc:silent
sealed trait StructureState
trait Empty extends StructureState
trait InHead extends StructureState
trait InBody extends StructureState

sealed trait TitleState
trait WithoutTitle extends TitleState
trait WithTitle extends TitleState

// Not a case class so external users cannot copy it
// and break invariants
final class Html[S <: StructureState, T <: TitleState](
    head: Vector[String],
    body: Vector[String]
) {
  // Head tags ---------------------------------------------

  def head(using S =:= Empty): Html[InHead, WithoutTitle] =
    Html(head, body)

  def title(
      text: String
  )(using S =:= InHead, T =:= WithoutTitle): Html[InHead, WithTitle] =
    Html(head :+ s"<title>$text</title>", this.body)

  def link(rel: String, href: String)(using S =:= InHead): Html[InHead, T] =
    Html(head :+ s"<link rel=\"$rel\" href=\"$href\"/>", body)

  // Body tags ---------------------------------------------

  def body(using S =:= InHead, T =:= WithTitle): Html[InBody, WithTitle] =
    Html(head, body)

  def h1(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<h1>$text</h1>")

  def p(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<p>$text</p>")

  // Interpreter ------------------------------------------

  override def toString(): String = {
    val h = head.mkString("  <head>\n    ", "\n    ", "\n  </head>")
    val b = body.mkString("  <body>\n    ", "\n    ", "\n  </body>")

    s"\n<html>\n$h\n$b\n</html>"
  }
}
object Html {
  val empty: Html[Empty, WithoutTitle] = Html(Vector.empty, Vector.empty)
}
```

The key point is that we factor the state into two components.
`StructureState` represents where in the overall structure we are (inside the `head`, inside the `body`, or inside neither).
`TitleState` represents the state when defining the elements inside the `head`, specifically whether we have a `title` element or not.
We could certainly represent this with one state type variable, but I find the factored representation both easier to work with and easier for other developers to understand.

Here's an example in use.

```scala mdoc
Html.empty.head
  .link("stylesheet", "styles.css")
  .title("Our Amazing Webpage")
  .body
  .h1("Where Amazing Exists")
  .p("Right here")
  .toString
```

Here's an example of the type system preventing an invalid construction, in this case the lack of a title.

```scala mdoc:fail
Html.empty.head
  .link("stylesheet", "styles.css")
  .body
  .h1("This Shouldn't Work")
```

These error messages are not great. We'll address this in Chapter [@sec:usability].

We can implement more complex protocols, such as those that can be represented by context-free or even context-sensitive grammars, using the same technique.


#### Exercise: HTML API Design {-}

I don't particularly like the HTML API we developed above, 
as the flat method call structure doesn't match the nesting in the HTML structure we're creating.
I would prefer to write the following.

```scala 
Html.empty
  .head(_.title("Our Amazing Webpage"))
  .body(_.h1("Where Amazing Happens").p("Right here"))
  .toString
```

We still require the `head` is specified before the `body`, 
but now the nesting of the method calls matches the nesting of the structure.
Notice we're still using a Church-encoded representation.

Can you think of how to implement this? 
You'll need to use indexed codata, and perhaps a bit of inspiration.
This is a very open ended question, so don't worry if you struggle with it!

<div class="solution">
Here's how I implemented it.
The structure is very similar to the original implementation,
but where we factored the state into type parameters 
I also factored the implementation into types.
Notice how we use `Head` and `Body` to accumulate the set of tags that make up the head and body respectively.
We still need to use indexed codata in some place, but we can avoid it in others.
For example, the `head` method simply requires a function of type `Head[WithoutTitle] => Head[WithTitle]`.

```scala mdoc:reset:silent
sealed trait StructureState
trait NeedsHead extends StructureState
trait NeedsBody extends StructureState
trait Complete extends StructureState

sealed trait TitleState
trait WithoutTitle extends TitleState
trait WithTitle extends TitleState

final class Head[S <: TitleState](contents: Vector[String]) {
  def title(text: String)(using S =:= WithoutTitle): Head[WithTitle] =
    Head(contents :+ s"<title>$text</title>")

  def link(rel: String, href: String): Head[S] =
    Head(contents :+ s"<link rel=\"$rel\" href=\"$href\"/>")

  override def toString(): String =
    contents.mkString("  <head>\n    ", "\n    ", "\n  </head>")
}
object Head {
  val empty: Head[WithoutTitle] = Head(Vector.empty)
}

final class Body(contents: Vector[String]) {
  def h1(text: String): Body =
    Body(contents :+ s"<h1>$text</h1>")

  def p(text: String): Body =
    Body(contents :+ s"<p>$text</p>")

  override def toString(): String =
    contents.mkString("  <body>\n    ", "\n    ", "\n  </body>")
}
object Body {
  val empty: Body = Body(Vector.empty)
}

final class Html[S <: StructureState](
    head: Head[?],
    body: Body
) {
  def head(f: Head[WithoutTitle] => Head[WithTitle])(using
      S =:= NeedsHead
  ): Html[NeedsBody] =
    Html(f(Head.empty), body)

  def body(f: Body => Body)(using S =:= NeedsBody): Html[Complete] =
    Html(head, f(Body.empty))

  override def toString(): String = {
    s"\n<html>\n${head.toString()}\n${body.toString()}\n</html>"

  }
}
object Html {
  val empty: Html[NeedsHead] = Html(Head.empty, Body.empty)
}
```

As always, we should show that is works.
Here's the output from the motivating example.

```scala mdoc
Html.empty
  .head(_.title("Our Amazing Webpage"))
  .body(_.h1("Where Amazing Happens").p("Right here"))
  .toString()
```
</div>

[html]: https://html.spec.whatwg.org/multipage/

[^valid-html]: The HTML specification allows for very lenient parsing of HTML. For example, if we don't define the `head` tag it will usually be inferred. However we aren't going to allow that kind of leniency in our API.


### Beyond Equality Constraints

Indexed data is all about equality constraints: proofs that some type parameter is equal to some type.
However we can go beyond equality constraints with contextual abstraction.
We can use [`<:<`][scala.<:<] for evidence of a subtyping relationship, 
and [`NotGiven`][scala.NotGiven] for evidence that no given instance exists (with which we can test that types are not equal, for example).
Beyond that, we can view any given instance as evidence.

Let's return to our example of length, force, and torque to see how this is useful.
In the exercise where we defined torque as force times length, we fixed the computation to have SI units.
The example code is below.

```scala
final case class Force[Unit](value: Double) {
  def *[L](length: Length[L])(using Unit =:= Newtons, L =:= Metres): Torque[NewtonMetres] =
    Torque(this.value * length.value)
}
```

This is a reasonable thing to do, as other units are insane, but there are a lot of insane people out there.
To accommodate other unit types we can create given instances that represent the results of operations of interest.
In this case we want to represent the result of multiplying a length unit by the force unit.
In code we can write the following.

```scala mdoc:reset:invisible
trait Metres
trait Newtons
trait NewtonMetres

final case class Torque[Unit](value: Double)
```
```scala mdoc:silent
// Weird units
trait Feet
trait Pounds
trait PoundsFeet

// An instance exists if A * B = C
trait Multiply[A, B, C]
object Multiply {
  given Multiply[Metres, Newtons, NewtonMetres] = new Multiply {}
  given Multiply[Feet, Pounds, PoundsFeet] = new Multiply {}
}
```

Now we can define `*` methods on `Length` and `Force` in terms of `Multiply`.

```scala mdoc:silent
final case class Length[L](value: Double) {
  def *[F, T](that: Force[F])(using Multiply[L, F, T]): Torque[T] =
    Torque(this.value * that.value)
}

final case class Force[F](value: Double) {
  def *[L, T](that: Length[L])(using Multiply[F, L, T]): Torque[T] =
    Torque(this.value * that.value)
}
```

Here's an example showing it works. 

```scala mdoc
Length[Metres](3) * Force[Newtons](4)

// What is this nonsense?
Length[Feet](3) * Force[Pounds](4)
```

Note that's it hard to think of `Multiply` as a type class, as it does not provide *any* methods.
Viewing it as evidence, however, does make sense.


#### Exercise: Commutivitiy {-}

In the example above we defined a `Multiply` type class to represent that metres times newtons gives newton metres.
Multiplication is commutative. If $A \times B = C$, then $B \times A = C$. 
However we have not represented this, and if we try newtons times metres, as in the example below, the code will fail.

```scala mdoc:fail
Force[Newtons](3) * Length[Metres](4)
```

Add evidence to `Multiply` that if `Multiply[A, B, C]` exists, then so does `Multiply[B, A, C]`, and show that it solves this problem.


<div class="solution">
```scala mdoc:reset:invisible
trait Metres
trait Newtons
trait NewtonMetres

final case class Torque[Unit](value: Double)
```

To solve this I defined a given instance called `commutative`, as shown below.

```scala mdoc:silent
// An instance exists if A * B = C
trait Multiply[A, B, C]
object Multiply {
  given Multiply[Metres, Newtons, NewtonMetres] = new Multiply {}
  
  // A * B == B * A
  given commutative[A, B, C](using Multiply[A, B, C]): Multiply[B, A, C] =
    new Multiply {}
}
```
```scala mdoc:invisible
final case class Length[L](value: Double) {
  def *[F, T](that: Force[F])(using Multiply[L, F, T]): Torque[T] =
    Torque(this.value * that.value)
}

final case class Force[F](value: Double) {
  def *[L, T](that: Length[L])(using Multiply[F, L, T]): Torque[T] =
    Torque(this.value * that.value)
}
```

Now the example works as expected.

```scala mdoc
Force[Newtons](3) * Length[Metres](4)
```
</div>


```scala mdoc:reset:silent
```
--->

## インデックス付き余データ

インデックス付き余データの基本的な考え方は、型で表現された特定の条件が満たされている場合にのみメソッドを呼び出せるようにすることである。もっと具体的に言えば、メソッドは型の等式によってガードされており、呼び出す際にはその等式が満たされていることを証明する必要がある。Scala では、コンテキスト抽象化機能である given インスタンスと using 句を用いてこれを実現する。

インデックス付き余データの探求を始めよう。まずは非常にシンプルな例からである。オフの時にのみオンにでき、オンの時にのみオフにできるスイッチを定義していく。余データなので、まずやることはインターフェースの定義である。

```scala mdoc:silent
trait Switch {
  def on: Switch
  def off: Switch
}
```

このインターフェースにはまだ何の制約も定義されていないため、スイッチがオンの状態でも `on` を呼び出すことができるし、逆も同様である。このような操作を制限を実装するための第一歩は `Switch` の状態を保持する型パラメータを追加することである。この型パラメータは `Switch` に格納されるいかなるデータにも対応していないため、ファントム型である。

```scala mdoc:reset:silent
trait Switch[A] {
  def on: Switch[A]
  def off: Switch[A]
}
```

次に、この型パラメータが特定の具体型である場合にのみメソッドを呼び出せるような制約を追加していく。この方法によって、インデックス付き余データはファントム型が単独でできることに加えて、型パラメータに指定された具体型をコンパイル時に検査し、その型に基づいて判断を行うことができるようになる。

この制約の実装手順にはふたつのステップがある。まずはオンとオフを表す型を定義する。

```scala mdoc:reset:silent
trait On
trait Off
```

次に、対象となる `Switch` のメソッドに制約を追加する。記述のしかたは以下のとおりである。

```scala mdoc:silent
trait Switch[A] {
  def on(using ev: A =:= Off): Switch[On]
  def off(using ev: A =:= On): Switch[Off]
}
```

これが確かに正しく動作することを示す実装を作成しよう。

```scala mdoc:silent
final case class SimpleSwitch[A]() extends Switch[A] {
  def on(using ev: A =:= Off): Switch[On] =
    SimpleSwitch()
  def off(using ev: A =:= On): Switch[Off] =
    SimpleSwitch()
}
object SimpleSwitch {
  val on: Switch[On] = SimpleSwitch()
  val off: Switch[Off] = SimpleSwitch()
}
```

以下はこれを正しく使用している例である。

```scala mdoc
SimpleSwitch.on.off
SimpleSwitch.off.on
```

誤った使い方をするとコンパイルエラーになる。

```scala mdoc:fail
SimpleSwitch.on.on
```

この制約はふたつの部分から成り立っている。ひとつは[@sec:type-classes]節で学んだ using 句、もうひとつは新たに登場した [`A =:= B`][scala.=:=] 構文である。`=:=` は型の等価性を表す。もし `A =:= B` という型の given インスタンスが存在するならば、型 `A` は型 `B` と等しいと言える。お望みなら `=:=[A, B]` というおなじみのプレフィックス記法を使ってもよい。これらの given インスタンスは自分で作成するものではなく、コンパイラが自動的に生成する。`on` メソッドでは、コンパイラに `A =:= Off` インスタンスの構築を要求しているが、これは `A` が `Off` である場合にのみ可能となる。つまり、`Switch` が `Off` の状態であるときのみこのメソッドを呼び出せる。状態を型へと持ち上げ、メソッド呼び出しを特定の状態にあるときのみに制限する。これがインデックス付き余データの核心をなすアイデアである。

これは型クラスとは異なるコンテキスト抽象化のもうひとつの使い方である。型クラスは操作を型に関連付ける。一方ここでは、ある型と別のもうひとつの型との何らかの関係性を証明している。もっと具体的に言えば、型パラメータがある特定の型と等しいことを証明している。この given インスタンスは、その関係性が正しいことをコンパイラが証明できる場合にのみ存在する。そのため、これらの given インスタンスは**エビデンス（evidence）**あるいは**ウィットネス（witness）**と呼ばれることもある。この新たな視点は型クラスにも広げることができる。型クラスも、ある型が特定のインターフェースを実装していることの証拠として捉えることができる。

#### 演習: トルク {-}

[@sec:indexed-types:phantom]節ではファントム型を使用して単位を表現する方法を見た。また、ファントム型に指定された具体型を調べる方法がなく、それに基づいて判断を行うことができないという制限にも直面した。インデックス付き余データを使用すれば、それが可能となる。

ファントム型を学んだときに使用した `Length` の定義を以下に再掲する。

```scala mdoc:silent
final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
```

課題は次のとおりである。

1. 力の単位を表すファントム型でパラメータ化された型 `Force` を実装せよ。
2. トルクの単位を表すファントム型でパラメータ化された型 `Torque` を実装せよ。
3. 力を表現する SI 単位として `Newtons` および `NewtonMetres` を定義せよ。
4. `Force` 型に、`Length` を受け取り `Torque` を返すメソッド `*` を実装せよ。これは、`Force` の単位が `Newtons` で、`Length` の単位が `Metres` の場合にのみ呼び出すことができ、結果である `Torque` の単位は `NewtonMetres` となる（トルクは力と長さの積である）。

<div class="solution">

`Force`、`Torque`、および単位型の定義は、例で見たパターンを繰り返すだけである。

```scala mdoc:silent
trait Newtons
trait NewtonMetres

final case class Force[Unit](value: Double)
final case class Torque[Unit](value: Double)
```

`Force` に `*` メソッドを定義するにあたっては、`Force` の `Unit` 型が `Newtons` であり、`Length` の `Unit` 型が `Metres` であるという制約が必要である。いずれも型の等価性を要求する制約なので、`=:=` を用いて表現できる。

```scala mdoc:reset:invisible
trait Metres
trait Feet
trait Newtons
trait NewtonMetres

final case class Length[Unit](value: Double) {
  def +(that: Length[Unit]): Length[Unit] =
    Length[Unit](this.value + that.value)
}
final case class Torque[Unit](value: Double)
```

```scala mdoc:silent
final case class Force[Unit](value: Double) {
  def *[L](length: Length[L])(using Unit =:= Newtons, L =:= Metres): Torque[NewtonMetres] =
    Torque(this.value * length.value)
}
```
</div>

### API プロトコル

API プロトコルはメソッドの呼び出し順序を定義する。メソッドはその順序で呼び出されなければならない。`Switch` におけるプロトコルは、`on` の後にのみ `off` を呼び出すことができ、`off` の後にのみ `on` を呼び出すことができる、というものである。このプロトコルは単純な有限状態機械である。これを図[@fig:indexed-types:switch]に示す。多くの一般的な型もこれと似たプロトコルをもっている。たとえば、ファイルは開かれて初めて読み取り可能になり、閉じられた後は読み取ることができない。

![The switch API protocol](src/pages/indexed-types/switch.pdf){#fig:indexed-types:switch}

インデックス付き余データを使えば、API プロトコルをコンパイル時に強制することができる。これらのプロトコルは有限状態機械であることが多い。`Switch` で行ったように、状態を表す単一の型パラメータで表現することができる。また、より便利な表現ができるのであれば複数の型パラメータを使用してもかまわない。

複数の型パラメータを使用する例を見てみよう。Web ページを定義する言語である [HTML][html] の小さなサブセットを表現する API を構築する。以下に HTML の例を示す。

```html
<!DOCTYPE html>
<html>
  <head><title>Our Amazing Web Page</title></head>
  <body>
    <h1>This Is Our Amazing Web Page</h1>
    <p>Please be in awe of its <strong>amazingness</strong></p>
  </body>
</html>
```

HTML では、ページのコンテンツに対して `<h1>` のようなタグでマークアップすることで意味を与える。たとえば、`<h1>` は大見出しを、`<p>` は段落を意味する。開始タグは対応する終了タグによって閉じられる。`<h1>` に対しては `</h1>`、`<p>` に対しては `</p>`　が終了タグとなる。

有効な HTML にはいくつかのルールがある[^valid-html]。ここでは以下のルールに焦点を当てる。

1. `html` タグ内には、`head` タグと `body` タグだけがこの順番で配置されなければならない
2. `head` タグ内には、`title` タグを必ずひとつだけ含まなければならず、それ以外の許可されたタグ（ここでは `link` のみをモデル化する）を好きな数だけ含むことができる
3. `body` タグ内には、許可されたタグ（ここでは`h1`と`p`のみをモデル化する）を好きな数だけ含むことができる

ここではチャーチ表現を用いるので、タグはメソッド呼び出しによって作成される。図[@fig:indexed-types:html]は API プロトコルの有限状態機械で表現したものである。これは以下に記述したとおり正規表現として表記すると読みやすくなる。

$$
\text{head}\ \text{link}^*\ \text{title}\ \text{link}^*\ \text{body}\ (\text{h1}\, |\, \text{p})^*
$$

![The HTML API protocol](src/pages/indexed-types/html.pdf){#fig:indexed-types:html}

コードはかなり反復的であるため、まず全体を提示し、その後に重要な部分を説明する。以下がその実装である。

```scala mdoc:silent
sealed trait StructureState
trait Empty extends StructureState
trait InHead extends StructureState
trait InBody extends StructureState

sealed trait TitleState
trait WithoutTitle extends TitleState
trait WithTitle extends TitleState

// 利用者が copy によって不変条件を破らないよう、case class は使わない
final class Html[S <: StructureState, T <: TitleState](
    head: Vector[String],
    body: Vector[String]
) {
  // Head タグ ---------------------------------------------

  def head(using S =:= Empty): Html[InHead, WithoutTitle] =
    Html(head, body)

  def title(
      text: String
  )(using S =:= InHead, T =:= WithoutTitle): Html[InHead, WithTitle] =
    Html(head :+ s"<title>$text</title>", this.body)

  def link(rel: String, href: String)(using S =:= InHead): Html[InHead, T] =
    Html(head :+ s"<link rel=\"$rel\" href=\"$href\"/>", body)

  // Body タグ ---------------------------------------------

  def body(using S =:= InHead, T =:= WithTitle): Html[InBody, WithTitle] =
    Html(head, body)

  def h1(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<h1>$text</h1>")

  def p(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<p>$text</p>")

  // インタープリタ ------------------------------------------

  override def toString(): String = {
    val h = head.mkString("  <head>\n    ", "\n    ", "\n  </head>")
    val b = body.mkString("  <body>\n    ", "\n    ", "\n  </body>")

    s"\n<html>\n$h\n$b\n</html>"
  }
}
object Html {
  val empty: Html[Empty, WithoutTitle] = Html(Vector.empty, Vector.empty)
}
```

状態をふたつの構成要素に分けている点に注目してほしい。`StructureState` は構造全体の中でどこにいるか（`head` 内、`body` 内、またはどちらでもないか）を表し、`TitleState` は `head` 内において `title` 要素を定義済みかどうかを表している。これらの状態をひとつの型変数で表現することも可能だが、分割された表現の方が扱いやすく、他の開発者にも理解しやすいだろう。

使い方の例を以下に示す。

```scala mdoc
Html.empty.head
  .link("stylesheet", "styles.css")
  .title("Our Amazing Webpage")
  .body
  .h1("Where Amazing Exists")
  .p("Right here")
  .toString
```

Here's an example of the type system preventing an invalid construction, in this case the lack of a title.

次に挙げるのは、型システムによって無効な構築を防ぐ例である。この例ではタイトルが欠落している。

```scala mdoc:fail
Html.empty.head
  .link("stylesheet", "styles.css")
  .body
  .h1("This Shouldn't Work")
```

これらが出力するエラーメッセージはわかりやすいとはいえない。その点については[@sec:usability]章で取り扱う。

これと同じ技法を用いることで、文脈自由文法あるいは文脈依存文法で表現されるような、より複雑なプロトコルも実装することができる。

#### 演習: HTML APIデザイン {-}

上で作成した HTML の API には修正したい点がある。平坦なメソッド呼び出しの構造が、作成している HTML のネスト構造と合っていないのである。以下のように書ける方が好ましい。

```scala 
Html.empty
  .head(_.title("Our Amazing Webpage"))
  .body(_.h1("Where Amazing Happens").p("Right here"))
  .toString
```

`head` が `body` より先に指定される必要がある点は変わらないが、メソッド呼び出しのネストは構造のネストと一致している。引き続きチャーチ表現を用いていることに注目してほしい。

この実装方法を考えられるだろうか。インデックス付き余データを使用する必要があり、インスピレーションもすこし必要かもしれない。これは非常に自由度の高い問題である。解法に迷っても気にすることはない。

<div class="solution">

以下に実装の一例を示す。構造は元の実装と非常によく似ているが、状態を複数の型パラメータに分けたのと同様に、実装を複数の型に分割している。ヘッドとボディを構成するタグを蓄積するために `Head` と `Body` をどのように使っているのか、よく確認してほしい。ある部分ではインデックス付き余データを引き続き使用する必要があるが、避けることが可能な部分もある。たとえば `head` メソッドは単に `Head[WithoutTitle] => Head[WithTitle]` 型の関数を要求するだけである。

```scala mdoc:reset:silent
sealed trait StructureState
trait NeedsHead extends StructureState
trait NeedsBody extends StructureState
trait Complete extends StructureState

sealed trait TitleState
trait WithoutTitle extends TitleState
trait WithTitle extends TitleState

final class Head[S <: TitleState](contents: Vector[String]) {
  def title(text: String)(using S =:= WithoutTitle): Head[WithTitle] =
    Head(contents :+ s"<title>$text</title>")

  def link(rel: String, href: String): Head[S] =
    Head(contents :+ s"<link rel=\"$rel\" href=\"$href\"/>")

  override def toString(): String =
    contents.mkString("  <head>\n    ", "\n    ", "\n  </head>")
}
object Head {
  val empty: Head[WithoutTitle] = Head(Vector.empty)
}

final class Body(contents: Vector[String]) {
  def h1(text: String): Body =
    Body(contents :+ s"<h1>$text</h1>")

  def p(text: String): Body =
    Body(contents :+ s"<p>$text</p>")

  override def toString(): String =
    contents.mkString("  <body>\n    ", "\n    ", "\n  </body>")
}
object Body {
  val empty: Body = Body(Vector.empty)
}

final class Html[S <: StructureState](
    head: Head[?],
    body: Body
) {
  def head(f: Head[WithoutTitle] => Head[WithTitle])(using
      S =:= NeedsHead
  ): Html[NeedsBody] =
    Html(f(Head.empty), body)

  def body(f: Body => Body)(using S =:= NeedsBody): Html[Complete] =
    Html(head, f(Body.empty))

  override def toString(): String = {
    s"\n<html>\n${head.toString()}\n${body.toString()}\n</html>"

  }
}
object Html {
  val empty: Html[NeedsHead] = Html(Head.empty, Body.empty)
}
```

As always, we should show that is works.
Here's the output from the motivating example.

いつもどおり、この実装が正しく動作することを示しておこう。この演習の冒頭に挙げたサンプルコードからの出力は次のとおりである。

```scala mdoc
Html.empty
  .head(_.title("Our Amazing Webpage"))
  .body(_.h1("Where Amazing Happens").p("Right here"))
  .toString()
```
</div>

[html]: https://html.spec.whatwg.org/multipage/

[^valid-html]: HTML 仕様は、HTML の解析について非常に寛容である。たとえば `head` タグを定義しなくても通常は補完される。しかし、ここで作成する API ではそのような寛容さは許可しない。

### 等価制約を超えて

インデックス付きデータはすべて等価制約、つまりある型パラメータが特定の型に等しいことを証明するものである。しかし、コンテキスト抽象を用いれば等価性以外の制約を実現することもできる。部分型関係のエビデンスには [`<:<`][scala.<:<] 、指定された given インスタンスが存在しないことのエビデンスには [`NotGiven`][scala.NotGiven] を使うことができる。これを用いれば、たとえば型が等しくないことをテストすることができる。さらに言えば、任意の given インスタンスをエビデンスとして扱うことが可能である。

その有用性を確認するために、長さ・力・トルクの例に戻ろう。トルクを力と長さの積として定義した演習では、計算を SI 単位に固定した。以下にそのコードを再掲する。

```scala
final case class Force[Unit](value: Double) {
  def *[L](length: Length[L])(using Unit =:= Newtons, L =:= Metres): Torque[NewtonMetres] =
    Torque(this.value * length.value)
}
```

他の単位は非合理的なのでこれは妥当な判断であるが、世の中にはそんな単位を好むおかしな人も多い。演算の結果を表す given インスタンスを作成することで、他の単位型にも対応することができる。ここでは、長さの単位と力の単位を掛け合わせた結果を表現したい。コードでは次のように書ける。

```scala mdoc:reset:invisible
trait Metres
trait Newtons
trait NewtonMetres

final case class Torque[Unit](value: Double)
```
```scala mdoc:silent
// 変な単位
trait Feet
trait Pounds
trait PoundsFeet

// A * B = C であればインスタンスが存在する
trait Multiply[A, B, C]
object Multiply {
  given Multiply[Metres, Newtons, NewtonMetres] = new Multiply {}
  given Multiply[Feet, Pounds, PoundsFeet] = new Multiply {}
}
```

そして、`Multiply` を利用して `Length` と `Force` に `*` メソッドを定義する。

```scala mdoc:silent
final case class Length[L](value: Double) {
  def *[F, T](that: Force[F])(using Multiply[L, F, T]): Torque[T] =
    Torque(this.value * that.value)
}

final case class Force[F](value: Double) {
  def *[L, T](that: Length[L])(using Multiply[F, L, T]): Torque[T] =
    Torque(this.value * that.value)
}
```

以下に、これが正しく動作することを示す例を挙げる。

```scala mdoc
Length[Metres](3) * Force[Newtons](4)

// 意味がわからないが
Length[Feet](3) * Force[Pounds](4)
```

`Multiply` は何のメソッドも提供しないので、これを型クラスと考えるのは無理がある。エビデンスと考えるのが妥当だろう。

#### 演習: 交換律 {-}

上の例では、メートルとニュートンを掛け合わせるとニュートンメートルが得られることを表現するために `Multiply` 型クラスを定義した。掛け算は交換律を満たす。もし $A \times B = C$ なら、$B \times A = C$ である。しかし、現在の定義ではこれを表現できていないため、以下の例のようにニュートンに対してメートルを掛けると、コードは失敗する。

```scala mdoc:fail
Force[Newtons](3) * Length[Metres](4)
```

`Multiply[A, B, C]` が存在するなら `Multiply[B, A, C]` も存在するように `Multiply` のエビデンスを追加し、それによってこの問題が解決することを示せ。

<div class="solution">
```scala mdoc:reset:invisible
trait Metres
trait Newtons
trait NewtonMetres

final case class Torque[Unit](value: Double)
```

以下では `commutative` という given インスタンスを定義することによってこれを解決している。

```scala mdoc:silent
// A * B = C であればインスタンスが存在する
trait Multiply[A, B, C]
object Multiply {
  given Multiply[Metres, Newtons, NewtonMetres] = new Multiply {}
  
  // A * B == B * A
  given commutative[A, B, C](using Multiply[A, B, C]): Multiply[B, A, C] =
    new Multiply {}
}
```
```scala mdoc:invisible
final case class Length[L](value: Double) {
  def *[F, T](that: Force[F])(using Multiply[L, F, T]): Torque[T] =
    Torque(this.value * that.value)
}

final case class Force[F](value: Double) {
  def *[L, T](that: Length[L])(using Multiply[F, L, T]): Torque[T] =
    Torque(this.value * that.value)
}
```

これで、先ほど失敗した例は想定どおり正しく動作する。

```scala mdoc
Force[Newtons](3) * Length[Metres](4)
```
</div>
