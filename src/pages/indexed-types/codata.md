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

```scala mdoc:reset:silent
trait Switch[A] {
  def on: Switch[A]
  def off: Switch[A]
}
```

This type parameter doesn't correspond to any data we store in `Switch`, so it is a phantom type.
This is the first part of implementing indexed codata.
We are now going to add constraints that say we can only call a certain method when this type parameter corresponds to a particular concrete type.
It is in this way that indexed codata goes beyond what phantom types alone can do: we can inspect, at compile-time, the type of a type parameter and make decisions based on this type.

Implementing these constraints has two parts.
The first is defining types to represent on and off. 

Here are the types.

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

The constraint is made of two parts: using clauses, which we learned about in [@sec:type-classes], and the [`A =:= B`][scala.=:=] construction, which is new. `=:=` represents a type equality. If a given instance `A =:= B` exists, then the type `A` is equal to the type `B`. (Note we can write this with the more familiar prefix notation `=:=[A, B]` if we prefer.) We never create these instances ourselves. Instead the compiler creates them for us. In the `on` method, we are asking the compiler to construct an instance `A =:= Off`, which can only be done if `A` *is* `Off`. This in turn means we can only call the method when the `Switch` is `Off`. This is the core idea of indexed codata: we reflect states as types, and restrict method calls to a subset of states.


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

An API protocol defines the order in which methods must be called. The protocol in the case of `Switch` is that we can only call `off` after calling `on` and vice versa. This protocol is a simple finite state machine, and illustrated in Figure [@img:indexed-types:switch]. Many common types have similar protocols. For example, files can only be read once they are opened and cannot be read once they have been closed.

![The switch API protocol](src/pages/indexed-types/switch.pdf) {#img:indexed-types:switch}

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
An opening tag is closed by a corresponding closing tag such as `</h1>` for `<h1>` and `</p>` for `<p>`.

There are several rules for valid HTML[^valid-html]. We're going to focus on the following:

1. Within the `html` tag there can only be a `head` and a `body` tag, in that order.
2. Within the `head` tag there must be exactly one `title`, and there can be any other number of allowed tags.
3. Within the `body` there can be any number of allowed tags.

We're going to use a Church-encoded representation for HTML.
As the code is fairly repetitive we will just present all the code and then discuss the important parts.
Here's the implementation.

```scala mdoc:silent
sealed trait StructureState
trait Empty extends StructureState
trait InHead extends StructureState
trait InBody extends StructureState

sealed trait TitleState
trait WithoutTitle extends TitleState
trait WithTitle extends TitleState

final class Html[S <: StructureState, T <: TitleState](
    head: List[String],
    body: List[String]
) {
  // Head tags ------------------------------------------------------------------

  def head(using S =:= Empty): Html[InHead, WithoutTitle] =
    Html(head, body)

  def title(
      text: String
  )(using S =:= InHead, T =:= WithoutTitle): Html[InHead, WithTitle] =
    Html(head :+ s"<title>$text</title>", this.body)

  def link(rel: String, href: String)(using S =:= InHead): Html[InHead, T] =
    Html(head :+ s"<link rel=\"$rel\" href=\"$href\"/>", body)

  // Body tags ------------------------------------------------------------------

  def body(using S =:= InHead, T =:= WithTitle): Html[InBody, WithTitle] =
    Html(head, body)

  def h1(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<h1>$text</h1>")

  def p(text: String)(using S =:= InBody): Html[InBody, T] =
    Html(head, body :+ s"<p>$text</p>")

  // Interpreter ----------------------------------------------------------------

  override def toString(): String = {
    val h = head.mkString("  <head>\n    ", "\n    ", "\n  </head>")
    val b = body.mkString("  <body>\n    ", "\n    ", "\n  </body>")

    s"\n<html>\n$h\n$b\n</head>"
  }
}
object Html {
  val empty: Html[Empty, WithoutTitle] = Html(List.empty, List.empty)
}
```

Here's an example in use.

```scala mdoc
Html.empty.head
  .title("Our Amazing Webpage")
  .body
  .h1("Where Amazing Exists")
  .p("Right here")
  .toString
```

Here's an example of the type system preventing an invalid construction.

```scala mdoc:fail
Html.empty.head
  .link("stylesheet", "styles.css")
  .body
  .h1("This Shouldn't Work")
```



- We start defining elements that go in the `head` of an HTML document. We'll only allow `title` and `link` elements, for simplicity.
- There must be exactly one `title` element, but there can be zero or more `link` elements.
- Once we start defining `body` elements, we cannot 


We can also implement more complex protcols, such as those that can be represented by context-free grammars, with more work. Let's see an example.

[html]: https://html.spec.whatwg.org/multipage/

[^valid-html]: The HTML specification allows for very lenient parsing of HTML. For example, if we don't define the `head` tag it will usually be inferred. However we aren't going to allow that kind of leniency in our API.
