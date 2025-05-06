<!--

## Codata Interpreters

In this section we'll explore codata interpreters, using a DSL for terminal interaction as a case study.
The terminal is familiar to most programmers, and terminal applications are common for developer focused tools. Most terminal features are controlled by writing so-called escape codes to the terminal. However, applications benefit from higher-level abstractions, motivating textual user interface (TUI) libraries that present a more ergonomic interface[^tuis]. Our library will showcase codata interpreters, monads, and the central role of designing for composition and reasoning. 


### The Terminal

The modern terminal is an accretion of features that started with the VT-100 in 1978 and continues [to this day][kitty-kp].
Most terminal features are accessed by reading and writing ANSI escape codes, which are sequence of characters starting with the escape character.
We will work only with escape codes that change the text style.
This allows us to produce interesting output, and raises all the design issues we want to address, but keeps the system simple.
The ideas here are extended to a more complete system in the [Terminus][terminus] library.

The code below is written so that with a single change it can pasted into a file and run with any recent version of Scala with just `scala <filename>`.
The required change is to add the `@main` annotation before the method `go`.
That is, change 

`def go(): Unit =`

to 

`@main def go(): Unit =`

(This is due to a limitation of the software that compiles the code in the book.)

The examples should work with any terminal from the last 40 odd years.
If you're on Windows you can use Windows Terminal, [WSL][wsl], or another terminal that runs on Windows such as [WezTerm][wezterm].


### Color Codes

We will start by writing color codes straight to the terminal.
This will introduce us to controlling the terminal, and show the problems of using ANSI escape codes directly.
Here's our starting point:

```scala mdoc:reset-object:silent
val csiString = "\u001b["

def printRed(): Unit =
  print(csiString)
  print("31")
  print("m")

def printReset(): Unit =
  print(csiString)
  print("0")
  print("m")

def go(): Unit =
  print("Normal text, ")
  printRed()
  print("now red text, ")
  printReset()
  println("and now back to normal.")
```

Try running the above code (e.g. add the `@main` annotation to `go`, save it to a file `ColorCodes.scala` and run `scala ColorCodes.scala`.) 
You should see text in the normal style for your terminal, followed by text colored red, and then some more text in the normal style.
The change in color is controlled by writing escape codes.
These are strings starting with `ESC` (which is the character `'\u001b'`) followed by `'['`.
This is the value of `csiString` (where CSI stands for Control Sequence Introducer).
The CSI is followed by a string indicating the text style to use, and ended with a `"m"`
The string `"\u001b[31m"` tells the terminal to set the text foreground color to red, and the 
string `"\u001b[0m"` tells the terminal to reset all text styling to the default.


### The Trouble with Escape Codes

Escape codes are simple for the terminal to process but lack useful structure for the programmer generating them.
The code above shows one potential problem: we must remember to reset the color when we finish a run of styled text. This problem is no different to that of remembering to free manually allocated memory, and the long history of memory safety problems in C programs show us that we cannot expect to do this reliably. Luckily, we're unlikely to crash our program if we forget an escape code!

To solve this problem we might decide to write functions like `printRed` below, which prints a colored string and resets the styling afterwards.

```scala mdoc:reset-object:silent
val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"

def printRed(output: String): Unit =
  print(redCode)
  print(output)
  print(resetCode)

def go(): Unit =
  print("Normal text, ")
  printRed("now red text, ")
  println("and now back to normal.")
```

Changing color is the not the only way that we can style terminal output. We can also, for example, turn text bold. Continuing the above design gives us the following.

```scala mdoc:reset-object:silent
val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"
val boldOnCode = s"${csiString}1m"
val boldOffCode = s"${csiString}22m"

def printRed(output: String): Unit =
  print(redCode)
  print(output)
  print(resetCode)

def printBold(output: String): Unit =
  print(boldOnCode)
  print(output)
  print(boldOffCode)

def go(): Unit =
  print("Normal text, ")
  printRed("now red text, ")
  printBold("and now bold.\n")
```

This works, but what if we want text that is *both* red and bold? We cannot express this with our current design, without creating methods for every possible combination of styles. Concretely this means methods like

```scala
def printRedAndBold(output: String): Unit =
  print(redCode)
  print(boldOnCode)
  print(output)
  print(resetCode)
```

This is not feasible to implement for all possible combinations of styles. The root problem is that our design is not compositional: there is no way to build a combination of styles from smaller pieces.


### Programs and Interpreters

