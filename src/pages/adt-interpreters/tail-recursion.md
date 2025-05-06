<!--

## Tail Recursive Interpreters

Structural recursion, as we have written it, uses the stack. This is not often a problem, but particularly deep recursions can lead to the stack running out of space. A solution is to write a **tail recursive** program. A tail recursive program does not need to use any stack space, and so is sometimes known as **stack safe**. Any program can be turned into a tail recursive version, which does not use the stack and therefore cannot run out of stack space.

<div class="callout callout-info">
#### The Call Stack {-}

Method and function calls are usually implemented using an area of memory known as the call stack, or just the stack for short.
Every method or function call uses a small amount of memory on the stack, called a stack frame.
When the method or function returns, this memory is freed and becomes available for future calls to use.

A large number of method calls, without corresponding returns, can require more stack frames than the stack can accommodate. When there is no more memory available on the stack we say we have overflowed the stack. In Scala a `StackOverflowError` is raised when this happens. 
</div>

In this section we will discuss tail recursion, converting programs to tail recursive form, and limitations and workarounds for the Scala's runtimes.


### The Problem of Stack Safety

Let's start by seeing the problem. In Scala we can create a repeated `String` using the `*` method.

```scala mdoc
"a" * 4
```

```scala mdoc:invisible
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int): Option[Int] =
      regexp match {
        case Append(left, right) =>
          loop(left, idx).flatMap(i => loop(right, i))
        case OrElse(first, second) =>
          loop(first, idx).orElse(loop(second, idx))
        case Repeat(source) =>
          loop(source, idx)
            .flatMap(i => loop(regexp, i))
            .orElse(Some(idx))
        case Apply(string) =>
          Option.when(input.startsWith(string, idx))(idx + string.size)
        case Empty =>
          None
      }

    // Check we matched the entire input
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  def apply(string: String): Regexp =
    Apply(string)
}
```

We can match such a `String` with a regular expression and `repeat`.

```scala mdoc
Regexp("a").repeat.matches("a" * 4)
```

However, if we make the input very long the interpreter will fail with a stack overflow exception.

```scala
Regexp("a").repeat.matches("a" * 20000)
// java.lang.StackOverflowError
```

This is because the interpreter calls `loop` for each instance of a repeat, without returning. However, all is not lost. We can rewrite the interpreter in a way that consumes a fixed amount of stack space, and therefore match input that is as large as we like.


### Tail Calls and Tail Position

Our starting point is **tail calls**. A tail call is a method call that does not take any additional stack space. Only method calls that are in **tail position** are candidates to be turned into tail calls. Even then, runtime limitations mean that not all calls in tail position will be converted to tail calls.

A method call in tail position is a call that immediately returns the value returned by the call.
Let's see an example.
Below are two versions of a method to calculate the sum of the integers from 0 to `count`.

```scala mdoc:silent
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

The method call to `isntTailRecursive` in

```scala
case n => n + isntTailRecursive(n - 1)
```

is not in tail position, because the value returned by the call is then used in the addition.
However, the call to `loop` in

```scala
case n => loop(n - 1, accum + n)
```

is in tail position because the value returned by the call to `loop` is itself immediately returned.
Similarly, the call to `loop` in

```scala
loop(count, 0)
```

is also in tail position.

A method call in tail position is a candidate to be turned into a tail call. Limitations of Scala's runtimes mean that not all calls in tail position can be made tail calls. Currently, only calls from a method to itself that are also in tail position will be converted to tail calls. This means

```scala
case n => loop(n - 1, accum + n)
```

is converted to a tail call, because `loop` is calling itself. However, the call

```scala
loop(count, 0)
```

is not converted to a tail call, because the call is from `isTailRecursive` to `loop`. 
This will not cause issues with stack consumption, however, because this call only happens once.

<div class="callout callout-info">
#### Runtimes and Tail Calls {-}

Scala supports three different platforms: the JVM, Javascript via Scala.js, and native code via Scala Native. Each platform provides what is known as a runtime, which is code that supports our Scala code when it is running. The garbage collector, for example, is part of the runtime.

At the time of writing none of Scala's runtimes support full tail calls. However, there is reason to think this may change in the future. [Project Loom](https://wiki.openjdk.org/display/loom/Main) should eventually add support for tail calls to the JVM. Scala Native is likely to support tail calls soon, as part of other work to implement continuations. Tail calls have been part of the Javascript specification for a long time, but remain unimplemented by the majority of Javascript runtimes. However, WebAssembly does support tail calls and will probably replace compiling Scala to Javascript in the medium term.
</div>

We can ask the Scala compiler to check that all self calls are in tail position by adding the `@tailrec` annotation to a method.
The code will fail to compile if any calls from the method to itself are not in tail position.

```scala mdoc:reset-object:fail
import scala.annotation.tailrec

@tailrec
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }
```

```scala mdoc:invisible
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

We can check the tail recursive version is truly tail recursive by passing it a very large input.
The non-tail recursive version crashes.

```scala
isntTailRecursive(100000)
// java.lang.StackOverflowError
```

The tail recursive version runs just fine.

```scala mdoc
isTailRecursive(100000)
```


### Continuation-Passing Style

Now that we know about tail calls, how do we convert the regular expression interpreter to use them? Any program can be converted to an equivalent program with all calls in tail position. This conversion is known as **continuation-passing style** or CPS for short. Our first step to understanding CPS is to understand **continuations**.

A continuation is an encapsulation of "what happens next". Let's return to our `Regexp` example. Here's the full code for reference.

```scala mdoc:silent
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int): Option[Int] =
      regexp match {
        case Append(left, right) =>
          loop(left, idx).flatMap(i => loop(right, i))
        case OrElse(first, second) =>
          loop(first, idx).orElse(loop(second, idx))
        case Repeat(source) =>
          loop(source, idx)
            .flatMap(i => loop(regexp, i))
            .orElse(Some(idx))
        case Apply(string) =>
          Option.when(input.startsWith(string, idx))(idx + string.size)
        case Empty =>
          None
      }

    // Check we matched the entire input
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  def apply(string: String): Regexp =
    Apply(string)
}
```

Let's consider the case for `Append` in `matches`.

