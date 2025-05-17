<!--

## Summary

We hit a big milestone in this chapter---we covered
our first type classes with fancy functional programming names:

 -  a `Semigroup` represents an addition or combination operation;
 -  a `Monoid` extends a `Semigroup` by adding an identity or "zero" element.

We can use `Semigroups` and `Monoids` by importing two things:
the type classes themselves, 
and the semigroup syntax to give us the `|+|` operator:

```scala mdoc:silent
import cats.Monoid
import cats.syntax.semigroup.* // for |+|
```

```scala mdoc
"Scala" |+| " with " |+| "Cats"
```

With the correct instances in scope,
we can set about adding anything we want:

```scala mdoc
Option(1) |+| Option(2)
```

```scala mdoc:silent
val map1 = Map("a" -> 1, "b" -> 2)
val map2 = Map("b" -> 3, "d" -> 4)
```

```scala mdoc
map1 |+| map2
```

```scala mdoc:silent
val tuple1 = ("hello", 123)
val tuple2 = ("world", 321)
```

```scala mdoc
tuple1 |+| tuple2
```

We can also write generic code that works with any type
for which we have an instance of `Monoid`:

```scala mdoc:silent
def addAll[A](values: List[A])
      (using monoid: Monoid[A]): A =
  values.foldRight(monoid.empty)(_ |+| _)
```

```scala mdoc
addAll(List(1, 2, 3))
addAll(List(None, Some(1), Some(2)))
```

`Monoids` are a great gateway to Cats.
They're easy to understand and simple to use.
However, they're just the tip of the iceberg
in terms of the abstractions Cats enables us to make.
In the next chapter we'll look at *functors*,
the type class personification of the beloved `map` method.
That's where the fun really begins!


```scala mdoc:reset:silent
```
--->

## まとめ

この章で我々は大きなマイルストーンに到達した。はじめての型クラスとして、ファンシーな関数型プログラミング用語を名前にもつ型クラスについて学んだ[^tn-monoids-summary-01]。

[^tn-monoids-summary-01]: 【訳注】 本書の第一版とも言える Scala with Cats では Cats への導入として `Show` や `Eq` を紹介する[@sec:cats]章がなかった。そのため、`Monoid` と `Semigroup` が本書で最初に触れる型クラスだった。

- `Semigroup` は加算や結合の操作を表す
- `Monoid` は `Semigroup` を拡張し、単位元もしくは零元を追加したものを表す

`Semigroup` や `Monoid` を使用するには、型クラス本体と、`|+|` 演算子を提供する `Semigroup` の構文をインポートすればよい。

```scala mdoc:silent
import cats.Monoid
import cats.syntax.semigroup.* // |+| のインポート
```

```scala mdoc
"Scala" |+| " with " |+| "Cats"
```

スコープに適切なインスタンスがありさえすれば、あらゆるものを結合できるようになる。

```scala mdoc
Option(1) |+| Option(2)
```

```scala mdoc:silent
val map1 = Map("a" -> 1, "b" -> 2)
val map2 = Map("b" -> 3, "d" -> 4)
```

```scala mdoc
map1 |+| map2
```

```scala mdoc:silent
val tuple1 = ("hello", 123)
val tuple2 = ("world", 321)
```

```scala mdoc
tuple1 |+| tuple2
```

また、`Monoid` インスタンスをもつ任意の型に対して動作する汎用的なコードを書くこともできる。

```scala mdoc:silent
def addAll[A](values: List[A])
      (using monoid: Monoid[A]): A =
  values.foldRight(monoid.empty)(_ |+| _)
```

```scala mdoc
addAll(List(1, 2, 3))
addAll(List(None, Some(1), Some(2)))
```

`Monoid` は理解が容易く、それに使いやすい。Cats の世界への導入として絶好の素材である。だが、モノイドは Cats が可能にする抽象化のほんの一端にすぎない。次の章では、多くの人に愛されている `map` メソッドの型クラス版である*ファンクター（functor）*を取り上げる。関数型プログラミングの本当の楽しさはまさにここから始まると言っていいだろう。
