#定义函数；添加运行时支持

由于上个部分的长时间延期万分抱歉，我只是太沉迷于其他事情了。正如上次所贴出的列表指示的，这部分将介绍定义函数和添加构建一个简单的“运行时库”。

###定义函数

一个没有任何函数或方法的程序语言算不上什么语言。并且恰巧的是，面向对象语言的所有部分都可以在过程式语言上被非常轻易的实现：方法不过就是比函数多了一个作为额外参数的对象而已。所以添加对函数的支持显然相当重要。

还是如此简单。让我们再次看看一些C语言和对应的汇编语言：
```cpp
    void foo()
    {
      puts("Hello world");
    }
```
.. 使用gcc输出的结果如下:
```cpp
    .globl foo
            .type   foo, @function
    foo:
            pushl   %ebp
            movl    %esp, %ebp
            subl    $8, %esp
            movl    $.LC0, (%esp)
            call    puts
            leave
            ret
            .size   foo, .-foo
```
函数调用是非常容易识别的，这留给我们一个小的对该函数的prolog和epilog：

prolog在堆栈上存储%ebp，然后将%esp复制到%ebp中。"leave"指令副作用，"ret"指令将返回地址弹出堆栈并返回到函数被调用的位置。为何它要存储%esp（堆栈指针）到%ebp中呢？嗯，一个明显的好处是你可以随意的弄乱堆栈，只要在最后将%ebp再复制回来就万事大吉。我们可以看到上述GCC利用的是，不释放它为函数调用申请的堆栈空间来再次存放，用"leave"指令来进行恢复堆栈指针——否则它就浪费了。

所以，如果那就是全部，这次的修改应该会相当简单。事实上，它们就是。

首先，我们修改Compiler#initiliaze来创建一个Hash将我们的函数保存进去：
```ruby
      def initialize
        @global_functions = {}
```
然后，我们添加一个方法来输出这个函数：
```ruby
      def output_functions
        @global_functions.each do |name,data|
          puts ".globl #{name}"
          puts ".type   #{name}, @function"
          puts "#{name}:"
          puts "\tpushl   %ebp"
          puts "\tmovl    %esp, %ebp"
          compile_exp(data[1])
          puts "\tleave"
          puts "\tret"
          puts "\t.size   #{name}, .-#{name}"
          puts
        end
      end
```
你也看到了我们包括".global"，".type"和".size"位吧。".global"表示这个函数在你生成的汇编代码的文件之外应该是可见的，这是非常重要的，如果你想要把多个目标文件链接到一起的话。".type"和".size"我相信主要用于调试，表明这个符号代表一个函数，和各自的大小。

除此之外，这个函数非常简单——它调用"compile_exp"来做实际的工作。

我们添加一个小的辅助方法来定义函数：
```ruby
      def defun name, args, body
        @global_functions[name] = [args,body]
      end
```
我们将这几行添加到#compile_exp中：
```ruby
        return if !exp || exp.size == 0
        return defun(*exp[1..-1]) if (exp[0] == :defun)
```
第一行放在这里一定程度上是为了代码的健壮性，一部分是为了让我们传递nil或空数组作为"没做任何事"，仅仅如你预期的可以编写一个空的函数声明。它在下一行的上下文中很有意义，这不检查它是否将要定义一个空函数。

你或许有也或许没有注意到，这件事让我们递归的定义函数。`[:defun,:foo,[:defun, :bar, []]]`是完全合法的。你可能还注意到这会导致定义两个函数都是全价可访问的。哦，好吧。它不会有任何妨碍，所以我们以后再处理它（要么阻止它，要么使内部函数只在外部函数内时可见——我还没决定我跟喜欢哪一个）。

所有还剩下的就是输出函数了，所以我把这段代码添加到#compile，正好放在output_constants前面：
```ruby
     output_functions
```
###添加运行时库支持

首先，我们把当前的#compile改成#compile_main。然后我们像下面这样重新定义#compile：
```ruby
      def compile(exp)
        compile_main([:do, DO_BEFORE, exp, DO_AFTER])
      end
```
然后，我们定义常量DO_BEFORE和DO_AFTER（如果你喜欢，可以把他们放到不同的文件中，现在我只是把它们放到文件顶部）：
```ruby
        DO_BEFORE= [:do,
          [:defun, :hello_world,[], [:puts, "Hello World"]]
        ]

        DO_AFTER= []
```
你就承认吧，你已经预期更多高级的东西。但这可能会偏离我们最初设定的目标。上面的代码是足够去顶一个完整的运行时库的。如果你想要的东西是需要用汇编或C语言来实现的，这个方法也是可行的：只连接到一个包含正在考虑的函数的目标文件，因为我们仍然尊重C语言的调用约定。

让我们来测试它。在`Compiler.new.compile(prog)`之前添加如下代码：
```ruby
    prog = [:hello_world]
```
编译并执行：
```sh
    $ ruby step4.rb >step4.s
    $ make step4
    cc    step4.s   -o step4
    $ ./step4
    Hello World
    $
```
你可以在[这里](http://www.hokstad.com/static/compiler/step4.rb)找到结果

###访问函数参数？
这里遗留了一个非常大的漏洞：访问函数参数。正好这需要一个巨大的改动。相信它不会被遗忘，这是第8部分的一个重要部分，我尽量在下面几个章节中不间隔那么长。