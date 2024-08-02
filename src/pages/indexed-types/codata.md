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
Let's start implementing the constraint we are after.
Our first step is to add a type parameter, which will hold the state of the `Switch`.

```scala mdoc:reset:silent
trait Switch[A] {
  def on: Switch[A]
  def off: Switch[A]
}
```

This type parameter doesn't correspond to any data we store in `Switch`, so it is sometimes called a phantom type.
This is the first part of implementing indexed codata.
We still need to add the constraints.
This has two parts: defining types to represent on and off, and then adding the constraints to the relevant methods on `Switch`.

Here are the types.

```scala mdoc:reset:silent
trait On
trait Off
```

Now we can add the constraints.

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
