## インタープリタとレイフィケーション Interpreters and Reification {#sec:interpreters:reification }

There are two different programming strategies at play in the regular expression code we've just written:

1. the interpreter strategy; and
2. the interpreter's implementation strategy of reification.

ここまでに書いた正規表現の実装では、二つの異なるプログラミング戦略が使われている。

1. インタープリタ戦略
2. レイフィケーションによるインタープリタの実装戦略

Remember the essence of the **interpreter strategy** is to separate description and action. Therefore, whenever we use the interpreter strategy we need at least two things: a description and an interpreter. Descriptions are programs; things that we want to happen. The interpreter runs the programs, carrying out the actions described within them.

**インタープリタ戦略**の本質が指示と実行の分離であることを思い出してほしい。インタープリタ戦略を用いるにあたっては、少なくとも指示の記述とインタープリタの二つが必要である。指示はプログラムであり、私たちが行いたいことを表している。インタープリタはそのプログラムを実行し、そこに記述された指示を実行に移す。

In the regular expression example, a `Regexp` value is a program. It is a description of a pattern we are looking for within a `String`.
The `matches` method is an interpreter. It carries out the instructions in the description, checking the pattern matches the entire input. We could have other interpreters, such as one that matches if at least some part of the input matches the pattern.

正規表現の例では `Regexp` オブジェクトがプログラムであり、`String` の中から探したいパターンを記述している。そして、`matches` メソッドがインタープリタである。記述された指示を実行に移し、入力全体がパターンにマッチするかどうか調べる。入力の一部がパターンにマッチするかどうかを判定するような別のインタープリタを作ることもできるだろう。

### インタープリタの構造 The Structure of Interpreters

All uses of the interpreter strategy have a particular structure to their methods.
There are three different kinds of methods:

1. **constructors**, or **introduction forms**, with type `A => Program`. Here `A` is any type that isn't a program, and `Program` is the type of programs. Constructors conventionally live on the `Program` companion object in Scala. We see that `apply` is a constructor of `Regexp`. It has type `String => Regexp`, which matches the pattern `A => Program` for a constructor. The other constructor, `empty`, is just a value of type `Regexp`. This is equivalent to a method with type `() => Regexp` and so it also matches the pattern for a constructor.

2. **combinators** have at least one program input and a program output. The type is similar to `Program => Program` but there are often additional parameters. All of `++`, `orElse`, and `repeat` are combinators in our regular expression example. They all have a `Regexp` input (the `this` parameter) and produce a `Regexp`. Some of them have additional parameters, such as `++` or `orElse`. For both these methods the single additional parameter is a `Regexp`, but it is not the case that additional parameters to a combinator must be of the program type. Conventionally these methods live on the `Program` type.

3. **destructors**, **interpreters**, or **elimination forms**, have type `Program => A`. In our regular expression example we have a single interpreter, `matches`, but we could easily add more. For example, we often want to extract elements from the input or find a match at any location in the input.

インタープリタ戦略におけるメソッドは一定の構造をもっている。メソッドには以下の三種類がある。

1. **コンストラクタ**または**導入形式（introduction form）**。`A => Program` という型をもつ。ここで `A` はプログラムではない任意の型で、`Program` は文字どおりプログラムを表す型である。コンストラクタは、Scalaでは慣習的に `Program` のコンパニオンオブジェクト上に定義される。正規表現の例では `apply` が `Regexp` のコンストラクタで、`String => Regexp` という型をもっていた。これはコンストラクタがもつべき型である `A => Program` に該当している。もうひとつのコンストラクタである `empty` は単なる `Regexp` 型の値だったが、これは `() => Regexp` 型のメソッドと等価であり、やはりコンストラクタとしての型をもっている。

2. **コンビネータ**。少なくともひとつのプログラムを入力として受け取り、プログラムを出力する。その型は `Program => Program` に近いものとなるが、追加のパラメータをもつことが少なくない。正規表現の例で見た `++`、`orElse`、および `repeat` はすべてコンビネータである。これらはすべて `this` パラメータとして `Regexp` 型の入力をもち、`Regexp` を生成する。`++` や `orElse` は追加パラメータももっている。それらは `Regexp` 型だったが、コンビネータの追加パラメータが常にプログラム型であるというわけではない。コンビネータは通常 `Program` のメソッドとして定義される。

3. **デストラクタ**、**インタープリタ**、または**除去形式（elimination form）**。`Program => A` という型をもつ。正規表現の例ではインタープリタは `matches` のひとつだけだったが、追加するのは簡単である。たとえば、マッチした文字列の一部を抽出したり、入力の特定の位置でマッチする文字列を見つけたり、といったものが考えられる。

This structure is often called an **algebra** or **combinator library** in the functional programming world. When we talk about constructors and destructors in an algebra we're talking at a more abstract level then when we talk about constructors and destructors on algebraic data types. A constructor of an algebra is an abstract concept, at the theory level in my taxonomy, that we can choose to concretely implement at the craft level with the constructor of an algebraic data type. There are other possible implementations. We'll see one later.

この構造は、関数型プログラミングの世界では代数やコンビネータライブラリと呼ばれることがよくある。代数におけるコンストラクタやデストラクタは、代数的データ型のコンストラクタやデストラクタについてよりも抽象的なレベルで語られる。代数のコンストラクタとは、本書の分類でいうと理論レベルの抽象概念であり、その具体的実装として、技法レベルでは代数的データ型のコンストラクタという選択肢がある。実装方法は他にも存在するが、それについては改めて説明するつもりである。

### レイフィケーションによるインタープリタ実装 Implementing Interpreters with Reification

