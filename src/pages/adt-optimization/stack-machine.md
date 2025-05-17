<!--

## Compilers and Virtual Machines

We've reified continuations and seen they contain a stack structure: each continuation contains a references to the next continuation, and continuations are constructed in a last-in first-out order. We'll now, once again, reify this structure. This time we'll create an explicit stack, giving rise to a stack-based **virtual machine** to run our code. We'll also introduce a compiler, transforming our code into a sequence of operations that run on this virtual machine. We'll then look at optimizing our virtual machine. As this code involves benchmarking, there is an [accompanying repository][stack-machine] that contains benchmarks you can run on your own computer.


### Virtual and Abstract Machines

A virtual machine is a computational machine implemented in software rather than hardware. A virtual machine runs programs written in some **instruction set**. The Java Virtual Machine (JVM), for example, runs programs written in Java bytecode.
Closely related are **abstract machines**. The two terms are sometimes used interchangeably but I'll make the distinction that a virtual machine has an implementation in software, while an abstract machine is a theoretical model without an implementation. Thus we can think of an abstract machine as a concept, and a virtual machine as a realization of a concept. This is a distinction we've made in many other parts of the book.

As an abstract machine, stack machines are represented by models such as push down automata and the SECD machine. From abstract stack machines we firstly get the concept itself of a stack machine. The two core operations for a stack are pushing a value on to the top of the stack, and popping the top value off the stack. Function arguments and results are both passed via the stack. So, for example, a binary operation like addition will pop the top two values off the stack, add them, and push the result onto the stack. Abstract stack machines also tell us that stack machines with a single stack are not universal computers. In other words, they are not as powerful as Turing machines. If we add a second stack, or some other form of additional memory, we have a universal computer. This informs the design of virtual machines based on a stack machine.

Stack machines are also very common virtual machines. The Java Virtual Machine is a stack machine, as are the .Net and WASM virtual machines. They are easy to implement, and to write compilers for. We've already seen how easy it is to implement an interpreter so why should we care about stack machines, or virtual machines in general? The usual answer is performance. Implementing a virtual machine opens up opportunities for optimizations that are difficult to implement in interpreters. Virtual machines also give us a lot of flexibility. It's simple to trace or otherwise inspect the execution of a virtual machine, which makes debugging easier. They are easy to port to different platforms and languages. Virtual machines are often very compact, as is the code they run. This makes them suitable for embedded devices. Our focus will be on performance. Although we won't go down the rabbit-hole of compiler and virtual machine optimizations, which would easily take up an entire book, we'll at least tip-toe to the edge and peek down.


### Compilation

Let's now briefly talk about compilation. A compiler transforms a program from one representation to another. In our case we will transform our programs represented as an algebraic data type of reified constructors and combinators into the instruction set for our virtual machine. The virtual machine itself is an interpreter for its instruction set. Computation always bottoms out in interpretation: a hardware CPU is nothing but an interpreter for it's machine code.

Notice there are two notions of program here, and two corresponding instruction sets: there is the program the structurally recursive interpreter executes, with an instruction set consisting of reified constructors and combinators, and there is the program we compile this into for the stack machine using the stack machine's instruction set. We will call these the interpreter program and instruction set, and stack machine program and instruction set respectively.

The structurally recursive interpreter is an example of a **tree-walking interpreter** or **abstract syntax tree (AST) interpreter**. The stack machine is an example of a **byte-code interpreter**.


## From Interpreter to Stack Machine

There are three parts to transforming an interpreter to a stack machine:

1. creating the instruction set the stack machine will run;
2. creating the compiler from interpreter programs to stack machine programs; and
3. implementing the stack machine to execute stack machine instructions.

Let's make this concrete by returning to our arithmetic interpreter.

```scala mdoc:silent
enum Expression {
  def +(that: Expression): Expression = Addition(this, that)
  def *(that: Expression): Expression = Multiplication(this, that)
  def -(that: Expression): Expression = Subtraction(this, that)
  def /(that: Expression): Expression = Division(this, that)

  def eval: Double =
    this match {
      case Literal(value)              => value
      case Addition(left, right)       => left.eval + right.eval
      case Subtraction(left, right)    => left.eval - right.eval
      case Multiplication(left, right) => left.eval * right.eval
      case Division(left, right)       => left.eval / right.eval
    }

  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
}
object Expression {
  def literal(value: Double): Expression = Literal(value)
}
```

Interpreter programs are defined by the interpreter instruction set

```scala
enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
}
```

Transforming the interpreter instruction set to the stack machine instruction set works as follows:

