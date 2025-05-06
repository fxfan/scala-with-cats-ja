<!--

## Summary

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

Regular `Functors` are by far the most common of these type classes,
but even then it is rare to use them on their own.
Functors form a foundational building block of
several more interesting abstractions that we use all the time.
In the following chapters we will look at two of these abstractions:
*monads* and *applicative functors*.

Functors for collections are extremely important, as they transform each element independently of the rest.
This allows us to parallelise or distribute
transformations on large collections,
a technique leveraged heavily in
"map-reduce" frameworks like [Hadoop][link-hadoop].
We will investigate this approach in more detail in the
map-reduce case study later in Section [@sec:map-reduce].

The `Contravariant` and `Invariant` type classes
are less widely applicable but are still useful
for building data types that represent transformations.
We will revisit them to discuss the `Semigroupal`
type class later in Chapter [@sec:applicatives].


-->

## まとめ

ファンクターは振る舞いの順序付けを表す。本章では三種類のファンクターを取り上げた。

- 通常の共変 `Functor` は `map` メソッドをもち、何らかのコンテキスト内の値に対して関数を適用できる。`map` を連続して呼び出すと、関数は*順序どおり*に適用される。その際、関数はひとつ前の関数の結果を引数として受け取る。
- `Contravariant` ファンクターは `contramap` メソッドをもち、関数的なコンテキストにおいて入力を受け取るチェーンの先頭に関数を挿入できる。`contramap` を連続して呼び出すと、それらの関数は `map` とは逆の順序で適用されるよう連結される。
- `Invariant` ファンクターは `imap` メソッドをもち、双方向の変換を表す。

通常の `Functor` はこれらの型クラスの中で圧倒的によく使われるが、それでも単独で使われることは稀である。ファンクターは、日常的に使われるもっと興味深い抽象概念の基礎となる。次章以降ではこれらの抽象概念のうちのふたつ、*モナド*と*アプリカティブファンクター*を見ていく。

コレクションに対するファンクターは、それぞれの要素を他の要素に依存せずに変換できる点において、極めて重要である。これにより、大規模なコレクションに対する変換を並列化したり、分散させたりすることが可能になる。この技術は [Hadoop][link-hadoop] のような MapReduce フレームワークで大いに活用されている。このアプローチについては、[@sec:map-reduce]節で取り組む MapReduce のケーススタディにおいて詳細に確認する。

`Contravariant` と `Invariant` 型クラスは、適用範囲こそ狭いが、変換を表すデータ型を構築する際に有用である。これらについては、[@sec:applicatives]章で `Semigroupal` 型クラスを議論するときに改めて取り上げる。