Now that we understand the components of an interpreter we can talk more clearly about the implementation strategy we used.
We used a strategy called **reification**, **defunctionalization**, **deep embedding**, or an **initial algebra**.

インタープリタの構成要素について理解したので、今回使った実装戦略についてより明確に説明することが可能となった。使った戦略は、**レイフィケーション（reification）**、**脱関数化（defunctionalization）**、**深い埋め込み（deep embedding）**、または**始代数（initial algebra）**と呼ばれる。

<!--
TODO: deep embeddingに対して日本語の訳語を当てた前例があまりない件の訳注
-->

Reification, in an abstract sense, means to make concrete what is abstract. Concretely, reification in the programming sense means to turn methods or functions into data. When using reification in the interpreter strategy we reify all the components that produce the `Program` type. This means reifying constructors and combinators.

具象化（reification）とは、広義には抽象的なものを具体的にすることを指す。プログラミングにおけるレイフィケーション（reification）とは、メソッドや関数をデータとして表現することを指す。インタープリタ戦略においては、`Program` 型を生成するすべての構成要素、すなわちコンストラクタやコンビネータをレイファイすることを意味している。

<!--
TODO: プログラミングにおけるreificationを「メソッドや関数をデータとして表現すること」と定義するのはやや今回の文脈が意識されすぎており、理解の妨げになると思う。訳註するか？
-->

Here are the rules for reification:

1. We define some type, which we'll call `Program`, to represent programs.
2. We implement `Program` as an algebraic data type.
3. All constructors and combinators become product types within the `Program` algebraic data type.
4. Each product type holds exactly the parameters to the constructor or combinator, including the `this` parameter for combinators.

以下にレイフィケーションのルールを挙げる。

1. `Program` と呼ぶ型を定義し、プログラムを表現する
2. `Program` を代数的データ型として実装する
3. すべてのコンストラクタとコンビネータは代数的データ型 `Program` を構成する直積型となる
4. 各直積型はコンストラクタもしくはコンビネータが受け取るパラメータと同じプロパティで構成される。コンビネータについては `this` パラメータもそこに含む

Once we've defined the `Program` algebraic data type, the interpreter becomes a structural recursion on `Program`.

`Program` を代数的データ型として定義すれば、インタープリタは `Program` に対する構造的再帰となる。

#### 演習: 算術演算 Exercise: Arithmetic {-}

Now it's your turn to practice using reification. Your task is to implement an interpreter for arithmetic expressions. An expression is:

- a literal number, which takes a `Double` and produces an `Expression`;
- an addition of two expressions;
- a substraction of two expressions;
- a multiplication of two expressions; or
- a division of two expressions;

ここでレイフィケーションを用いる練習をしよう。課題は、算術式のインタープリタを実装することである。式は以下のようなものとする。

- 数値リテラル。`Double` を受け取って `Expression` を生成する
- 二つの式の加算
- 二つの式の減算
- 二つの式の乗算
- 二つの式の除算

Reify this description as a type `Expression`.

以上の記述を `Expression` 型としてレイファイせよ。

<div class="solution">
The trick here is to recognize how the textual description relates to code, and to apply reification correctly.

ポイントは、テキストによる記述がコードとどのように関連するかを理解し、正しくレイフィケーションを適用することである。

```scala mdoc:silent 
enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>

Now implement an interpreter `eval` that produces a `Double`. This interpreter should interpret the expression using the usual rules of arithmetic.

続いて、`Double` 型の値を生成する `eval` インタープリタを実装せよ。インタープリタは一般的な算術規則に従って式を解釈すること。

<div class="solution">
インタープリタは構造的再帰である。

```scala mdoc:reset:silent 
enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)
  
  def eval: Double =
    this match {
      case Literal(value)              => value
      case Addition(left, right)       => left.eval + right.eval
      case Subtraction(left, right)    => left.eval - right.eval
      case Multiplication(left, right) => left.eval * right.eval
      case Division(left, right)       => left.eval / right.eval
    }
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```
</div>

Add methods `+`, `-` and so on that make your system a bit nicer to use. Then write some expressions and show that it works as expected.

ライブラリの使い勝手を向上させるため `+` や `-` などのメソッドを追加せよ。また、いくつかの式を記述し、それが期待どおりに動作することを確認せよ。

<div class="solution">
Here's the complete code.

以下が完成したコードである。

```scala mdoc:reset:silent
enum Expression {
  case Literal(value: Double)
  case Addition(left: Expression, right: Expression)
  case Subtraction(left: Expression, right: Expression)
  case Multiplication(left: Expression, right: Expression)
  case Division(left: Expression, right: Expression)

  def +(that: Expression): Expression =
    Addition(this, that)

  def -(that: Expression): Expression =
    Subtraction(this, that)

  def *(that: Expression): Expression =
    Multiplication(this, that)

  def /(that: Expression): Expression =
    Division(this, that)

  def eval: Double =
    this match {
      case Literal(value)              => value
      case Addition(left, right)       => left.eval + right.eval
      case Subtraction(left, right)    => left.eval - right.eval
      case Multiplication(left, right) => left.eval * right.eval
      case Division(left, right)       => left.eval / right.eval
    }
}
object Expression {
  def apply(value: Double): Expression =
    Literal(value)
}
```

Here's an example showing use, and that the code is correct.

また、以下は、その使い方およびコードが正しいことを示す例である。

```scala mdoc:silent
val fortyTwo = ((Expression(15.0) + Expression(5.0)) * Expression(2.0) + Expression(2.0)) / Expression(1.0)
```
```scala mdoc
fortyTwo.eval
```
</div>
