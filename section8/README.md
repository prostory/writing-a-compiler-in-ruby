# 赋值，算法和比较

Last time we looked at an improved way of handling loops etc. using anonymous functions. But most of that is relatively limited if there's no way of modifying variables. Sure, you can get away with recursion and not allow mutation or even other side effects at all. It might satisfy some functional programming purists, but it would not satisfy me and it would also require some fairly sophisticated optimizations to get decent performance.

I like the principle of reducing side effects as much as possible, and pushing side effects up, so much so that it's one of the factors I pointed out in my post about reducing coupling through unit tests. But I don't like sacrificing simplicity and/or performance for the sake of a "purer" approach.

So lets introduce assignment, and at the same time add in basic arithmetic and comparisons.

### Assignment

We'll cheat and keep what we introduce into the actual compiler as minimal as possible. That means assignments go into the compiler, and arithmetic and comparisons are confined to a new C runtime for simplicity for now. It is temporary, I promise. Just another shortcut.

Why don't we give assignment the same treatment? Well, we could, if we introduced a transparent way of passing the address of a variable as an argument to a function, but it would still mean compiler changes, would be slower and it seems it wouldn't actually bring any benefits.

Another reason for not taking that route is that it opens up a can of worns: If we allow taking the address of stack variables, it gets easy to shoot yourself in the foot by accessing it after the stack frame is dead. We could accept that risk (it's not like this compiler is a bastion of safety at the moment, with mostly no typing and no error checking to speak of), or we can introduce "environments" and guarantee that any variable you take the address of will live on the heap and stay alive until you lose the reference... But that's more complicated than I'd like right now. Environments is one of the approaches to keeping local variables "alive" in the heap that would let us do proper closures instead of mere lowly anonymous functions, though, so we might eventually bite the bullet.

Adding assignment to the compiler is trivial anyway.

Here's the first try:
```ruby
      def compile_assign scope, left, right 
        source = compile_eval_arg(scope, right) 
        atype, aparam = get_arg(scope,left) 
        raise "Expected a variable on left hand side of assignment" if atype != :arg 
        puts "\tmovl\t#{source},#{PTR_SIZE*(aparam+2)}(%ebp)" 
        return [:subexpr] 
      end 
```
Notice our first error message... We'll eventually need to bite the bullet and add lots more of those, as well as handling them a bit better than just letting the compiler fail with an exception message and a stack trace, but it's a start.

What this function does is quite straightforward: It compiles the right hand expression, then copies the result into the variable.

Can you spot the problem?

It's not a bug, it's a feature! Right... It only allows assignment to a local variable. Whether that is indeed a bug or a feature is open for discussion - many languages does allow for assignment to function/method arguments as well as local (and sometimes global) arguments. I promise we'll look at this closer later, but for now it's simply not important.

We then update #compile_exp by adding this:
```ruby
       return compile_assign(scope,*exp[1..-1]) if (exp[0] == :assign) 
```
And test it with this (the output needs to be linked with the arithmetic functions in the next step, or rather it at least needs :sub):
```ruby
    prog = [:call, [:lambda, [:i],
        [:do,
          [:printf, "Testing sub and assign (before): %ld\\n", :i],
          [:assign, :i, [:sub, :i, 1]],
          [:printf, "Testing sub and assign (after): %ld\\n", :i],
        ]
      ], [10] ]
```
### Arithmetic and comparisons

Here's our new C runtime:
```cpp
    signed int add(signed int a, signed int b)
    {
      return a + b;
    }
    
    signed int sub(signed int a, signed int b)
    {
      return a - b;
    }
    
    signed int div(signed int a, signed int b)
    {
      return a / b;
    }
    
    signed int mul(signed int a, signed int b)
    {
      return a * b;
    }
    
    signed int ne(signed int a, signed int b)
    {
      return a != b;
    }
    
    signed int eq(signed int a, signed int b)
    {
      return a == b;
    }
    
    signed int not(signed int a)
    {
      return !a;
    }
    
    // Note that our "and" won't shortcircuit, as the
    // evaluation happens before this function is called.
    signed int and(signed int a, signed int b)
    {
      return a && b;
    }
    
    signed int gt(signed int a, signed int b)
    {
      return a > b;
    }
    
    signed int lt(signed int a, signed int b)
    {
      return a < b;
    }
```
Not very exciting is it?

I'm not going to try to make it very exciting, but it's worth considering what would be involved in adding compilation of it, though I won't complete that now. Lets look at a single example, "lt", and show the process, though you should be used to it by now. gcc -S on just the "lt" function gives:
```cpp
    lt:
        pushl   %ebp
        movl    %esp, %ebp
        movl    8(%ebp), %eax
        cmpl    12(%ebp), %eax
        setl    %al
        movzbl  %al, %eax
        popl    %ebp
        ret
```
(Uh-oh, what's that setl / movzbl stuff? This is an optimization - cmpl compares the two operands and set a number of condition codes. "set[something]" sets a value in a single byte register %al based on the condition codes - in this case "l" for less. "movzbl" then takes that byte, extends the value up to 32 bits by filling the remaining bits with 0 and puts it into %eax - don't worry about it)

To add this to the compiler we'd do compile_eval_arg on the left and right arguments. After doing it to the left, we'd need to store the result somewhere temporarily, in case we try to use %eax again. We'd then need to issue the cmpl/setl/movzbl instructions, and we'd be done.

The function could look something like this:
```ruby
    def compile_lt scope,left,right
        l = compile_eval_arg(scope, left)
        puts "\tmovl\t#{l},%ecx"
        r = compile_eval_arg(scope, right)
        puts "\tcmpl\t%ecx, %eax"
        puts "\tsetl\t%al"
        puts "\tmovzbl %al,%eax"
        return [:subexpr] 
    end
```
Note that this is completely untested - I haven't even checked that I got the operands to cmpl in the right order.

More importantly, this illustrates one reason for not jumping into this and spending time on doing the same thing for all of these functions right away: Notice that we needed a temporary register? Notice how that was wasteful, because we could have just put things into %ecx right away? Notice how this is a recipe for a lot of pain, because we might have even more complex situations where it is handy (though not strictly necessary) to have even more temporary registers?

Register allocation is a huge subject in itself. It boils down to figuring out the most efficient way of stuffing a large number of temporary and long lived variables into a very scarce (on the x86 in particular) resource, namely registers. You want to make the most of them, because access to registers is generally far faster than memory accesses, and because many architectures have instructions that requires the use of specific registers for specific purposes.

It gets complicated fast, because while using a register when you can is good, "spilling" - that is the process of storing a variable back to memory and loading another variable it it's place - when you have more variables in use than registers can get even more expensive than accessing things straight from memory in the first place. Figuring out what combination of variables is most efficient to keep in registers or move in and out of registers is something that can make grown men cry, and many simple compilers just ignore this problem and pick a very simple and inefficient approach.

At least reading the wikipedia article linked to above is worthwhile if you want to know more about this, though a few searches should turn up a lot of papers. I don't intend to add any advanced register allocation to this compiler until it's far closer to being functionally complete. But we will need to deal with some of it, but it's something I'd rather defer doing a proper implementation of until more functionality is in place for the simple reason that doing it well is a hell of a lot of work, and doing a half assed job at it will at least make things work, even though performance will suffer.

### Wrapping it up

Since there's now a separate file for the C runtime support, I've tar'ed up the files for step8 together with a Makefile here. If you have Ruby, make and gcc installed you should be able to just untar it and run make to get the "step8" test binary:
```sh
    $ make
    ruby step8.rb >step8.s
    as   -o step8.o step8.s
    cc    -c -o runtime.o runtime.c
    cc   step8.o runtime.o   -o step8
    $ ./step8 
    Testing sub and assign (before): 10
    Testing sub and assign (after): 9
    $ 
```