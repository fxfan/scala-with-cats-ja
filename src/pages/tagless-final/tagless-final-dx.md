<!--

## A Better Encoding

```scala mdoc:invisible
type Validation[A] = A => Either[String, A]

// The validation rule that always succeeds
def succeed[A](value: A): Either[String, A] = Right(value)

trait Controls[Ui[_]] {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout[Ui[_]] {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

The basic implementation of tagless final has quite a poor developer experience. Consider the refactoring of our example below.

```scala mdoc:silent:nest
def name[Ui[_]](controls: Controls[Ui]): Ui[String] =
  controls.textInput("What is your name?", "John Doe")
  
def rating[Ui[_]](controls: Controls[Ui]): Ui[Int] =
  controls.choice(
    "Tagless final is the greatest thing ever",
    Seq(
      "Strongly disagree" -> 1,
      "Disagree" -> 2,
      "Neutral" -> 3,
      "Agree" -> 4,
      "Strongly agree" -> 5
    )
  )
  
def quiz[Ui[_]](
    controls: Controls[Ui],
    layout: Layout[Ui]
): Ui[(String, Int)] =
  layout.and(name(controls), rating(controls))
```

This style of code quickly becomes tedious to write. The method signatures are quite involved, and passing the program algebras from method to method is annoying busy work.

An improvement is to make the program algebras `given` instances. If we define accessors

```scala mdoc:silent
object Controls {
  def apply[Ui[_]](using controls: Controls[Ui]): Controls[Ui] =
    controls
}

object Layout {
  def apply[Ui[_]](using layout: Layout[Ui]): Layout[Ui] =
    layout
}
```

we can then write

```scala mdoc:silent:nest
def name[Ui[_]: Controls]: Ui[String] =
  Controls[Ui].textInput("What is your name?", "John Doe")
  
def rating[Ui[_]: Controls]: Ui[Int] =
  Controls[Ui].choice(
    "Tagless final is the greatest thing ever",
    Seq(
      "Strongly disagree" -> 1,
      "Disagree" -> 2,
      "Neutral" -> 3,
      "Agree" -> 4,
      "Strongly agree" -> 5
    )
  )
  
def quiz[Ui[_]: Controls: Layout]: Ui[(String, Int)] =
  Layout[Ui].and(name, rating)