- each constructor interpreter instruction corresponds to stack machine instruction carrying exactly the same data; and
- each combinator interpreter instruction has a corresponding stack machine instruction that carries only non-recursive data. Recursive data, which is executed by recursive calls to the interpreter, will be represented by data on the stack machine's stack.

Turning to the arithmetic interpreter's instruction set, we see that `Literal` is our sole constructor and thus has a mirror in our stack machine's instruction set. Here I've named the interpreter instruction set `Op` (short for "operation"), and shortened the name from `Literal` to `Lit` to make it clearer which instruction set we are using.

```scala
enum Op {
  case Lit(value: Double)
}
```

The other instructions are all combinators. They also all only contain values of type `Expression`, and hence in the stack machine the corresponding values will be found on the stack. This gives us the complete stack machine instruction set.

```scala mdoc:silent
enum Op {
  case Lit(value: Double)
  case Add
  case Sub
  case Mul
  case Div
}
```

This completes the first step of the process. The second step is to implement the compiler. The secret to compiling for a stack machine is to transfrom instructions into **reverse polish notation (RPN)**. In RPN operations follow their operands. So, instead of writing `1 + 2` we write `1 2 +`. This is exactly the order in which a stack machine works. To evaluate `1 + 2` we should first push `1` onto the stack, then push `2`, and finally pop both these values, perform the addition, and push the result back to the stack. RPN also does not need nesting. To represent `1 + (2 + 3)` in RPN we simply use `2 3 + 1 +`. Doing away with brackets means that stack machine programs can be represented as a linear sequence of instructions, not a tree. Concretely, we can use `List[Op]`.

How we should we implement the conversion to RPN. We are performing a transformation on an algebraic data type, our interpreter instruction set and therefore we can use structural recursion. The following code shows one way to implement this. It's not very efficient (appending lists is a slow operation) but this doesn't matter for our purposes.

```scala
def compile: List[Op] =
  this match {
    case Literal(value) => List(Op.Lit(value))
    case Addition(left, right) =>
      left.compile ++ right.compile ++ List(Op.Add)
    case Subtraction(left, right) =>
      left.compile ++ right.compile ++ List(Op.Sub)
    case Multiplication(left, right) =>
      left.compile ++ right.compile ++ List(Op.Mul)
    case Division(left, right) =>
      left.compile ++ right.compile ++ List(Op.Div)
  }
```

We now are left to implement the stack machine. We'll start by sketching out the interface for the stack machine.

```scala
final case class StackMachine(program: List[Op]) {
  def eval: Double = ???
}
```

In this design the program is fixed for a given `StackMachine` instance, but we can run the program multiple times.

Now we'll implement `eval`. It is a structural recursion over an algebraic data type, in this case the `program` of type `List[Op]`. It's a little bit more complicated than some of the structural recursions we have seen, because we need to implement the stack as well. We'll represent the stack as a `List[Double]`, and define methods to push and pop the stack.


```scala
final case class StackMachine(program: List[Op]) {
  def eval: Double = {
    def pop(stack: List[Double]): (Double, List[Double]) =
      stack match {
        case head :: next => (head, next)
        case Nil =>
          throw new IllegalStateException(
            s"The data stack does not have any elements."
          )
      }

    def push(value: Double, stack: List[Double]): List[Double] =
      value :: stack

    ???
  }
}
```

Now we can define the main stack machine loop. It takes as parameters the program and the stack, and is a structural recursion over the program.

```scala
def eval: Double = {
  // pop and push defined here ...

  def loop(stack: List[Double], program: List[Op]): Double =
    program match {
      case head :: next =>
        head match {
          case Op.Lit(value) => loop(push(value, stack), next)
          case Op.Add =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Sub =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Mul =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Div =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
        }

      case Nil => stack.head
    }

  loop(List.empty, program)
}
```

I've implemented a simple benchmark for this code (see [the repository][stack-machine]) and it's roughly five times slower than the interpreter we started with. Clearly some optimization is needed.

[stack-machine]: https://github.com/scalawithcats/stack-machine


### Effectful Interpreters

One of the reasons for using the interpreter strategy is to isolate effects, such as state or input and output. An interpreter can be effectful without impacting the ability to reason about or compose the programs the interpreter runs. Sometimes the effects are the entire point of the interpreter as the program may describe effectful actions, such as parsing network data or drawing on a screen, which the interpreter then carries out. Sometimes effects may just be optimizations, which is how we are going to use them in our arithmetic stack machine.

