#定义函数；添加运行时支持

Sorry for the long delay since the last part, I've simply been too caught up in other stuff. As the list from last time indicated, this part will cover defining functions and adding support for building a simple "runtime library".

###Defining functions

A programming language without either functions or methods isn't much of a language. And as it happens, all parts of an object oriented language can be implemented easily enough in terms of a procedural language: A method isn't much more than a function that takes an object as an extra argument. So adding support for functions is obviously rather critical.

It is also very simple. Lets take a look at some C and the resulting assembly again:
```ruby
    void foo()
    {
      puts("Hello world");
    }
```
.. results in this, with gcc:
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
The function call is easy enough to recognize, and that leaves us with a small prolog and epilog to the function:

The prolog stores %ebp onto the stack, and then copies %esp into %ebp. "leave" undoes the damage, and "ret" pops the return address off the stack and returns to where the function was called from. Why does it store %esp (the stack pointer) in %ebp? Well, one obvious advantage is that you can mess up the stack as much as you like, and just copy %ebp back again at the end and all is well. We can see above that GCC takes advantage of that by not freeing the stack space it allocated for the call to puts again, and leaving that to the "leave" instruction - it'd just be wasteful otherwise.

So if that's all, the changes should be fairly simple. And they are.

First we modify Compiler#initialize to create a Hash to store our functions in:
```ruby
      def initialize
        @global_functions = {}
```
Then we add a method to output the functions:
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
```language
```
You see we include the ".globl" and ".type" and ".size" bits too. ".globl" indicates that the function should be visible outside of the specific file you're assembling, which is important if you want to link multiple object files together. ".type" and ".size" I believe are there primarily for debugging, indicating the the symbol represents a function, and the size respectively.

Apart from that, this function is pretty trivial - it calls "compile_exp" to do the actual work.

We add a little helper to define the functions:
```ruby
      def defun name, args, body
        @global_functions[name] = [args,body]
      end
```
We add a couple of lines to #compile_exp:
```ruby
        return if !exp || exp.size == 0
        return defun(*exp[1..-1]) if (exp[0] == :defun)
```
The first line is there partly for robustness, partly to let us pass nil or empty arrays as "do nothing", just as you'd expect to be able to write an empty function declaration. It makes sense in the context of the next line, that doesn't check if it's about to define an empty function.

You may or may not have noticed that this thing lets us define functions recursively. [:defun,:foo,[:defun, :bar, []]] is completely legal. You may also have noticed that this currently leads to defining two functions that are both globally accessible. Oh well. It doesn't do any harm, so we'll deal with that later ("that" being either preventing it, or making the inner function only visible inside the outer function - I haven't decided yet which I prefer).

All that remains is to output the functions, so we add this to #compile, right before output_constants:
```ruby
     output_functions
```
###Adding support for a runtime library

First we change the name of the current #compile to #compile_main. Then we redefine #compile as follows:
```ruby
      def compile(exp)
        compile_main([:do, DO_BEFORE, exp, DO_AFTER])
      end
```
Then we define the constants DO_BEFORE and DO_AFTER (put them in a separate file if you prefer, for now I've just put them at the top):
```ruby
        DO_BEFORE= [:do,
          [:defun, :hello_world,[], [:puts, "Hello World"]]
        ]

        DO_AFTER= []
```
Just admit it, you'd expected something more advanced. But that would defeat the goals set out at the start. The above is sufficient to define a proper runtime library. If you want something that'd need to be implemented in assembler or C, that works too: Just link to an object file containing the functions in question, since we still respect C calling conventions.

Lets test it. Add this before Compiler.new.compile(prog):
```ruby
    prog = [:hello_world]
```
And compile and run:
```shell
    $ ruby step4.rb >step4.s
    $ make step4
    cc    step4.s   -o step4
    $ ./step4
    Hello World
    $
```
You can find the result here

###Accessing function arguments?

This leaves one big gaping omission: Accessing function arguments. As it happens that is a fairly large change. Trust that it won't be forgotten, it's a major part of part 8, and I'll try not leaving so long between the next few parts.