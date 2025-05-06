<!--

## The State Monad {#sec:monad:state}

[`cats.data.State`][cats.data.State]
allows us to pass additional state around as part of a computation.
We define `State` instances representing atomic state operations
and thread them together using `map` and `flatMap`.
In this way we can model mutable state in a purely functional way,
without using actual mutation.

### Creating and Unpacking State

Boiled down to their simplest form,
instances of `State[S, A]` represent functions of type `S => (S, A)`.
`S` is the type of the state and `A` is the type of the result.

```scala mdoc:silent
import cats.data.State
```

```scala mdoc:silent
val a = State[Int, String]{ state =>
  (state, s"The state is $state")
}
```

In other words, an instance of `State` is a function
that does two things:

- transforms an input state to an output state;
- computes a result.

We can "run" our monad by supplying an initial state.
`State` provides three methods---`run`, `runS`, and `runA`---that return
different combinations of state and result.
Each method returns an instance of `Eval`,
which `State` uses to maintain stack safety.
We call the `value` method as usual to extract the actual result:

```scala mdoc
// Get the state and the result:
val (state, result) = a.run(10).value

// Get the state, ignore the result:
val justTheState = a.runS(10).value

// Get the result, ignore the state:
val justTheResult = a.runA(10).value
```

### Composing and Transforming State

As we've seen with `Reader` and `Writer`,
the power of the `State` monad comes from combining instances.
The `map` and `flatMap` methods thread the state from one instance to another.
Each individual instance represents an atomic state transformation,
and their combination represents a complete sequence of changes:

```scala mdoc:invisible:reset-object
import cats.data.State
val a = State[Int, String]{ state =>
  (state, s"The state is $state")
}
```
```scala mdoc:silent
val step1 = State[Int, String]{ num =>
  val ans = num + 1
  (ans, s"Result of step1: $ans")
}

val step2 = State[Int, String]{ num =>
  val ans = num * 2
  (ans, s"Result of step2: $ans")
}

val both = for {
  a <- step1
  b <- step2
} yield (a, b)
```

```scala mdoc
val (state, result) = both.run(20).value
```

As you can see, in this example the final state
is the result of applying both transformations in sequence.
State is threaded from step to step
even though we don't interact with it in the for comprehension.

The general model for using the `State` monad
is to represent each step of a computation as an instance
and compose the steps using the standard monad operators.
Cats provides several convenience constructors for creating primitive steps:

  - `get` extracts the state as the result;
  - `set` updates the state and returns unit as the result;
  - `pure` ignores the state and returns a supplied result;
  - `inspect` extracts the state via a transformation function;
  - `modify` updates the state using an update function.

```scala mdoc
val getDemo = State.get[Int]
getDemo.run(10).value

val setDemo = State.set[Int](30)
setDemo.run(10).value

val pureDemo = State.pure[Int, String]("Result")
pureDemo.run(10).value

val inspectDemo = State.inspect[Int, String](x => s"${x}!")
inspectDemo.run(10).value

val modifyDemo = State.modify[Int](_ + 1)
modifyDemo.run(10).value
```

We can assemble these building blocks using a for comprehension.
We typically ignore the result of intermediate stages
that only represent transformations on the state:

```scala mdoc:silent:reset-object
import cats.data.State
import State._
```

```scala mdoc
val program: State[Int, (Int, Int, Int)] = for {
  a <- get[Int]
  _ <- set[Int](a + 1)
  b <- get[Int]
  _ <- modify[Int](_ + 1)
  c <- inspect[Int, Int](_ * 1000)
} yield (a, b, c)

val (state, result) = program.run(1).value
```

### Exercise: Post-Order Calculator

The `State` monad allows us to implement
simple interpreters for complex expressions,
passing the values of mutable registers along with the result.
We can see a simple example of this by implementing
a calculator for post-order integer arithmetic expressions.

In case you haven't heard of post-order expressions before
(don't worry if you haven't),
they are a mathematical notation
where we write the operator *after* its operands.
So, for example, instead of writing `1 + 2` we would write:

```scala
1 2 +
```

Although post-order expressions are difficult for humans to read,
they are easy to evaluate in code.
All we need to do is traverse the symbols from left to right,
carrying a *stack* of operands with us as we go:

- when we see a number, we push it onto the stack;

- when we see an operator, we pop two operands off the stack,
  operate on them, and push the result in their place.

This allows us to evaluate complex expressions without using parentheses.
For example, we can evaluate `(1 + 2) * 3)` as follows:

```scala
1 2 + 3 * // see 1, push onto stack
2 + 3 *   // see 2, push onto stack
+ 3 *     // see +, pop 1 and 2 off of stack,
          //        push (1 + 2) = 3 in their place
3 3 *     // see 3, push onto stack
3 *       // see 3, push onto stack
*         // see *, pop 3 and 3 off of stack,
          //        push (3 * 3) = 9 in their place
```

Let's write an interpreter for these expressions.
We can parse each symbol into a `State` instance
representing a transformation on the stack
and an intermediate result.
The `State` instances can be threaded together using `flatMap`
to produce an interpreter for any sequence of symbols.

Start by writing a function `evalOne` that
parses a single symbol into an instance of `State`.
Use the code below as a template.
Don't worry about error handling for now---if
the stack is in the wrong configuration,
it's OK to throw an exception.

```scala mdoc:reset:silent
import cats.data.State

type CalcState[A] = State[List[Int], A]

def evalOne(sym: String): CalcState[Int] = ???
```

If this seems difficult,
think about the basic form of the `State` instances you're returning.
Each instance represents a functional transformation
from a stack to a pair of a stack and a result.
You can ignore any wider context and focus on just that one step:

```scala mdoc:invisible
def someTransformation(input: List[Int]): List[Int] = input
def someCalculation: Int = 123
```

```scala mdoc:silent
State[List[Int], Int] { oldStack =>
  val newStack = someTransformation(oldStack)
  val result   = someCalculation
  (newStack, result)
}
```

Feel free to write your `Stack` instances in this form
or as sequences of the convenience constructors we saw above.

<div class="solution">
The stack operation required is different for operators and operands.
For clarity we'll implement `evalOne` in terms of two helper functions,
one for each case:

```scala mdoc:invisible:reset-object
import cats.data.State

type CalcState[A] = State[List[Int], A]
```
```scala
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```

Let's look at `operand` first.
All we have to do is push a number onto the stack.
We also return the operand as an intermediate result:

```scala mdoc:silent
def operand(num: Int): CalcState[Int] =
  State[List[Int], Int] { stack =>
    (num :: stack, num)
  }
```

The `operator` function is a little more complex.
We have to pop two operands off the stack 
(having the second operand at the top of the stack)i
and push the result in their place.
The code can fail if the stack doesn't have enough operands on it,
but the exercise description allows us to throw an exception in this case:

```scala mdoc:silent
def operator(func: (Int, Int) => Int): CalcState[Int] =
  State[List[Int], Int] {
    case b :: a :: tail =>
      val ans = func(a, b)
      (ans :: tail, ans)

    case _ =>
      sys.error("Fail!")
  }
```

```scala mdoc:invisible
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```
</div>

`evalOne` allows us to evaluate single-symbol expressions as follows.
We call `runA` supplying `Nil` as an initial stack,
and call `value` to unpack the resulting `Eval` instance:

```scala mdoc
evalOne("42").runA(Nil).value
```

We can represent more complex programs using `evalOne`, `map`, and `flatMap`.
Note that most of the work is happening on the stack,
so we ignore the results of the intermediate steps for `evalOne("1")` and `evalOne("2")`:

```scala mdoc
val program = for {
  _   <- evalOne("1")
  _   <- evalOne("2")
  ans <- evalOne("+")
} yield ans

program.runA(Nil).value
```

Generalise this example by writing an `evalAll` method
that computes the result of a `List[String]`.
Use `evalOne` to process each symbol,
and thread the resulting `State` monads together using `flatMap`.
Your function should have the following signature:

```scala mdoc:silent
def evalAll(input: List[String]): CalcState[Int] =
  ???
```

<div class="solution">
We implement `evalAll` by folding over the input.
We start with a pure `CalcState` that returns `0` if the list is empty.
We `flatMap` at each stage,
ignoring the intermediate results as we saw in the example:

```scala mdoc:invisible:reset-object
import cats.data.State

type CalcState[A] = State[List[Int], A]
def operand(num: Int): CalcState[Int] =
  State[List[Int], Int] { stack =>
    (num :: stack, num)
  }
def operator(func: (Int, Int) => Int): CalcState[Int] =
  State[List[Int], Int] {
    case b :: a :: tail =>
      val ans = func(a, b)
      (ans :: tail, ans)

    case _ =>
      sys.error("Fail!")
  }
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```
```scala mdoc:silent
import cats.syntax.applicative._ // for pure

def evalAll(input: List[String]): CalcState[Int] =
  input.foldLeft(0.pure[CalcState]) { (a, b) =>
    a.flatMap(_ => evalOne(b))
  }
```