There are many inefficiencies in the stack machine we have just created. A `List` is a poor choice of data structure for both the stack and program. We can avoid a lot of pointer chasing and memory allocation by using a fixed size `Array`. The program never changes in size, and we can simply allocate a large enough stack that resizing it becomes very unlikely. We can also avoid the indirection of pushing and popping and operate directly on the stack array.

The code below shows a simple implementation, which in my benchmarking is about thirty percent faster than the tree-walking interpreter.

```scala
final case class StackMachine(program: Array[Op]) {
  // The data stack
  private val stack: Array[Double] = Array.ofDim[Double](256)

  def eval: Double = {
    // sp points to first free element on the stack
    // stack(sp - 1) is the first element with data
    //
    // pc points to the current instruction in program
    def loop(sp: Int, pc: Int): Double =
      if (pc == program.size) stack(sp - 1)
      else
        program(pc) match {
          case Op.Lit(value) =>
            stack(sp) = value
            loop(sp + 1, pc + 1)
          case Op.Add =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a + b)
            loop(sp - 1, pc + 1)
          case Op.Sub =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a - b)
            loop(sp - 1, pc + 1)
          case Op.Mul =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a * b)
            loop(sp - 1, pc + 1)
          case Op.Div =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a / b)
            loop(sp - 1, pc + 1)
        }

    loop(0, 0)
  }
}
```


### Further Optimization

The above optimization is, to me, the most obvious and straightforward to implement. In this section we'll attempt to go further, by looking at some of the optimizations described in the literature. We'll see that there is not always a straight path to faster code.

The benchmark I used is the simple recursive Fibonacci. Calculating the $n^{th}$ Fibonacci number produces a large expression for a modest choice of $n$. I used a value of 25, and the expression has over one million elements. Notably the expressions only involve addition, and the only literals in use are zero and one. This limits the applicability of the optimizations to a wider range of inputs, but the intention is not to produce an optimized interpreter for this specific case but rather to discuss possible optimizations and issues that arise when attempting to optimize an interpreter in general.

We'll look at four different optimizations, which all use the optimized stack machine above as their base:

- **Algebraic simplification** performs simplifications at compile-time to produce smaller expressions. A small expression should require fewer interpreter steps and hence be faster. The only simplification I used was replacing $x + 0$ or $0 + x$ with $x$. This occurs frequently in the Fibonacci series. Since the expressions we are working with have no variables or control flow we could simplify the entire expression to a single literal at compile-time. This would be an extremely good optimization but rather defeats the purpose of trying to generalize to other applications.

- **Byte code** replaces the `Op` algebraic data type with a single byte. The hope here is that the smaller representation will lead to better cache utilization, and possibly a faster `match` expression, and therefore a faster overall interpreter. In this representation literals are also stored in a separate array of `Doubles`. More on this later.

- **Stack caching** stores the top of the stack in a variable, which we hope will be allocated to a register and therefore be extremely fast to access. The remainder of the stack is stored in an array as above. Stack caching involves more work when pushing values on to the stack, as we must copy the value from the top into the array, but less work when popping values off the stack. The hope is that the savings will outweigh the costs.

- **Superinstructions** replace common sequences of instructions with a single instruction. We already do this to an extent; a typical stack machine would have separate instructions for pushing and popping, but our instruction set merges these into the arithmetic operations. I used two superinstructions: one for incrementing a value, which frequently occurs in the Fibonacci, and one for adding two values from the stack and a literal.

Below are the benchmarks results obtained on an AMD Ryzen 5 3600 and an Apple M1, both running JDK 21. Results are shown in operations per second. The Baseline interpreter is the one using structural recursion. The Stack interpreter uses a `List` to represent the stack and program. The Optimized Stack represents the stack and program as arrays. The other interpreters build on the Optimized Stack interpreter and add the optimizations described above. The All interpreter has all the optimizations.

+--------------------------+---------+---------+---------+---------+
| Interpreter              | Ryzen 5 | Speedup | M1      | Speedup |
+==========================+=========+=========+=========+=========+
| Baseline                 | 2754.43 | 1       | 3932.93 | 1       |
+--------------------------+---------+---------+---------+---------+
| Stack                    | 676.43  | 0.25    | 1004.16 | 0.26    |
+--------------------------+---------+---------+---------+---------+
| Optimized Stack          | 3631.19 | 1.32    | 2953.21 | 0.75    |
+--------------------------+---------+---------+---------+---------+
| Algebraic Simplification | 1630.93 | 0.59    | 4818.45 | 1.23    |
+--------------------------+---------+---------+---------+---------+
| Byte Code                | 4057.11 | 1.47    | 3355.75 | 0.85    |
+--------------------------+---------+---------+---------+---------+
| Stack Caching            | 3698.10 | 1.34    | 3237.17 | 0.82    |
+--------------------------+---------+---------+---------+---------+
| Superinstructions        | 3706.10 | 1.35    | 4689.02 | 1.19    |
+--------------------------+---------+---------+---------+---------+
| All                      | 7612.45 | 2.76    | 7098.06 | 1.80    |
+--------------------------+---------+---------+---------+---------+

