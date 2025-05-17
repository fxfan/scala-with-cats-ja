<!--

# Foldable and Traverse {#sec:foldable-traverse}

In this chapter we'll look at two type classes
that capture iteration over collections:

  - `Foldable` abstracts the familiar
    `foldLeft` and `foldRight` operations;
  - `Traverse` is a higher-level abstraction
    that uses `Applicatives` to iterate
    with less pain than folding.

We'll start by looking at `Foldable`,
and then examine cases where folding becomes complex
and `Traverse` becomes convenient.


-->

# `Foldable` と `Traverse` {#sec:foldable-traverse}

この章ではコレクション走査を抽象化するふたつの型クラスについて説明する。

  - `Foldable` は、本書ではすでにおなじみの `foldLeft` および `foldRight` を抽象化する。
  - `Traverse` は、アプリカティブを利用し、畳み込みの煩雑さを軽減しながら走査を行う高次の抽象化である。

まずは `Foldable` から見ていこう。その後、畳み込みが複雑になり `Traverse` がその力を発揮するケースについて考察する。
