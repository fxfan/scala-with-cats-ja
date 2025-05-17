<!--

## Conclusions

In this chapter we took a first look at type classes.
We saw the components that make up a type class:

- A `trait`, which is the type class

- Type class instances, which are given instances.

- Type class usage, which uses using clauses.

We saw that type classes can be composed from components using type class composition.
This is one form of metaprogramming in Scala, 
where we can get the compiler to do work for us based on our program's types.

We can view type classes as marrying codata with tools to select and compose implementations based on type. 
We can also view type classes as shifting implementation from the definition site to the call site.
Finally, can see type classes as a mechanism for ad-hoc polymorphism, allowing us to define common functionality for otherwise unrelated types.

Type classes were first described in @10.1007/3-540-19027-9_9 and @10.1145/75277.75283. @10.1145/1869459.1869489 details the encoding of type classes in Scala 2, and compares Scala's and Haskell's approach to type classes. Note that type classes are not restricted to Haskell and Scala. For examples, Rust's traits are essentially type classes.

As we have seen, Scala's support for type classes is based on implicit parameters (known as using clauses in Scala 3). Implicit parameters [@10.1145/325694.325708] were motivated by a desire to decompose type classes into smaller orthogonal language features, but they have been shown to be useful for other tasks. @10.1145/3360589 surveys different uses of implicits in Scala. See @OLIVEIRA_GIBBONS_2010 for a particularly mind-bending example. We'll see some of these different uses in later chapters.

Scala 3 has a few language features related to contextual abstraction that we haven't mentioned in this chapter. Context functions [@10.1145/3158130] allow functions to have using clauses. They are something the community is still exploring, and well defined use cases have yet to emerge. Generic derivation allows us to write code that generates type classes instances. Although this is extremely useful I think it's conceptually quite simple and doesn't warrant space in this book.


-->

## まとめ

この章では、型クラスについて紹介し、それが以下の要素から構成されることを学んだ。

- `trait` として実現される型クラスの本体
- given インスタンスとして用いられる型クラスインスタンス
- using 句を用いた型クラスの利用

型クラスが合成を通じて複数の要素から構成できることも見てきた。Scala においてこれはメタプログラミングの一種で、プログラムの型に基づいてコンパイラに作業をさせることができる。

型クラスは、余データと、型に基づいて実装を選択し合成するためのツールとを融合させたものとして捉えることができる。また、型クラスは実装を定義時点から呼び出し時点へと移行させるものとも言える。そして最後に、型クラスはアドホック多相を実現する仕組みとして、互いに関連のない型に共通の機能を定義することを可能にする。

型クラスは @10.1007/3-540-19027-9_9 および @10.1145/75277.75283 で初めて言及された。@10.1145/1869459.1869489 は Scala2 における型クラスのエンコーディングについて詳述し、Scala と Haskell の型クラスに対するアプローチを比較している。型クラスは Haskell や Scala だけに限定されたものではない。たとえば Rust のトレイトは本質的に型クラスと同じものである。

これまで見てきたように、Scala における型クラスのサポートは、暗黙パラメータ（ Scala3 では using 句として知られている）に基づいている。暗黙パラメータ [@10.1145/325694.325708] は型クラスをより小さく独立した言語機能に分解するために導入されたが、他の用途にも役立つことが示されている。@10.1145/3360589 では Scala における暗黙パラメータのさまざまな利用方法を調査している。 @OLIVEIRA_GIBBONS_2010 に特に興味深い例が記載されているので参照してほしい。それらの異なる使用方法のいくつかについては、後の章で詳しく見ていこうと思う。

Scala3 がもっているコンテキスト抽象化に関連する言語機能の中には、本章で言及していないものもいくつかある。コンテキスト関数 [@10.1145/3158130] は、関数が using 句をもつことを可能にする。それらの機能については、まだコミュニティで模索中であり、はっきりしたユースケースが定まっていない。ジェネリック導出（generic derivation）を使えば、型クラスインスタンスを生成するコードを記述できる。これは非常に有用だが、概念的にはかなり単純であり、本書で詳しく扱うほどの内容ではないと考えている。

<!-- TODO: generic derivationの訳語「ジェネリック導出」は仮。翻訳事例を探す -->