```scala
case Append(left, right) =>
  loop(left, idx).flatMap(i => loop(right, i))
```

What happens next when we call `loop(left, idx)`? Let's give the name `result` to the value returned by the call to `loop`. The answer is we run `result.flatMap(i => loop(right, i))`. We can represent this as a function, to which we pass `result`:

```scala
(result: Option[Int]) => result.flatMap(i => loop(right, i))
```

This is exactly the continuation, reified as a value. 

As is often the case, there is a distinction between the concept and the representation. 
The concept of continuations always exists in code.
A continuation means "what happens next". 
In other words, it is the program's control flow.
There is always some concept of control flow, even if it is just "the program halts". 
We can represent continuations as functions in code.
This transforms the abstract concept of continuations into concrete values in our program, and hence reifies them.

Now that we know about continuations, and their reification as functions, we can move on to continuation-passing style.
In CPS we, as the name suggests, pass around continuations.
Specifically, each function or method takes an extra parameter that is a continuation.
Instead of returning a value it calls that continuation with the value.
This is another example of duality, in this case between returning a value and calling a continuation.

Let's see how this works.
We'll start with a simple example written in the normal style, also known as **direct style**.

```scala mdoc
(1 + 2) * 3
```

To rewrite this in CPS style we need to create replacements for `+` and `*` with the extra continuation parameter.

```scala mdoc:silent
type Continuation = Int => Int

def add(x: Int, y: Int, k: Continuation) = k(x + y)
def mul(x: Int, y: Int, k: Continuation) = k(x * y)
```

Now we can rewrite our example in CPS. `(1 + 2)` becomes `add(1, 2, k)`, but what is `k`, the continuation?
What we do next is multiply the result by `3`. Thus the continuation is `a => mul(a, 3, k2)`. 
What is the next continuation, `k2`?
Here the program finishes, so we just return the value with the identity continuation `b => b`.
Put it all together and we get

```scala mdoc
add(1, 2, a => mul(a, 3, b => b))
```

Notice that every continuation call is in tail position in the CPS code.
This means that code written in CPS can potentially consume no stack space.

Now we can return to the interpreter loop for `Regexp`.
We are going to CPS it, so we need to add an extra parameter for the continuation.
In this case the contination accepts and returns the result type of `loop`: `Option[Int]`.

```scala
def matches(input: String): Boolean = {
  // Define a type alias so we can easily write continuations
  type Continuation = Option[Int] => Option[Int]

  def loop(regexp: Regexp, idx: Int, cont: Continuation): Option[Int] =
  // etc...
}
```

Now we go through each case and convert it to CPS. Each continuation we construct must call `cont` as its final step.
This is tedious and a bit error-prone, so good tests are helpful.

```scala
def matches(input: String): Boolean = {
  // Define a type alias so we can easily write continuations
  type Continuation = Option[Int] => Option[Int]

  def loop(
      regexp: Regexp,
      idx: Int,
      cont: Continuation
  ): Option[Int] =
    regexp match {
      case Append(left, right) =>
        val k: Continuation = _ match {
          case None    => cont(None)
          case Some(i) => loop(right, i, cont)
        }
        loop(left, idx, k)

      case OrElse(first, second) =>
        val k: Continuation = _ match {
          case None => loop(second, idx, cont)
          case some => cont(some)
        }
        loop(first, idx, k)

      case Repeat(source) =>
        val k: Continuation =
          _ match {
            case None    => cont(Some(idx))
            case Some(i) => loop(regexp, i, cont)
          }
        loop(source, idx, k)

      case Apply(string) =>
        cont(Option.when(input.startsWith(string, idx))(idx + string.size))
        
      case Empty =>
        cont(None)
    }

  // Check we matched the entire input
  loop(this, 0, identity).map(idx => idx == input.size).getOrElse(false)
}
```

Every call in this interpreter loop is in tail position. However Scala cannot convert these to tail calls because the calls go from `loop` to a continuation and vice versa. To make the interpreter fully stack safe we need to add **trampolining**. 


#### Exercise: CPS Arithmetic {-}

In a previous exercise we wrote an interpreter for arithmetic expressions. Your task now is to CPS this interpreter.
For reference, the definition of an arithmetic expression is:

- a literal number, which takes a `Double` and produces an `Expression`;
- an addition of two expressions;
- a substraction of two expressions;
- a multiplication of two expressions; or
- a division of two expressions;

<div class="solution">
The continuations have a slightly different structure to the regular expression example.
In the regular expression example, all the information needs by a continuation is either found in the parameter to the continuation (the index) or values extracted via pattern matching.
In the arithmetic code we need values from previous continuations that are not passed as parameters.
This is to compute binary operations like additions.
The solution is to capture these values within the environment of the closure that represents the continuation.

```scala mdoc:reset:silent
type Continuation = Double => Double

enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)

  def eval: Double = {
    def loop(expr: Expression, cont: Continuation): Double =
      expr match {
        case Literal(value) => cont(value)
        case Addition(left, right) =>
          loop(left, l => loop(right, r => cont(l + r)))
        case Subtraction(left, right) =>
          loop(left, l => loop(right, r => cont(l - r)))
        case Multiplication(left, right) =>
          loop(left, l => loop(right, r => cont(l * r)))
        case Division(left, right) =>
          loop(left, l => loop(right, r => cont(l / r)))
      }

    loop(this, identity)
  }
  
  def +(that: Expression): Expression =
    Addition(this, that)

  def -(that: Expression): Expression =
    Subtraction(this, that)

  def *(that: Expression): Expression =
    Multiplication(this, that)

  def /(that: Expression): Expression =
    Division(this, that)
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>


### Trampolining

Earlier we said that CPS utilizes the duality between function calls and returns: instead of returning a value we call a function with a value. This allows us to transform our code so it only has calls in tail positions. However, we still have a problem with stack safety. Scala's runtimes don't support full tail calls, so calls from a continuation to `loop` or from `loop` to a continuation will use a stack frame. We can use this same duality to avoid using the stack by, instead of making a call, returning a value that reifies the call we want to make. This idea is the core of trampolining. Let's see it in action, which will help clear up what exactly this all means.

Our first step is to reify all the method calls made by the interpreter loop and the continuations.
There are three cases: calls to `loop`, calls to a continuation, and, to avoid an infinite loop, the case when we're done.

```scala
type Continuation = Option[Int] => Call

