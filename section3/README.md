#链接表达式；添加子表达式
我希望尽早发布这篇文章，但是我这周却忙的不可开交。尽管忙于清理我的旧的资料，到目前为止也才差不多每半小时贴一次。总之，这是第三部分，并且在最后我贴出来一张粗略的列表介绍接下来将要讲解的部分。我试着把几个发布的意图更长，更有意义的部分（这个部分是一个好的示例，因为它非常简短）分组到一起，所以我已经把原来的三十个左右压缩到了20个（虽然那仅仅是我已经写完了的，但并不意味着工作已经完成了）。

###带有"do"的链接表达式

目前为止，从上次的第二版可以执行一个单独的表达式，也就是那样了。不是很有用。所以我需要添加一种方法把表达式链接起来就像一个函数的函数体一样。根据目前情况来看，真的是很简单。我将添加一个关键字——"do"。它将会接受一个不做限制的（即，实际上，只受限制于内存）表达式列表并计算它们。它将像下面这样：
```ruby
    prog = [:do,
       [:printf,"Hello"],
       [:printf," "],
       [:printf,"World\\n"]
    ]
```
它真的是非常地简单。在#compile_exp的开始部分，我们添加如下代码：
```ruby
        if exp[0] == :do
          exp[1..-1].each { |e| compile_exp(e) }
          return
        end
```
在这里递归将发生巨大的作用——最后为一个下降的树结构，所以编译表达式的核心方法也会被越来越深得节点调用，尤其是处理完全的子表达式，这是我们的下一个目标。

这里的测试我们想要使它工作：
```ruby
    prog = [:printf,"'hello world' takes %ld bytes\\n",[:strlen, "hello world"]],
```
######添加自表达式，第一版

这里是第一个变更。在#get_arg中，我们在函数的开头处添加：
```ruby
        # Handle strings or subexpressions
        if a.is_a?(Array)
          compile_exp(a)
          return nil # What should we return?
        end
```
如果你尝试去用上面的代码来编译那个测试，你将会从gcc中得到错误提示，因为我们期望get_arg返回为字符串常量返回一个数字序列而不是空，但这样的子表达式显然无法工作。

######添加自表达式，第二版：返回值
GCC是如何处理的，我们获取下面代码汇编代码：
```cpp
int main()
{
  printf("'Hello world' takes %ld bytes\n",foo("Hello world"));
}
```
... 这段代码编译的结果（仅仅贴出main相关的部分）：
```cpp
        subl    $20, %esp
        movl    $.LC0, (%esp)
        call    foo
        movl    %eax, 4(%esp)
        movl    $.LC1, (%esp)
        call    printf
        addl    $20, %esp
```
...这表明它很简单。gcc首先调用子表达式（foo），然后期望函数的返回值在寄存器"%eax"中。因此，需要被拷贝到堆栈上而不是字符串常量的地址。

首先，让我们修复#get_arg:
```ruby
      def get_arg(a)
        # Handle strings or subexpressions
        if a.is_a?(Array)
          compile_exp(a)
          return [:subexpr]
         end
        seq = @string_constants[a]
        return seq if seq
        seq = @seq
        @seq += 1
        @string_constants[a] = seq
        return [:strconst,seq]
      end
```
唯一的变更就是"return"表达式，用来指示我们返回什么——这个将在以后进行扩展。

其余的变更是几乎重写了compile_exp的剩余部分。我们遍历get_arg的结果并直接输出，而不是仅仅收集它。（这就是为什么堆栈校正也要修改，因为"arg"数组已经去除了）：
```ruby
        stack_adjustment = PTR_SIZE + (((exp.length-1+0.5)*PTR_SIZE/(4.0*PTR_SIZE)).round) * (4*PTR_SIZE)

        puts "\tsubl\t$#{stack_adjustment}, %esp" if exp[0] != :do
        exp[1..-1].each_with_index do |a,i|
          atype, aparam = get_arg(a)
          if exp[0] != :do
            if atype == :strconst
              param = "$.LC#{aparam}"
            else
              param = "%eax"
            end
            puts "\tmovl\t#{param},#{i>0 ? i*4 : ""}(%esp)"
          end
        end
```
正如你所看到的更改并不是那么复杂。我们仅仅只是检查从#get_arg返回的值并取出一个字符串常量或根据%eax而定。这部分将会大幅度的扩展随着我们添加更多我们可以返回的不同的类型的值。

你可以在[这里]找到最新的代码。

###即将到来的部分
这些仅仅是几乎已经预写的部分。我预期一旦我需要去开始赶写新的部分时，重点将放在一个简单的解析器，来使编译器可以尽快实现自举（即可以编译它自己）。

* 第4步：介绍运行时，定义函数
* 第5步：处理字符串以外的字面量
* 第6步：If ... then ... else
* 第7步：循环结构
* 第8步：匿名函数（lambda）
* 第9步：用匿名函数重访循环，访问函数参数
* 第10步：添加赋值和简单的算术操作
* 第11步：一个干净的"while"循环
* 第12步：测试语言：编写一个简单的文本识别程序来使输入更整洁
* 第13步：重构代码生成器并开始抽象出目标结构
* 第14步：各种概念和未来发展方向的讨论
* 第15步：添加数组
* 第16步：本地变量和多个作用域
* 第17步：访问可变长参数
* 第18步：重访文本识别器——清理测试新功能，并运行它自己
* 第19步：确定编译器自举所需要的结构
* 第20步：开始一个完整的解析器

[这里]:http://hokstad.com/static/compiler/step3.rb