To solve the problem above we need `printRed` and `printBold` to accept not a `String` to print but a program to run. 
We don't need to know what these programs do; we just need a way to run them.
Then the combinators `printRed`, `printBold`, and so on, can also return programs.
These programs will set the style appropriately before running their program parameter, and reset it after the parameter program has finished running.
By accepting and returning programs the combinators have the property of closure, meaning that type of the input (a program) is the same as the type of the outpt. Closure in turn makes composition possible.

How should we represent a program?
We will choose codata and in particular functions, the simplest form of codata.
In the code below we define the type `Program[A]`, which is a function `() => A`.
The interpreter, which is the thing that runs programs, is just function application.
To make it clearer when we are running programs I have a created method `run` that does just that.


```scala mdoc:reset-object:silent
type Program[A] = () => A

val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"
val boldOnCode = s"${csiString}1m"
val boldOffCode = s"${csiString}22m"

def run[A](program: Program[A]): A = program()

def print(output: String): Program[Unit] =
  () => Console.print(output)

def printRed[A](output: Program[A]): Program[A] =
  () => {
    run(print(redCode))
    val result = run(output)
    run(print(resetCode))
    
    result
  }


def printBold[A](output: Program[A]): Program[A] = 
  () => {
    run(print(boldOnCode))
    val result = run(output)
    run(print(boldOffCode))
    
    result
  }


def go(): Unit =
  run(() => {
    run(print("Normal text, "))
    run(printRed(print("now red text, ")))
    run(printBold(print("and now bold ")))
    run(printBold(printRed(print("and now bold and red.\n"))))
  })
```

Notice that we have the usual structure for an algebra, which we first met in Section [@sec:interpreters:structure]:

1. we have a constructor in `print`;
2. we have two combinators in `printRed` and `printBold`; and
3. we have an interpreter in `run`.

This code works, for the example we have chosen, but there are two issues: composition and ergonomics.
That we have a problem with composition is perhaps surprising, as that's the problem we set out to solve.
We have made the system compositional in some aspects, but there are still ways in which it does not work correctly.
For example, take the following code:

```scala mdoc:compile-only
run(printBold(() => {
  run(print("This should be bold, "))
  run(printBold(print("as should this ")))
  run(print("and this.\n"))
}))
```

We would expect output like 

**This should be bold, as should that and this**

but we get 

**This should be bold, as should this** and this. 

The inner call to `printBold` resets the bold styling when it finishes, which means the surrounding call to `printBold` does not have effect on later statements.

The issue with ergonomics is that this code is tedious and error-prone to write. We have to pepper calls to `run` in just the right places, and even in these small examples I found myself making mistakes. This is actually another failing of composition, because we don't have methods to combine together programs. For example, we don't have methods to say that the program above is the sequential composition of three sub-programs.

We can solve the first problem by keeping track of the state of the terminal. If `printBold` is called within a state that is already printing bold it should do nothing, otherwise it should update the state to indicate bold styling has been turned on. This means the type of programs changes from `() => A` to `Terminal => (Terminal, A)`, where `Terminal` holds the current state of the terminal.

To solve the second problem we're looking for a way to sequentially compose programs. Remember programs have type `Terminal => (Terminal, A)` and pass around the state in `Terminal`. When you hear the phrase "sequentially compose", or see that type, your monad sense might start tingling. You are correct: this is an instance of the state monad, which we first met in Section [@sec:monad:state]. 

Using Cats we can define

```scala mdoc:reset:silent
import cats.data.State
type Program[A] = State[Terminal, A]
```

assuming some suitable definition of `Terminal`. Let's accept this definition for now, and focus on defining `Terminal`.

`Terminal` has two pieces of state: the current bold setting and the current color. (The real terminal has much more state, but these are representative and modelling additional state does not introduce any new concepts.) The bold setting could simply be a toggle that is either on or off, but when we come to the implementation it will be easier to work with a counter that records the depth of the nesting. The current color must be a stack. We can nest color changes, and the color should change back to the surrounding color when a nested level exits. Concretely, we should be able to write code like

```scala
printBlue(.... printRed(...) ...)
```

and have output in blue or red as we would expect.

Given this we can define `Terminal` as

```scala mdoc:silent
final case class Terminal(bold: Int, color: List[String]) {
  def boldOn: Terminal = this.copy(bold = bold + 1)
  def boldOff: Terminal = this.copy(bold = bold - 1)
  def pushColor(c: String): Terminal = this.copy(color = c :: color)
  // Only call this when we know there is at least one color on the
  // stack
  def popColor: Terminal = this.copy(color = color.tail)
  def peekColor: Option[String] = this.color.headOption
}
```

