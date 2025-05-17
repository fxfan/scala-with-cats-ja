<!--

# Indexed Types

In this chapter we look at **indexed types**. An indexed type is a type constructor, so a type like `F[_]`, along with a set of types that can fill in the constructor's type parameters. Let's say this set of types is `Int`, `String`, and `Option[Double]`. Then, for a type constructor `F` we can construct an indexed type from the set `F[Int]`, `F[String]`, and `F[Option[Double]]`. 
The types `Int`, `String`, and `Option[Double]` act as indices into the set 
`F[Int]`, `F[String]`, and `F[Option[Double]]`, hence the name.
The type constructor `F` can be either data and codata. 

The description above is very abstract, and doesn't help us understand how indexed types are useful. We'll see a lot of details and examples in this chapter, but let's start with a more useful high-level overview. We can think of indexed types as working with proofs that a type parameter is equal to a particular element from the set of indices. Indexed *data* provides this evidence when we destructure it, while indexed *codata* requires this evidence when we call methods. Remember the definition of algebras we gave in Section [@sec:interpreters:reification], where we said an algebra consists of three different kinds of methods: constructors, combinators, and interpreters. Indexed types allows us to do two things:

- We can restrict where constructors and combinators can be used. We can think of representing some state using a type parameter of `F`, and we can only call particular methods when we are in the correct state. In this case we are working with **indexed codata**.

- We restrict the types produced by interpreters, enabling us to create type-safe interpreters that guarantee they only encounter particular states when they run. Again these constraints are represented using type parameters. In this case we are working with **indexed data**.

Indexed data are more usually known as **generalized algebraic data types**. Indexed codata are sometimes known as **typestate**. Both can make use of what is known as **phantom types**. Indeed, an early name for indexed data was **first-class phantom types**. As you might expect, indexed data and indexed codata are dual to one another. 


-->

# インデックス付き型

この章では**インデックス付き型（indexed type）**について見ていく。インデックス付き型は型コンストラクタであり、`F[_]` のような型と、その型パラメータを埋めることのできる型の集合からなる。この型の集合を仮に `Int`、`String`、および `Option[Double]` としよう。この場合、型コンストラクタ `F` について、 `F[Int]`、`F[String]` および `F[Option[Double]]` というインデックス付き型を構成できる。このとき、型 `Int`、`String`、`Option[Double]` は `F[Int]`、`F[String]`、`F[Option[Double]]` という型の集合における索引として機能するため、`F` はインデックス付き型と呼ばれる。型コンストラクタ `F` はデータと余データどちらもかまわない。

この説明は非常に抽象的であり、インデックス付き型がどのように役立つのかを理解する助けにはならない。そこで本章では、より多くの詳細や例を見ることになるが、まずはもっと役に立ちそうな高レベルの概観から始めよう。インデックス付き型は、ある型パラメータがインデックス集合の特定の要素に等しいという証明に基づいて動作するものと考えることができる。インデックス付きの*データ*では、それを分解する際にこのエビデンスを提供し、インデックス付きの*余データ*では、メソッドを呼び出す際にこのエビデンスを必要とする。[@sec:interpreters:reification]節で示した代数の定義を思い出してほしい。そこでは、代数とはコンストラクタ、コンビネータ、インタープリタという三種類の異なるメソッドから構成されるものとした。インデックス付き型を用いると、次のふたつのことが可能になる。

- コンストラクタとコンビネータの使用箇所を制限できる。`F` の型パラメータを用いて状態を表し、適切な状態にあるときにのみ特定のメソッドを呼び出せるようにできる。このケースで扱うのは**インデックス付き余データ（indexed codata）**である。

- インタープリタによって生成される型を制約し、実行したときに特定の状態にしか出会わないことを保証された型安全なインタープリタを作成できる。この制約も型パラメータを使用して表現される。このケースで扱うのは**インデックス付きデータ（indexed data）**である。

インデックス付きデータは**一般化代数的データ型（generalized algebraic data type; GADT）**という名前で知られていることが多い。一方、インデックス付き余データは**型状態（typestate）**と呼ばれることがある。どちらも、いわゆる**ファントム型**を使用することができる。実際、インデックス付きデータは**ファーストクラス・ファントム型**と呼ばれていたこともある。予想されているかもしれないが、インデックス付きデータとインデックス付き余データは双対の関係にある。
