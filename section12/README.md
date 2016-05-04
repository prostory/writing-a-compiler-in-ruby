# 添加数组的支持

At this point it's worth looking briefly at what is required to implement something more "serious" without a lot of excessive pain. It's really quite easy:
We really want variable length arguments
We want primitives that allow us to create and access arrays. Lets call them "array" and "index", and we want to make sure "index" works with "assign" or provide an alternative to get the address of the slot in the array.
We'd like local variables.
Why did I make these specific choices? Variable length argument lists makes it possible to implement a lot of functions in a more user friendly manner. Most importantly, implementing things like a dynamic object system with a minimal number of primitives is far easier if you have a way of turning an array into a list of arguments and the other way around - it allows the implementation of methods like a Ruby like #send Arrays should be pretty obvious. More flexible would be a Ruby like #pack or #unpack or ways of accessing raw memory. However arrays are sufficient to get most of the advantages, and simple enough to implement very easily without adding more typing etc. We do eventually need to get to typing, but Local variables should also be pretty obvious. We can emulate them now, by wrapping functions in another function, and have the "real" function have extra arguments, like below, but it's hardly satisfying:
```lisp
(defun real_func (arg,localvar1,localvar2) ( ... ) )

(defun func (arg) (real_func arg 0 0))
```
The three points above will therefore be the focus of the next couple of parts.
### Adding arrays

Let's deal with the trivial bit first, finally adding something to the runtime library:
#### Implementing "array"

```ruby
DO_BEFORE= [:do,
  [:defun, :array, [:size],[:malloc,[:mul,:size,4]]
] 
```
We're just calling the C-library's "malloc" and multiplying the input by 4 (yes, I'm assuming 32 bit pointers again here, so sue me - certainly this needs to be abstracted out at some point, but by then this code will be unrecognizable). Note that depending on platform and typical array size this can be horribly inefficient for small allocations, but that's for later. So we can create arrays, but not access them. We could easily enough do this in C, BUT there's one constraint: We want to be able to assign to the array with the "assign" primitive instead of calling a special function to access arrays.
#### Quick interlude for object purists

Yes, this is not object oriented. Yes, we could hide it, and bootstrap classes and a #send method or similar, and implement everything in terms of that. But why? What we are working on now is for all intents of purposes the "syntax tree" of the language - we'll soon add a parser, and it's easy enough to hide non-object oriented constructs like this then, or make it a special "privileged" set of instructions to facilitate implementation of the runtime "kernel" of the language. You'll see the benefits clearer once we get around to implementing an object system, even though it can be limited to being used in a handful of places easily enough.
#### Implementing "index"

The syntax will be [:index, array, index value]. Here's the first stab to add to the Compiler class:
```ruby
  def compile_index scope,arr,index
    source = compile_eval_arg(scope, arr)
    @e.movl(source,:edx)
    source = compile_eval_arg(scope, index)
    @e.save_result(source)
    @e.sall(2,:eax)
    @e.addl(:eax,:edx)
    return [:indirect,:edx]
  end
```
Notice that this gets quite a bit of x86 specific stuff back in compiler.rb, but we'll worry about that later. The logic is fairly straightforward:

* Evaluate the first argument to get at the address of the array
* Move that address into a temporary register - I picked %edx. The reason we do this is that the compiler doesn't have any register allocation at all yet, and %eax is used for return values regardless.
* Evaluate the second argument.
* Multiply the second argument by four to get the number of bytes for the offset from the start of the array (note that we use "sall" which is a left arithmetic shift - shifts are cheaper than multiplications on most architectures; in the tradition of this series I haven't verified if that makes a difference on x86)
* Add the start address to the offset.
* The result is [:indirect,:edx] - we'll need to add support for the new use of indirect, indicating that %edx does not hold the actual value, but a pointer to it.

We hook this into #compile_exp like this:
```ruby
   return compile_index(scope,*exp[1..-1]) if (exp[0] == :index)
```
So, lets deal with :indirect. We need to change two places. First and foremost, we need to add the following line to #compile_eval_arg:
```ruby
  @e.emit(:movl,"(%#{aparam.to_s})",@e.result_value) if atype == :indirect
```
This takes care of the case when and :index occurs anywhere but as the target of an :assign. It reads the value pointed to. Then we need to change #compile_assign to store the result in a different way when :indirect is used. Here's a diff of #compile_assign
```ruby
   def compile_assign scope, left, right
     source = compile_eval_arg(scope, right)
     atype, aparam = get_arg(scope,left)
-    raise "Expected an argument on left hand side of assignment" if atype != :arg
-    @e.save_to_arg(source,aparam)
+    if atype == :indirect
+      @e.emit(:movl,source,"(%#{aparam})")
+    elsif atype == :arg
+      @e.save_to_arg(source,aparam)
+    else
+      raise "Expected an argument on left hand side of assignment"
+    end
     return [:subexpr]
   end
```
An example:
```ruby
require 'compiler'

prog = [:do,
  [[:lambda, [:i], [:do,
        [:assign, :i, [:array, 2]],
        [:printf, "i=%p\n",:i],
        [:assign, [:index, :i, 0], 2],
        [:assign, [:index, :i, 1], 42],
        [:printf, "i[0]=%ld, i[1]=%ld\n",[:index,:i,0],[:index,:i,1]]
      ]
    ],10]
]

Compiler.new.compile(prog)
```
Here's the [:index, :i, 0] expression, with some comments:
```asm
    movl    8(%ebp), %eax     ; Move "i" to %eax
    movl    %eax, %edx        ; Save it to %edx
    movl    $0, %eax          ; The 0 from [:index, :i, 0]
    sall    $2, %eax          ; Multiply by 4
    addl    %eax, %edx        ; Add the offset to the address saved in %edx
    movl    $2, (%edx)        ; Copy the value 2 into the address of the beginning of "i"
```
It's a pretty good example of complete lack of optimization, and both some trivial optimization of the original program as well as peephole optimization of the assembly could reduce it to next to nothing, but it works... So lets run it:
```sh
$ make testarray
ruby testarray.rb >testarray.s
as   -o testarray.o testarray.s
testarray.s: Assembler messages:
testarray.s:97: Warning: unterminated string; newline inserted
cc    -c -o runtime.o runtime.c
gcc -o testarray testarray.o runtime.o
$ ./testarray 
i=0x889a008
i[0]=2, i[1]=42
```
Get the source

You can download the source here.