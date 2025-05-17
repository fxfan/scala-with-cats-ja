<!--

## Conclusions

In this chapter we explored two main techniques for optimizing interpeters: algebraic simplification of programs, and interpretation in a virtual machine.

Our regular expression derivative algorithm is taken from [Regular-expression derivatives reexamined][rere].
What we didn't explore, but we should if we really care about performance, is compiling regular expressions to a finite state machine, another kind of virtual machine.
Regular expression derivatives are very easy to implement and nicely illustrate the point of algebraic simplification.
However we have to recompute the derivative on each input character.
If we instead compile the regular expression to a finite state machine ahead of time, we save time when parsing input.
The details of this algorithm are in the paper.


This work is based on [Derivatives of Regular Expressions][regexp-deriv]. Derivatives of Regular Expressions was published in 1964. Although the style of the paper will be immediately recognizable to anyone familiar with the more theoretical end of computer science, anachronisms like "State Diagram Construction" are a reminder that this work comes from the very beginnings of the discipline. Regular expression derivatives can be extended to context-free grammars and therefore used to implement parsers. This is explored in [Parsing with Derivatives][parsing-deriv].

[regexp-deriv]: https://dl.acm.org/doi/pdf/10.1145/321239.321249
[rere]: https://www.khoury.northeastern.edu/home/turon/re-deriv.pdf
[parsing-deriv]: https://matt.might.net/papers/might2011derivatives.pdf


A lot of work has looked at systematically transforming an interpreter into a compiler and virtual machine.
[From Interpreter to Compiler and Virtual Machine: A Functional Derivation][interpreter-to-compiler] is an earlier example. [Calculating Correct Compilers][calculating-correct] is more recent, and follow-up papers extend the technique in a number of directions.

[interpreter-to-compiler]: https://www.brics.dk/RS/03/14/BRICS-RS-03-14.pdf
[calculating-correct]: https://www.cambridge.org/core/journals/journal-of-functional-programming/article/calculating-correct-compilers/70AA17724EBCA4182B1B2B522362A9AF


Interpreter and their optimization is an enormous area of work. It also one I find very interesting, so I've been a bit more through in collecting references for this section.

We looked at four techniques for optimization: algebraic simplification, byte code, stack caching, and superinstructions. 
Algebraic simplification is as old as algebra, and something familiar to any secondary school student. 
In the world of compilers, different aspects of algebraic simplification are known as constant folding, constant propagation, and common subexpression elimination. 
Byte code is probably as old as interpreters, and dates back to at least the 1960s in the form of [p-code]. 
[Stack Caching for Interpreters][stack-caching] introduces the idea of stack caching, and shows some rather more complex realizations than the simple system I used. 
Superinstructions were introduced in [Optimizing an ANSI C interpreter with superoperators][superoperators].
[Towards Superinstructions for Java Interpreters][towards-super] is a nice example of applying superinstructions to a interpreted JVM. 

Let's now talk about instruction dispatch, which is area we did not consider for optimization.
Instruction dispatch is the process by which the interpreter chooses the code to run for a given interpreter instruction. 
[The Structure and Performance of Efficient Interpreters][spei] argues that instruction dispatch makes up a major portion of an interpreter's execution time.
The approach we used is generally called switch dispatch in the literature.
There are several alternative approaches.
Direct threaded dispatch is described in [Threaded Code][threaded-code]. Direct threading represents an instruction by the function that implements it. This requires first-class functions and full tail calls. It is generally considered the fastest form of dispatch. Notice that it relies on the duality between data and functions.
Subroutine threading is like direct threading, but uses normal calls and returns instead of tail calls.
In indirect threaded code (described in [Indirect Threaded Code][indirect-threaded-code]), each bytecode is the index into a lookup table that points to the implementing function.

Stack machines are not the only virtual machine used for implementing interpreters. Register machines are the most common alternative. The Lua virtual machine, for example, is a register machine. [Virtual Machine Showdown: Stack Versus Registers][stacks-vs-registers] compares the two and concludes that register machines are faster. However they are more complex to implement.

If you're interested in the design considerations in a general purpose stack based instruction set, [Bringing the Web up to Speed with WebAssembly][wasm] is the paper for you. It covers the design of WebAssembly, and the rationale behind the design choices. An interpreter for WebAssembly is described in [A Fast In-Place Interpreter for WebAssembly][wasm-interp]. Notice how often tail calls arise in the discussion!


[stack-caching]: https://dl.acm.org/doi/pdf/10.1145/207110.207165
[p-code]: https://en.wikipedia.org/wiki/P-code_machine
[superoperators]: https://dl.acm.org/doi/abs/10.1145/199448.199526
[towards-super]: https://core.ac.uk/download/pdf/297029962.pdf 
[spei]: https://jilp.org/vol5/v5paper12.pdf 
[threaded-code]: https://dl.acm.org/doi/pdf/10.1145/362248.362270
[indirect-threaded-code]: http://figforth.org.uk/library/Indirect.Threaded.Code.p330-dewar.pdf 

[wasm]: https://dl.acm.org/doi/pdf/10.1145/3062341.3062363
[stacks-vs-registers]: https://dl.acm.org/doi/pdf/10.1145/1328195.1328197 
[wasm-interp]: https://dl.acm.org/doi/pdf/10.1145/3563311


-->

## まとめ

この章では、インタープリタの最適化におけるふたつの主要な手法である、プログラムの代数的簡略化と仮想マシンによる解釈について考察した。

正規表現の微分アルゴリズムは [Regular-expression derivatives reexamined][rere] から採用した。今回は触れなかったが、性能を本当に重視するのであれば、正規表現の有限状態機械へのコンパイルを検討したほうがよい。有限状態機械はある種の仮想マシンである。正規表現の微分は実装が非常に簡単で、代数的簡略化の意義をよく示している。ただし、この手法では入力文字ごとに微分を再計算する必要がある。一方で、正規表現を事前に有限状態機械にコンパイルしておけば、入力の解析時に時間を節約できる。このアルゴリズムの詳細は論文に記載されている。