There are a few lessons to take from this. The most important, in my opinion, is that *performance is not compositional*. The results of applying two optimizations is not simply the sum of applying the optimizations individually. You can see that most of the optimizations on their own make little or no change to performance relative to the Optimized Stack interpreter. Taken together, however, they make a significant improvement.

Basic structural recursion, the Baseline interpreter, is surprisingly fast; a bit slower than the Optimized Stack interpreter on the Ryzen 5 but faster on the M1. A stack machine emulates the processor's built-in call stack. The native call stack is extremely fast, so we need a good reason to avoid using it.

Details really matter in optimization. We see the choice of data structure makes a massive difference between the Stack and Optimized Stack interpreters. An earlier version of the Byte Code interpreter had worse performance than the Optimized Stack. As best I could tell this was because I was storing literals alongside byte code, and loading a `Double` from an `Array[Byte]` (using a `ByteBuffer`) was slow. Superinstructions are very dependent on the chosen superinstructions. The superinstruction to add two values from the stack plus a literal had little effect on it's own; in fact the interpreter with this single superinstruction was much slower on the Ryzen 5.

Compilers, and JIT compilers in particular, are difficult to understand. I cannot explain why, for example, the Algebraic Simplification interpreter is so slow on the Ryzen 5. This interpreter does strictly less work than the Optimized Stack interpreter. Just like the interpreter optimizations I implemented, compiler optimizations apply in restricted cases that the algorithms recognize. If code does not match the patterns the algorithms look for, the optimizations will not apply, which can lead to strange performance cliffs. My best guess is that something about my implementation caused me to run afoul of such an issue.

Finally, differences between platforms are also significant. It's hard to know how much this due to differences in the computer's architecture, and how much is down to differences in the JVM. Either way, be aware of which platform or platforms you expect the majority of users to run on, and don't naively assume performance on one platform will directly translate to another.


```scala mdoc:reset:silent
```
--->

## コンパイラと仮想マシン

前節では、継続をレイフィケーションし、それがスタック構造をもつことを見てきた。各継続は次の継続への参照を含み、継続は後入れ先出しの順序で構築される。ここでは、その構造を再びレイフィケーションする。今回は明示的なスタックを作成し、コードを実行するスタックベースの**仮想マシン（virtual machine）**を構築する。さらに、コードをこの仮想マシン上で動作する一連の操作に変換するコンパイラを導入する。その後、この仮想マシンの最適化について考える。このコードには性能測定が必要なので、自分のコンピュータ上でベンチマークを実行できる[関連リポジトリ][stack-machine]が用意されている。

### 仮想機械と抽象機械

仮想マシン（仮想機械）とは、ハードウェアではなくソフトウェアで実現された計算機である。仮想マシンは特定の**命令セット**で記述されたプログラムを実行する。たとえば、Java 仮想マシン（JVM）は Java バイトコードで記述されたプログラムを実行する。

これと密接に関連しているのが**抽象機械（abstract machine）**である。このふたつの用語は同じ意味で用いられることもあるが、本書では区別を設ける。仮想マシンはソフトウェアとして実装されたものとし、抽象機械は実装をもたない理論的なモデルとする。つまり、抽象機械は概念、仮想マシンは概念を実現したものと考えることができる。この区別は、本書の他の部分にわたって広く用いられる。

抽象機械としてのスタックマシンは、プッシュダウン・オートマトンや SECD マシンといったモデルによって表現される。抽象的なスタックマシンからは、まずスタックマシンという概念そのものが得られる。スタックの主要なふたつの操作は、値をスタックの一番上にプッシュすることと、一番上の値をポップすることである。関数の引数と戻り値はどちらもスタックを介して受け渡しされる。たとえば、加算のような二項演算では、スタックからふたつの値をポップし、それらを足し合わせ、その結果をスタックにプッシュする。抽象的なスタックマシンは、スタックをひとつしかもたないスタックマシンが汎用計算機ではないことも教えてくれる。言い換えると、そのようなスタックマシンはチューリングマシンと同等の計算能力をもたない。スタックをもうひとつ追加するか、あるいは別の形態の追加メモリがあれば、スタックマシンは汎用計算機となる。この考え方は、スタックマシンに基づいた仮想マシンの設計に影響を与える。

