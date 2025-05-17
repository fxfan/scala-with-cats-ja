<!--

## Conventions Used in This Book {-}

This book contains a lot of technical information and program code.
We use the following typographical conventions
to reduce ambiguity and highlight important concepts:

### Typographical Conventions {-}

New terms and phrases are introduced in *italics*.
After their initial introduction they are written in normal roman font.

Terms from program code, filenames, and file contents,
are written in `monospace font`.
Note that we do not distinguish between singular and plural forms.
For example, we might write `String` or `Strings` to refer to `java.lang.String`.

References to external resources are written as [hyperlinks][link-underscore].
References to API documentation are written
using a combination of hyperlinks and monospace font,
for example: [`scala.Option`][scala.Option].

### Source Code {-}

Source code blocks are written as follows.
Syntax is highlighted appropriately where applicable:

```scala mdoc:silent
object MyApp extends App {
  println("Hello world!") // Print a fine message to the user!
}
```

Most code passes through [mdoc][link-mdoc] to ensure it compiles.
mdoc uses the Scala console behind the scenes,
so we sometimes show console-style output as comments:

```scala mdoc
"Hello Cats!".toUpperCase
```

### Callout Boxes {-}

We use two types of *callout box* to highlight particular content:

<div class="callout callout-info">
Tip callouts indicate handy summaries, recipes, or best practices.
</div>

<div class="callout callout-warning">
Advanced callouts provide additional information
on corner cases or underlying mechanisms.
Feel free to skip these on your first read-through---come
back to them later for extra information.
</div>


```scala mdoc:reset:silent
```
--->

## 本書の表記 {-}

本書には技術的な情報やプログラムコードが多数含まれている。文章構成を明確にし、重要な概念を強調するために、以下のような表記規約を用いる。

### 表記規約 {-}

新たに導入される用語や語句は*イタリック体*で示す。初出以降は通常フォントで表記される。

プログラムコード中の用語、ファイル名、ファイルの内容は、`等幅フォント（monospace font）` で表記する。単数形と複数形の区別はしていない点に注意してほしい。たとえば `java.lang.String` を指して `String` や `Strings` と表記することがある。

外部リソースへの参照は[ハイパーリンク][link-underscore]によって示される。API ドキュメントへの参照は、[`scala.Option`][scala.Option] のようにハイパーリンクと等幅フォントを組み合わせて表記する。

### ソースコード {-}

ソースコードのブロックは次のように記述される。可能な場合はシンタックスハイライトが適用される。

```scala mdoc:silent
object MyApp extends App {
  println("Hello world!") // Print a fine message to the user!
}
```

ほとんどのコードは [mdoc][link-mdoc] を通じて処理され、コンパイル可能であることが保証されている。
mdoc は内部的に Scala コンソールを使用するため、ときには出力をコメント形式で示すこともある。

```scala mdoc
"Hello Cats!".toUpperCase
```

### 強調枠 {-}

特定の内容を際立たせるために、次の二種類の*強調枠（callout box）*を使用する。

<div class="callout callout-info">
ティップス枠は、簡単な要約やコツ、ベストプラクティスを示す。
</div>

<div class="callout callout-warning">
補足枠は、例外的なケースや基盤となるしくみに関する追加情報を提供する。初読では読み飛ばしても構わない。必要になったときに戻って参照してほしい。
</div>
