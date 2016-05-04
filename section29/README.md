# 在我们新的Fixnum上操作

Where we left off last time, we transform numbers into Fixnum objects, and turn + and other operators into method calls. In theory at least - we haven't actually tested that functionality properly in practice.

#### Putting them to work

So lets start with the simplest possible test: Can we call Fixnum#+ at all?

Let's compile 5 + 3. Nope. Doesn't work. For starters, we need to add require statements for Numeric and Integer (in 1c51f99)

But that's not enough. So lets try dipping down to our low level syntax, and compile this decidedly non-Ruby test:
```ruby
        %s(printf "test: %d\n" (callm (call __get_fixnum 5) + ((call __get_fixnum 3))))
```
The (callm ...) is explicitly meant to do roughly what we should expect the 5 + 3 to get turned into.

(You can compile this with ./compile [filename] in the working directory, and get /tmp/[filename] out if everything worked)

To get it to return something other than junk, let's actually implement Fixnum#+ still using runtime.c to provide add() (in 326ae54):
```ruby
      def + other
        %s(call add (@value (callm other __get_raw)))
      end
```
With that in place there's just one little wrinkle to get the example above to compile: Our %s() syntax can't handle operators. So for the sake of convenience, let us fix that by allowing any valid method name (in a7d4c31) by modifying sexp.rb:
```ruby
       def parse_exp
         ws
    -    ret = expect(Atom, Int, Quoted) || parse_sexp
    +    ret = expect(Atom, Int, Quoted, Methodname) || parse_sexp
         ws
         return ret
       end
```
At this point you should be able to compile and run the test above, and get the expected test: 8 output. So why is 5 + 3 on its own segmentation faulting?

Luckily we have a debugging tool in the --norequire --parsetree options:
```ruby
    %s(do
      (callm (sexp (call __get_fixnum 5)) + (sexp (call __get_fixnum 3)))
    )
```
And there we see an ugly pitfall of this syntax. Spotted it? The second argument to callm is a parameter list enclosed in parantheses, but here we pass it another list/array that happens to be an expression. It does reveal rather weak error handling and consistency checking for this syntax that we probably should do something about, but for now let us focus on the operators.

With a one line fix we get the output we need (in 1d9df53:):
```ruby
    --- a/transform.rb
    +++ b/transform.rb
    @@ -86,7 +86,7 @@ class Compiler
           next :skip if e[0] == :sexp
     
           if OPER_METHOD.member?(e[0].to_s)
    -        e[3] = e[2]
    +        e[3] = E[e[2]]
             e[2] = e[0]
             e[0] = :callm
           end
```
And now the parse output looks like this:
```ruby
    %s(do
      (callm (sexp (call __get_fixnum 5)) + (
          (sexp (call __get_fixnum 3))))
    )
    }}}                                     
```
    Which is far more reasonable, and actually doesn't segfault. So let us test
    that it returns the right value too:
```ruby
    a = 5 + 3
    
    %s(printf "%d\n" a)
```
... and that works. And that is wrong. Sigh. What we forgot above is that this whole mess means we have to re-wrap integers into an object every time we get one back. The example above should have been:
```ruby
    a = 5 + 3
    
    %s(printf "%d\n" (callm a __get_raw))
```
And that still fails. Until we do this to Fixnum#+ (in 50fcf26)
```ruby
       def + other
       -    %s(call add (@value (callm other __get_raw))) 
       +    %s(call __get_fixnum ((call add (@value (callm other __get_raw)))))
       end
```
And now our latest test should work.

I made you look through this intensely frustrating sequence for a reason - it's easy to present the more polished description of this process, but it's important to keep an eye on just how many details are there to trip you up when generating code and trying to debug things based on effects on generated code through several layers like this. It is one of the hardest parts of writing a compiler.

You may have noticed that despite a growing number of unit-tests for the parser components, there's far fewer tests for the code generation, despite the fact that it is far trickier to track down bugs there. One of the upcoming parts will start addressing the difficulties of unit-testing the code generation.