where we use `List` to represent the stack of color codes. (We could also use a mutable stack, as working with the state monad ensures the state will be threaded through our program.) I've also defined some convenience methods to simplify working with the state.

With this in place we can write the rest of the code, which is shown below. Compared to the previous code I've shortened a few method names and abstracted the escape codes.
Remember this code can be directly executed by `scala`. Just copy it into a file (e.g. `Terminal.scala`), add the `@main` annotation to `go`, and run `scala Terminal.scala`. 

```scala mdoc:reset-object:silent
//> using dep org.typelevel::cats-core:2.13.0

import cats.data.State
import cats.syntax.all.*

object AnsiCodes {
  val csiString: String = "\u001b["

  def csi(arg: String, terminator: String): String =
    s"${csiString}${arg}${terminator}"

  // SGR stands for Select Graphic Rendition. 
  // All the codes that change formatting are SGR codes.
  def sgr(arg: String): String =
    csi(arg, "m")

  val reset: String = sgr("0")
  val boldOn: String = sgr("1")
  val boldOff: String = sgr("22")
  val red: String = sgr("31")
  val blue: String = sgr("34")
}

final case class Terminal(bold: Int, color: List[String]) {
  def boldOn: Terminal = this.copy(bold = bold + 1)
  def boldOff: Terminal = this.copy(bold = bold - 1)
  def pushColor(c: String): Terminal = this.copy(color = c :: color)
  // Only call this when we know there is at least one color on the
  // stack
  def popColor: Terminal = this.copy(color = color.tail)
  def peekColor: Option[String] = this.color.headOption
}
object Terminal {
  val empty: Terminal = Terminal(0, List.empty)
}

type Program[A] = State[Terminal, A]
object Program {
  def print(output: String): Program[Unit] =
    State[Terminal, Unit](
      terminal => (terminal, Console.print(output))
    )

  def bold[A](program: Program[A]): Program[A] =
    for {
      _ <- State.modify[Terminal] { terminal =>
        if terminal.bold == 0 then Console.print(AnsiCodes.boldOn)
        terminal.boldOn
      }
      a <- program
      _ <- State.modify[Terminal] { terminal =>
        val newTerminal = terminal.boldOff
        if terminal.bold == 0 then Console.print(AnsiCodes.boldOff)
        newTerminal
      }
    } yield a

  // Helper to construct methods that deal with color
  def withColor[A](code: String)(program: Program[A]): Program[A] =
    for {
      _ <- State.modify[Terminal] { terminal =>
        Console.print(code)
        terminal.pushColor(code)
      }
      a <- program
      _ <- State.modify[Terminal] { terminal =>
        val newTerminal = terminal.popColor
        newTerminal.peekColor match {
          case None    => Console.print(AnsiCodes.reset)
          case Some(c) => Console.print(c)
        }
        newTerminal
      }
    } yield a

  def red[A](program: Program[A]): Program[A] =
    withColor(AnsiCodes.red)(program)

  def blue[A](program: Program[A]): Program[A] =
    withColor(AnsiCodes.blue)(program)

  def run[A](program: Program[A]): A =
    program.runA(Terminal.empty).value
}

def go(): Unit = {
  val program =
    Program.blue(
      Program.print("This is blue ") >>
        Program.red(Program.print("and this is red ")) >>
        Program.bold(Program.print("and this is blue and bold "))
    ) >>
      Program.print("and this is back to normal.\n")

  Program.run(program)
}

```

Having defined the structure of `Terminal`, the majority of the remaining code manipulates the `Terminal` state. Most of the methods on `Program` have a common structure that specifies a state change before and after the main program runs.

Notice we don't need to implement combinators like `flatMap` or `>>` because we get them from the `State` monad. This is one of the big benefits of reusing abstractions like monads: we get a full library of methods without doing additional work.


### Composition and Reasoning

In Section [@sec:what-is-fp] I argued that the core of functional programming is reasoning and composition. Both of these are central to this case study. We've explicitly designed the DSL for ease of reasoning. Indeed that's the whole point of creating a DSL instead of just spitting control codes at the terminal. An example is how we paid attention to making sure nested calls work as we'd expect. Composition comes in at two levels: both our design and our implementation are compositional. Within the case study we discussed compositionality in the design. Implementationally, a `Program` is a composition of the state monad and the functions inside the state monad. The state monad provides the sequential flow of the `Terminal` state, and the functions provide the domain specific actions.


### Codata and Extensibility

We made a seemingly arbitrary choice to use a codata interpreter. Let's now explore this choice and its implications.

