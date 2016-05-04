# 匿名函数: lamba / call

自从上次发布后，我要从Lisp，Scheme和朋友那里偷师（以及来自其他各个方面的——我觉得没有必要独创该项目。 不管怎样，直到我们走的更远），现在是时候开始介绍一些更强大的概念了。

### 一些延迟求值和匿名函数怎样？

Lambdas，或者说匿名函数，可以被像值一样传递并且在你方便的时候调用（或根本不去调用）。一般来说，他们可以从周围的作用域中访问变量对该函数获取“约束”作为一个环境来允许它传递状态。这就是一个[闭包](http://en.wikipedia.org/wiki/Closure_%28computer_science%29)。这一次我们将要做的不是带来完全的闭包支持，而是它的起始，我们会在将来获得完全的闭包支持。

让我们清楚：编程语言中几乎所有的东西，它们都是语法糖。你可以认为它们等效于定义一个包含单一方法的类来调用它和环境的实例变量，例如（或者你可以把闭包作为[构建一个替代对象系统的方式](http://strlen.com/bla/index.html)来对待——由成为我发现[Amiga E](http://strlen.com/e/index.html)的英雄[Wouter van Oortmerssen](http://strlen.com/)探索的概念。如果你是一个编程语言极客，你需要看一下Wouter这些年所做的东西）——许多概念理论上都是交叉的。

（作为进一步的题外话，你有没有注意到许多像我一样痴迷编程语言的人都有一些可怕的倾向，如构造语句片段并滥用标点符号随意地插入任何地方，或者只有我是这样？）

但不用这么麻烦，并且使用小函数会弄乱你的命名空间。lambdas让你可以内联地定义它们，也可以返回一个代表该函数替代执行它的值。

现在让我们添加这些架构：
```lisp
(lambda (args) body)

(call f (args))
```
该模板将返回这个函数——现在只是它的地址——代替执行它的函数体。`call`将调用它的地址传递它给定的参数，这里没什么惊喜。

那么这与”真正的“闭包有什么区别呢？

真正的闭包棘手的部分是，如果你从周围的作用域引用变量，它们应该为下一次调用”保持存活“。这就是说，它们注定要被保存无论你后来何时调用这个闭包。应该清楚的是仅仅返回函数的地址是不足以实现这个效果的。所以，让我们看一下实现它的一种方式，那么你就可以得到一个相关工作的想法（它并不庞大，但也不是微不足道的）。

我们需要创建一个”环境“，即某个地方用来存储那些闭包创建的应该生存在整个函数的生存周期的变量。这个环境必须存在于堆上，并且每个包围着该lambda表达式的函数的调用必须产生一个新的环境。

返回的是什么必须允许你去访问这个环境。你可以通过创建一个”对象“并使环境代表实例变量来做这件事。或着你可以使用一个”thunk“——一个小的生成函数包括一个指向该对象的指针并加载它到一个预定义的位置在调用该匿名函数之前，或是任何其他的方法。你需要决定什么变量要加到环境中。这可能是在”lambda“表达式执行时可访问的”一切事物“，或者是你可以扫描lambda表达式体并明确的只放入一些被一个或多个在函数中的lambda表达式使用的变量到环境中。后者可能会有更多的空间效率，但是稍微需要更多的工作。

好了，那么现在让我们开始实现”匿名函数“。想往常一样，我会一步一步的通过这些修改，但需要注意的是我也对一些无关紧要的修改做了一些小的清理。我不会再过那些无关紧要的修改，因为那之后越弄越乱。

第一部分是处理”lambda“表达式的代码：
```ruby
      def compile_lambda args, body
        name = "lambda__#{@seq}"
        @seq += 1
        compile_defun(name, args,body)
        puts "\tmovl\t$#{name},%eax"
        return [:subexpr]
      end
```
这段代码是相当自我解释的。它做的所有事情就是创建一个`lambda__[number]`形式的函数名，我们将用它去引用该函数，因为我们不会真正的内联地输出它。没有什么会真的妨碍我们内联的输出它，但我发现它很乱，所以我将只把他当做另外的函数对待。然后，我们调用compile_defun去产生另一个命名函数——所以它只对用户来说是匿名的。然后，我们把该函数的地址移动到`%eax`中，这个是我们建立我们的子表达式运行的结果的地方。请注意，这是另一条捷径——最终我们需要做更复杂的事来处理更复杂的表达式，但寄存器分配是一个复杂的主题（然而也可以这样做，把所有事情都放到栈中工作，但这样会很慢）。

最好我们返回`[:subexpr]`来指明调用者到哪里或怎么发现该表达式的结果。

接下来我们会做一些重构。你可能已经注意到了`#compile_exp`中处理不同参数类型的内部循环变得非常丑陋。所以我们把它提取了出来：
```ruby
      def compile_eval_arg arg
        atype, aparam = get_arg(arg)
        return "$.LC#{aparam}" if atype == :strconst
        return "$#{aparam}" if atype == :int
        return aparam.to_s if atype == :atom
        return "%eax"
      end
```
注意到这里也出现了`:atom`。这使我们可以取得C函数的地址并把它们传递到`:call`中。添加该功能是如此简单，为什么我不去做呢。要添加该功能，我们在`#get_arg`中添加如下代码：
```ruby
       return [:atom, a] if (a.is_a?(Symbol))
```
下一步，作为重构的一部分，`:call`几乎是由它自己构成：
```ruby
      def compile_call func, args
        stack_adjustment = PTR_SIZE + (((args.length+0.5)*PTR_SIZE/(4.0*PTR_SIZE)).round) * (4*PTR_SIZE)
    
        puts "\tsubl\t$#{stack_adjustment}, %esp"
        args.each_with_index do |a,i|
          param = compile_eval_arg(a)
          puts "\tmovl\t#{param},#{i>0 ? i*4 : ""}(%esp)"
        end
    
        res = compile_eval_arg(func)
        res = "*%eax" if res == "%eax" # Ugly. Would be nicer to retain some knowledge of what "res" contains                            
        puts "\tcall\t#{res}"
        puts "\taddl\t$#{stack_adjustment}, %esp"
        return [:subexpr]
      end
```
这段代码可能看起来很熟悉。这是因为它就是`#compile_exp`的内部代码，用`#compile_eval_arg`来代替一些更丑陋的代码。另外一个主要的修改是它同样调用compile_eval_arg来获取该函数，并且用”*“对`%eax`做了一些奇怪的事情。

你现在要么感觉很困惑，要么你可能开始看到一些可能性——做的很漂亮也会搬起石头砸自己的脚。上述代码等效于把任意表达式转换成一个指针，并不做任何校验就跳转到它。这使得它非常容易跳转到一个随机的地址从而引起段错误。顺便说一句，它还会使事情很容易向一个类系统的虚拟指针列表，和一些其他的事情。安全不得不晚点到来。而事实证明，”*“在间接的”call“的前面是必要的。

那么现在`#compile_exp`看起来像什么样子呢？很明显答案是”像地狱般的整洁“：
```ruby
     def compile_do(*exp)
        exp.each { |e| compile_exp(e) }
        return [:subexpr]
      end
    
      def compile_exp(exp)
        return if !exp || exp.size == 0
        return compile_do(*exp[1..-1]) if exp[0] == :do
        return compile_defun(*exp[1..-1]) if (exp[0] == :defun)
        return compile_ifelse(*exp[1..-1]) if (exp[0] == :if)
        return compile_lambda(*exp[1..-1]) if (exp[0] == :lambda)
        return compile_call(exp[1],exp[2]) if (exp[0] == :call)
        return compile_call(exp[0],exp[1..-1])
      end
```
漂亮，难道不是吗？`compile_call`，正如事实证明它几乎跟`compile_exp`的内部所做的完全相同，除了把一些代码外包给其他的函数之外。

那么，我们做一个小小的测试：
```ruby
    prog = [:do,
      [:call, [:lambda, [], [:puts, "Test"]], [] ]
    ]
```
（太棒了，不算太惊天动地）

编译并运行：
```sh
    $ ruby step6.rb >step6.s
    $ make step6
    cc    step6.s   -o step6
    $ ./step6
    Test
```
这一步的完全的代码在[这里](http://hokstad.com/static/compiler/step6.rb)

### 后面的部分

自从上次我把三个部分组合起来，这里有一个对剩余预写的"只是需要清理"的部分的更新列表——我想我最好抽出一些时间马上去写一些新的东西，因为我相当肯定，我可能会以把下面的几个部分组合起来结束，当我找到足够的时间去清理它们的时候。

* 第7步：重复循环访问匿名函数，访问函数的参数
* 第8步：添加赋值操作和基本的算法
* 第9步：一个更整洁的”while“循环
* 第10步：测试语言：编写一个简单的文本识别程序来使输入更整洁
* 第11步：重构代码生成器并开始抽象出目标结构
* 第12步：各种概念和未来发展方向的讨论
* 第13步：添加数组
* 第14步：本地变量和多个作用域
* 第15步：访问可变长参数
* 第16步：重访文本识别器——清理测试新功能，并运行它自己
* 第17步：确定编译器自举所需要的结构
* 第18步：开始一个完整的解析器