スタックマシンは仮想マシンとしても非常に一般的で、Java、.Net、WASM の仮想マシンはいずれもスタックマシンである。スタックマシンは実装が容易であり、コンパイラを作成するのも簡単である。インタープリタの実装がいかに容易であるかはすでに見たが、ではなぜスタックマシンや仮想マシン全般に関心をもつべきなのかといえば、よく言われる理由はパフォーマンスである。仮想マシンを実装することでインタープリタでは実現が難しい最適化への道が拓ける。仮想マシンは高い柔軟性も提供してくれる。仮想マシンの実行を追跡、調査するのは簡単でデバッグしやすい。さらに、異なるプラットフォームや言語に移植するのも簡単である。仮想マシンおよびその上で実行されるコードはいずれも非常にコンパクトであることが多く、組み込み機器に適している。

ここではパフォーマンスに焦点を当てる。コンパイラや仮想マシンの最適化は軽く一冊の本を費やせる分野であり詳細に踏み込むつもりはないが、その入り口に足を踏み入れ、中をのぞき込んでみたい。

### コンパイル

まずはコンパイルについて簡単に説明しよう。コンパイラは、プログラムをある表現形式から別の表現形式へ変換するものである。本書の場合、レイフィケーションされたコンストラクタとコンビネータからなる代数的データ型として表現されたプログラムを、仮想マシン用の命令セットへと変換する。仮想マシン自体は、その命令セットのインタープリタである。計算は最終的にはかならずインタープリタによる実行に行き着く。ハードウェアの CPU もまた、機械語を解釈するインタープリタに過ぎない。

ここではプログラムの概念がふたつと、それに対応するふたつの命令セットがあることに注目してほしい。ひとつ目は、構造的に再帰的なインタープリタが実行するプログラムであり、その命令セットはレイフィケーションされたコンストラクタとコンビネータから成る。そしてもうひとつは、そのプログラムをスタックマシン用の命令セットにコンパイルして得られるプログラムである。本書では、これらをそれぞれ「インタープリタプログラムと命令セット」「スタックマシンプログラムと命令セット」と呼ぶことにする。

構造的に再帰的なインタープリタは、**ツリーウォーク型インタープリタ**または**抽象構文木（AST）インタープリタ**の一例である。一方、スタックマシンは**バイトコードインタープリタ**の例である。

## インタープリタからスタックマシンへ

インタープリタからスタックマシンへの変換は次の三つの部分から構成される。

1. スタックマシンが実行する命令セットの作成
2. インタープリタプログラムをスタックマシンプログラムに変換するコンパイラの作成
3. 命令を実行するスタックマシンの実装

以前実装した算術インタープリタを振り返って、これを具体的に考えてみよう。

```scala mdoc:silent
enum Expression {
  def +(that: Expression): Expression = Addition(this, that)
  def *(that: Expression): Expression = Multiplication(this, that)
  def -(that: Expression): Expression = Subtraction(this, that)
  def /(that: Expression): Expression = Division(this, that)

  def eval: Double =
    this match {
      case Literal(value)              => value
      case Addition(left, right)       => left.eval + right.eval
      case Subtraction(left, right)    => left.eval - right.eval
      case Multiplication(left, right) => left.eval * right.eval
      case Division(left, right)       => left.eval / right.eval
    }

  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
}
object Expression {
  def literal(value: Double): Expression = Literal(value)
}
```

インタープリタプログラムはインタープリタ用の命令セットによって定義される。

```scala
enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
}
```

インタープリタの命令セットをスタックマシンの命令セットに変換する際の対応関係は以下のとおりである。

- インタープリタ命令におけるコンストラクタは、それぞれまったく同じデータをもつスタックマシン命令に対応する
- インタープリタ命令におけるコンビネータは、再帰的でないデータのみをもつスタックマシン命令に対応する。再帰的なデータは、インタープリタの再帰呼び出しによって実行されるものだが、スタックマシンではスタック上のデータとして表現される

算術インタープリタの命令セットに目を向けよう。`Literal` が唯一のコンストラクタであり、スタックマシンの命令セットにもこれに対応するものが必要だということがわかる。ここでは、インタープリタの命令セットを `Op`（"operation" の略）と名付け、インタープリタの命令セットとの区別を明確にするために `Literal` を `Lit` に短縮している。