We described codata as programming to an interface. The interface for functions is essentially one method: the ability to apply them. This corresponds to the single interpretation we have for `Program`: run it and carry out the effects therein. If we wanted to have multiple interpretations (such as logging the `Terminal` state or saving the output to a buffer) we would need to have a richer interface. In Scala this would be a `trait` or `class` exposing more than one method.

Keen readers will recall that data makes it easy to add new interpreters but hard to add new operations, while codata makes it easy to add new operations but hard to add new interpreters. We see that in action here. For example, it's trivial to add a new color combinator by defining a method like the below.

```scala
def green[A](program: Program[A]): Program[A] =
  withColor(AnsiCodes.sgr("32"))(program)
```

However, changing `Program` to something that allows more interpretations requires changing all of the existing code.

Another advantage of codata is that we can mix in arbitrary other Scala code. For example, we can use `map` like shown below.

```scala
Program.print("Hello").map(_ => 42)
```

Using the native representation of programs (i.e. functions) gives us the entire Scala language for free. In a data representation we have to reify every kind of expression we wish to support. There is a downside to this as well: we get Scala semantics whether we like them or not. A codata representation would not be appropriate if we wanted to make an exotic language that worked in a different way.

We could factor the interpreter in different ways, and it would still be a codata interpreter. For example, we could put a method to write to the terminal on the `Terminal` type. This would give us a bit more flexibility as changing the implementation of `Terminal` could, say, write to a network socket or a terminal embedded in a browser. We still have the limitation that we cannot create truly different interpretations, such as serializing programs to disk, with the codata approach. We'll address this limitation in the next section where we look at tagless final.


[^tuis]: If you're interested in [TUI][tui] libraries you might like to look at the brilliantly named [ratatui](https://github.com/ratatui/ratatui)  for Rust, [brick](https://github.com/jtdaugherty/brick) for Haskell, or [Textual](https://textual.textualize.io/) for Python.

[terminus]: https://www.creativescala.org/terminus/
[wsl]: https://learn.microsoft.com/en-us/windows/wsl/about
[wezterm]: https://wezfurlong.org/wezterm/index.html
[tui]: https://en.wikipedia.org/wiki/Text-based_user_interface
[fp]: @/posts/2020-07-05-what-and-why-fp.md


```scala mdoc:reset:silent
```
--->

## 余データ的インタープリタ

本節では、ターミナルでの対話のための DSL を題材として余データ的インタープリタを探求する。ターミナルは多くのプログラマにとって馴染み深く、そこで使われる CLI アプリケーションは開発者向けのツールとして一般的である。ターミナルの機能はいわゆるエスケープシーケンスを書き込むことで制御されることがよくある。しかし、もっと高水準な抽象が提供されれば、アプリケーションの利便性は向上する。そこで、より使いやすいインターフェースを提供するテキストユーザインターフェース（TUI）ライブラリを作りたい[^tuis]。本節で構築するライブラリは、余データ的インタープリタやモナド、そして合成と推論のための設計が果たす中心的な役割について見せてくれるはずである。


### ターミナル

現在のターミナルは、1978年に登場した VT-100 に始まり[今日に至るまで][kitty-kp]機能が集積してできたものである。多くのターミナル機能は ANSI エスケープシーケンスの読み書きによって利用できる。エスケープシーケンスとは、エスケープ文字を先頭とする文字列のことをいう。ここでは、文字スタイルを変更するためのエスケープシーケンスだけを扱う。これにより、システムをシンプルに保ったまま、興味深い成果を得るとともに設計上の論点をひととおり明らかにできる。ここで示すアイデアは、[Terminus][terminus] ライブラリの中で、より本格的なシステムへと拡張されている。

以下に示すコードは、わずかな変更を加えるだけで、ファイルに貼りつけて Scala の最近のバージョンで `scala <ファイル名>` としてそのまま実行できるように書かれている。必要な変更とは `go` 関数の前に `@main` アノテーションを追加することである。つまり、

`def go(): Unit =`

を 

`@main def go(): Unit =`

に変えればよい（これは、本書に記述されたコードをコンパイルするソフトウェア[^tn-tagless-final-codata-01]の制約による）。

例は過去40年ほどのターミナルであればどれでも動作するはずである。Windows 環境では Windows Terminal や [WSL][wsl]、あるいは [WezTerm][wezterm] のような Windows 上で動作するターミナルを使えばよい。

### カラーコード

ターミナルに直接カラーコードを書き込むところから始めよう。これによりターミナル制御の基本を学ぶことができ、同時に ANSI エスケープシーケンスを直接使用することの問題点が明らかとなる。以下が出発点となるコードである。

