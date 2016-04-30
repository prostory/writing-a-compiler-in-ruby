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
Recursion will play a great role here - you're descending a tree structure after all, so the core method to compile an expression will be called on deeper and deeper nodes as well, especially to handle proper sub-expressions, which is our next goal.

Adding sub-expressions, take 1

Here's the test we'd like to make work:
```ruby
    prog = [:printf,"'hello world' takes %ld bytes\\n",[:strlen, "hello world"]],
```
Here's the first change. In #get_arg, we add this at the start:
```ruby
        # Handle strings or subexpressions
        if a.is_a?(Array)
          compile_exp(a)
          return nil # What should we return?
        end
```
If you try to make the code above compile the test, you'll get an error from gcc, because we expect get_arg to return a sequence number for the string constants and nothing else, but that clearly doesn't work for sub expressions.

Adding sub-expressions, take 2: Return values

So how does GCC handle this. Getting the assembly for this:

int main()
{
  printf("'Hello world' takes %ld bytes\n",foo("Hello world"));
}
... results in this (just the relevant part of main):
```cpp
        subl    $20, %esp
        movl    $.LC0, (%esp)
        call    foo
        movl    %eax, 4(%esp)
        movl    $.LC1, (%esp)
        call    printf
        addl    $20, %esp
```
... which shows that it's pretty straightforward. Gcc first calls the sub expression (foo), and then expects the return value of a function to be in the register "%eax", and so that needs to be copied onto the stack instead of the address of a string constant.

First up let's fix #get_arg:
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
The only changes are the "return" expressions, where we indicate what is returned - this will expand considerably later.

The remaining change is pretty much a rewrite of the rest of compile_exp. Instead of just collecting the results of get_arg, we iterate over it and output directly (which is why the stack adjustment's also change, since the "args" array has gone away):
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
As you can see the change isn't that complex. We just check the return value from #get_arg and pick a string constant or %eax depending. This part will expand considerably as we add more different types of things that can be returned.

You can find the latest version here

###Upcoming parts

These are the almost ready pre-written parts only. I expect once I need to start catching up on new parts the focus will be on a simple parser with the goal of making the compiler self-hosted (i.e. able to compile itself) as quickly as possible.

>Step 4: Introducing a runtime, and defining functions
>Step 5: Handle literals other than strings
>Step 6: If ... then ... else
>Step 7: Looping constructs
>Step 8: Anonymous functions (lambda)
>Step 9: Revisiting loops with anonymous functions, accessing >function arguments
>Step 10: Adding assignment and basic arithmetic
>Step 11: A cleaner "while" loop
>Step 12: Testing the language: Writing a simple text mangling >program to make input cleaner
>Step 13: Refactoring the code generation and starting to >abstract out the target architecture
>Step 14: Discussion of various concepts and future direction
>Step 15: Adding arrays
>Step 16: Local variables and multiple scopes
>Step 17: Accessing variable length arguments
>Step 18: Revisiting the text mangler - cleanups to test new >functionality, and run it on itself
>Step 19: Identifying the constructs needed to self host the compiler
>Step 20: A start on a proper parser