enum Call {
  case Loop(regexp: Regexp, index: Int, continuation: Continuation)
  case Continue(index: Option[Int], continuation: Continuation)
  case Done(index: Option[Int])
}
```

Now we update `loop` to return instances of `Call` instead of making the calls directly.

```scala
def loop(regexp: Regexp, idx: Int, cont: Continuation): Call =
  regexp match {
    case Append(left, right) =>
      val k: Continuation = _ match {
        case None    => Call.Continue(None, cont)
        case Some(i) => Call.Loop(right, i, cont)
      }
      Call.Loop(left, idx, k)

    case OrElse(first, second) =>
      val k: Continuation = _ match {
        case None => Call.Loop(second, idx, cont)
        case some => Call.Continue(some, cont)
      }
      Call.Loop(first, idx, k)

    case Repeat(source) =>
      val k: Continuation =
        _ match {
          case None    => Call.Continue(Some(idx), cont)
          case Some(i) => Call.Loop(regexp, i, cont)
        }
      Call.Loop(source, idx, k)

    case Apply(string) =>
      Call.Continue(
        Option.when(input.startsWith(string, idx))(idx + string.size),
        cont
      )

    case Empty =>
      Call.Continue(None, cont)
  }
```

This gives us an interpreter loop that returns values instead of making calls, and so does not consume stack space.
However, we need to actually make these calls at some point, and doing this is the job of the trampoline.
The trampoline is simply a tail recursive loop that makes calls until it reaches `Done`.

```scala
def trampoline(next: Call): Option[Int] =
  next match {
    case Call.Loop(regexp, index, continuation) =>
      trampoline(loop(regexp, index, continuation))
    case Call.Continue(index, continuation) =>
      trampoline(continuation(index))
    case Call.Done(index) => index
  }
```

Now every call has a corresponding return, so the stack usage is limited. 
Our interpreter can handle input of any size, up to the limits of available memory.

Here's the complete code for reference.

```scala mdoc:reset:silent
// Define a type alias so we can easily write continuations
type Continuation = Option[Int] => Call

enum Call {
  case Loop(regexp: Regexp, index: Int, continuation: Continuation)
  case Continue(index: Option[Int], continuation: Continuation)
  case Done(index: Option[Int])
}

enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int, cont: Continuation): Call =
      regexp match {
        case Append(left, right) =>
          val k: Continuation = _ match {
            case None    => Call.Continue(None, cont)
            case Some(i) => Call.Loop(right, i, cont)
          }
          Call.Loop(left, idx, k)

        case OrElse(first, second) =>
          val k: Continuation = _ match {
            case None => Call.Loop(second, idx, cont)
            case some => Call.Continue(some, cont)
          }
          Call.Loop(first, idx, k)

        case Repeat(source) =>
          val k: Continuation =
            _ match {
              case None    => Call.Continue(Some(idx), cont)
              case Some(i) => Call.Loop(regexp, i, cont)
            }
          Call.Loop(source, idx, k)

        case Apply(string) =>
          Call.Continue(
            Option.when(input.startsWith(string, idx))(idx + string.size),
            cont
          )

        case Empty =>
          Call.Continue(None, cont)
      }

    def trampoline(next: Call): Option[Int] =
      next match {
        case Call.Loop(regexp, index, continuation) =>
          trampoline(loop(regexp, index, continuation))
        case Call.Continue(index, continuation) =>
          trampoline(continuation(index))
        case Call.Done(index) => index
      }

    // Check we matched the entire input
    trampoline(loop(this, 0, opt => Call.Done(opt)))
      .map(idx => idx == input.size)
      .getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  def apply(string: String): Regexp =
    Apply(string)
}
```


#### Exericse: Trampolined Arithmetic {-}

Convert the CPSed arithmetic interpreter we wrote earlier to a trampolined version.

<div class="solution">
The process to produce this code is very similar to the regular expression example.
We just identify all the different types of calls (which are the same as the regular expression example) and reify them.

```scala mdoc:reset:silent
type Continuation = Double => Call

enum Call {
  case Continue(value: Double, k: Continuation)
  case Loop(expr: Expression, k: Continuation)
  case Done(result: Double)
}

enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)

  def eval: Double = {
    def loop(expr: Expression, cont: Continuation): Call =
      expr match {
        case Literal(value) => Call.Continue(value, cont)
        case Addition(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l + r, cont))
          )
        case Subtraction(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l - r, cont))
          )
        case Multiplication(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l * r, cont))
          )
        case Division(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l / r, cont))
          )
      }

    def trampoline(call: Call): Double =
      call match {
        case Call.Continue(value, k) => trampoline(k(value))
        case Call.Loop(expr, k)      => trampoline(loop(expr, k))
        case Call.Done(result)       => result
      }

    trampoline(loop(this, x => Call.Done(x)))
  }

  def +(that: Expression): Expression =
    Addition(this, that)

  def -(that: Expression): Expression =
    Subtraction(this, that)

  def *(that: Expression): Expression =
    Multiplication(this, that)

  def /(that: Expression): Expression =
    Division(this, that)
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>


### When Tail Recursion is Easy

Doing a full CPS conversion and trampoline can be quite involved. Some methods can made tail recursive without so large a change.
Remember these examples we looked at earlier?

```scala mdoc:reset-object:silent
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

The tail recursive version doesn't seem to involve the complexity of CPS. How can we relate this to what we've just learned, and when can we avoid the work of CPS and trampolining?

Let's use substitution to show how the stack is used by each method, for a small value of `count`.

```scala
isntTailRecursive(2)
// expands to
(2 match {
  case 0 => 0
  case n => n + isntTailRecursive(n - 1)
})
// expands to
(2 + isntTailRecursive(1))
// expands to
(2 + (1 match {
        case 0 => 0
        case n => n + isntTailRecursive(n - 1)
      }))
