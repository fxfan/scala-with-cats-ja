## まとめ Summary

Functors represent sequencing behaviours.
We covered three types of functor in this chapter:

- Regular covariant `Functors`, with their `map` method,
  represent the ability to apply functions
  to a value in some context.
  Successive calls to `map`
  apply these functions in *sequence*,
  each accepting the result of its predecessor
  as a parameter.

- `Contravariant` functors, with their `contramap` method,
  represent the ability to "prepend" functions
  to a function-like context.
  Successive calls to `contramap`
  sequence these functions in the opposite order to `map`.

- `Invariant` functors, with their `imap` method,
  represent bidirectional transformations.

ファンクターは振る舞いの順序付けを表す。本章では三種類のファンクターを取り上げた。

- 通常の共変 `Functor` は `map` メソッドをもち、何らかのコンテキスト内の値に対して関数を適用できる。`map` を連続して呼び出すことで、前の関数の結果を次の関数の引数として渡し、*順序どおりに*それらの関数を適用する。
- `Contravariant` ファンクターは `contramap` メソッドをもち、関数的なコンテキストに対して入力の先頭に関数を挿入できる。`contramap` を連続して呼び出すことで、`map` とは逆の順序でそれらの関数が適用されるように連結する。
- `Invariant` ファンクターは `imap` メソッドをもち、双方向の変換を表す。

Regular `Functors` are by far the most common of these type classes,
but even then it is rare to use them on their own.
Functors form a foundational building block of
several more interesting abstractions that we use all the time.
In the following chapters we will look at two of these abstractions:
*monads* and *applicative functors*.

通常の `Functor` はこれらの型クラスの中でも圧倒的によく使われるが、それでも単独で使われることは稀である。ファンクターは、日常的に使われるより興味深い抽象概念の基礎を成す。次章以降ではこれらの抽象概念のうちの二つ、*モナド*と*アプリカティブファンクター*を見ていく。

Functors for collections are extremely important, as they transform each element independently of the rest.
This allows us to parallelise or distribute
transformations on large collections,
a technique leveraged heavily in
"map-reduce" frameworks like [Hadoop][link-hadoop].
We will investigate this approach in more detail in the
map-reduce case study later in Section [@sec:map-reduce].

コレクションに対するファンクターは、それぞれの要素を他の要素に依存せずに変換できる点において、極めて重要である。これにより、大規模なコレクションに対する変換を並列化したり、分散させたりすることが可能になる。この技術は[Hadoop][link-hadoop]のようなMapReduceフレームワークで大いに活用されている。このアプローチについては、[@sec:map-reduce]節で取り組むMapReduceのケーススタディの中で、詳細に確認する。

The `Contravariant` and `Invariant` type classes
are less widely applicable but are still useful
for building data types that represent transformations.
We will revisit them to discuss the `Semigroupal`
type class later in Chapter [@sec:applicatives].

`Contravariant` と `Invariant` 型クラスは、適用範囲こそ狭いが、変換を表すデータ型を構築する際には役立つ。これらについては、[@sec:applicatives]章で `Semigroupal` 型クラスを議論する際に再度取り上げる。
