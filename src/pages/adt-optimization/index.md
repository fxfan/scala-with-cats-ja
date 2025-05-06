<!--

# Optimizing Interpreters and Compilers

In a previous chapter we introduced interpreters as a key strategy in functional programming.
In many cases simple structurally recursive interpreters are sufficient.
However, in a few cases we need more performance than they can offer so in this chapter we'll turn to optimization.
This is a huge subject, which we cannot hope to cover in just one book chapter.
Instead we'll focus on two techniques that I believe use key ideas found in more complex techniques: algebraic manipulation and compilation to a virtual machine.

We'll start looking at algebraic manipulation, returning to the regular expression example we used earlier.
We'll then move to virtual machine, this time using a simple arithmetic interpreter example. 
We'll see how we can compile code to a stack machine, and then look at some of the optimizations that are available when we 
use a virtual machine.


-->

# インタープリタとコンパイラの最適化

以前の章で、インタープリタを関数型プログラミングにおける重要な戦略として紹介した。多くの場合、単純な構造的再帰によるインタープリタで十分でだが、いくつかのケースではそれ以上のパフォーマンスが必要になる。そこで、この章では最適化について取り組む。このテーマは非常に幅広く、ひとつの章ですべてを網羅することは望むべくもない。そこで、複雑な技法から取り出した重要なアイデアを利用したふたつの技法、代数的操作と仮想マシンへのコンパイルに焦点を当てる。

まず、以前に使用した正規表現の例を用いて、代数的操作について見ていく。その後、仮想マシンの話に移る。ここでは単純な算術インタープリタの例を使用する。コードをどのようにスタックマシンへとコンパイルするかを学び、仮想マシンを用いる際に適用することのできる最適化をいくつか見ていく。