// expands to
(2 + (1 + isntTailRecursive(n - 1)))
// expands to
(2 + (1 + (0 match {
             case 0 => 0
             case n => n + isntTailRecursive(n - 1)
           })))
// expands to
(2 + (1 + (0)))
// expands to
3
```

Here each set of brackets indicates a new method call and hence a stack frame allocation.

Now let's do the same for `isTailRecursive`.

```scala
isTailRecursive(2)
// expands to
(loop(2, 0))
// expands to
(2 match {
   case 0 => 0
   case n => loop(n - 1, 0 + n)
 })
// expands to
(loop(1, 2))
// call to loop is a tail call, so no stack frame is allocated 
// expands to
(1 match {
   case 0 => 2
   case n => loop(n - 1, 2 + n)
 })
// expands to
(loop(0, 3))
// call to loop is a tail call, so no stack frame is allocated 
// expands to
(0 match {
   case 0 => 3
   case n => loop(n - 1, 3 + n)
 })
// expands to
(3)
// expands to
3
```

The non-tail recursive function computes the result `(2 + (1 + (0)))`
If we look closely, we'll see that the tail recursive version computes `(((2) + 1) + 0)`, which simply accumulates the result in the reverse order.
This works because addition is associative, meaning `(a + b) + c == a + (b + c)`.
This is our first criteria for using the "easy" method for converting to a tail recursive form: the operation that accumulates results must be associative.

This doesn't explain, though, how we come to realize that addition is the correct operation to use. The second criteria is that we don't need any memory beyond the partial result calculated from the data we've already seen. Some implications of this are that we can stop at any time and have a usable result, and that we are only applying a single operation to the data. This is not the case in the regular expression example. For example, we have the following code in the `Append` case:

```scala
case Append(left, right) =>
  loop(left, idx).flatMap(i => loop(right, i))
```

To compute the result for the `Append` we need to compute and combine results from both `left` and `right`. So when we have computed the result for `right` we need to remember both the result from `left` and that we're combining the two results using the rule for `Append` rather than, say, `OrElse`. It's remembering this that is exactly what the continuation does, and what stops us from using the easy method we saw when summing the elements of a list.

So, in summary, if we are applying only a single associative operation to data we can use the simple method for writing a tail recursive method:

1. define an structurally recursive loop with an additional parameter that is the partial result or accumulator;
2. in the base cases return the accumulator; and
3. in the recursive cases update the accumulator and call the loop in tail position.

You might be wondering how we handle tree-shaped data with this technique. One consequence of an associative operation is that we can transform any sequence of operations into a list-shaped sequence. If, for example, we have an expression tree that suggests we should call operations in the order `(1 + 2) + (3 + 4)` (where I'm using `+` to indicate the operation) we can rewrite that to `(((1 + 2) + 3) + 4)` via associativity. So we can transform our tree into a list and then apply the recipe above.


```scala mdoc:reset:silent
```
--->

## 末尾再帰インタープリタ

構造的再帰は、これまでにも述べたとおりスタックを使用する。これが問題になることはあまりないが、特に深い再帰ではスタックオーバーフローを引き起こす可能性がある。ひとつの解決策は**末尾再帰**のプログラムを書くことである。末尾再帰のプログラムはスタック領域を使用しないので、**スタックセーフ**であると表現されることもある。どのようなプログラムでも末尾再帰の形に変換することができる。

<div class="callout callout-info">
#### コールスタック {-}  

通常、メソッドや関数の呼び出しは**コールスタック**または単に**スタック**と呼ばれるメモリ領域を使用して実装される。メソッドや関数の呼び出しごとに**スタックフレーム**と呼ばれる小さなメモリ領域がスタック上に確保される。メソッドや関数がリターンすると、このメモリは解放され、次の呼び出しで再利用可能になる。  

しかし、多数のメソッド呼び出しがリターンすることなく積み重なっていくと、スタックが確保できる以上のフレームを要求されることがある。スタック上に利用可能なメモリがなくなった状態は、スタックをオーバーフローした、と表現される。Scala では、このような状況になると `StackOverflowError` が発生する。
</div>

この節では、末尾再帰、プログラムを末尾再帰の形に変換する方法、そして Scala ランタイムにおける制約とその回避策について議論する。

### スタックの安全性に関する問題

まずは、どのような問題があるのかを見てみよう。Scala には、同じ文字列を繰り返し並べた新しい文字列を作る `*` メソッドがある。これを用いて `String` オブジェクトを作成する。

```scala mdoc
"a" * 4
```

```scala mdoc:invisible
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int): Option[Int] =
      regexp match {
        case Append(left, right) =>
          loop(left, idx).flatMap(i => loop(right, i))
        case OrElse(first, second) =>
          loop(first, idx).orElse(loop(second, idx))
        case Repeat(source) =>
          loop(source, idx)
            .flatMap(i => loop(regexp, i))
            .orElse(Some(idx))
        case Apply(string) =>
          Option.when(input.startsWith(string, idx))(idx + string.size)
        case Empty =>
          None
      }

    // 入力全体がマッチしたかどうかを確認する
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  def apply(string: String): Regexp =
    Apply(string)
}
```

`Regexp` の `repeat` メソッドを使えば、そのような文字列にマッチする正規表現を作ることができる。

```scala mdoc
Regexp("a").repeat.matches("a" * 4)
```

だが、入力が極端に長くなると、インタープリタはスタックオーバーフローを発生させて失敗する。

```scala
Regexp("a").repeat.matches("a" * 20000)
// java.lang.StackOverflowError
```

こうなるのは、繰り返しの正規表現が入力にひとつマッチするたびに、インタープリタがリターンをはさむことなく `loop` を呼び出すからである。しかし、解決策がないわけではない。使用するスタック領域を一定に保つようにインタープリタを書き換えれば、どのような大きさの入力にも対応できるようになる。

### 末尾呼び出しと末尾位置

出発点となるのは **末尾呼び出し（tail call）** である。末尾呼び出しは、新たなスタック領域を消費しないメソッド呼び出しである。**末尾位置（tail position）** にあるメソッド呼び出しのみが、末尾呼び出しに変換される候補となる。ただし、ランタイムの制約により、末尾位置にあるすべての呼び出しが末尾呼び出しに変換されるとは限らない。[^tn-adt-interpreters-tail-recursion-01]

[^tn-adt-interpreters-tail-recursion-01] 【訳注】 「末尾呼び出し」は、単に「末尾位置にある関数呼び出し」を指して使われることが多いと思うが、本書では異なる意味で使われているようである。一般的に、末尾呼び出しは、新しいスタック領域を消費しないようにコンパイラやランタイムを実装することが理屈の上ではできる。末尾呼び出しを行う側の関数が用いるスタックフレームは、末尾呼び出し時点ではもはや用済みであり、再利用できるからである。Scala の場合、末尾位置にある自己再帰呼び出しについてのみ、コンパイルによってループ相当の JVM バイトコードへと変換し、スタックセーフを実現する。本書における「末尾呼び出しに変換される」という表現からは、「末尾呼び出し」が「コンパイラやランタイムによってスタックフレームを消費しない形に最適化された関数呼び出し」のことを指しているように思われる。

あるメソッド呼び出しの戻り値が、そのままその呼び出し元の戻り値となる場合、その呼び出しは末尾位置にあるという。例を見てみよう。0から `count` までの整数を合計するメソッドについて二通りの実装を以下に示す。

```scala mdoc:silent
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

