<!--

# Preface {-}

Some twenty years ago I started my first job in the UK.
This job involved a commute by train, giving me about an hour a day to read without distraction.
Around about the same time I first heard about *Structure and Interpretation of Computer Programs*, referred to as the "wizard book" and spoken of in reverential terms.
It sounded like the just the thing for a recent graduate looking to become a better developer.
I purchased a copy and spent the journey reading it, doing most of the exercises in my head.
*Structure and Interpretation of Computer Programs* was already an old book at this time, and it's programming style was archaic.
However it's core concepts were timeless and it's fair to say it absolutely blew my mind, putting me on a path I'm still on today.

Another notable stop on this path occured some ten years ago when Dave and I started writing *Scala with Cats*.
In *Scala with Cats* we attempted to explain the core type classes found in the Cats library, and their use in building software.
I'm proud of the book we wrote together, but time and experience showed that type classes are only a small piece of the puzzle of building software in a functional programming style.
We needed a much wider scope if we were to show people how to effectively build software with all the tools that functional programming provides.
Still, writing a book is a lot of work, and we were busy with other projects, so *Scala with Cats* remained largely untouched for many years.

Around 2020 I got the itch to return to *Scala with Cats*.
My initial plan was simply to update the book for Scala 3.
Dave was busy with other projects so I decided to go alone.
As the writing got underway I realized I really wanted to cover the additional topics I thought were missing.
If *Scala with Cats* was a good book, I wanted to aim to write a great book; one that would contain almost everything I had learned about building software.
The title *Scala with Cats* no longer fit the content, and hence I adopted a new name for what is largely a new book.
The result, *Functional Programming Strategies in Scala with Cats*, is what you are reading now.
I hope you find it useful, and I hope that just maybe some young developer will find this book inspiring the same way I found *Structure and Interpretation of Computer Programs* inspiring all those years ago.


## Preface from Scala with Cats {-}

The aims of this book are two-fold:
to introduce monads, functors, and other functional programming patterns
as a way to structure program design,
and to explain how these concepts are implemented in [Cats][link-cats].

Monads, and related concepts, are the functional programming equivalent
of object-oriented design patterns---architectural building blocks
that turn up over and over again in code.
They differ from object-oriented patterns in two main ways:

- they are formally, and thus precisely, defined; and
- they are extremely (extremely) general.

This generality means they can be difficult to understand.
*Everyone* finds abstraction difficult.
However, it is generality that allows concepts like monads
to be applied in such a wide variety of situations.

In this book we aim to show the concepts in a number of different ways,
to help you build a mental model
of how they work and where they are appropriate.
We have extended case studies, a simple graphical notation,
many smaller examples, and of course the mathematical definitions.
Between them we hope you'll find something that works for you.

Ok, let's get started!


-->

# 序文 {-}

二十年ほど前、私はイギリスで初めての仕事に就いた。職場へは電車で通勤する必要があったため、一日に一時間ほど邪魔の入らない読書の時間を得ることができた。初めて『計算機プログラムの構造と解釈』のことを耳にしたのは、ちょうどその頃である。この本は表紙の絵柄から「魔術師本（Wizard Book）」とも呼ばれ、開発者たちのあいだで特別な敬意をもって語られていた。学校を卒業したばかりの、優れた開発者を目指していた自分にとって、それはまさに最適な一冊のように思われた。私はその本を購入し、通勤のあいだに読み進めた。ほとんどの演習問題は頭の中で解いた。『計算機プログラムの構造と解釈』は当時すでに古典であり、そのプログラミングスタイルは時代遅れだったが、その核心となる概念は時代を超えるものだった。それはまさに衝撃的で、自分の進む道を決定づけるきっかけとなった。私はその道をいまでも歩み続けている。

その道程におけるもうひとつの重要なできごとは、十年ほど前に Dave とともに『Scala with Cats』の執筆を始めたことである。同書では Cats ライブラリに含まれる主要な型クラスとそれらを用いたソフトウェア構築について解説しようとした。私たちがともに書き上げたこの本には誇りを感じている。だが、時を経て経験を重ねるうちに、型クラスは関数型プログラミングによるソフトウェア構築というパズルの一片にすぎないことが明らかになった。関数型プログラミングが提供するすべてのツールを活用してソフトウェアを効果的に構築する方法を示すには、対象範囲をもっと広げなければならない。とはいえ、一冊の本を書くのは大変な作業であり、私たちが他のプロジェクトに忙殺されていたこともあり、『Scala with Cats』は何年ものあいだほとんど手を加えられないままだった。

2020年ごろ、もう一度『Scala with Cats』に取り組みたいという気持ちが私の中に湧いてきた。当初の計画は、Scala3 に対応させるための単なる改訂にすぎなかった。Dave は他の仕事で忙しかったため、私は単独で執筆を進めることにした。だが、執筆を進めるうちに、前著に欠けていた話題を盛り込みたいと思っている自分に気づいた。『Scala with Cats』が良い本だったとすれば、今回は素晴らしい本を目指したかった。私がソフトウェア構築について学んできたことのほぼすべてを詰め込んだ一冊にしたかった。その内容にとって『Scala with Cats』というタイトルはもはやふさわしくなかったので、内容に見合った新しい名前を採用した。その『Functional Programming Strategies in Scala with Cats』こそ、いまあなたが読んでいるこの本である。この本があなたの助けとなることを願っている。そして、かつて私が『計算機プログラムの構造と解釈』に心を打たれたように、若い開発者の心を動かす本になれば幸いである。

## 『Scala with Cats』版 序文  {-}

本書の目的はふたつある。第一に、モナド、ファンクター、およびその他の関数型プログラミングのパターンを、プログラム設計の構造化手法として紹介すること。第二に、それらの概念が [Cats][link-cats] においてどのように実装されているかを知ってもらうことである。

関数型プログラミングにおけるモナドやそれに関連する概念は、オブジェクト指向でいうところのデザインパターンに相当する。いずれも、アーキテクチャ上の構成要素として、コードの中に繰り返し登場する。ただし、オブジェクト指向のパターンとは主に次の二点において異なる。

- 定義が形式的であり、それゆえ正確であること
- 極めて（本当に極めて）一般性が高いこと

この一般性の高さゆえに、これらの概念は理解しづらいものになっている。抽象概念は誰にとっても難しい。しかし、モナドなどの概念がこれほど多様な場面に応用できるのは、この一般性のおかげでもある。

本書では、これらの概念がどのように機能しどこで使えるのかというメンタルモデルを構築できるよう手助けしたい。そのためにこれらの概念をさまざまな角度から見ていく。具体的には、発展的なケーススタディやいくつもの小さな例、簡潔な図式表記、そしてもちろん数学的な定義も用意している。どれかひとつでも読者のみなさんの理解の助けになるものがあれば幸いである。

では、始めよう。
