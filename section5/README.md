# 数字字面量；if .. then .. else

上次我承诺会更频繁的发布这些文章，但却没有找什么时间。作为回报，我花了一些时间把5,6和7部分那些比较短的章节结合起来。我们开始看看：

###处理数字字面量

目前为止，到目前为止我们的程序只把数字作为一种意外的副产品而不做任何形式的类型检查而处理，甚至只有一种外部函数可以产生它们。

所以，让我们看看gcc是怎样处理C语言中从integers到long long等数据类型的。我们仍然现在它为32位x86平台。
```cpp
    void foo1(unsigned short a) {}
    void foo2(signed short a) { }
    void foo3(unsigned int a) {}
    void foo4(signed int a) { }
    void foo5(unsigned long a) {}
    void foo6(signed long a) {}

    void foo7(unsigned long long a) { }
    void foo8(signed long long  a) {}

    int main()
    {
      foo1(1);
      foo2(2);
      foo3(3);
      foo4(4);
      foo5(5);
      foo6(6);
      foo7(7);
    }
```
我忽略了大部分的gcc输出代码，如果你比较介意的话，你可以自己执行`gcc -S`来查看。有趣的地方是对不同函数的调用，它们都是以如下形式结束的：
```cpp
        movl    $1, (%esp)
        call    foo1
        movl    $2, (%esp)
        call    foo2
        movl    $3, (%esp)
        call    foo3
        movl    $4, (%esp)
        call    foo4
        movl    $5, (%esp)
        call    foo5
        movl    $6, (%esp)
        call    foo6
        movl    $7, (%esp)
        movl    $0, 4(%esp)
        call    foo7
```
换句话说，为了至少一个函数调用的目的，gcc把除了long long类型之外的所有类型都作为一个单独的32位的值。为图方便，我们忽略long long类型并把所有值都转化为32位，因为这意味着我们可以完全忽略类型。

懒惰是一种美德啊！

我们为何还要忽略浮点类型呢？因为你可以非常容易的编写一个只包含整数运算的编译器，现在添加浮点运算只是在浪费时间，反正事情无论怎样都会改变的。

此外，在我们年轻的时我们可没有这许多花哨的FPU(Float Point Unit，浮点运算单元)啊。即便这样我们仅仅使用整数固点运算也可以做的非常好了。

那么，我们究竟怎样修改呢？

在`#get_arg`中，在我们处理字符串常量之前，我们添加如下代码：
```ruby
      return [:int, a] if (a.is_a?(Fixnum))
```
在`#compile_exp`中，我们添加如下代码来使用从`#get_arg`返回的值：
```ruby
      elsif atype == :int then param = "$#{aparam}"
```
太好了，我们完成了。这真是太容易了。

下面添加一个测试：
```ruby
    prog = [:do,
      [:printf,"'hello world' takes %ld bytes\\n",[:strlen, "hello world"]],
      [:printf,"The above should show _%ld_ bytes\\n",11]
    ]
```
### 中场休息：对原始数据类型的一些思考

“纯”面向对象语言在其他方面都很好，除了在相对较低级别的代码生成器外，在我看来。无论你是否想实现一个纯面向对象语言，我都坚信首先实现一个可以工作在原始数据类型上的原语是非常值得的。然后，你就可以决定是否要在后面的阶段从用户角度完全隐藏这些原语，或着找到一种可以使它们看起来像对象的方法，或者把它们隐式地转换回后端用户所需类型。

需要注意的是，MRI（Matz Ruby Interpreter，松本行弘的Ruby解释器）就是这样做的：例如，数字在内部就是跟“真的”对象使用不同方式处理的，但是解释器并没有做到最好从用户的角度隐藏这个事实。我个人认为MRI做的还远远不够好。

###If ... then ... else

如果没有一些形式的条件逻辑，你不可能走的更远。一些形式的`if .. then .. else`存在于大部分语言中，想必是很有用的。现在，我们将要按照C语言的方式实现` [:if, condition, if-arm, else-arm]`，换句话说，0和null指针计算为false，其他地都计算为true。

再次以一个简单的测试开始：
```cpp
    void foo() {}
    void bar() {}

    int main()
    {
      if (baz()) {
        foo();
      } else {
        bar();
      }
    }
```
下面给出相关的部分：
```cpp
        call    baz
        testl   %eax, %eax
        je  .L6
        call    foo
        jmp .L10
    .L6:
        call    bar
    .L10:
```
`if .. then .. else`是一个非常普遍的编译模式，跨越了大量的语言和体系结构：

计算在条件发生时的表达式，以某种方式测试该条件（这里的`testl`——对各种体系结构来说这里的变化通常是`cmp`对比指令或者是从自身减去寄存器）。`testl`对左操作数和右操作数进行对比，并设置各种标志。接下来，对else分支做条件判断。换句话说，我们可以检查这个条件是否为非真。在这种状况下，它会执行`je`操作，即`如果为真就跳转`。换句话说，如果前面的比较标记的结果是相等的（注意，在大多数CPU上，大量的指令会设置条件代码，而不是仅仅显式测试）。然后执行`if-arm`。