In the meantime, it is now finally time to do what we came here for, and re-implement runtime.c as compiler intrinsics + Ruby.

#### What does add/sub/etc. do anyway?

Time to break out our old friend gcc, and look at some asm output. To start with, here's add:
```asm
    .globl add
        .type   add, @function
    add:
        pushl   %ebp
        movl    %esp, %ebp
        movl    12(%ebp), %edx
        movl    8(%ebp), %eax
        addl    %edx, %eax
        popl    %ebp
        ret
        .size   add, .-add
```
Oh. That was simple. It "just" sets up the stack frame, moves the arguments into %edx and %eax respectively, adds them, and frees the stack frame.

We've put up with runtime.c and do a function call on every add for that? Of course, when we start trying to implement the Ruby semantics, an extra function call doesn't look all that bad (we'll have a lot of fun trying to optimize that out at some point in the future...), since there'll be a full blown method call to begin with.

Not all of them are as simple (do gcc -o runtime.s -S runtime.c and take a look at runtime.s yourself), but none of the few runtime functions we have are all that much work. So let us get to it.

#### Adding 'add'

First we add a :add keyword in the Compiler class, and include a new file to hold our arithmetic keywords so we don't keep messing up compiler.rb (in dc2bbd8)
```ruby
    diff --git a/compiler.rb b/compiler.rb
    index 00c857f..059abf4 100644
    --- a/compiler.rb
    +++ b/compiler.rb
    @@ -10,6 +10,8 @@ require 'transform'
     require 'set'
     require 'print_sexp'
     
    +require 'compile_arithmetic'
    +
     class Compiler
       attr_reader :global_functions
       attr_accessor :trace
    @@ -21,7 +23,7 @@ class Compiler
                        :do, :class, :defun, :defm, :if, :lambda,
                        :assign, :while, :index, :let, :case, :ternif,
                        :hash, :return,:sexp, :module, :rescue, :incr, :block,
    -                   :required
    +                   :required, :add
                       ]
```
We then add a simple implementation in compile_arithmetic.rb:
```ruby
      def compile_add(scope, left, right)
        @e.with_register do |reg|
          src = compile_eval_arg(scope,left)
          @e.movl(src,reg)
          @e.save_result(compile_eval_arg(scope,right))
          @e.addl(reg, :eax)
        end
        [:subexpr]
      end
```
Note that this is flawed in many ways: We don't do any simplification even when both arguments are constants; we don't do anything to try to avoid using two registers (if one of the arguments is an integer, we can evaluate the other expression first, and just addl the integer to this result.

We also for good measure comment out add() from runtime.c

As usual, we're being lazy and the above works, as you can show by compiling this:
```ruby
    %s(printf "test: %d\n" (add 5 3))
```
Next up we need to make use of this in Fixnum... And we'll find it doesn't work. It turns out the above version is too naive - our with_register is a "fake" start to a register allocator - it only hands out %edx. That's fine for now. But what isn't fine is that it doesn't actually allocate it in any way. You'll find it gets clobbered elsewhere.

So we'll need to implement very basic register allocation. For now we'll just add %ecx to our set of registers to hand out, and actually keep track of them, and raise an error if we run out (which we will sooner or later - at which point we'll need to implement more sophisticated register allocation).

We modify emitter.rb like this (in d84e905):
```ruby
    @@ -83,6 +83,7 @@ class Emitter
         @out = out
         @basic_main = false
         @section = 0 # Are we in a stabs section?
    +    @free_registers = [:edx,:ecx]
       end
    
    
    @@ -333,9 +334,16 @@ class Emitter
       end
    
       def with_register
    -    # FIXME: This is a hack - for now we just hand out :edx,
    -    # we don't actually do any allocation
    -    yield(:edx)
    +    # FIXME: This is a hack - for now we just hand out :edx or :ecx
    +    # and we don't handle spills.
    +
    +    @allocated_registers ||= Set.new
    +    free = @free_registers.shift
    +    raise "Register allocation FAILED" if !free
    +    @allocated_registers << free
    +    yield(free)
    +    @allocated_registers.delete(free)
    +    @free_registers << free
       end
```
Basically we've changed from just blindly handing out %edx for code that needs a spare register, to handing out one of %edx or %ecx depending on which one is already in use. But if we try to allocate more than two, we're screwed. The reason I haven't added more registers is because I've not wanted to try to figure out to what extent they're "safe". E.g. we use %eax as our default scratch register all over the place.

The normal way of handle running out of register is "register spilling". That is, we need to "spill" the registers into variables (this is reasonable if we've used the registers to cache variables), or push them onto the stack. The latter adds complexities in that it means we need to ensure we keep track of it when handling offsets to the stack pointer.

For now this will do.

If you try to compile this, you should find it works:
```ruby
    a = 5 + 3
    
    %s(printf "%d\n" (callm a __get_raw))
```
(This of course reveals that we now really should have some IO functions that can handle other types - so that will be one of the things to handle in one of the upcoming parts).

Subtracting 'sub' from the runtime.

So lets do the same thing for sub. I won't go over every detail as it's pretty much identical to add. The meat is in 0c6de44. Let's test it.
```ruby
    a = 7 - 2
    
    %s(printf "%d\n" (callm a __get_raw))
```
Compiling this gives -5. What gives?

Well, the evaluation order monster reared its head. We didn't spot this for compile_add since addition is commutative. So lets look at compile_sub and figure out a fix we can hopefully also apply to compile_add (this might sound unnecessary, but consider that expected evaluation order matters a great deal in a language with side effects, and few languages have more potential side effects than one in which you can, if you please, redefine even basic arithmetic operators, or modify classes as they are being defined...)
```ruby
      def compile_sub(scope, left, right)
        @e.with_register do |reg|
          src = compile_eval_arg(scope,left)
          @e.movl(src,reg)
          @e.save_result(compile_eval_arg(scope,right))
          @e.subl(reg, :eax)
        end
        [:subexpr]
      end
```
We evaluate the left hand, then save it to a register, and we evaluate the right hand. Then we subtract the left hand from the right, and that's obviously wrong...

But if we switch the order, we leave the result in another register than our "result" register %eax, and end up having to move it. We can fix that by evaluating the right side before the left, but that's the wrong evaluation order. We can try to save the result from the left hand evaluation in %eax and save the result of the right hand in %edx, which would solve the problem if it wasn't for the fact that we likely will clobber %eax when evaluating the left hand side.

What we're seeing here is that choices made for simplicity early on are starting to really affect the code-generation. Here are two possible alternatives:

* Not treat %eax as a scratch register and allow allocating it. This is still problematic as it will get clobbered by a lot of calls we might want to make to external code. It may still be the best choice in some cases.
* Allow returning results in something other than %eax. This is a much more viable choice for many situations. If we can return reg from compile_sub, we push the problem up one level, and its quite possible that the next step up can make use of %edx directly. We may want to go further and not restrict it to variables, as that would allow us to get rid of many instances of that awful call *%eax thing that we currently do for calls.

It may be time to look into this in one of the forthcoming parts - as much as a focus of this series has been about being "lazy" and deferring changes like these, we're now at a point where not being able to do the above actually makes the code more convoluted.

For this part, lets just bite the bullet and fix the order and do a movl at the end for compile_sub (and this allows us to leave #compile_add untouched, since the evaluation order of the operands to the Ruby + operator are left intact):
```ruby
      def compile_sub(scope, left, right)
        @e.with_register do |reg|
          src = compile_eval_arg(scope,left)
          @e.movl(src,reg)
          @e.save_result(compile_eval_arg(scope,right))
          @e.subl(:eax,reg)
          @e.save_result(reg)
        end
        [:subexpr]
      end
```
This works, but is not very DRY, when compared with compile_add, so lets add a helper, since we'll keep relying on this same pattern for the remaining bits and pieces (in a23f0d0):
```ruby
     def compile_2(scope, left, right)
        @e.with_register do |reg|
          src = compile_eval_arg(scope,left)
          @e.movl(src,reg)
          @e.save_result(compile_eval_arg(scope,right))
          yield reg
        end
        [:subexpr]
      end
    
      def compile_add(scope, left, right)
        compile_2(scope,left,right) do |reg|
          @e.addl(reg, :eax)
        end
      end
    
      def compile_sub(scope, left, right)
        compile_2(scope,left,right) do |reg|
          @e.subl(:eax,reg)  
          @e.save_result(reg)
        end
      end
```
#### Div and mul

We'll mostly ignore mul, since it's pretty much the same as add except using the imull instruction. See 5036b94 for the details.

Div, however, looks curious, when we disassemble the code generated by gcc for runtime.c. With the normal prolog and epilog stripped out from the div function:
```asm
        movl    8(%ebp), %eax
        movl    %eax, -8(%ebp)
        movl    -8(%ebp), %edx
        movl    %edx, %eax
        sarl    $31, %edx
        idivl   12(%ebp)
        leave
        ret
```
What is going on here? For starters, it doesn't look very optimized. 8(%ebp) and 12(%ebp) are the operands passed in to the function. The moves back and worth between %eax, -8(%ebp) and %edx looks fairly wasteful.

As it turns out, idivl performs signed division of a double-word (64 bit on i386) combination of %edx and %eax registers by a third register or a memory location. The sarl instruction does an Arithmethic Shift Right on a Long.

For those who are still not up on assembler, consider an "arithmetic shift" effectively moving the bits of the binary representation of the value one step in the specified direction. The prefix "arithmetic" is usually used on assembler instructions to indicate that the shift copies in the furthermost bit in the "empty" slots. So if you shift one bit right with an arithmetic shift right, it is equivalent to dividing by two regardless whether or not the value is negative or positive, as the left-most (furthermost for a shift right) bit is the sign bit.

Here is a page that illustrates it with a little graphic

So what is happening here is that it is filling all of %edx by the sign bit from %eax in order to handle negative values correctly.

Lets take a stab at implementing that. I end up with this horrible monstrosity, which is another reason to take another look at the code generation shortly (in 95d642e)
```ruby
      def compile_div(scope, left, right)
        # FIXME: We really want to be able to request
        # %edx specifically here, as we need it for idivl.
        # Instead we work around that below if we for some
        # reason don't get %edx.
        @e.with_register do |reg|
          src = compile_eval_arg(scope,left)
          @e.movl(:eax,reg)
```
So far, so good. We've just evaluated the left hand, and moved the result out of the way. Then we evaluate the right hand.
```ruby
          @e.save_result(compile_eval_arg(scope,right))
```
And then the ugly stuff starts. Depending on what registers we got allocated, we spit out variout variations up to and including temporarily pushing %edx onto the stack, since we need to have %edx free. This is a mess. One way to make it simpler would be to be able to request two registers, of which one should be %edx and the other doesn't matter. Then we could be assured a cleaner way of setting this up. So another task for when we clean up the code generator shortly.
```ruby
          @e.with_register do |r2|
            if (reg == :edx)
              @e.movl(:eax,r2)
              @e.movl(:edx,:eax)
              divby = r2
            else
              divby = reg
              if (r2 != :edx)
                save = true
                @e.pushl(:edx)
              end
              @e.movl(reg, :edx)
              @e.movl(:eax,reg)
              @e.movl(:edx,:eax)
            end
            @e.sarl(31, :edx)
            @e.idivl(divby)
    
            # NOTE: This clobber the remainder made available by idivl,
            # which we'll likely want to be able to save in the future
            @e.popl(:edx) if save
          end
        end
        [:subexpr]
      end
```
### Next time

It is finally time for the rest of the operators (for now), and wiping out runtime.c!