以下の `isntTailRecursive` 呼び出しは、戻り値が加算の中で用いられているため、末尾位置にない。

```scala
case n => n + isntTailRecursive(n - 1)
```

一方、以下の `loop` 呼び出しは、戻り値が即座にひとつ前の `loop` 呼び出しの戻り値として返却されるので、末尾位置にあると言える。

```scala
case n => loop(n - 1, accum + n)
```

同様に、以下の `loop` 呼び出しも末尾位置にある。

```scala
loop(count, 0)
```

末尾位置にあるメソッド呼び出しは末尾呼び出し最適化の候補である。Scala ランタイムの制約により、末尾位置にある呼び出しがすべて最適化されるわけではない。現在のところ、あるメソッドが末尾位置でそのメソッド自身を呼び出す場合のみ、それは最適化の対象となる。

先ほどの `isTailRecursive` の例で言えば、以下の部分には末尾呼び出し最適化が適用される。

```scala
case n => loop(n - 1, accum + n)
```

だが、以下の部分は `isTailRecursive` から `loop` を呼び出しているので、最適化されない。

```scala
loop(count, 0)
```

とはいえ、 これは一回の `isTailRecursive` につき一度しか発生しない呼び出しなので、スタック消費の観点で問題を起こすことはない。

<div class="callout callout-info">
#### ランタイムと末尾呼び出し {-}

Scala は、JVM 、Scala.js を介した JavaScript、そして Scala Native を介したネイティブコードという三つの異なるプラットフォームをサポートし、プラットフォーム毎にランタイムを提供している。ランタイムとは、Scala コードの実行時にそれを支えるコードのことをいう。たとえばガベージコレクタはランタイムの一部である。  

執筆時点では、Scala のどのランタイムも完全な末尾呼び出し最適化をサポートしていない。しかし、将来的には状況が変わる可能性がある。JVM においては、[Project Loom](https://wiki.openjdk.org/display/loom/Main) によってようやく末尾呼び出し最適化がサポートされる見込みである。Scala Native では、継続の実装作業の一環として、近いうちにサポートされると考えられる。JavaScript においては、末尾呼び出し最適化はずっと以前から仕様に含まれているが、大半の JavaScript ランタイムはこれを実装していない。一方で、WebAssembly は末尾呼び出し最適化をサポートしている。中期的には Scala から JavaScript へのコンパイルは、WebAssembly へのコンパイルへと取って代わられるだろう。
</div>

メソッドに `@tailrec` アノテーションを付けることで、自己呼び出しがすべて末尾位置にあることを Scala コンパイラに確認させることができる。このアノテーションを付けたメソッド内で自己呼び出しが末尾位置にない場合、コンパイルエラーとなる。

```scala mdoc:reset-object:fail
import scala.annotation.tailrec

@tailrec
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }
```

```scala mdoc:invisible
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

末尾再帰を使って書かれているコードが確かに末尾再帰になっているかは、大きな入力値を渡すことで確かめることができる。末尾再帰でないほうのコードはクラッシュする。

```scala
isntTailRecursive(100000)
// java.lang.StackOverflowError
```

末尾再帰版は正しく動作する。

```scala mdoc
isTailRecursive(100000)
```

### 継続渡しスタイル

これで末尾呼び出し最適化については把握できたが、これを利用する形に正規表現インタープリタを変換するにはどうすればよいだろうか。どのようなプログラムでも、**継続渡しスタイル**略して CPS と呼ばれる形に変換することで、すべての関数呼び出しを末尾位置にもってきた等価なプログラムを得ることができる。まずは**継続**について知ることが、CPS を理解するための最初のステップとなる。

継続とは「次に何が起きるのか」をカプセル化したものである。`Regexp` の例に戻ろう。以下にそのコード全体を再掲する。

```scala mdoc:silent
enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int): Option[Int] =
      regexp match {
        case Append(left, right) =>
          loop(left, idx).flatMap(i => loop(right, i))
        case OrElse(first, second) =>
          loop(first, idx).orElse(loop(second, idx))
        case Repeat(source) =>
          loop(source, idx)
            .flatMap(i => loop(regexp, i))
            .orElse(Some(idx))
        case Apply(string) =>
          Option.when(input.startsWith(string, idx))(idx + string.size)
        case Empty =>
          None
      }

    // 入力全体にマッチするかどうかチェック
    loop(this, 0).map(idx => idx == input.size).getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  def apply(string: String): Regexp =
    Apply(string)
}
```

`matches` メソッドにおける `Append` のケースについて考えてみよう。

```scala
case Append(left, right) =>
  loop(left, idx).flatMap(i => loop(right, i))
