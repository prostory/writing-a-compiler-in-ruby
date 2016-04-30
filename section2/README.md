# 函数调用 / Hello World

我选中Ruby作为我的实现语言时相当武断。在这个阶段，这个选择还不是那么重要，但是我真的喜欢Ruby语言。

然而，有些时候，有一系列步骤会让语言和编译器连接的更紧密。我的意思是我想要编译器可以自举：它应该有能力编译它自己。这意味着编译器要么不得不至少支持Ruby语言的一个子集，要么有翻译步骤让编译器能够理解以该语言重写编译器的部分。尽管这没有规定实现的语言，这意味着确保你所实现的语言跟你想要达到的目标是颇为接近的至少总是值得的，除非你对于使编译器自举毫无兴趣。

如果你想要你的编译器可以自举，在你的实现种严格地避免复杂的语言结构也是很重要的。请记住，你需要支持每种编译器需要的语言结构，如果你使用太多的花哨的功能的话，要么你不得不在编译器种支持同样的功能，要么不得不冒险去做巨大的改变当你准备跳转的时候。这可一点都不好玩。

把Ruby作为实现语言的有一个明显的好处，然而（并且它与其他确定语言如Lisp都有这个好处），就是你可以非常容易的构建字面量树——在Ruby种使用Array或Hash和在Lisp中使用列表。

这意味着我们可以以“冒充”抽象语法树作为数组开始，可以轻便的跳过编写解析器的步骤。好耶！代价就是丑陋的语法，但暂时还可以忍受。

### Hello World

下面的代码展示的是Hello World:

```ruby
	 [:puts,"Hello World"]
```

在这里我们需要处理的非常简单：我们需要可以把一个参数推送到堆栈上，并且我们需要可以调用一个函数。

所以，让我们来看看怎样用x86汇编语言来实现它。我使用"gcc -S"来编译如下C程序：
```cpp
	int main()
	{
	   puts("Hello World");
	}
```
我们来看下输出。下面我已经复制了其中有意义的代码，我在这里通过查看上次的输出来发现发生的改变：
```cpp
            .section        .rodata
    .LC0:
            .string "Hello World"
            .text
    [...]
            subl    $4, %esp
            movl    $.LC0, (%esp)
            call    puts
            addl    $4, %esp
    [...]
```
如果你了解一些汇编语言，即使你从前没有用x86工作过，这个也是很容易抓住的：

它定义了一个字符串常量

在代码中它通过从堆栈指针减去4在堆栈上分配了4个字节。

然后，它把字符串的地址移动到堆栈上，到刚刚分配的4个字节中。

我们调用"puts"，通常由glibc提供（所有的这些系列都假定在Linux odds上你已经安装了gcc/gas + glibc）

之后，我们通过加回4个字节来清空堆栈。所以，我们该怎样在我们的编译器种使用它呢？首先，我们需要一种处理字符串常量的方式，所以我们把这个添加到上次的Compiler类种。我在这里提供了完全的[Ruby代码]，所以你不用去猜要复制或粘贴什么代码到那个位置。
```ruby
      def initialize
        @string_constants = {}
        @seq = 0
      end

      def get_arg(a)
        # For now we assume strings only
        seq = @string_constants[a]
        return seq if seq
        seq = @seq
        @seq += 1
        @string_constants[a] = seq
        return seq
      end
```
这段代码简单的把一个字符串常量映射到一个整型值，我们会使用这个整型值指向这个标签。你不能将多个字符串常量映射到同一个标签然后还能一次获取输出。按这种方法做并不是必须的，但是使用Hash来代替Array来确保独一性是一个微小的优化。
这里是我添加的用来输出字符串常量的函数：
```ruby
      def output_constants
        puts "\t.section\t.rodata"
        @string_constants.each do |c,seq|
          puts ".LC#{seq}:"
          puts "\t.string \"#{c}\""
        end
      end
```
那就只剩下编译调用本身了：
```ruby
      def compile_exp(exp)
        call = exp[0].to_s

        args = exp[1..-1].collect {|a| get_arg(a)}

        puts "\tsubl\t$4,%esp"

        args.each do |a|
          puts "\tmovl\t$.LC#{a},(%esp)"
        end

        puts "\tcall\t#{call}"
        puts "\taddl\t$4, %esp"
      end
```
你可能已经注意到这里有一个丑陋的不一致：上面这段代码假装可以处理多个参数，但是后面它只是从堆栈指针中减去了4而已，并且它无法对堆栈指针应用移位操作，所以多个参数只会覆盖彼此。

一会儿我们再修复它。我们的简单的一个参数的Hello World程序上面的代码就可以工作了。

