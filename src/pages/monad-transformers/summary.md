<!--

## Summary

In this chapter we introduced monad transformers,
which eliminate the need for nested for comprehensions and pattern matching
when working with "stacks" of nested monads.

Each monad transformer, such as `FutureT`, `OptionT` or `EitherT`,
provides the code needed to merge its related monad with other monads.
The transformer is a data structure that wraps a monad stack,
equipping it with `map` and `flatMap` methods
that unpack and repack the whole stack.

The type signatures of monad transformers are written from the inside out,
so an `EitherT[Option, String, A]` is a wrapper for an `Option[Either[String, A]]`.
It is often useful to use type aliases
when writing transformer types for deeply nested monads.

With this look at monad transformers,
we have now covered everything we need to know about monads
and the sequencing of computations using `flatMap`.
In the next chapter we will switch tack
and discuss two new type classes, `Semigroupal` and `Applicative`,
that support new kinds of operation such as `zipping`
independent values within a context.


-->

## まとめ

この章ではモナド変換子を紹介した。モナド変換子を使えば、ネストしたモナドのスタックを扱う際のネストした for 内包表記やパターンマッチが不要になる。

`FutureT`、`OptionT`、`EitherT` などの各モナド変換子は、それぞれが対応するモナドを他のモナドと統合するためのコードを提供する。変換子はモナドスタックをラップするデータ構造であり、そのスタック全体を `map` や `flatMap` メソッドで展開し、再び組み立てる機能を備えている。

モナド変換子の型シグネチャは内側から外側に向けて書かれる。たとえば `EitherT[Option, String, A]` であれば、それは `Option[Either[String, A]]` をラップしたものである。深くネストしたモナドを扱う変換子の型を書く場合は、型エイリアスが役に立つことが多い。

モナド変換子について見たことで、モナドおよび `flatMap` を用いた計算の順序付けについて知るべきことはすべてカバーした。次の章では話題を変え、`Semigroupal` と `Applicative` というふたつの型クラスについて新たに議論する。これらの型クラスは、コンテキストに包まれている互いに独立した値を `zipping` するといった、本書ではこれまで取り上げてこなかった新しいタイプの操作をサポートしてくれる。
