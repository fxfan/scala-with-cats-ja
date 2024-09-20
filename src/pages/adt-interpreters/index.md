# Reified Interpreters {#sec:interpreters}

The interpreter strategy is perhaps the most important in all of functional programming. The central idea is to **separate description from action**. When we use the interpreter strategy our program consists of two parts: the description, instructions, or program that describes what we want to do, and the interpreter that carries the actions in the description. In this chapter we'll start exploring the design and implementation of interpreters, focusing on implementations using algebraic data types. 

インタープリタ戦略は、おそらく関数型プログラミングにおいて最も重要な戦略だろう。中心となるアイデアは**指示と実行の分離**である。インタープリタ戦略を用いたプログラムは、実行したい内容を指示する部分と、その指示に従って実際にアクションを実行する「インタープリタ」という二つの部分から構成される。この章では、インタープリタの設計と実装について探る。その中でも、特に代数的データ型を用いた実装に焦点を当てる。

Interpreters arise whenever there is this distinction between description and action. You may think an interpreter is a complex piece requiring a lot of development effort, but I hope to show you this is not the case. You probably already use lots of interpreters in your daily coding without realizing it. For example, consider the code below which is taken from a web framework called [Krop][krop]

指示と実行を区別する場面には必ずインタープリタが登場する。インタープリタは複雑で多大な開発労力を要するものと思われるかもしれないが、実際にはそうではないことを伝えたい。おそらく、皆さんは日頃のコーディングでそれと気付かずに多くのインタープリタを既に使用しているはずである。たとえば、[Krop][krop]というWebフレームワークから抜き出した以下のコードを考えてみよう。

```scala
val route =
  Route(
    Request.get(Path.root / "user" / Param.int),
    Response.ok(Entity.text)
  ).handle(userId => s"You asked for the user ${userId.toString}")
```

This defines a route, which matches `GET` requests for the path `"/user/<int>"`, and responds with an `Ok` containing text. This kind of routing library is ubiquitous in web frameworks, is simple to write, and yet contains everything we need for the interpreter strategy.

このコードは、パス `"/user/<int>"` に対するGETリクエストにマッチし、テキストのボディをもった `Ok` レスポンスを返すルートを定義している。この種のルーティングライブラリはWebフレームワークにおいて普遍的で、簡単に実装できるものだが、実はインタープリタ戦略に必要な要素をすべて含んでいる。

Interpreters are so important because they are the key to enabling compositionality and reasoning, particularly while allowing effects. For example, imagine implementing a graphics library using the interpreter strategy. A program simply describes what we want to draw on the screen, but critically it does not draw anything. The interpreter takes this description and creates the drawing described by it. We can freely compose descriptions only because they do not carry out any effects. For example, if we have a description that describes a circle, and one for a square, we can compose them by saying we should draw the circle next to the square thereby creating a new description. If we immediately drew pictures there would be nothing to compose with. Similarly, it's easier to reason about pictures in this system because a program describes exactly what will appear on the screen, and there is no state from prior drawing that we need to worry about.

インタープリタが重要なのは、それが副作用を許容しながらも合成可能性と推論を可能にする鍵となる考え方だからである。たとえば、インタープリタ戦略を使ってグラフィックスライブラリを実装することを考えてみよう。プログラムは、描画したい内容を単に指示するが、重要な点として、その段階では実際には何も描画しない。インタープリタがこの指示を受け取り、それに基づいて描画を行う。指示部分は副作用を伴わないので自由に合成できる。たとえば、円を描画する指示と四角形を描画する指示があれば、それらを組み合わせて「四角形の横に円を描く」といった新たな指示を作り出すことができる。もし即座に描画が行われていたら合成は行えない。同様に、この仕組みの下では、プログラムは画面に何が表示されるかをそのまま記述しており、それ以前の描画による状態を気にする必要がないので、推論が容易である。

Throughout this chapter we will explore the interpreter strategy by building a series of interpreters for regular expressions. We've chosen to use regular expressions because they are already familiar to many and they are simple to work with. This means we can focus on the details of the interpreter strategy without getting caught up in problem specific details, but we still end up with a realistic and useful result.

この章では、正規表現のための一連のインタープリタを構築することで、インタープリタ戦略を探求していく。正規表現を選んだのは、それが多くの人にとって馴染みのあるものであり、使い方がシンプルだからである。これにより、実現する機能（ここでは正規表現）固有の詳細にとらわれることなく、インタープリタ戦略の細部に集中することができ、ついでに現実的で役に立つ結果を得ることができる。

We'll start with a basic implementation strategy that uses algebraic data types and structural recursion. We'll then look at transformations to turn our interpreter into a version that avoids using the stack and hence avoids the possibility of stack overflow.

まずは、代数的データ型と構造的再帰を用いた基本的な実装戦略から始める。その後、スタックを使わないことでスタックオーバーフローの可能性を回避するバージョンへとインタープリタを変換する方法について見ていく。

[krop]: https://github.com/creativescala/krop