```

`loop(left, idx)` 呼び出しに続いて起きることは何だろうか。`loop` の戻り値が `result` に格納されたとすると、実行されるのは `result.flatMap(i => loop(right, i))` である。この処理は `result` を受け取る関数として次のように表現することができる。

```scala
(result: Option[Int]) => result.flatMap(i => loop(right, i))
```

これがまさに、データとして表現された継続である。

概念とその表現は往々にして異なるものである。継続という概念はコードの中に常に存在している。継続とは「次に何が起こるか」を指すものであり、言い換えればプログラムの制御フローそのものである。たとえあるプログラムがただ停止するだけであったとしても、そこには常に制御フローの概念が存在している。継続はコードの中で関数として表現することができる。これにより、継続という抽象的な概念はプログラム内の具体的な値へと変換、すなわちレイフィケーションされる。

継続およびそれが関数としてレイフィケーションされることについて理解したところで、次は継続渡しスタイル（CPS）の話に進もう。CPS では、その名前が示すように、継続を引数として受け渡す。具体的に言えば、各関数やメソッドは継続を追加のパラメータとして受け取り、値を返す代わりに、その値を渡して継続を呼び出す。これもまた双対性の一例で、今回のケースでは値を返すことと継続を呼び出すことが双対の関係にある。

Let's see how this works.
We'll start with a simple example written in the normal style, also known as **direct style**.

CPS の仕組みを例で見てみよう。まずは通常のスタイルで書かれたシンプルな例からスタートする。このようなスタイルは**直接スタイル**と呼ばれる。

```scala mdoc
(1 + 2) * 3
```

これを CPS で書き換えるためには、 `+` と `*` にパラメータとして継続を付け加えた関数を作成する必要がある。

```scala mdoc:silent
type Continuation = Int => Int

def add(x: Int, y: Int, k: Continuation) = k(x + y)
def mul(x: Int, y: Int, k: Continuation) = k(x * y)
```

これで先ほどの式を CPS に書き換えることができる。`(1 + 2)` は `add(1, 2, k)` となり、次にすべきことは加算の結果に `3` を掛けることなので、継続 `k` は `a => mul(a, 3, k2)` となる。次の継続 `k2` はどんな処理だろうか。プログラムはここで終了するので、恒等継続 `b => b` を用いてただ値を返せばよい。以上をまとめると次のプログラムが得られる。

```scala mdoc
add(1, 2, a => mul(a, 3, b => b))
```

CPS のコードでは継続の呼び出しがすべて末尾位置にあることに注目してほしい。つまり CPS で書かれたコードはスタック領域を消費せずに実行できる可能性をもっているのである。

`Regexp` におけるインタープリタループの話に戻ろう。これを CPS に書き換えるために、継続を受け取るパラメータを追加する。今回の場合、継続の引数と戻り値の型はどちらも、`loop` の結果型である `Option[Int]` となる。

```scala
def matches(input: String): Boolean = {
  // 継続を書きやすくするために型エイリアスを定義する
  type Continuation = Option[Int] => Option[Int]

  def loop(regexp: Regexp, idx: Int, cont: Continuation): Option[Int] =
  // etc...
}
```

続いて、各ケースを CPS へと書き換えていく。構築する継続は必ずその最終ステップとして `cont` を呼び出さなくてはならない。この変換作業は面倒でミスが発生しやすいため、適切なテストを用意することが重要である。

```scala
def matches(input: String): Boolean = {
  // 継続を書きやすくするために型エイリアスを定義する
  type Continuation = Option[Int] => Option[Int]

  def loop(
      regexp: Regexp,
      idx: Int,
      cont: Continuation
  ): Option[Int] =
    regexp match {
      case Append(left, right) =>
        val k: Continuation = _ match {
          case None    => cont(None)
          case Some(i) => loop(right, i, cont)
        }
        loop(left, idx, k)

      case OrElse(first, second) =>
        val k: Continuation = _ match {
          case None => loop(second, idx, cont)
          case some => cont(some)
        }
        loop(first, idx, k)

      case Repeat(source) =>
        val k: Continuation =
          _ match {
            case None    => cont(Some(idx))
            case Some(i) => loop(regexp, i, cont)
          }
        loop(source, idx, k)

      case Apply(string) =>
        cont(Option.when(input.startsWith(string, idx))(idx + string.size))
        
      case Empty =>
        cont(None)
    }

  // 入力全体にマッチするかどうかチェック
  loop(this, 0, identity).map(idx => idx == input.size).getOrElse(false)
}
```

Every call in this interpreter loop is in tail position. However Scala cannot convert these to tail calls because the calls go from `loop` to a continuation and vice versa. To make the interpreter fully stack safe we need to add **trampolining**. 

このインタープリタでは `loop` 呼び出しはすべて末尾位置に置かれている。だが、`loop` から継続、そして継続から `loop` という相互呼び出しの形になっているため、Scala はこれらに対して末尾呼び出し最適化を行うことができない。このインタープリタを完全にスタックセーフにするには**トランポリン化**を行う必要がある。

#### 演習: CPS の算術式インタープリタ {-}

前回の演習で作成した算術式のインタープリタを CPS へと書き換えよ。参考までに算術式の定義を再掲する。

- 数値リテラル。`Double` を受け取って `Expression` を生成する
- ふたつの式の加算
- ふたつの式の減算
- ふたつの式の乗算
- ふたつの式の除算

<div class="solution">
この課題における継続の構造は、正規表現の例とはすこし異なる。正規表現の例では、継続が必要とする情報はすべて、継続のパラメータまたはパターンマッチによって抽出された値に含まれていた。一方、算術計算のコードで加算のような二項演算を行うには、ひとつ前の継続がもっている値をパラメータ以外の方法で受け取る必要がある。これを解決する方法は、継続を表すクロージャの環境内にそれらの値をキャプチャすることである。

```scala mdoc:reset:silent
type Continuation = Double => Double

enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)

  def eval: Double = {
    def loop(expr: Expression, cont: Continuation): Double =
      expr match {
        case Literal(value) => cont(value)
        case Addition(left, right) =>
          loop(left, l => loop(right, r => cont(l + r)))
        case Subtraction(left, right) =>
          loop(left, l => loop(right, r => cont(l - r)))
        case Multiplication(left, right) =>
          loop(left, l => loop(right, r => cont(l * r)))
        case Division(left, right) =>
          loop(left, l => loop(right, r => cont(l / r)))
      }

    loop(this, identity)
  }
  
  def +(that: Expression): Expression =
    Addition(this, that)

  def -(that: Expression): Expression =
    Subtraction(this, that)

  def *(that: Expression): Expression =
    Multiplication(this, that)

  def /(that: Expression): Expression =
    Division(this, that)
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>


