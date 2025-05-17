<!--

## Summary

While monads and functors are the most widely used
sequencing data types we've covered in this book,
semigroupals and applicatives are the most general.
These type classes provide a generic mechanism
to combine values and apply functions within a context,
from which we can fashion monads and a variety of other combinators.

`Semigroupal` and `Applicative` are
most commonly used as a means of
combining independent values such as
the results of validation rules.
Cats provides the `Validated` type
for this specific purpose,
along with apply syntax as
a convenient way to express
the combination of rules.

We have almost covered
all of the functional programming concepts
on our agenda for this book.
The next chapter covers `Traverse` and `Foldable`,
two powerful type classes for converting between data types.
After that we'll look at several case studies
that bring together all of the concepts from Part I.


-->

## まとめ

本書で扱った計算の連結を可能とするデータ型のうちモナドとファンクターがもっとも広く使用されるが、もっとも汎用的なのは Semigroupal とアプリカティブである。これらの型クラスは、値同士を結合しコンテキスト内で関数を適用するための汎用的なメカニズムを提供する。また、そこからモナドや他のさまざまなコンビネータを作り出すことができる。

`Semigroupal` と `Applicative` は、バリデーションの結果など互いに独立した値同士を結合する手段として、もっとも一般的に使用される。Cats ライブラリは、特にこの目的のために `Validated` 型を提供し、バリデーションルールの結合を表現する便利な方法として `apply` 構文も用意している。

本書のテーマに掲げた関数型プログラミングの概念はこれでほぼ網羅した。次の章では、データ型間の変換を行う強力な型クラスである `Traverse` と `Foldable` について説明する。その後、第1部で紹介したすべての概念を活用するいくつかのケーススタディを見ていく。
