# 变长参数

### Supporting variable length arguments

The C way handles variable length arguments is to let the caller push as much as they'd like onto the stack. It's then up to the callee to make sure they don't access too much, and there's really no way for the C function to know how many arguments have been pushed other than through inspection of the arguments themselves. Not a very pleasant situation. It works because the arguments are pushed right to left, so that the leftmost argument is always at the top of the stack.

This makes variadic functions like printf() reasonably easy to write, but it is also a never-ending source of bugs, since the printf() format string can indicate there are more arguments on the stack than is really the case (most modern C compilers as a result warn about mismatches between printf() format strings and arguments, but this doesn't help you if you write your own. Not great.

Of course, so far we've not really care about safety much, but that doesn't mean I don't want to get there, and in this case it makes it easier to write code too. There are relatively few ways out of this that are not horribly inefficient: A marker that indicate you're at the end of the argument list; a pointer indicating the last position; a count.

None of them are particularly pleasing, but they do the work. Then there's the issue of how to access the extra arguments.

In Ruby it's done by adding a final argument prepended with "*". This works like the "splat" operator in expressions in that it turns anything from then on into an array. In some ways it simplifies matters in that the compiler or interpreter "only" need to know how to stuff the remaining arguments into an array and push a pointer to the array itself onto the stack instead.

We're not quite so lucky as to have a fully formed object system to help us yet. What are the alternatives?

We're going to go with a halfway solution that is pleasing because it allows us to keep interoperability with C:


* We push everything on the stack as always
* We modify :defun (and by extension :lambda) so that the parameter list may contain arrays. If it does, then the first element in any sub-array is the argument name, and the second is directives/modifiers that affect the treatment of that argument. For now we specify one directive: ":rest". :rest will act like the splat operator, and will mean that accessing this variable will give us the address of where this argument would be on the stack normally.
* We arbitrarily declare that %ebx will contain the address of the first byte past the last argument on the stack if :rest was used for this function. I chose %ebx because some Googling revealed that this is considered a "caller saved" register that the callee can (and will) trash at will - it doesn't have any specified content normally. This has a  benefit: We can still safely call C functions, including variadic C functions, and they'll just trash %ebx and ignore it.
* The :rest declared argument is just a normal argument variable for all intents and purposes.
* For now we just set up %ebx for all function calls, and don't do any checks for overflows in the functions that use the :rest argument - we can add that later.
* Implementing something like the Ruby behavior can be done later - for example by iterating over the :rest variable, or by pointing to it as the "backing store" for a higher level concept. Eventually we'll probably want to "hide" some of the current language structures behind safer, higher level constructs.

One issue: Calling IN to variadic functions written in this language from C can't be safely done unless the other function arguments contain enough information to deduce the number of arguments, or without using a wrapper function that sets up %ebx.

As you'll see, setting up %ebx is going to be pretty trivial. We also add a new function, :numargs, that simply returns the number of arguments. It can be calculated like this:
```asm
 movl %ebx, %eax
 subl %ebp, %eax
 shrl $2, %eax
```
Before I start going through the code: since I now have a repository at Github, you can see the commit for this code and download the the full code from here:

* [full repository](http://github.com/vidarh/writing-a-compiler-in-ruby/tree/master)
* [the commit of the main functionality for variadic functions](https://github.com/vidarh/writing-a-compiler-in-ruby/commit/17e8c48c9d93b04afd1076309ca8803c8319115a)
* [a fixed typo and missing function from the original commit](https://github.com/vidarh/writing-a-compiler-in-ruby/commit/be64ace0e467fd190b44831249fa82f5a7e82af7)
* [a cleanup for :numargs](https://github.com/vidarh/writing-a-compiler-in-ruby/commit/7cb52bf18e557d7b57ab753a9a2df9ad96debc0d)
* [a test program-- this is the last commit for this part; download/pull from this commit if you want the code as at the end of this article](https://github.com/vidarh/writing-a-compiler-in-ruby/commit/116dd7f3e4ae4a7fba3ce3543f071a2ccebb77e7)

You can also <"watch" the repository to keep track of whenever I update it.

The biggest change this time is getting the infrastructure in place to handle modifiers to the arguments. The "Arg" class will take care of the arguments from now on:
```ruby
    class Arg
     attr_reader :name,:rest
     def initialize name, *modifiers
     @name = name
     modifiers.each do |m|
     @rest = true if m == :rest
     end
     end
     
     def rest?; @rest; end 
     def type
     rest? ? :argaddr : :arg
     end
    end
```
The next change we have to make is updating the Function class to instantiate Arg's, and it's #get_arg method to handle :numargs so you can get access to the number of arguments:
```ruby
    class Function
     attr_reader :args,:body
     def initialize args,body
      @body = body
      @rest = false
      @args = args.collect do |a|
       arg = Arg.new(*[a].flatten)
       @rest = true if arg.rest?
       arg
      end
     end
     
     def rest?; @rest; end
     def get_arg(a)
      if a == :numargs
       return [:int,args.size] if !rest?
       return [:numargs]
      end
     
      args.each_with_index do |arg,i|
       return [arg.type,i] if arg.name == a
      end
     
      return nil
     end
    end
```
Note that we return a constant :int if :rest isn't used, as a tiny little optimization.

If you look through the rest of the commit on Github you'll see mostly changes to thread the rest? method and `get_arg` changes up into the scope handling. The next interesting bit is this change to LocalVarScope's get_arg:
```ruby
     def get_arg a
       a = a.to_sym
    -  return [:lvar,@locals[a]] if @locals.include?(a)
    +  return [:lvar,@locals[a] + (rest? ? 1 : 0)] if @locals.include?(a)
       return @next.get_arg(a) if @next
       return [:addr,a] # Shouldn't get here normally
      end
```
We add one to the offset because we'll later change the emitter to copy :numargs from %ebx and onto the stack when inside the function so that it's accessible as a local variable. We don't strictly speaking need to push it on the stack unless we need the register AND actually use :numargs, but we'll leave avoiding that as an optimization for later.

The next interesting bit is this cleaned up #compile_eval_arg (I really need to find more meaningful names for these methods... sigh):
```ruby
     def compile_eval_arg scope,arg
      atype, aparam = get_arg(scope,arg)
      return aparam if atype == :int
      return @e.addr_value(aparam) if atype == :strconst
      case atype
      when :numargs
       @e.movl("-4(%ebp)",@e.result_value)
      when :argaddr:
        @e.load_arg_address(aparam)
    ...
```
(I've omitted the end, as the rest were just cleanups - see the commit).

The new thing here is supporting accessing :numargs, and having a way to get the address of an argument. This is used to get the start of the array of the remaining arguments.

The changes in the emitter are minor:

with_stack will add a "movl(args,:ebx)" if numargs are required, and #func will add "pushl(:ebx)" to push the numargs value onto the stack if required, and then we have #load_arg_addres:
```ruby
def load_arg_address(aparam) 
  leal(local_arg(aparam),:eax) 
 end 
```
In a separate commit I decided to do this change to Function#get_arg:
```ruby
 return rest? ? [:lvar,-1],[:int,args.size]
```
.. combined with stripping the asm for :numargs out of #compile_eval_arg. This change was made for the purpose of removing the asm, but also to treat :numargs more like a variable than a language keyword, though that distinction isn't particularly strong.

### A test program

Here's a program that demonstrates the new functionality:
```ruby
require 'compiler'

prog = prog = [:do,
 [:defun, :f, [:test,[:arr, :rest]],[:let,[:i],
   [:assign, :i, 0],
   [:while, [:lt,:i,[:sub,:numargs,1]], [:do,
     [:printf, "test=%ld, i=%ld, numargs=%ld, arr[i]=%ld\n",:test,:i,:numargs,[:index, :arr, :i]],
     [:assign, :i, [:add, :i, 1]],
    ]
   ]
  ]
 ],
 [:defun,:g,[:i,:j],[:let,[:k],
   [:assign,:k,42],
   [:printf,"numargs=%ld, i=%ld,j=%ld,k=%ld\n",:numargs,:i,:j,:k]
  ]
 ],
 [:f,123, 42,43,45],
 [:g,23,67]
]

Compiler.new.compile(prog)
```
... and the expected output:
```sh
[vidarh@dev writing-a-compiler-in-ruby]$ make testargs
ruby testargs.rb >testargs.s
as  -o testargs.o testargs.s
testargs.s: Assembler messages:
testargs.s:156: Warning: unterminated string; newline inserted
cc  -c -o runtime.o runtime.c
cc  testargs.o runtime.o  -o testargs
[vidarh@dev writing-a-compiler-in-ruby]$ ./testargs 
test=123, i=0, numargs=4, arr[i]=42
test=123, i=1, numargs=4, arr[i]=43
test=123, i=2, numargs=4, arr[i]=45
numargs=2, i=23,j=67,k=42
[vidarh@dev writing-a-compiler-in-ruby]$ 
```
To reiterate, the full source is available in this Github repository. To get the source at the state as of the endo f this article, pull or download from this commit