```scala mdoc:reset-object:silent
val csiString = "\u001b["

def printRed(): Unit =
  print(csiString)
  print("31")
  print("m")

def printReset(): Unit =
  print(csiString)
  print("0")
  print("m")

def go(): Unit =
  print("Normal text, ")
  printRed()
  print("now red text, ")
  printReset()
  println("and now back to normal.")
```

上述のコードを試してみよう。たとえば、`go` 関数に `@main` アノテーションを追加してファイル `ColorCodes.scala` に保存し、`scala ColorCodes.scala` を実行すればよい。ターミナルの通常のスタイルで表示されるテキストに続いて赤色のテキストが表示され、その後、通常のスタイルに戻ったテキストが続くのが見られるはずである。

色の変更はエスケープシーケンスの書き込みによって制御されている。そのシーケンスは `ESC`（文字 `'\u001b'`）に `'['` が続く文字列で、これが `csiString` の値である。CSI は Control Sequence Introducer を意味している。CSI の後に、使用したいテキストスタイルを指示する文字列を続け、最後に `"m"` を付ける。文字列 `"\u001b[31m"` はターミナルに対して文字色を赤にするよう指示するもので、文字列 `"\u001b[0m"` は、すべてのテキストスタイルをデフォルトに戻すよう指示するものである。

### エスケープシーケンスの問題点

エスケープシーケンスは、ターミナルが処理するのは単純だが、それを生成するプログラマにとっては有用な構造を欠いている。上記のコードにはひとつの潜在的な問題が示されている。スタイル付きのテキストを出力し終えたら、忘れずに色をリセットしなければならないということである。この問題は、手動で確保したメモリを解放し忘れないようにする問題と何ら変わらない。そして、C言語プログラムのメモリ安全性問題に関する長い歴史が示すとおり、この種の作業を人間が確実にこなすことは期待できない。幸いなことに、エスケープシーケンスを忘れてもプログラムがクラッシュする可能性は低いが。

この問題を解決するために、次のような `printRed` 関数を書くことが考えられる。この関数は、文字列を赤色で出力し、その後スタイルをリセットする。


```scala mdoc:reset-object:silent
val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"

def printRed(output: String): Unit =
  print(redCode)
  print(output)
  print(resetCode)

def go(): Unit =
  print("Normal text, ")
  printRed("now red text, ")
  println("and now back to normal.")
```

ターミナル出力のスタイリングは文字色を変えることだけではない。たとえば、文字を太字にすることもできる。上記の設計を引き継ぐとコードは次のようになる。

```scala mdoc:reset-object:silent
val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"
val boldOnCode = s"${csiString}1m"
val boldOffCode = s"${csiString}22m"

def printRed(output: String): Unit =
  print(redCode)
  print(output)
  print(resetCode)

def printBold(output: String): Unit =
  print(boldOnCode)
  print(output)
  print(boldOffCode)

def go(): Unit =
  print("Normal text, ")
  printRed("now red text, ")
  printBold("and now bold.\n")
```

これでも動作はする。しかし、テキストを赤色かつ太字で出力したい場合はどうだろうか。現在の設計ではこれを表現する方法がなく、すべてのスタイルの組み合わせに対して個別の関数を作るしかない。具体的には、次のような関数を書く必要がある。

```scala
def printRedAndBold(output: String): Unit =
  print(redCode)
  print(boldOnCode)
  print(output)
  print(resetCode)
```

すべてのスタイルの組み合わせについてこのような実装を行うのは現実的でない。根本的な問題は、現在の設計が合成的でないことである。小さな部品を合成してスタイルの組み合わせを構築する手段が存在しない。

### プログラムとインタープリタ

上記の問題を解決するには、`printRed` や `printBold` が、出力対象となる `String` ではなく実行するプログラムを受け取るようにする必要がある。それらのプログラムが具体的に何をするかを知る必要はない。それらを実行する手段があればよい。そうすれば、`printRed` や `printBold` といったコンビネータもまたプログラムを返すことができるようになる。これらのコンビネータが返すプログラムは、まず適切にスタイルを設定し、それから受け取ったプログラムを実行し、終了後にスタイルをリセットする。

プログラムを受け取りプログラムを返すことにより、これらのコンビネータは**閉包性（closure）**をもつ。入力（プログラム）の型と出力の型が同じということである。閉包性があることによって合成が可能となる。

プログラムはどのように表現すればよいだろうか。ここでは余データ、特にそのもっとも単純な形である関数を選ぶ。以下のコードでは型 `Program[A]` を `() => A` という関数として定義している。インタープリタ、すなわちプログラムを実行するものは、単なる関数適用である。ここでは、プログラムを実行していることがより明確にわかるように、単に関数適用するだけの `run` メソッドを作成した。