</div>

We can use `evalAll` to conveniently evaluate multi-stage expressions:

```scala mdoc
val multistageProgram = evalAll(List("1", "2", "+", "3", "*"))

multistageProgram.runA(Nil).value
```

Because `evalOne` and `evalAll` both return instances of `State`,
we can thread these results together using `flatMap`.
`evalOne` produces a simple stack transformation and
`evalAll` produces a complex one, but they're both pure functions
and we can use them in any order as many times as we like:

```scala mdoc
val biggerProgram = for {
  _   <- evalAll(List("1", "2", "+"))
  _   <- evalAll(List("3", "4", "+"))
  ans <- evalOne("*")
} yield ans

biggerProgram.runA(Nil).value
```

Complete the exercise by implementing an `evalInput` function that
splits an input `String` into symbols, calls `evalAll`,
and runs the result with an initial stack.

<div class="solution">
We've done all the hard work now.
All we need to do is split the input into terms
and call `runA` and `value` to unpack the result:

```scala mdoc:silent
def evalInput(input: String): Int =
  evalAll(input.split(" ").toList).runA(Nil).value
```

```scala mdoc
evalInput("1 2 + 3 4 + *")
```
</div>


```scala mdoc:reset:silent
```
--->

## `State` モナド {#sec:monad:state}

[`cats.data.State`][cats.data.State] は、計算の一部として状態を扱うことを可能にする。アトミックな状態操作を表す `State` インスタンスをいくつか定義し、それらを `map` と `flatMap` でつなぎ合わせることで計算全体を表現する。このようにして、実際の状態変化を伴わずに、純粋関数型の方法で可変状態をモデル化することができる。

### `State` の作成と展開

突き詰めれば、`State[S, A]` のインスタンスは `S => (S, A)` 型の関数を表す。`S` は状態の型、`A` は結果の型である。

```scala mdoc:silent
import cats.data.State
```

```scala mdoc:silent
val a = State[Int, String]{ state =>
  (state, s"The state is $state")
}
```

言い換えれば、`State` インスタンスとは次のふたつの処理を行う関数である。

- 入力の状態を出力の状態に変換する
- 結果を計算する

初期状態を与えれば `State` モナドを実行することができる。`State` には `run`、`runS`、`runA` という三つのメソッドがあり、それぞれ状態と結果の異なる組み合わせを返す。各メソッドが返すのは `Eval` インスタンスで、`State` はそれによってスタックセーフ性を保っている。最終的な結果を取り出すには、通常どおり `value` メソッドを呼び出す。

```scala mdoc
// 状態と結果を取得する
val (state, result) = a.run(10).value

// 結果を無視し、状態だけを取得する
val justTheState = a.runS(10).value

// 状態を無視し、結果だけを取得する
val justTheResult = a.runA(10).value
```

### `State` の合成と変換

`Reader` や `Writer` と同様に、`State` モナドの強力さはインスタンス同士を組み合わせる能力にある。`map` と `flatMap` があるインスタンスから別のインスタンスへと状態を受け渡していく。各インスタンスはひとつのアトミックな状態変換を表し、それらの組み合わせによって一連の完全な状態変更の流れが形成される。

```scala mdoc:invisible:reset-object
import cats.data.State
val a = State[Int, String]{ state =>
  (state, s"The state is $state")
}
```
```scala mdoc:silent
val step1 = State[Int, String]{ num =>
  val ans = num + 1
  (ans, s"Result of step1: $ans")
}

val step2 = State[Int, String]{ num =>
  val ans = num * 2
  (ans, s"Result of step2: $ans")
}

val both = for {
  a <- step1
  b <- step2
} yield (a, b)
```

```scala mdoc
val (state, result) = both.run(20).value
```

見てのとおり、この例における最終的な状態はふたつの変換を順番に適用した結果である。for 内包表記では状態に直接関与していないにもかかわらず、状態はステップからステップへ受け渡されている。

`State` モナド利用の一般的なモデルは、計算の各ステップを `State` インスタンスとして表現し、標準的なモナド演算子を使ってそれらのステップを合成することである。Cats は、基本的な構成要素となるステップを作成するための便利なコンストラクタをいくつか提供している。

  - `get`: 状態を結果として取り出す
  - `set`: 状態を更新し、結果として `Unit` を返す
  - `pure`: 状態を無視し、指定された値をそのまま結果として返す
  - `inspect`: 状態を関数によって変換した上で結果として取り出す
  - `modify`: 関数を使用して状態を更新する