```scala
enum Op {
  case Lit(value: Double)
}
```

それ以外の命令はすべてコンビネータである。これらはすべて `Expression` 型の値しか保持していないので、それに対応する値はスタックマシンではスタック上に存在する。このことから、スタックマシンの完全な命令セットは次のように定義される。

```scala mdoc:silent
enum Op {
  case Lit(value: Double)
  case Add
  case Sub
  case Mul
  case Div
}
```

これで最初のステップは完了である。次のステップではコンパイラを実装する。スタックマシン向けのコンパイルの秘訣は、命令を**逆ポーランド記法（reverse polish notation, RPN）**に変換することである。RPN では演算子がオペランドの後に来る。たとえば `1 + 2` を `1 2 +` と表現する。これはスタックマシンの動作順序そのものである。`1 + 2` を評価するには、まず `1` をスタックにプッシュし、次に `2` をプッシュし、最後にこれらの値を両方ポップして加算を行い、結果をスタックに戻す。RPN ではネストも必要ない。たとえば `1 + (2 + 3)` を RPN で表現すると `2 3 + 1 +` となる。括弧を除去できるため、スタックマシンプログラムはツリーではなく直線的な一連の命令として表現できる。具体的には `List[Op]` で表現することが可能である。

RPN への変換をどのように実装すればよいか考えよう。ここではインタープリタの命令セットは代数的データ型である。それに対して変換を行うのだから、構造的再帰を用いることができる。以下のコードはその一例である。リスト同士の連結は遅く、それを用いるこの実装は効率的ではないが、本書の目的にとって問題にはならない。

```scala
def compile: List[Op] =
  this match {
    case Literal(value) => List(Op.Lit(value))
    case Addition(left, right) =>
      left.compile ++ right.compile ++ List(Op.Add)
    case Subtraction(left, right) =>
      left.compile ++ right.compile ++ List(Op.Sub)
    case Multiplication(left, right) =>
      left.compile ++ right.compile ++ List(Op.Mul)
    case Division(left, right) =>
      left.compile ++ right.compile ++ List(Op.Div)
  }
```

これで残りはスタックマシンを実装するだけとなった。スタックマシンのインターフェースを大まかに描き出すところから始めよう。

```scala
final case class StackMachine(program: List[Op]) {
  def eval: Double = ???
}
```

この設計では `StackMachine` インスタンスごとにプログラムが固定されるが、そのプログラムは複数回実行することができる。

次に `eval` を実装する。この関数は代数的データ型に対する構造的再帰である。今回のケースでは `List[Op]` 型の `program` がその代数的データ型にあたる。ただし、スタックを実装する必要がある分だけ、これまで見てきた構造的再帰よりもすこし複雑である。スタックは `List[Double]` 型データとして表現し、値をプッシュおよびポップするメソッドを定義する。

```scala
final case class StackMachine(program: List[Op]) {
  def eval: Double = {
    def pop(stack: List[Double]): (Double, List[Double]) =
      stack match {
        case head :: next => (head, next)
        case Nil =>
          throw new IllegalStateException(
            s"The data stack does not have any elements."
          )
      }

    def push(value: Double, stack: List[Double]): List[Double] =
      value :: stack

    ???
  }
}
```

これで、スタックマシンのメインループを定義できる。このループは、プログラムとスタックをパラメータとして受け取り、プログラムに対する構造的再帰として動作する。

```scala
def eval: Double = {
  // ここに pop と push が定義されている

  def loop(stack: List[Double], program: List[Op]): Double =
    program match {
      case head :: next =>
        head match {
          case Op.Lit(value) => loop(push(value, stack), next)
          case Op.Add =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Sub =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Mul =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
          case Op.Div =>
            val (a, s1) = pop(stack)
            val (b, s2) = pop(s1)
            val s = push(a + b, s2)
            loop(s, next)
        }

      case Nil => stack.head
    }

  loop(List.empty, program)
}
```

このコード用に簡単なベンチマーク（[GitHubリポジトリ][stack-machine]を参照）を実装してみたところ、当初のインタープリタよりも約5倍遅いことがわかった。明らかに何らかの最適化が必要である。

[stack-machine]: https://github.com/scalawithcats/stack-machine


### 副作用をもつインタープリタ