このアイデアは、1964年に発表された [Derivatives of Regular Expressions][regexp-deriv] に基づいている。計算機科学の理論的分野に精通した人ならすぐにこの論文のスタイルを認識できるだろう。「状態図の構築」などのアナクロな言葉が登場するのは、この研究が計算機科学の黎明期に行われた名残である。正規表現の微分は文脈自由文法にも拡張可能であり、パーサの実装にも利用できる。それについては [Parsing with Derivatives][parsing-deriv] で論じられている。

[regexp-deriv]: https://dl.acm.org/doi/pdf/10.1145/321239.321249
[rere]: https://www.khoury.northeastern.edu/home/turon/re-deriv.pdf
[parsing-deriv]: https://matt.might.net/papers/might2011derivatives.pdf


多くの研究によって、インタープリタをコンパイラや仮想マシンに体系的に変換する手法が探求されてきた。[From Interpreter to Compiler and Virtual Machine: A Functional Derivation][interpreter-to-compiler] はその初期の例である。より最近の例としては [Calculating Correct Compilers][calculating-correct] があり、後続の関連論文では、この手法をさまざまな方向に拡張している。

[interpreter-to-compiler]: https://www.brics.dk/RS/03/14/BRICS-RS-03-14.pdf
[calculating-correct]: https://www.cambridge.org/core/journals/journal-of-functional-programming/article/calculating-correct-compilers/70AA17724EBCA4182B1B2B522362A9AF


インタープリタとその最適化は極めて広大な研究分野である。私自身が強い興味をもっている分野であることから、この節では参考文献をやや多めに集めた。

本章では、代数的簡略化、バイトコード、スタックキャッシング、スーパーインストラクションという4つの最適化手法を見てきた。代数的簡略化は代数と同じくらい古い歴史をもち、中高校生にも馴染みのあるものである。コンパイラの分野では、代数的簡略化の別の側面が、定数畳み込み（constant folding）・定数伝播（constant propagation）・共通部分式除去（common subexpression elimination）として知られている。バイトコードはおそらくインタープリタと同じくらい古く、少なくとも1960年代の [p-code] の形にまで遡ることができる。[Stack Caching for Interpreters][stack-caching] ではスタックキャッシングのアイデアが紹介されており、本章で扱ったシンプルなシステムよりもかなり複雑な実現方法を示している。スーパーインストラクションは [Optimizing an ANSI C interpreter with superoperators][superoperators] で紹介された。[Towards Superinstructions for Java Interpreters][towards-super] は JVM のインタープリタ実行にスーパーインストラクションを適用した好例である。

次に命令ディスパッチについて話そう。これは最適化としては考慮しなかった領域である。命令ディスパッチとは、インタープリタが自らを構成する処理ロジックの中から特定の命令に対応するものを選んで実行するプロセスを指す。[The Structure and Performance of Efficient Interpreters][spei] は、命令ディスパッチがインタープリタの実行時間の大部分を占めると主張している。本章で使用した方法は、文献では一般にスイッチディスパッチと呼ばれているものである。これにはいくつかの代替手法が存在する。そのひとつである直接スレッディングについては [Threaded Code][threaded-code] に記述されている。この手法では、命令はそれを実装する関数として表現される。これにはファーストクラス関数と末尾呼び出し最適化が必要であり、一般に最速のディスパッチ形式とされている。この手法はデータと関数の双対性に依存していることに注意が必要である。サブルーチンスレッディングは直接スレッディングに似ているが、末尾呼び出しの代わりに通常の呼び出しと戻りを使用する。[Indirect Threaded Code][indirect-threaded-code] で論じられている間接スレッデッドコードでは、実装関数を指すルックアップテーブルのインデックスがバイトコードとして用いられる。

インタープリタの実装に使われる仮想マシンはスタックマシンだけではない。レジスタマシンはもっとも一般的な代替手法である。たとえば Lua の仮想マシンはレジスタマシンとして実装されている。[Virtual Machine Showdown: Stack Versus Registers][stacks-vs-registers] は両者を比較し、レジスタマシンのほうが速いと結論付けている。ただし、実装はレジスタマシンのほうが複雑である。

汎用的なスタックベースの命令セットの設計に興味があるなら、[Bringing the Web up to Speed with WebAssembly][wasm] を読むとよいだろう。この論文では WebAssembly の設計とその設計が選択された背景について述べられている。また、WebAssembly 用のインタープリタについては [A Fast In-Place Interpreter for WebAssembly][wasm-interp] で解説されている。その議論の中で末尾呼び出しについて頻繁に言及されていることにも注目してほしい。

[stack-caching]: https://dl.acm.org/doi/pdf/10.1145/207110.207165
[p-code]: https://en.wikipedia.org/wiki/P-code_machine
[superoperators]: https://dl.acm.org/doi/abs/10.1145/199448.199526
[towards-super]: https://core.ac.uk/download/pdf/297029962.pdf 
[spei]: https://jilp.org/vol5/v5paper12.pdf 
[threaded-code]: https://dl.acm.org/doi/pdf/10.1145/362248.362270
[indirect-threaded-code]: http://figforth.org.uk/library/Indirect.Threaded.Code.p330-dewar.pdf 

[wasm]: https://dl.acm.org/doi/pdf/10.1145/3062341.3062363
[stacks-vs-registers]: https://dl.acm.org/doi/pdf/10.1145/1328195.1328197 
[wasm-interp]: https://dl.acm.org/doi/pdf/10.1145/3563311