### トランポリン化

先ほど述べたとおり、CPS は関数の呼び出しとリターンの双対性を利用し、値を返す代わりにその値を関数に渡して呼び出している。これにより、コードはすべての再帰呼び出しが末尾位置に来る形へと変換される。しかし、これでスタックセーフに関する課題が解決したわけではない。Scala ランタイムは末尾呼び出し最適化を完全にはサポートしておらず、継続からの `loop` 呼び出しや `loop` からの継続呼び出しにはスタックフレームが消費される。

この問題を解決するには、再び双対性を利用し、呼び出しを行う代わりに、行いたい呼び出しを表現する値を返せばよい。これがトランポリン化の基本的なアイデアである。具体的にどのように動作するのか見てみよう。そうすれば、ここで言わんとしていることが明確になるだろう。

最初のステップは、インタープリタループや継続によって行われるすべてのメソッド呼び出しをレイフィケーションすることである。ここで扱うケースは三つある。

- `loop` 呼び出し
- 継続の呼び出し
- 無限ループを防ぐための処理の完了

```scala
type Continuation = Option[Int] => Call

enum Call {
  case Loop(regexp: Regexp, index: Int, continuation: Continuation)
  case Continue(index: Option[Int], continuation: Continuation)
  case Done(index: Option[Int])
}
```

次に、`loop` が呼び出しを直接行う代わりに `Call` インスタンスを返すよう変更する。

```scala
def loop(regexp: Regexp, idx: Int, cont: Continuation): Call =
  regexp match {
    case Append(left, right) =>
      val k: Continuation = _ match {
        case None    => Call.Continue(None, cont)
        case Some(i) => Call.Loop(right, i, cont)
      }
      Call.Loop(left, idx, k)

    case OrElse(first, second) =>
      val k: Continuation = _ match {
        case None => Call.Loop(second, idx, cont)
        case some => Call.Continue(some, cont)
      }
      Call.Loop(first, idx, k)

    case Repeat(source) =>
      val k: Continuation =
        _ match {
          case None    => Call.Continue(Some(idx), cont)
          case Some(i) => Call.Loop(regexp, i, cont)
        }
      Call.Loop(source, idx, k)

    case Apply(string) =>
      Call.Continue(
        Option.when(input.startsWith(string, idx))(idx + string.size),
        cont
      )

    case Empty =>
      Call.Continue(None, cont)
  }
```

これでインタープリタループは呼び出しを行う代わりに値を返す形となり、スタック領域を消費しない。しかし、これらの呼び出しはどこかの時点で実際に実行されなければならない。それをするのがトランポリンの役割である。トランポリンは、`Done` に到達するまで繰り返すシンプルな末尾再帰処理である。

```scala
def trampoline(next: Call): Option[Int] =
  next match {
    case Call.Loop(regexp, index, continuation) =>
      trampoline(loop(regexp, index, continuation))
    case Call.Continue(index, continuation) =>
      trampoline(continuation(index))
    case Call.Done(index) => index
  }
```

ループおよび継続の呼び出しはすべて即座にリターンするので、スタックの消費量は限定される。このインタープリタはメモリが許すかぎりどんなに大きなサイズの入力でも処理することができる。

以下に完全なコードを示す。

```scala mdoc:reset:silent
// 継続を書きやすくするために型エイリアスを定義する
type Continuation = Option[Int] => Call

enum Call {
  case Loop(regexp: Regexp, index: Int, continuation: Continuation)
  case Continue(index: Option[Int], continuation: Continuation)
  case Done(index: Option[Int])
}

enum Regexp {
  def ++(that: Regexp): Regexp =
    Append(this, that)

  def orElse(that: Regexp): Regexp =
    OrElse(this, that)

  def repeat: Regexp =
    Repeat(this)

  def `*` : Regexp = this.repeat

  def matches(input: String): Boolean = {
    def loop(regexp: Regexp, idx: Int, cont: Continuation): Call =
      regexp match {
        case Append(left, right) =>
          val k: Continuation = _ match {
            case None    => Call.Continue(None, cont)
            case Some(i) => Call.Loop(right, i, cont)
          }
          Call.Loop(left, idx, k)

        case OrElse(first, second) =>
          val k: Continuation = _ match {
            case None => Call.Loop(second, idx, cont)
            case some => Call.Continue(some, cont)
          }
          Call.Loop(first, idx, k)

        case Repeat(source) =>
          val k: Continuation =
            _ match {
              case None    => Call.Continue(Some(idx), cont)
              case Some(i) => Call.Loop(regexp, i, cont)
            }
          Call.Loop(source, idx, k)

        case Apply(string) =>
          Call.Continue(
            Option.when(input.startsWith(string, idx))(idx + string.size),
            cont
          )

        case Empty =>
          Call.Continue(None, cont)
      }

    def trampoline(next: Call): Option[Int] =
      next match {
        case Call.Loop(regexp, index, continuation) =>
          trampoline(loop(regexp, index, continuation))
        case Call.Continue(index, continuation) =>
          trampoline(continuation(index))
        case Call.Done(index) => index
      }

    // 入力全体にマッチするかどうかチェック
    trampoline(loop(this, 0, opt => Call.Done(opt)))
      .map(idx => idx == input.size)
      .getOrElse(false)
  }

  case Append(left: Regexp, right: Regexp)
  case OrElse(first: Regexp, second: Regexp)
  case Repeat(source: Regexp)
  case Apply(string: String)
  case Empty
}
object Regexp {
  val empty: Regexp = Empty

  def apply(string: String): Regexp =
    Apply(string)
}
```


#### 演習: 算術式インタープリタのトランポリン化 {-}

CPS で記述された算術式インタープリタをトランポリン化せよ。

<div class="solution">
この解答コードを生み出すプロセスは正規表現の例とだいたい同じである。異なる種類の呼び出しをすべて特定し、それらをレイフィケーションすればよい。呼び出しの種類は正規表現の例と変わらない。