インタープリタ戦略を使用する理由のひとつは、状態や入出力のような副作用を分離するためである。インタープリタは副作用をもつことができるが、実行するプログラムについて論理的な推論や合成を行う能力がそれによって損なわれることはない。ときには、副作用そのものがインタープリタの主目的となる。プログラムがネットワークデータの解析や画面への描画といった副作用的なアクションを記述し、それをインタープリタが実行する場合がそうである。一方で、副作用が単なる最適化に過ぎない場合もある。今回の算術スタックマシンではこの形で副作用を使用する。

先ほど作成したスタックマシンには多くの非効率が存在する。`List` はスタックとプログラムいずれのデータ構造としても適切な選択とは言えない。固定されたサイズをもつ `Array` を用いれば、多くのポインタ追跡やメモリ割当を回避できる。プログラムのサイズは変化しないし、十分な大きさのスタックを確保しておけばリサイズが発生する可能性を極めて小さくできる。また、プッシュやポップのような間接的な操作を避け、スタック配列を直接操作することも可能である。

以下のコードはその単純な実装である。この実装は、私のベンチマークではツリーウォーク型インタープリタより約30%高速であることが確認された。

```scala
final case class StackMachine(program: Array[Op]) {
  // The data stack
  private val stack: Array[Double] = Array.ofDim[Double](256)

  def eval: Double = {
    // sp はスタック上の最初の空き領域を指すインデックス
    // stack(sp - 1) がスタックの一番上にあるデータ
    //
    // pc はプログラム内の現在の命令を指すインデックス
    def loop(sp: Int, pc: Int): Double =
      if (pc == program.size) stack(sp - 1)
      else
        program(pc) match {
          case Op.Lit(value) =>
            stack(sp) = value
            loop(sp + 1, pc + 1)
          case Op.Add =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a + b)
            loop(sp - 1, pc + 1)
          case Op.Sub =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a - b)
            loop(sp - 1, pc + 1)
          case Op.Mul =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a * b)
            loop(sp - 1, pc + 1)
          case Op.Div =>
            val a = stack(sp - 1)
            val b = stack(sp - 2)
            stack(sp - 2) = (a / b)
            loop(sp - 1, pc + 1)
        }

    loop(0, 0)
  }
}
```


### さらなる最適化

上記の最適化がもっとも明白で実装しやすいものと思われる。本節では、文献で紹介されているいくつかの手法に目を向け、最適化ををさらに進めてみることにする。速いコードに直結する道が常に存在するわけではないことがわかるだろう。

使用したベンチマークは単純な再帰的フィボナッチ計算である。控えめな大きさの $n$ を選んだとしても、$n$ 番目のフィボナッチ数の計算には大きな式が生成される。$n = 25$ なら式の要素数は100万を超える。注目すべきは、式が加算のみで構成され、使用されるリテラルが `0` と `1` だけだということで、これにより広範な入力に対する最適化の適用可能性は制限される。だが、目的は今回のケースに特化して最適化されたインタープリタを作成することではない。最適化の可能性、そして一般的にインタープリタの最適化を試みたときに生じる問題について議論することである。

ここでは4つの異なる最適化手法を取り上げる。いずれも先述の最適化されたスタックマシンを基盤としている。

- **代数的簡略化（algebraic simplification）**はコンパイル時に簡略化を行って小さな式を生成する。式が小さければ、インタープリタのステップ数は減少し、動作はより高速になるはずである。今回行った簡略化は $x + 0$ および $0 + x$ の $x$ への置き換えだけである。フィボナッチ計算ではこのような式が頻繁に現れる。今回の式は変数や制御フローを含まないため、コンパイル時に式全体を単一のリテラルに置き換えることもできる。これは非常に優れた最適化になるが、他の用途への汎用化を試みるという目的にはそぐわない。

- **バイトコード (byte code)**では `Op` という代数的データ型を1バイトで表現する。これによって期待されるのは、この小さな表現によりキャッシュ効率が向上し `match` 式が高速化されることで、インタプリタ全体の動作も速くなることである。この表現では、リテラルは `Double` 型の配列に格納される。詳細は後述する。

- **スタックキャッシング (stack caching)**では、スタックトップを変数に格納し、これがレジスタに割り当てられることによるアクセスの高速化を期待する。スタックの残りの部分はこれまでどおり配列に格納される。値をスタックにプッシュする際には変数に格納されたスタックトップを配列にコピーする必要があるため作業量は増えるが、ポップする際の作業量は減る。これによる処理の軽減がコストを上回ることを期待する。

