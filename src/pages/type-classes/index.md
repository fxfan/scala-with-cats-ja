# コンテキスト抽象化 Contextual Abstraction {#sec:type-classes}

All but the simplest programs depend on the **context** in which they run. The number of available CPU cores is an example of context provided by the computer, which a program might adapt to by changing how work is distributed. Other forms of context include configuration read from files and environment variables, and (and we'll see at lot of this later) values created at compile-time, such as serialization formats, in response to the type of some method parameters.

あらゆるプログラムは、特に単純なものを除けば、それが実行される**コンテキスト**に依存している。利用可能なCPUコア数は、コンピュータが提供するコンテキストの一例であり、プログラムはそれに応じてタスクの分配方法を調整するかもしれない。コンテキストの他の形式としては、ファイルや環境変数から読み取られる設定情報、そしてシリアライズ形式のようにメソッドパラメータの型に応じてコンパイル時に生成される値などがある。後者については改めて詳しく述べるつもりである。

Scala is one of the few languages that provides features for **contextual abstraction**, known as **implicits** in Scala 2 or **given instances** in Scala 3. In Scala these features are intimately related to types; types are used to select between different available given instances and drive construction of given instances at compile-time.

Scalaは**コンテキスト抽象化**のための機能を提供する数少ない言語のひとつである。その機能はScala2では**implicit**、Scala3では**givenインスタンス**として知られている。Scalaにおけるこれらの機能は型と密接に関連している。利用可能なgivenインスタンスの中から必要なものを選択することや、givenインスタンスの構築は、コンパイル時に型を使用して行われる。

<!-- TODO: givenインスタンスの構築がコンパイル時に行われているかのような表現を避ける -->

Most Scala programmers are less confident with the features for contextual abstraction than with other parts of the language, and they are often entirely novel to programmers coming from other languages. Hence this chapter will start by reviewing the abstractions formely known as implicits: given instances and using clauses. We will then look at one of their major uses, **type classes**[]. Type classes allow us to extend existing types with new functionality, without using traditional inheritance, and without altering the original source code. Type classes are the core of Cats, which we will be exploring in the next part of this book.

多くのScalaプログラマは、コンテキスト抽象化の機能を他の言語機能ほど使いこなせていない場合が多く、他の言語から来たプログラマにとっては完全に新しい概念であることも多い。そのため、この章では、かつて `implicit` と呼ばれていた抽象化や、givenインスタンス、そして `using` 句について振り返ることから始める。そして、それらの主要な用途のひとつである**型クラス**[^type-class-defn]について見ていく。 型クラスを使うと、従来の継承を使わず、元のソースコードを変更することもなく、既存の型を拡張し、新しい機能を追加することができる。型クラスは、この本の次のパートで学んでいくCatsの中心的な要素である。

<!--
Type classes work well with another programming pattern: *algebraic data types*.
These are closed systems of types that we use to represent data or concepts.
Because the systems are closed (and therefore cannot be extended by other users),
we can process them using pattern matching
and the compiler will check the exhaustiveness of our case clauses.

There are two other patterns we need to cover in this chapter.
*Value classes* provide a way to wrap up
generic data types like `Strings` and `Ints`
and give them specific meanings in a given context.
The extra type information is useful when type classes.
*Type aliases* are another pattern that
provide aliases for large, complex types.
-->

[^type-class-defn]: 「クラス」という言葉は、厳密には、ScalaやJavaにおける `class` を意味するものではない
