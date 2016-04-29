# 函数调用 / Hello World

我选中Ruby作为我的实现语言时相当武断。在这个阶段，这个选择还不是那么重要，但是我真的喜欢Ruby语言。

然而，有些时候，有一系列步骤会让语言和编译器连接的更紧密。我的意思是我想要编译器可以自举：它应该有能力编译它自己。这意味着编译器要么不得不至少支持Ruby语言的一个子集，要么有翻译步骤让编译器能够理解以该语言重写编译器的部分。尽管这没有规定实现的语言，这意味着确保你所实现的语言跟你想要达到的目标是颇为接近的至少总是值得的，除非你对于使编译器自举毫无兴趣。

It is also worthwhile to seriously avoid using complex language constructs in your implementation if you want the compiler to become self-hosted. Keep in mind that you need to support every language construct the compiler will need, and so if you use lots of fancy functionality you'll either have to support equivalent functionality in the compiler, or run the risk of having to do massive changes to the structure when you're ready to take the jump. Not fun.

One distinct benefit of Ruby as an implementation language, however (and it shares this with certain other languages, such as Lisp), is that you can build literal tree structures very easily - in Ruby using Array or Hash and in Lisp using lists.

It means I can start out by "faking" the abstract syntax tree as arrays, and neatly skip over the step of writing a parser. Yay! The cost is ugly, but temporarily bearable, syntax.