- **スーパーインストラクション (super instruction)**では、よく使われる命令のシーケンスをひとつの命令に置き換える。すでにある程度はこの考え方を取り入れている。典型的なスタックマシンはプッシュとポップで別々の命令をもつが、今回の命令セットではこれを算術演算へと統合している。ここでは、フィボナッチ計算で頻繁に現れる値のインクリメントと、スタック上の値とリテラルのふたつを加算する操作をスーパーインストラクションとして追加した。

以下は、AMD Ryzen 5 3600 と Apple M1 のベンチマーク結果である。どちらも JDK 21 を使用している。結果は秒あたりの操作回数で示されている。Baseline は構造的再帰を使用しているインタープリタで、Stack インタープリタはスタックとプログラムを `List` で表現している。Optimized Stack はスタックとプログラムを配列で表現しており、その他のインタープリタは Optimized Stack を基盤として上述の最適化を追加している。All インタープリタはすべての最適化を取り入れたものである。

+--------------------------+---------+---------+---------+---------+
| Interpreter              | Ryzen 5 | Speedup | M1      | Speedup |
+==========================+=========+=========+=========+=========+
| Baseline                 | 2754.43 | 1       | 3932.93 | 1       |
+--------------------------+---------+---------+---------+---------+
| Stack                    | 676.43  | 0.25    | 1004.16 | 0.26    |
+--------------------------+---------+---------+---------+---------+
| Optimized Stack          | 3631.19 | 1.32    | 2953.21 | 0.75    |
+--------------------------+---------+---------+---------+---------+
| Algebraic Simplification | 1630.93 | 0.59    | 4818.45 | 1.23    |
+--------------------------+---------+---------+---------+---------+
| Byte Code                | 4057.11 | 1.47    | 3355.75 | 0.85    |
+--------------------------+---------+---------+---------+---------+
| Stack Caching            | 3698.10 | 1.34    | 3237.17 | 0.82    |
+--------------------------+---------+---------+---------+---------+
| Superinstructions        | 3706.10 | 1.35    | 4689.02 | 1.19    |
+--------------------------+---------+---------+---------+---------+
| All                      | 7612.45 | 2.76    | 7098.06 | 1.80    |
+--------------------------+---------+---------+---------+---------+

この結果から学べることはいくつかあるが、もっとも重要なのは*性能は合成的ではない*という点だと思う。ふたつの最適化を適用した結果は、それぞれの最適化を個別に適用した場合の性能向上の単純な合計になるわけではない。ほとんどの最適化は Optimized Stack インタープリタと比較して個別ではほとんど性能に影響を与えていない。しかし、それらを組み合わせると大幅な改善がもたらされる。

基本的な構造的再帰を用いる Baseline インタープリタは驚くほど高速で、Ryzen 5 では Optimized Stack インタープリタよりすこし遅いが、M1 では逆に高速である。スタックマシンはプロセッサの組み込みコールスタックをエミュレートする。ネイティブのコールスタックは極めて高速なので、それを使わないことには相応の理由が必要になる。

最適化では細部が非常に重要である。Stack と Optimized Stack の間に見られる大きな違いは、データ構造の選択によって生み出されたものである。Byte Code インタープリタの初期バージョンは Optimized Stack よりも性能が悪かった。リテラルをバイトコードと一緒に保存していたのと、`Double` 値を `Array[Byte]` から `ByteBuffer` を使用して読み込む処理が低速だったのが原因だと思われる。スーパーインストラクションは、どのような命令を統合したか、その選択に大きく依存する。スタック上の値にリテラルを加算するスーパーインストラクションは単体ではほとんど効果がなく、むしろ Ryzen 5 では性能が低下した。

コンパイラ、特に JIT コンパイラは理解が難しい。たとえば、Algebraic Simplification インタープリタが Ryzen 5 で非常に遅い理由は説明できない。このインタープリタが行う作業は Optimized Stack インタープリタと比べて明らかに少ない。今回実装したインタープリタ最適化と同様に、コンパイラの最適化はアルゴリズムによって認識される特定のケースでのみ機能する。アルゴリズムが期待するパターンにコードが合致しない場合、最適化は適用されず、奇妙な性能の急落を招くことがある。おそらく、今回の実装の何らかの部分がそのような問題に引っかかったのだろう。

最後に、プラットフォーム間の違いも重要である。性能の違いのうち、どの程度がコンピュータのアーキテクチャによるもので、どの程度が JVM の違いによるものなのかを知るのは難しい。いずれにせよ、大多数のユーザがどのプラットフォームでプログラムを実行することになるのかを意識し、あるプラットフォームでの性能が他のプラットフォームにそのまま適用されるとナイーブに想定しないようにすべきである。
