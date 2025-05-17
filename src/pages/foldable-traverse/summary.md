<!--

## Summary

In this chapter we were introduced to `Foldable` and `Traverse`,
two type classes for iterating over sequences.

`Foldable` abstracts
the `foldLeft` and `foldRight` methods we know
from collections in the standard library.
It adds stack-safe implementations of these methods
to a handful of extra data types,
and defines a host of situationally useful additions.
That said, `Foldable` doesn't introduce much
that we didn't already know.

The real power comes from `Traverse`,
which abstracts and generalises
the `traverse` and `sequence` methods we know from `Future`.
Using these methods we can turn an `F[G[A]]` into a `G[F[A]]`
for any `F` with an instance of `Traverse`
and any `G` with an instance of `Applicative`.
In terms of the reduction we get in lines of code,
`Traverse` is one of the most powerful patterns in this book.
We can reduce `folds` of many lines down to a single `foo.traverse`.

-----

...and with that,
we've finished all of the theory in this book.
There's plenty more to come, though,
as we put everything we've learned into practice
in a series of in-depth case studies in Part II!


-->

## まとめ

この章ではシーケンスの反復処理を行うふたつの型クラス `Foldable` と `Traverse` について学んだ。

`Foldable` は、標準ライブラリのコレクションでおなじみの `foldLeft` や `foldRight` を抽象化する。いくつかの追加データ型にスタックセーフな実装を提供し、便利なメソッドを数多く追加定義している。とはいえ、`Foldable` は既存の知識に新たな要素を大きく加えるものではない。

本当に強力なのは `Traverse` である。`Traverse` は、`Future` でおなじみの `traverse` や `sequence` メソッドを抽象化し一般化する。これらのメソッドを使えば、`Traverse` インスタンスをもつ任意の `F` と、`Applicative` インスタンスをもつ任意の `G` に対して、`F[G[A]]` を `G[F[A]]` に変換できる。コード行数の削減に関しては、`Traverse` は本書でもっとも強力なパターンのひとつである。多くの行数を要する `fold` を、一行の `foo.traverse` にまで簡潔にできる。

-----

これで、本書における理論の解説はすべて終了した。しかし、内容はまだまだ続く。第二部では、学んだすべてのことを実践する一連の詳細なケーススタディを見ていく。