```scala mdoc:reset-object:silent
type Program[A] = () => A

val csiString = "\u001b["
val redCode = s"${csiString}31m"
val resetCode = s"${csiString}0m"
val boldOnCode = s"${csiString}1m"
val boldOffCode = s"${csiString}22m"

def run[A](program: Program[A]): A = program()

def print(output: String): Program[Unit] =
  () => Console.print(output)

def printRed[A](output: Program[A]): Program[A] =
  () => {
    run(print(redCode))
    val result = run(output)
    run(print(resetCode))
    
    result
  }


def printBold[A](output: Program[A]): Program[A] = 
  () => {
    run(print(boldOnCode))
    val result = run(output)
    run(print(boldOffCode))
    
    result
  }


def go(): Unit =
  run(() => {
    run(print("Normal text, "))
    run(printRed(print("now red text, ")))
    run(printBold(print("and now bold ")))
    run(printBold(printRed(print("and now bold and red.\n"))))
  })
```

このコードには、[@sec:interpreters:structure]節で初めて見たような、代数的な構造が備わっていることに注目してほしい。

1. `print` がコンストラクタ
2. `printRed` と `printBold` がコンビネータ
3. `run` がインタープリタ

先ほど用いた例においてこのコードは正しく動作するが、問題がふたつある。ひとつは合成、もうひとつは使い勝手である。私たちはここでまさに合成の問題を解決しようとしていたはずなので、そこに問題があるというのは意外に思えるかもしれない。確かに、ある側面ではこのシステムは合成的になった。だが、それでもなお正しく機能しないケースがある。たとえば、次のコードを見てほしい。

```scala mdoc:compile-only
run(printBold(() => {
  run(print("ここは太字になるはずで、"))
  run(printBold(print("ここも太字のはず。")))
  run(print("ついでにここも太字。\n"))
}))
```

出力は次のようになる想定だが、

**ここは太字になるはずで、ここも太字のはず。ついでにここも太字。**

実際には次のようになる。

**ここは太字になるはずで、ここも太字のはず。**ついでにここも太字。

内側の `printBold` 呼び出しが終了時に太字スタイルをリセットしてしまい、それにより外側の `printBold` の効果がその後のステートメントに及ばなくなってしまう。

使い勝手の問題というのは、このコードを書くのが煩雑かつミスを招きやすいという点にある。`run` の呼び出しを正しい場所にちりばめる必要があり、この程度の小さな例であっても、筆者自身いくつか間違えた。実のところこれは合成に関するもうひとつの問題でもある。この問題はプログラムを組み合わせるためのメソッドが存在しないことに起因するからである。たとえば、上述のプログラムが三つのサブプログラムの逐次的合成であることを記述する方法がない。

最初の問題はターミナルの状態を追跡することで解決できる。たとえば、`printBold` がすでに太字出力中の状態で呼び出された場合には何もせず、そうでなければ太字スタイルが設定されたことを示すよう状態を更新すればよい。これは、プログラムの型が `() => A` から `Terminal => (Terminal, A)` に変わることを意味する。`Terminal` はターミナルの現在の状態を保持する型である。

もうひとつの問題を解決するには、プログラムを逐次的に合成する方法が求められる。プログラムは `Terminal => (Terminal, A)` という型をもち、状態を `Terminal` の中にいれて引き回すということを思い出そう。「逐次的に合成する」という言葉やこのような型を見て、そこにモナドが関わってくることを感じ取ったかもしれない。そのとおり、これは[@sec:monad:state]節で見た状態モナドの一例である。

Cats を使えば `Program[A]` は以下のように定義できる。

```scala mdoc:reset:silent
import cats.data.State
type Program[A] = State[Terminal, A]
```

ただし、これは `Terminal` が適切に定義されていることを仮定している。まずはこの定義を受け入れ、`Terminal` の定義に焦点を移そう。

`Terminal` は、現在の太字設定と現在の文字色というふたつの状態をもっている。　実際のターミナルはもっと多くの状態をもつが、このふたつをその代表的なものと考えておく。他の状態をモデル化したとしても新しい概念が増えるわけではない。太字設定はオンとオフを切り替えられる単純なトグルでもよいが、実装の都合を考えると、ネストの深さを記録するカウンタとしたほうが取り扱いやすい。現在の文字色はスタックで表現する必要がある。文字色の変更はネストさせることができ、ネストから抜けるときには色を元に戻すべきだからである。具体的に言えば、以下のようなコードを記述でき、期待通りに青と赤を切り替えながら出力されるようにしたい。