```

This is the encoding of tagless final that is common in the Scala community, but there is still a lot of notational overhead for the developer who has to write this code.
We can use Scala language features to reduce the overhead of writing code using a tagless final style to the point where is a simple as standard code.

We'll use a combination of five techniques:

1. creating a base type for program algebras;
2. making the program type a type member;
3. defining a type for programs;
4. defining constructors on companion objects; and
5. using extension methods for combinators.

This is quite involved, but each step is relatively simple. Let's see how it works.

Our first step is to create a base type for algebras. This is just a trait like

```scala mdoc:silent:nest
trait Algebra[Ui[_]]
```

Our program algebras extend this trait.

```scala mdoc:silent
trait Controls[Ui[_]] extends Algebra[Ui[_]]{
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout[Ui[_]] extends Algebra[Ui[_]]{
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

Now we make the program type a type member.

```scala mdoc:silent:nest
trait Algebra {
  type Ui[_]
}

trait Controls extends Algebra {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout extends Algebra {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

At this point we've made sufficient changes that our example program is meaningfully changed.
Our starting point was

```scala
def quiz[Ui[_]: Controls: Layout](
    controls: Controls[Ui],
    layout: Layout[Ui]
): Ui[(String, Int)] =
  Layout[Ui].and(
    Controls[Ui].textInput("What is your name?", "John Doe"),
    Controls[Ui].choice(
      "Tagless final is the greatest thing ever",
      Seq(
        "Strongly disagree" -> 1,
        "Disagree" -> 2,
        "Neutral" -> 3,
        "Agree" -> 4,
        "Strongly agree" -> 5
      )
    )
  )
```

With the changes above we can instead write

```scala
def quiz(using alg: Controls & Layout): alg.Ui[(String, Int)] =
  alg.and(
    alg.textInput("What is your name?", "John Doe"),
    alg.choice(
      "Tagless final is the greatest thing ever",
      Seq(
        "Strongly disagree" -> 1,
        "Disagree" -> 2,
        "Neutral" -> 3,
        "Agree" -> 4,
        "Strongly agree" -> 5
      )
    )
  )
```

The key changes are:

1. the program algebras are a single parameter to the method, which is possible because they extend a common base type;
2. the `Ui` type parameter is no longer needed, as it has become a type member; and
3. we must now use a dependent method to specify the result type.

Our next step is to define a type for programs. Programs are conceptually functions from an algebra to a program type, so we can define such a type.

```scala mdoc:silent
trait Program[-Alg <: Algebra, A] {
  def apply(alg: Alg): alg.Ui[A]
}
```

Pay particular attention to the result type, `alg.Ui[A]`. As `Program` requires a dependent method type it cannot be a standard function.

The example now becomes

```scala mdoc:silent
val quiz =
  new Program[Controls & Layout, (String, Int)] {
    def apply(alg: Controls & Layout) =
      alg.and(
        alg.textInput("What is your name?", "John Doe"),
        alg.choice(
          "Tagless final is the greatest thing ever",
          Seq(
            "Strongly disagree" -> 1,
            "Disagree" -> 2,
            "Neutral" -> 3,
            "Agree" -> 4,
            "Strongly agree" -> 5
          )
        )
      )
  }
```

Programs are now values instead of methods. 
Notice that first type parameter of `Program` declares all the program algebras the program requires. 
It's still quite involved to write this code, though we can simplify it a bit by using the *single abstract method* technique, which means a `trait` with a single abstract method (like `Program`) can be implemented with a function.

```scala mdoc:silent:nest
val quiz: Program[Controls & Layout, (String, Int)] =
  (alg: Controls & Layout) =>
    alg.and(
      alg.textInput("What is your name?", "John Doe"),
      alg.choice(
        "Tagless final is the greatest thing ever",
        Seq(
          "Strongly disagree" -> 1,
          "Disagree" -> 2,
          "Neutral" -> 3,
          "Agree" -> 4,
          "Strongly agree" -> 5
        )
      )
    )
```

Programs-as-values is the key that unlocks the next two improvements. The first is to define constructors as methods on companion objects. 

```scala mdoc:silent
object Controls {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Program[Controls, String] =
    alg => alg.textInput(label, placeholder, validation)

  def choice[A](
    label: String, 
    options: Seq[(String, A)]
  ): Program[Controls, A] =
    alg => alg.choice(label, options)
}
```

This works because methods can now return programs.

The second and final improvement is to define extension methods for combinators. Since we only have one combinator, `and`, that means a single extension method.

```scala mdoc:silent
extension [Alg <: Algebra, A](p: Program[Alg, A]) {
  def and[Alg2 <: Algebra, B](
    second: Program[Alg2, B]
  ): Program[Alg & Alg2 & Layout, (A, B)] =
    alg => alg.and(p(alg), second(alg))
}
```

Pay particular attention to how the types are defined for this extension method. We define the extension on a `Program` requiring algebras `Alg`. The parameter to the `and` method is a `Program` requiring algebras `Alg2`. The result requires algebras `Alg & Alg2 & Layout`, which is the union of the algebras required by the two programs and the `Layout` algebra. In this way the combinators build up the algebras required for the program.

The net result is that users can write

```scala mdoc:silent:nest
val quiz  =
  Controls
    .textInput("What is your name?", "John Doe")
    .and(
      Controls.choice(
        "Tagless final is the greatest thing ever",
        Seq(
          "Strongly disagree" -> 1,
          "Disagree" -> 2,
          "Neutral" -> 3,
          "Agree" -> 4,
          "Strongly agree" -> 5
        )
      )
    )
```

which looks just like normal code. The type of `quiz` shows that type inference has correctly inferred all the needed program algebras.

```scala mdoc
quiz
```

This encoding requires more work from the library developer. However this is a one off cost, and result is that library users write much simpler code. For most applications of tagless final I think this is an appropriate trade off.


```scala mdoc:reset:silent
```
--->

## よりよいエンコーディング

```scala mdoc:invisible
type Validation[A] = A => Either[String, A]

// 常に成功する検証ルール
def succeed[A](value: A): Either[String, A] = Right(value)

trait Controls[Ui[_]] {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout[Ui[_]] {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

Tagless Final の基本的な実装は、開発者体験としてはやや残念なものだった。先ほどの例をリファクタリングすることを考えてみよう。

```scala mdoc:silent:nest
def name[Ui[_]](controls: Controls[Ui]): Ui[String] =
  controls.textInput("What is your name?", "John Doe")
  
def rating[Ui[_]](controls: Controls[Ui]): Ui[Int] =
  controls.choice(
    "Tagless final is the greatest thing ever",
    Seq(
      "Strongly disagree" -> 1,
      "Disagree" -> 2,
      "Neutral" -> 3,
      "Agree" -> 4,
      "Strongly agree" -> 5
    )
  )
  
def quiz[Ui[_]](
    controls: Controls[Ui],
    layout: Layout[Ui]
): Ui[(String, Int)] =
  layout.and(name(controls), rating(controls))
```

このスタイルのコードはすぐに書くのが煩雑になってしまう。メソッドシグネチャはかなり込み入っており、プログラム代数をメソッド間で渡す作業は煩わしい手間でしかない。

これを改善する方法のひとつは、プログラム代数を `given` インスタンスとして定義することである。たとえば次のようなアクセサを定義する。

```scala mdoc:silent
object Controls {
  def apply[Ui[_]](using controls: Controls[Ui]): Controls[Ui] =
    controls
}

object Layout {
  def apply[Ui[_]](using layout: Layout[Ui]): Layout[Ui] =
    layout
}
```

そうすると次のように書くことができる。

```scala mdoc:silent:nest
def name[Ui[_]: Controls]: Ui[String] =
  Controls[Ui].textInput("What is your name?", "John Doe")
  
def rating[Ui[_]: Controls]: Ui[Int] =
  Controls[Ui].choice(
    "Tagless final is the greatest thing ever",
    Seq(
      "Strongly disagree" -> 1,
      "Disagree" -> 2,
      "Neutral" -> 3,
      "Agree" -> 4,
      "Strongly agree" -> 5
    )
  )
  
def quiz[Ui[_]: Controls: Layout]: Ui[(String, Int)] =
  Layout[Ui].and(name, rating)
```

これは Scala コミュニティで一般的に用いられている Tagless Final のエンコーディングであるが、実際にこのコードを書く開発者にとっては、まだ記法上の負担が大きい。しかし Scala の言語機能を活用すれば、Tagless Final 形式のコーディングの負担を、通常のコードとほとんど変わらないレベルにまで減らすことができる。

そのために、次の5つの技法を組み合わせて用いる。

1. プログラム代数の基底型を作ること
2. プログラム型を抽象型メンバーとして定義すること
3. プログラムを表す型を定義すること
4. コンストラクタをコンパニオンオブジェクト上に定義すること
5. コンビネータのために拡張メソッドを定義すること

やや込み入ってはいるが、各ステップ自体は比較的単純である。どのように機能するのかを見ていこう。

最初のステップは、代数の基底型を作ることである。これは次のような単なるトレイトである。

```scala mdoc:silent:nest
trait Algebra[Ui[_]]
```

プログラム代数はこのトレイトを継承する。

```scala mdoc:silent
trait Controls[Ui[_]] extends Algebra[Ui[_]]{
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout[Ui[_]] extends Algebra[Ui[_]]{
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

次に、プログラム型を抽象型メンバーとして定義する。

```scala mdoc:silent:nest
trait Algebra {
  type Ui[_]
}

trait Controls extends Algebra {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Ui[String]

  def choice[A](label: String, options: Seq[(String, A)]): Ui[A]
}

trait Layout extends Algebra {
  def and[A, B](first: Ui[A], second: Ui[B]): Ui[(A, B)]
}
```

この時点で例題プログラムにはすでに有意義な変化が生じる。当初の出発点は次のようなコードだった。

```scala
def quiz[Ui[_]: Controls: Layout](
    controls: Controls[Ui],
    layout: Layout[Ui]
): Ui[(String, Int)] =
  Layout[Ui].and(
    Controls[Ui].textInput("What is your name?", "John Doe"),
    Controls[Ui].choice(
      "Tagless final is the greatest thing ever",
      Seq(
        "Strongly disagree" -> 1,
        "Disagree" -> 2,
        "Neutral" -> 3,
        "Agree" -> 4,
        "Strongly agree" -> 5
      )
    )
  )
```

上述の変更により、このコードは以下のように書き直すことができる。

```scala
def quiz(using alg: Controls & Layout): alg.Ui[(String, Int)] =
  alg.and(
    alg.textInput("What is your name?", "John Doe"),
    alg.choice(
      "Tagless final is the greatest thing ever",
      Seq(
        "Strongly disagree" -> 1,
        "Disagree" -> 2,
        "Neutral" -> 3,
        "Agree" -> 4,
        "Strongly agree" -> 5
      )
    )
  )
```

主な変更点は以下のとおりである。

1. プログラム代数が共通の基底型を継承しているため、メソッドの単一の引数として受け取れるようになった
2. `Ui` 型が抽象型メンバーとして定義され、型パラメータが不要となった
3. 戻り値の型を指定するために依存メソッド型（dependent method types）を使う必要が生じた

次のステップではプログラムを表す型を定義する。プログラムは概念的にはプログラム代数を受け取りプログラム型を返す関数である。そういう関数のような型を以下のように定義できる。

```scala mdoc:silent
trait Program[-Alg <: Algebra, A] {
  def apply(alg: Alg): alg.Ui[A]
}
```

戻り値型が `alg.Ui[A]` であることに特に注意を払ってほしい。`Program` は戻り値に依存メソッド型を要求するので、普通の関数型としては定義することはできない。

これで例題プログラムは次のようになる。

```scala mdoc:silent
val quiz =
  new Program[Controls & Layout, (String, Int)] {
    def apply(alg: Controls & Layout) =
      alg.and(
        alg.textInput("What is your name?", "John Doe"),
        alg.choice(
          "Tagless final is the greatest thing ever",
          Seq(
            "Strongly disagree" -> 1,
            "Disagree" -> 2,
            "Neutral" -> 3,
            "Agree" -> 4,
            "Strongly agree" -> 5
          )
        )
      )
  }
```

これでプログラムはメソッドではなく値として定義されるようになった。`Program` の最初の型パラメータには、そのプログラムが必要とするすべてのプログラム代数が指定されることに注目してほしい。

このコードを書くのはいまだにやや煩雑だが、*単一抽象メソッド（single abstract method）*というテクニックを用いることで、すこしシンプルにできる。これは、`Program` のように抽象メソッドをひとつだけもつトレイトを関数で実装できるという Scala の機能である。

```scala mdoc:silent:nest
val quiz: Program[Controls & Layout, (String, Int)] =
  (alg: Controls & Layout) =>
    alg.and(
      alg.textInput("What is your name?", "John Doe"),
      alg.choice(
        "Tagless final is the greatest thing ever",
        Seq(
          "Strongly disagree" -> 1,
          "Disagree" -> 2,
          "Neutral" -> 3,
          "Agree" -> 4,
          "Strongly agree" -> 5
        )
      )
    )
```

「値としてのプログラム」は、続くふたつの改善への扉を開く鍵となる。そのひとつ目はコンストラクタをコンパニオンオブジェクトのメソッドとして定義することである。

```scala mdoc:silent
object Controls {
  def textInput(
      label: String,
      placeholder: String,
      validation: Validation[String] = succeed
  ): Program[Controls, String] =
    alg => alg.textInput(label, placeholder, validation)

  def choice[A](
    label: String, 
    options: Seq[(String, A)]
  ): Program[Controls, A] =
    alg => alg.choice(label, options)
}
```

メソッドがプログラムを返せるようになったことで、このような書き方が可能となる。

ふたつ目の、そして全体として最後の改善は、コンビネータのための拡張メソッドを定義することである。今回はコンビネータが `and` ひとつしかないので、拡張メソッドもひとつだけ定義する。

```scala mdoc:silent
extension [Alg <: Algebra, A](p: Program[Alg, A]) {
  def and[Alg2 <: Algebra, B](
    second: Program[Alg2, B]
  ): Program[Alg & Alg2 & Layout, (A, B)] =
    alg => alg.and(p(alg), second(alg))
}
```

この拡張メソッドにおける型の定義を特に注意深く見てほしい。この拡張はプログラム代数 `Alg` を要求する `Program` に対して定義されている。`and` メソッドに渡す引数はそれとは別の `Program` 値で、プログラム代数 `Alg2` を要求する。その結果として得られるプログラムは代数 `Alg & Alg2 & Layout` を要求する。これはふたつのプログラムが要求する代数と `Layout` との交差型である。コンビネータは、プログラムが要求する代数をこのようにして組み立てる。

最終的な結果として、ユーザは次のようなコードを書くことができる。

```scala mdoc:silent:nest
val quiz  =
  Controls
    .textInput("What is your name?", "John Doe")
    .and(
      Controls.choice(
        "Tagless final is the greatest thing ever",
        Seq(
          "Strongly disagree" -> 1,
          "Disagree" -> 2,
          "Neutral" -> 3,
          "Agree" -> 4,
          "Strongly agree" -> 5
        )
      )
    )
```

これは見た目にも通常のコードと変わらない。`quiz` の型を見れば、必要なプログラム代数がすべて型推論によって正しく導出されていることがわかる。

```scala mdoc
quiz
```

このエンコーディングではライブラリ開発者側により多くの作業が必要となる。だがそれは一度きりのコストであり、その代わりとしてライブラリ利用者ははるかに簡潔なコードを書くことができる。Tagless Final のほとんどの用途において、このトレードオフは妥当なものだと考えている。
