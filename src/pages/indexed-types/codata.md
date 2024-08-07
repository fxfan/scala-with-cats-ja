## Indexed Codata

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

The constraint is made of two parts: using clauses, which we learned about in [@sec:type-classes], and the [`A =:= B`][scala.=:=] construction, which is new. `=:=` represents a type equality. If a given instance `A =:= B` exists, then the type `A` is equal to the type `B`. (Note we can write this with the more familiar prefix notation `=:=[A, B]` if we prefer.) We never create these instances; instead the compiler creates them for us. In the `on` method, we are asking the compiler to construct an instance `A =:= Off`, which can only be done if `A` is `Off`. This in turn means we can only call the method when the `Switch` is `Off`. This is the core idea of indexed codata: we reflect states as types, and restrict method calls to a subset of states.


### API Protocols