```scala mdoc
val getDemo = State.get[Int]
getDemo.run(10).value

val setDemo = State.set[Int](30)
setDemo.run(10).value

val pureDemo = State.pure[Int, String]("Result")
pureDemo.run(10).value

val inspectDemo = State.inspect[Int, String](x => s"${x}!")
inspectDemo.run(10).value

val modifyDemo = State.modify[Int](_ + 1)
modifyDemo.run(10).value
```

for 内包表記を使ってこれらの構成要素を組み立てることができる。中間ステップが状態の変換を表しているだけである場合、その結果は無視するのが一般的である。

```scala mdoc:silent:reset-object
import cats.data.State
import State._
```

```scala mdoc
val program: State[Int, (Int, Int, Int)] = for {
  a <- get[Int]
  _ <- set[Int](a + 1)
  b <- get[Int]
  _ <- modify[Int](_ + 1)
  c <- inspect[Int, Int](_ * 1000)
} yield (a, b, c)

val (state, result) = program.run(1).value
```

### 演習: 後置記法の計算機

`State` モナドを使うことで、複雑な式に対するシンプルなインタープリタを実装できる。その実装では、可変レジスタの値を結果と一緒に受け渡しながら処理を進める。シンプルな例として、後置記法による整数の算術式計算機の実装について見ていこう。

後置記法について聞いたことがなくても心配いらない。これは、演算子をオペランドの後に記述する数学的表記法である。たとえば、`1 + 2` と書く代わりに次のように書く。

```scala
1 2 +
```

後置記法の式は人間にとっては読みづらいが、コードで評価するのは簡単である。オペランドを入れる*スタック*を用意し、シンボルを左から右へと読み進めながら、次のように操作するだけでよい。

- 数字が見つかったら、それをスタックにプッシュする
- 演算子が見つかったら、スタックからオペランドをふたつポップし、それらに対して計算を行い、結果をスタックにプッシュする

これにより、括弧を使わずに複雑な式を評価できる。たとえば `(1 + 2) * 3` なら次のようになる。

```scala
1 2 + 3 * // 1 を見つけ、スタックにプッシュ
2 + 3 *   // 2 を見つけ、スタックにプッシュ
+ 3 *     // + を見つけ, 1 と 2 をスタックからポップし、
          //     足し合わせた結果である 3 をスタックにプッシュする
3 *       // 3 を見つけ、スタックにプッシュ
*         // * を見つけ, 3 と 3 をスタックからポップし、
          //     掛け合わせた結果である 9 をスタックにプッシュする
```

このような式を評価するインタープリタを書いてみよう。`State` インスタンスをスタック上での変換および中間結果を表すものとし、各シンボルを読み取ってこれに変換する。それらの `State` インスタンスを `flatMap` でつなげれば、任意のシンボルの連なりを処理するインタープリタを作成できる。

まずは、単一のシンボルを読み取って `State` インスタンスに変換する `evalOne` 関数を作成せよ。以下のコードをテンプレートとして使用すること。エラーハンドリングについては今は考慮しなくてよい。スタックが不正な状態にある場合、例外を投げてもかまわない。

```scala mdoc:reset:silent
import cats.data.State

type CalcState[A] = State[List[Int], A]

def evalOne(sym: String): CalcState[Int] = ???
```

もし難しく感じるなら、返却する `State` インスタンスの基本的な形について考えてみよう。各 `State` インスタンスは、スタックを受け取りスタックと結果のペアを返す関数的な変換を表している。広い文脈は無視して、そのひとつのステップに集中すればよい。

```scala mdoc:invisible
def someTransformation(input: List[Int]): List[Int] = input
def someCalculation: Int = 123
```

```scala mdoc:silent
State[List[Int], Int] { oldStack =>
  val newStack = someTransformation(oldStack)
  val result   = someCalculation
  (newStack, result)
}
```

`Stack` インスタンスは、このような形式で自由に書いてもかまわないし、上で見た便利なコンストラクタのシーケンスとして記述してもよい。

<div class="solution">
見つかったのが演算子かオペランドかによって必要なスタック操作は異なる。わかりやすさのため、ケース毎にひとつずつヘルパー関数を用意し、それを使って `evalOne` を実装しよう。

```scala mdoc:invisible:reset-object
import cats.data.State

type CalcState[A] = State[List[Int], A]
```
```scala
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```

