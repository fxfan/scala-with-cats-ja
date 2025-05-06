<!--

## Summary

In this chapter we've seen monads up-close.
We saw that `flatMap` can be viewed
as an operator for sequencing computations,
dictating the order in which operations must happen.
From this viewpoint, `Option` represents a computation
that can fail without an error message,
`Either` represents computations that can fail with a message,
`List` represents multiple possible results,
and `Future` represents a computation
that may produce a value at some point in the future.

We've also seen some of
the custom types and data structures that Cats provides,
including `Id`, `Reader`, `Writer`, and `State`.
These cover a wide range of use cases.

Finally, in the unlikely event that
we have to implement a custom monad,
we've learned about defining our own instance using `tailRecM`.
`tailRecM` is an odd wrinkle that is a concession to building
a functional programming library that is stack-safe by default.
We don't need to understand `tailRecM` to understand monads,
but having it around gives us benefits
of which we can be grateful when writing monadic code.


--->

## まとめ

この章ではモナドについて詳しく見てきた。`flatMap` は、複数の計算を順序付けてつなげる演算子と見ることができた。この観点から見ると、`Option` はエラーメッセージなしで失敗する可能性のある計算を、`Either` はメッセージ付きで失敗するかもしれない計算を、`List` は複数の結果をもつ可能性のある計算を、そして `Future` は値を将来のある時点で返すかもしれない計算を、それぞれ表している。

また、Cats が提供する独自の型やデータ構造についても学んだ。そこには `Id`、`Reader`、`Writer`、`State` が含まれており、幅広いユースケースをカバーしている。

最後に、滅多にないとは思うが、独自のモナドを実装しなければならない時のために、`tailRecM` を実装して独自の `Monad` インスタンスを定義する方法を学んだ。`tailRecM` は特殊な仕組みで、デフォルトでスタックセーフな関数型プログラミングライブラリを構築するための妥協点として存在する。モナドを理解するために `tailRecM` の理解は必須ではないが、モナディックなコードを書く際にその恩恵を受けられるのはありがたいことである。
