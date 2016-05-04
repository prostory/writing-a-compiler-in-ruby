# 使用lambda/call和函数参数

I've combined two of the planned parts this time, what was in the list from last time as parts 7 and 8.

### Making use of lambda / call

We can implement loops using recursion "manually", but adding lambda's now should make it possible to create a slightly cleaner version by actually defining a "while" function:
```lisp
       (defun while (cond body) 
          (if (call cond ()) (do (call body ()) (while cond body) ()))
        )
```
In a way, this isn't so far from Ruby blocks, except that we don't provide the syntactic sugar that allows the blocks to be defined without any extra vocabulary, so in use our "while" function would look like this:
```lisp
       (while (lambda () (cond)) (lambda () (body)))
```
Not terrible, and far better than earlier, but still not very clean. We'll do better shortly, but for now lets start with a pre-requisite that prevents the code above from actually working in any meaningful sense of the word:

We finally have to add support for using the arguments passed.

In our Ruby based syntax, the while function using function arguments looks like this:
```ruby
     [:defun, :while, [:cond, :body],
        [:if, [:apply, :cond, []], [:do,
            [:apply, :body, []],
            [:while, :cond, :body]
          ],
          []
        ]
      ]
```
As a sidebar, indirectly this also means we're providing a hackish way of providing local variables. "Proper" local variables is largely syntactic sugar again. Most stuff in programming is syntactic sugar:
```lisp
      (call (lambda (i) (code here)) (0))
```
The code above defines the local variable (i), visible to the code inside, and initializes it to 0. Not pretty, but as usual we defer pretty until later. As usual it's also highly inefficient, since it depends on creating a brand new function, but we'll deal with that later.

### Adding function arguments

But lets move on and actually figure out what changes to make to access the function arguments. These preparations also pave the way for proper local variables and more down the line.

First we'll add a "scope" object, and do some minor refactoring. The "scope" defines which sets of variables are actually visible to use at any time.
```ruby
    class Function
      attr_reader :args,:body
      def initialize args,body
        @args = args
        @body = body
      end
    end
    
    class Scope
      def initialize compiler,func
        @c = compiler
        @func = func
      end
    
      def get_arg a
        a = a.to_sym
        @func.args.each_with_index {|arg,i| return [:arg,i] if arg == a }
        return [:atom,a]
      end
    end
```
The reason we separate Function and Scope here is that I'll later introduce scopes that match other things than functions, such as classes etc.

Part of the refactoring mentioned is to thread the scope objects through the compiler, so that scope changes are transparent (by just passing the new scope down).

We hook Scope#get_arg in into Compiler#get_arg by changing:
```ruby
        return [:atom, a] if (a.is_a?(Symbol))
```
into this:
```ruby
        return scope.get_arg(a) if (a.is_a?(Symbol)) 
```
`#output_functions` changes into this, to start a new scope for each function:
```ruby
     def output_functions
        @global_functions.each do |name,func|
         puts ".globl #{name}"
         puts ".type   #{name}, @function"
         puts "#{name}:"
         puts "\tpushl   %ebp"
         puts "\tmovl    %esp, %ebp"
         compile_exp(Scope.new(self,func,func.body)
         puts "\tleave"
         puts "\tret"
         puts "\t.size   #{name}, .-#{name}"
         puts
        end
      end   
```
And #compile_defun changes to create a proper Function object instead of just an array of arguments and the body:
```ruby
      def compile_defun scope,name, args, body
        @global_functions[name] = Function.new(args,body)
        return [:subexpr]
      end 
```
Then we need to actually support accessing the arguments. Again we resort to "gcc -S" to find out how. This:
```cpp
    void bar(const char * str, unsigned long arg, unsigned long arg2)
    {
      printf(str,arg,arg2);
    }
```
turns into:
```cpp
    bar:
            pushl   %ebp
            movl    %esp, %ebp
            subl    $24, %esp
            movl    16(%ebp), %eax
            movl    %eax, 8(%esp)
            movl    12(%ebp), %eax
            movl    %eax, 4(%esp)
            movl    8(%ebp), %eax
            movl    %eax, (%esp)
            call    printf
            leave
            ret
```
As you can see, the arguments are accessed relative to %ebp, which has been loaded with a copy of %esp at the beginning of the function. Why the offset of 8 for the first argument? Well, the return address gets pushed onto the stack, creating an offset of 4, and the the old value of %ebp is pushed on, giving us an offset of 8 to access the arguments. Remember that %esp grows down in memory, which is why we're adding offsets to get past the last entries pushed onto the stack.

That leads to this addition to #compile_eval_arg as the last "if" check:
```ruby
       if atype == :arg
          puts "\tmovl\t#{PTR_SIZE*(aparam+2)}(%ebp),%eax"
        end
```
Last but not least we create a Function object for "main" by modifying the call to #compile_exp in #compile_main:
```ruby
       @main = Function.new([],[])
        compile_exp(Scope.new(self,@main),exp)
```
Time to test how it works:
```ruby
    prog = [:do,
      [:defun,:myputs,[:foo],[:puts,:foo]],
      [:myputs,"Demonstrating argument passing"],
    ]
```
Then:
```sh
    $ ruby step7.rb >step7.s
    $ make step7
    cc    step7.s   -o step7
    $ ./step7
    Demonstrating argument passing
```
The latest version is here