```scala
printBlue(.... printRed(...) ...)
```

以上をふまえると `Terminal` は次のように定義できる。

```scala mdoc:silent
final case class Terminal(bold: Int, color: List[String]) {
  def boldOn: Terminal = this.copy(bold = bold + 1)
  def boldOff: Terminal = this.copy(bold = bold - 1)
  def pushColor(c: String): Terminal = this.copy(color = c :: color)
  // 文字色が最低ひとつはスタックに積まれているときにのみ呼び出す
  def popColor: Terminal = this.copy(color = color.tail)
  def peekColor: Option[String] = this.color.headOption
}
```

ここではカラーコードのスタックを表現するのに `List` を使っている。状態モナドを用いることによって、プログラム全体に状態が適切に伝播することが保証されているため、可変スタックを使っても構わない。また、状態操作を簡潔にするために、補助メソッドもいくつか定義してある。

準備は整ったので、残りのコードを書いていこう。以下にそのコードを示す。前回のコードと比べて、メソッド名をいくつか短くし、エスケープシーケンスの抽象化を行っている。前述のとおり、このコードは `scala` コマンドでそのまま実行できる。ファイル（たとえば `Terminal.scala`）に貼りつけて `go` 関数に `@main` アノテーションを追加し、`scala Terminal.scala` を実行すればよい。

```scala mdoc:reset-object:silent
//> using dep org.typelevel::cats-core:2.13.0

import cats.data.State
import cats.syntax.all.*

object AnsiCodes {
  val csiString: String = "\u001b["

  def csi(arg: String, terminator: String): String =
    s"${csiString}${arg}${terminator}"

  // SGR は Select Graphic Rendition の略
  // スタイルを変更するエスケープシーケンスはすべて SGR である
  def sgr(arg: String): String =
    csi(arg, "m")

  val reset: String = sgr("0")
  val boldOn: String = sgr("1")
  val boldOff: String = sgr("22")
  val red: String = sgr("31")
  val blue: String = sgr("34")
}

final case class Terminal(bold: Int, color: List[String]) {
  def boldOn: Terminal = this.copy(bold = bold + 1)
  def boldOff: Terminal = this.copy(bold = bold - 1)
  def pushColor(c: String): Terminal = this.copy(color = c :: color)
  // 文字色が最低ひとつはスタックに積まれているときにのみ呼び出す
  def popColor: Terminal = this.copy(color = color.tail)
  def peekColor: Option[String] = this.color.headOption
}
object Terminal {
  val empty: Terminal = Terminal(0, List.empty)
}

type Program[A] = State[Terminal, A]
object Program {
  def print(output: String): Program[Unit] =
    State[Terminal, Unit](
      terminal => (terminal, Console.print(output))
    )

  def bold[A](program: Program[A]): Program[A] =
    for {
      _ <- State.modify[Terminal] { terminal =>
        if terminal.bold == 0 then Console.print(AnsiCodes.boldOn)
        terminal.boldOn
      }
      a <- program
      _ <- State.modify[Terminal] { terminal =>
        val newTerminal = terminal.boldOff
        if terminal.bold == 0 then Console.print(AnsiCodes.boldOff)
        newTerminal
      }
    } yield a

  // 文字色を取り扱うメソッドを構築するためのヘルパーメソッド
  def withColor[A](code: String)(program: Program[A]): Program[A] =
    for {
      _ <- State.modify[Terminal] { terminal =>
        Console.print(code)
        terminal.pushColor(code)
      }
      a <- program
      _ <- State.modify[Terminal] { terminal =>
        val newTerminal = terminal.popColor
        newTerminal.peekColor match {
          case None    => Console.print(AnsiCodes.reset)
          case Some(c) => Console.print(c)
        }
        newTerminal
      }
    } yield a

  def red[A](program: Program[A]): Program[A] =
    withColor(AnsiCodes.red)(program)

  def blue[A](program: Program[A]): Program[A] =
    withColor(AnsiCodes.blue)(program)

  def run[A](program: Program[A]): A =
    program.runA(Terminal.empty).value
}

def go(): Unit = {
  val program =
    Program.blue(
      Program.print("This is blue ") >>
        Program.red(Program.print("and this is red ")) >>
        Program.bold(Program.print("and this is blue and bold "))
    ) >>
      Program.print("and this is back to normal.\n")

  Program.run(program)
}

```

`Terminal` の構造を定義したので、残りのコードの大部分は `Terminal` の状態操作となる。`Program` オブジェクトに定義したメソッドの多くは、メインのプログラムを実行する前後に状態変更を行うという共通の構造をもっている。