关于这段代码还有一些其他需要注意的地方：
它没有检查这个函数是否存在——我们让gcc/gas来做这些工作，尽管这意味着无用的错误信息。

我们事实上可以调用任何我们可以链接到的函数，只要它使用的是一个字符串参数。
这里有大量的丑陋的代码真的应该被抽象出来。例如，我该怎样持有函数调用的地址，所有的那些内联的汇编代码都与编译器到i386平台紧密联结起来了，等等。
是时候尝试[运行编译器的第二步]了。它应该会生成如下代码：
```cpp
            .text
    .globl main
            .type   main, @function
    main:
            leal    4(%esp), %ecx
            andl    $-16, %esp
            pushl   -4(%ecx)
            pushl   %ebp
            movl    %esp, %ebp
            pushl   %ecx
            subl    $4,%esp
            movl    $.LC0,(%esp)
            call    puts
            addl    $4, %esp
            popl    %ecx
            popl    %ebp
            leal    -4(%ecx), %esp
            ret
            .size   main, .-main
            .section        .rodata
    .LC0:
            .string "Hello World"
```
这里是测试的方法：
```shell
    [vidarh@dev compiler]$ ruby step2.rb >hello.s
    [vidarh@dev compiler]$ gcc -o hello hello.s
    [vidarh@dev compiler]$ ./hello
    Hello World
    [vidarh@dev compiler]$
```
所以，我们要怎样处理多于一个的参数呢？

我不会再展示C语言代码和相应的汇编代码了——做一些不同数量的参数的函数调用并查看它们的结果是很容易的。相反，我会直接更改compile_exp(这里是完整[源码])。
```ruby
      PTR_SIZE=4

      def compile_exp(exp)
        call = exp[0].to_s

        args = exp[1..-1].collect {|a| get_arg(a)}

        # gcc on i386 does 4 bytes regardless of arguments, and then
        # jumps up 16 at a time. We will blindly do the same.
        stack_adjustment = PTR_SIZE + (((args.length+0.5)*PTR_SIZE/(4.0*PTR_SIZE)).round) * (4*PTR_SIZE)
        puts "\tsubl\t$#{stack_adjustment}, %esp"
        args.each_with_index do |a,i|
          puts "\tmovl\t$.LC#{a},#{i>0 ? i*PTR_SIZE : ""}(%esp)"
        end

        puts "\tcall\t#{call}"
        puts "\taddl\t$#{stack_adjustment}, %esp"
      end
```
所以，这是怎么回事呢？这里的改变并不大：

我们根据参数的个数来调整堆栈，而不是分配一个特定数量的堆栈空间（上个版本中是4个字节）。坦率地说，我承认我不知道为何gcc使用它所指定的调整——在这个阶段并不重要，尽管我猜测他可能是为了堆栈的对齐。

后面的优化/清空，如果你不确定它们为何那样做，最好不要做任何修改。
正如你所看到的，参数被一个接一个移动到堆栈上。我们仍然假定所有的指针都是同样的大小（i386平台上是4个字节）。

你还可以看到，函数的参数是被转移到堆栈上的。所以，最左边的参数是在最低的地址上分配的。如果你没有使用过任何汇编语言编程，如果你无法再你的头脑中构思它，画框来构思它。请记住，在这里堆栈在内存中通常是向下增长的。当分配空间时，我们在内存中向下移动堆栈指针，然后我们在内存中将参数向上复制到堆栈上（使用%esp的索引越来越高，就好像你正在访问数组一样）。

这里已经足够编译像下面的代码：
```ruby
    [:printf,"Hello %s\\n","World"]
```
下一步要做什么呢？

这是向前的一小步，我保证接下来的步骤将会越来越有用，作为编译可用的程序所需要构建的数量事实上是很小的。我也会是接下来的步骤更简洁并且处理“更丰富的”附加信息。正如我已经解释过很多次的动机，当我们继续的时候，我会尽可能为那些更简单的功能代码尽可能更少的添加解释。

接下来，我将扩展编译器来处理多个参数，其次是链接语句，子表达式，从子表达式处理返回值等等。

估计还有一打左右的类似的复杂的步骤，我们将有函数定义，参数传递，条件语句，一个运行时库，甚至一个功能受限的"lambda"操作给我们带来匿名函数（完全的闭包后续会到来）。

再有几步，我们将足够来编译一个简单的文本识别的程序来给定比Ruby的Array和符号稍微更漂亮的语法输入（仅仅只是稍微，一个完全的解析器不会很快上桌）。

[Ruby代码]:http://hokstad.com/static/compiler/step2.rb
[运行编译器的第二步]:http://hokstad.com/static/compiler/step2.rb
[源码]:http://hokstad.com/static/compiler/step2b.rb