```scala mdoc:reset:silent
type Continuation = Double => Call

enum Call {
  case Continue(value: Double, k: Continuation)
  case Loop(expr: Expression, k: Continuation)
  case Done(result: Double)
}

enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)

  def eval: Double = {
    def loop(expr: Expression, cont: Continuation): Call =
      expr match {
        case Literal(value) => Call.Continue(value, cont)
        case Addition(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l + r, cont))
          )
        case Subtraction(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l - r, cont))
          )
        case Multiplication(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l * r, cont))
          )
        case Division(left, right) =>
          Call.Loop(
            left,
            l => Call.Loop(right, r => Call.Continue(l / r, cont))
          )
      }

    def trampoline(call: Call): Double =
      call match {
        case Call.Continue(value, k) => trampoline(k(value))
        case Call.Loop(expr, k)      => trampoline(loop(expr, k))
        case Call.Done(result)       => result
      }

    trampoline(loop(this, x => Call.Done(x)))
  }

  def +(that: Expression): Expression =
    Addition(this, that)

  def -(that: Expression): Expression =
    Subtraction(this, that)

  def *(that: Expression): Expression =
    Multiplication(this, that)

  def /(that: Expression): Expression =
    Division(this, that)
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>

### 末尾再帰が容易なケース

完全な CPS への変換やトランポリン化にはかなり手間がかかることがある。しかし、それほど大きな変更を加えずに末尾再帰にすることができる場合もある。先ほど見た以下の例を思い出そう。

```scala mdoc:reset-object:silent
def isntTailRecursive(count: Int): Int =
  count match {
    case 0 => 0
    case n => n + isntTailRecursive(n - 1)
  }

def isTailRecursive(count: Int): Int = {
  def loop(count: Int, accum: Int): Int =
    count match {
      case 0 => accum
      case n => loop(n - 1, accum + n)
    }
    
  loop(count, 0)
}
```

この末尾再帰バージョンは、CPS のような複雑さを伴っていないように見える。これを、これまで学んだ内容に対してどのように位置付ければよいだろうか。また、どのような場合に CPS やトランポリンを用いる手間を省くことができるのだろうか。

それぞれのメソッドがどのようにスタックを使用するのか、小さな `count` 値を例とし、代入（substitution）を使って確認してみよう。

```scala
isntTailRecursive(2)
// expands to
(2 match {
  case 0 => 0
  case n => n + isntTailRecursive(n - 1)
})
// expands to
(2 + isntTailRecursive(1))
// expands to
(2 + (1 match {
        case 0 => 0
        case n => n + isntTailRecursive(n - 1)
      }))
// expands to
(2 + (1 + isntTailRecursive(n - 1)))
// expands to
(2 + (1 + (0 match {
             case 0 => 0
             case n => n + isntTailRecursive(n - 1)
           })))
// expands to
(2 + (1 + (0)))
// expands to
3
```

このコードにおいて、括弧は新しいメソッド呼び出しとそれによるスタックフレームの割り当てを表している。

同じことを `isTailRecursive` でも行ってみよう。

```scala
isTailRecursive(2)
// expands to
(loop(2, 0))
// expands to
(2 match {
   case 0 => 0
   case n => loop(n - 1, 0 + n)
 })
// expands to
(loop(1, 2))
// call to loop is a tail call, so no stack frame is allocated 
// expands to
(1 match {
   case 0 => 2
   case n => loop(n - 1, 2 + n)
 })
// expands to
(loop(0, 3))
// call to loop is a tail call, so no stack frame is allocated 
// expands to
(0 match {
   case 0 => 3
   case n => loop(n - 1, 3 + n)
 })
// expands to
(3)
// expands to
3
```

非末尾再帰の関数は `(2 + (1 + (0)))` という計算を行って結果を得る。よく見ると、末尾再帰バージョンでは `(((2) + 1) + 0)` を計算しており、単に結果を逆順に蓄積しているだけである。これがうまくいくのは、加算が結合律を満たしている、すなわち `(a + b) + c == a + (b + c)` が成り立つからである。結果を蓄積する演算が結合的であること、それが末尾再帰形への変換を簡単な方法で行うための第一の条件である。

第二の条件は、メモリ上に保持する必要のある情報を、計算の途中結果以外にはもたないことである。この条件が意味することはいくつかあるが、たとえば、途中で処理を停止しても利用可能な結果が得られること、各データに計算が適用されるのはそれぞれ一度だけということなどが挙げられる。正規表現の例ではこれが成り立たない。たとえば `Append` のケースを処理する以下のコードを考えてみよう。

```scala
case Append(left, right) =>
  loop(left, idx).flatMap(i => loop(right, i))
```

`Append` の結果を得るためには、`left` と `right` 両方の結果を計算し組み合わせる必要がある。`right` の結果を計算した時点で、`left` の結果も記憶している必要があるし、両方の結果が他でもない `Append` のルールに従って組み合わされるということも覚えていなければならない。この「覚えておく」という行為こそが、まさに継続が果たしている役割であり、そのようなケースでは、リストの要素を合計する際に用いた簡単な方法を適用することはできない。

まとめると、結合的な演算を各データに対して一度だけ適用する場合、以下の方法で簡単に末尾再帰のメソッドを記述できる。

1. 途中結果（蓄積変数）をパラメータとして加えた構造的再帰のループを定義する
2. 基本ケースでは蓄積変数をそのまま返す
3. 再帰ケースでは蓄積変数を更新し、末尾位置でループを呼び出す

この方法をツリー構造のデータに適用する方法が気になるかもしれない。演算は結合的であることが前提なのだから、任意の演算のシーケンスはリスト状のシーケンスに変換することができる。たとえば `(1 + 2) + (3 + 4)` という順序で演算を適用するツリー構造の式は（ここでは演算を表す記号として `+` を用いる）、結合律に基づき `(((1 + 2) + 3) + 4)` という形に書き換えることができる。したがって、ツリーはリストに変換して上記の手順を適用すればよい。

```scala
case Append(left, right) =>
  loop(left, idx).flatMap(i => loop(right, i))
```