ここで注目すべきなのは、`flatMap` や `>>` といったコンビネータを自分で実装する必要がないという点である。これらは `State` モナドから自動的に得られる。これはモナドのような抽象を再利用することで得られる大きな利点のひとつである。追加の実装をしなくても豊富なメソッド群をすぐに使うことができる。

### 合成と推論

[@sec:what-is-fp]節では関数型プログラミングの核心は推論と合成であると述べた。このふたつは本ケーススタディにおいても中心的な位置を占めている。ここでは推論を容易にするために DSL を明示的に設計した。制御コードをターミナルに直接吐き出すのではなく DSL を構築したのは、すべてまさにそのためである。その一例が、ネストされた呼び出しが期待どおりに動作するよう注意を払った点である。

合成はふたつのレベルで現れる。今回の設計と実装、いずれもが合成的である。設計における合成性についてはケーススタディの中ですでに論じた。実装においても、`Program` は状態モナドとその中に含まれる関数の合成である。状態モナドは `Terminal` 状態の逐次的な変化の流れを、関数はドメイン固有の動作を、それぞれ提供してくれる。

### 余データと拡張性

余データ的インタープリタを選んだのは一見すると恣意的に思えるかもしれない。ここでは、その選択の理由と含意について探っていきたい。

余データは「インターフェースに対してプログラムを書く」ものであると述べた。関数におけるインターフェースは基本的にひとつのメソッド、すなわち関数適用の能力である。これは、`Program` に対する解釈が、それを実行して内部に記述された効果を発動する、というひとつだけであったことと対応している。もし、`Terminal` の状態をログに記録したり、出力をバッファに保存したりといった複数の解釈をもたせたいのであれば、もっと豊かなインターフェースが必要となる。Scala においては、複数のメソッドを公開する `trait` や `class` がそれにあたる。

注意深い読者は、データと余データにおける拡張性のトレードオフを思い出すかもしれない。データでは、新しいインタープリタを追加するのは容易だが、新しい操作を追加するのは難しい。これに対して余データでは、新しい操作の追加は容易だが、新しいインタープリタを追加するのは難しい。そのことは今まさに実例で示されている。たとえば、新しい文字色のコンビネータを追加するのはごく簡単である。次のようにメソッドを定義すればよい。

```scala
def green[A](program: Program[A]): Program[A] =
  withColor(AnsiCodes.sgr("32"))(program)
```

しかし `Program` に複数の解釈を許すような変更を加える場合、既存のコードすべてを書き換える必要が生じる。

余データのもうひとつの利点は Scala の任意のコードを混ぜ込めることである。たとえば、次のように `map` を使うことができる。

```scala
Program.print("Hello").map(_ => 42)
```

プログラムを関数として表現することで、Scala の言語機能すべてがそのまま使える。データによる表現で同じことを行おうとすれば、サポートしたいすべての構文要素をレイフィケーションしなければならない。もっとも、これには Scala のセマンティクスが望むと望まざるとにかかわらずそのままついてくるという欠点もある。もし独自のセマンティクスをもった異質な言語を作りたいのであれば、余データによる表現は適切ではない。

インタープリタの分割のしかたはいろいろあるが、それでもなお余データ的インタープリタであることに変わりはない。たとえば、ターミナルへの書き出し用のメソッドを `Terminal` 型にもたせることもできる。こうすれば実装に柔軟性が生まれ、`Terminal` の実装を変更することで、出力先をネットワークソケットやブラウザ上の仮想ターミナルなどに切り替えることも可能になる。しかしそれでも、プログラムをディスクにシリアライズするといった、まったく異なる種類の解釈を行うことは、余データ的手法では難しい。この制限については、次節で Tagless Final という手法を紹介することで対処していく。


[^tuis]: [TUI][tui] ライブラリに興味があるなら、Rust 向けにはラタトゥイユをもじったユーモラスな名前の [ratatui](https://github.com/ratatui/ratatui)、Haskell 向けには [brick](https://github.com/jtdaugherty/brick)、Python 向けには [Textual](https://textual.textualize.io/) を見てみるとよいかもしれない。

[^tn-tagless-final-codata-01] 【訳注】 mdoc

[terminus]: https://www.creativescala.org/terminus/
[wsl]: https://learn.microsoft.com/en-us/windows/wsl/about
[wezterm]: https://wezfurlong.org/wezterm/index.html
[tui]: https://en.wikipedia.org/wiki/Text-based_user_interface
[fp]: @/posts/2020-07-05-what-and-why-fp.md