まずは `operand` から見ていこう。必要なのは読み取ったオペランドをスタックに積むことだけである。同時に、中間結果としても、そのオペランドを用いる。

```scala mdoc:silent
def operand(num: Int): CalcState[Int] =
  State[List[Int], Int] { stack =>
    (num :: stack, num)
  }
```

`operator` 関数はもうすこし複雑である。スタックからオペランドをふたつポップし、計算結果をスタックにプッシュしなければならない。スタックの一番上にあるのは二番目のオペランドだという点に気をつけること。スタック上にオペランドがふたつ以上ない場合、処理は失敗する。この演習では、失敗のハンドリング方法として、例外を投げることを認めている。

```scala mdoc:silent
def operator(func: (Int, Int) => Int): CalcState[Int] =
  State[List[Int], Int] {
    case b :: a :: tail =>
      val ans = func(a, b)
      (ans :: tail, ans)

    case _ =>
      sys.error("Fail!")
  }
```

```scala mdoc:invisible
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```
</div>

`evalOne` を使うと、次のように単一シンボルの式を評価できる。初期スタックとして `Nil` を渡して `runA` を呼び出し、結果として得られる `Eval` インスタンスを `value` を使って展開する。

```scala mdoc
evalOne("42").runA(Nil).value
```

もっと複雑なプログラムも `evalOne`、`map`、`flatMap` を使うことで表現できる。処理のほとんどはスタック上で行われるため、`evalOne("1")` や `evalOne("2")` といった中間ステップの結果は無視している。

```scala mdoc
val program = for {
  _   <- evalOne("1")
  _   <- evalOne("2")
  ans <- evalOne("+")
} yield ans

program.runA(Nil).value
```

この例を一般化し、`List[String]` の結果を計算する `evalAll` メソッドを作成せよ。`evalOne` を使って各シンボルを処理し、結果として得られる `State` モナドを `flatMap` でつなげること。作成する関数のシグネチャは以下のとおりとする。

```scala mdoc:silent
def evalAll(input: List[String]): CalcState[Int] =
  ???
```

<div class="solution">
`evalAll` の実装では入力に対して畳み込みを行う。`0` をコンテキストに包んだだけの純粋な `CalcState` を初期状態とする。入力されたリストが空であれば `0` が返される。各ステップでは `flatMap` を行う。その際、中間結果は前の例で見たように無視する。

```scala mdoc:invisible:reset-object
import cats.data.State

type CalcState[A] = State[List[Int], A]
def operand(num: Int): CalcState[Int] =
  State[List[Int], Int] { stack =>
    (num :: stack, num)
  }
def operator(func: (Int, Int) => Int): CalcState[Int] =
  State[List[Int], Int] {
    case b :: a :: tail =>
      val ans = func(a, b)
      (ans :: tail, ans)

    case _ =>
      sys.error("Fail!")
  }
def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case num => operand(num.toInt)
  }
```
```scala mdoc:silent
import cats.syntax.applicative._ // pure

def evalAll(input: List[String]): CalcState[Int] =
  input.foldLeft(0.pure[CalcState]) { (a, b) =>
    a.flatMap(_ => evalOne(b))
  }
```

</div>

`evalAll` を使えば、複数ステップからなる式を手軽に評価できる。

```scala mdoc
val multistageProgram = evalAll(List("1", "2", "+", "3", "*"))

multistageProgram.runA(Nil).value
```

`evalOne` と `evalAll` はどちらも `State` インスタンスを返すので、これらの結果を `flatMap` でつなげることができる。`evalOne` はスタックの単純な変換を、`evalAll` は複雑な変換を生成するが、複雑さにかかわらずどちらも純粋関数である。順序を問わずいくつでも連結が可能である。

```scala mdoc
val biggerProgram = for {
  _   <- evalAll(List("1", "2", "+"))
  _   <- evalAll(List("3", "4", "+"))
  ans <- evalOne("*")
} yield ans

biggerProgram.runA(Nil).value
```

`evalInput` 関数を実装し、この演習課題を完成させよ。この関数は、入力された文字列をシンボルに分割して `evalAll` を呼び出し、その結果に初期スタックを与えて実行するものとする。

<div class="solution">
難しい部分はすべて終えている。あとは入力をシンボルに分割し、`runA` と `value` を呼び出して結果を展開するだけである。

```scala mdoc:silent
def evalInput(input: String): Int =
  evalAll(input.split(" ").toList).runA(Nil).value
```

```scala mdoc
evalInput("1 2 + 3 4 + *")
```
</div>