跳过`else-arm`到全部的`if..then..else`构造结束之后。为`else-arm`放置一个标签和代码来执行它。放置一个标签来标记结束。这里会有很多变化，例如根据估算哪个条件最有可能并了解是否有遗漏或选取特定体系结构上的最便宜的分支来重新排列if/else分支，上述即是现在的一切所需。

这个编译看起来又是非常简单——这真的只是对上述的一个直接翻译：
```ruby
      def ifelse cond, if_arm,else_arm
        compile_exp(cond)
         puts "\ttestl   %eax, %eax"
         @seq += 2
         else_arm_seq = @seq - 1
         end_if_arm_seq = @seq
         puts "\tje  .L#{else_arm_seq}"
         compile_exp(if_arm)
         puts "\tjmp .L#{end_if_arm_seq}"
         puts ".L#{else_arm_seq}:"
         compile_exp(else_arm)
         puts ".L#{end_if_arm_seq}:"
      end
```
它非常具有自我说明性——它基本上为每个条件都调用`compile_exp`方法，`if-arm`和`else-arm`并且编织成上面的代码，使用`@seq`产生唯一的标签。

为了钩住它，我们把如下代码添加到`#compile_exp`中的`return defun ....`之后：
```ruby
      return ifelse(*exp[1..-1]) if (exp[0] == :if)
```
如下是一个简的测试：
```ruby
    prog = [:do,
      [:if, [:strlen,""],
        [:puts, "IF: The string was not empty"],
        [:puts, "ELSE: The string was empty"]
      ],
      [:if, [:strlen,"Test"],
        [:puts, "Second IF: The string was not empty"],
        [:puts, "Second IF: The string was empty"]
      ]
    ]
```
如下是测试的结果

像往常一样，你要这样运行它：
```sh
    $ ruby step5.rb >step5.s
    $ make step5
    cc    step5.s   -o step5
    $ ./step5
    ELSE: The string was empty
    Second IF: The string was not empty
    $
```
### 关于循环的一些思考

添加无条件循环到语言中是微不足道的，但我们需要吗？答案严格来说是不。我们已经可以使用递归来做循环，为什么还要再破坏语言呢？

虽然这是一个很好的解决方案，但是我们需要尾部递归消除，我不准备现在进入这个话题。尾递归消除，或更通俗的形式——尾调用消除——本质上是指确认一个函数的退出情况表现为一个相同或更少的参数的调用，其次是返回它的返回值。如果是这样的话，你可以使用正在调用的函数的参数来重新使用当前函数调用帧，后面跟着一个`jmp`指令代替`call`指令。`jmp`不会将返回地址压入堆栈，所以当跳转到返回的函数时，它将返回到调用该函数的地方，而不是到当前函数。

这实现了两件事情：第一，栈不会增长。第二，你节省了一些宝贵的周期。与适当的其他优化向结合，包括尾调用消除，你可以实现这样的循环，而不必担心堆栈溢出：
```ruby
    [:defun, :loop, [], [:do,
      [:puts, "I am all loopy"],
      [:loop]
    ],

    [:loop]
```
换句话说，尾调用消除意味着所有`(defun foo () (do bar foo))`形式的函数，上堆栈的使用量与尾调用的深度成正比，在那个点上为1帧的深度。

使用当前的编译器运行上述代码，它将会迅速的溢出堆栈并崩溃，太糟糕了。

我感到非常困扰。一想到堆栈会随着每一次迭代而不断增长，我都感觉糟透了。

现在，我们忽略这一点，且不再做任何滥用堆栈的事情。相反，我们稍后实现一个完整的循环架构作为一个原语。至少现在——如果我添加尾调用消除，我们可以考虑再把它弄出来并作为替代运行时库的一部分。

那么，我们可以做非终止循环。那对我们现在就不好了吗？

是的，我们也已经可以做while循环了：
```lisp
    (defun some-while-loop () (if condition (some-while-loop) ()))
```

虽然不是很满意，但它是可以工作的。对于一个可以接受的解决办法来说，它实在是太丑了，但是，这样一个`while`语言是在列表上实现的。

（为了更好的领会，我是用类Lisp语法来表示的）

我不是Lisp程序员。看到这么多的括号我都感觉头疼。但是当我们的语言实际上没有自己的语法时用Lisp语法是相当方便的。当是时候编写一个完整的解析器时我们的编译器就有了自己的语法。顺便说一句，大部分我们已经实现或着将要实现的将忍受一些类似于严重碎裂的Lisp语法——事实证明，Lisp架构是非常强大的，如果你能忍受它的语法的话。即使你不想要用它来编程，学习更多的Lisp和Scheme（Lisp的一个分支）也是值得的。

我个人不会留心我的建议，大多数我将使用的类Lisp想法可能都是基于在非常有限的时间内学习Lisp所得到的不完全的理解。
