# 本地变量

It's been a few months, and apart from my side step into parser land I haven't had much time to keep posting this series, in part because of work, and part because of repeated illness that after being poked and prodded and x-rayed and being subjected to ultrasound (no, I'm not pregnant, nor is anything about to rupture) and being tapped for what feels like several litres of blood turns out to likely 'only' be infectious mononucleosis. Luckily I've avoided the sometimes lengthy bouts of fatigue, but 4 rounds with several days of fevers and fatigue in just a few months hasn't been entertaining, and if I'm unlucky I might keep getting them occasionally for a few more months. Fun.

Anyway, enough moaning. Last time, I implemented support for arrays, and mentioned local variables as one of the remaining targets:

### Local variables

Lets first figure out how to do it, by resorting to our trusted old method of seeing what gcc does to this:
```cpp
int foo() {
  unsigned int i;
  i = 42;
}
```
This is the abbreviated result of gcc -s:
```asm
foo:
 pushl %ebp
 movl %esp,%ebp
 subl $16, %esp
 movl $42, -4(%ebp)
 leave
 ret
```
Figuring out what's going on shouldn't be too hard:


* First we're simply saving %ebp on the stack.  
* Then we're taking a copy of the stack pointer.   
* We're subtracting 16 from the stack pointer. Effectively we're allocating 16 bytes on the stack for our own use.  
* The next instruction is the "i = 42". gcc have placed "i" at the top 4 bytes of the 16 that was allocated. Note that it's using an offest of %ebp (the saved copy of the stack pointer) because the stack pointer could easily be manipulated further by pushing arguments on the stack for function calls etc., and using %ebp means the compiler doesn't need to keep track of all that and adjust the offset accordingly.  
* "leave" is simplified way of moving %ebp back into %esp (resetting the stack to where it was, essentially freeing all stack space allocated in the function), popping the old %ebp off the stack again (effectively restoring the value used for local variables etc. in the surrounding scope).  
* "ret" as we know just returns to the address at the top of the stack.  

There's not all that much to it.
Our first step is to introduce a new type of scope. Previously I added the Scope class to keep track of named parameters to a function. Now we have another form: A set of variables tied to a local scope. Why not tie it directly to the function, and just add it to the Scope class? Consider Ruby blocks or the lambda's we've already introduced: Many languages have constructs that introduce new scopes inside a function that may only encompass part of that function. Adding a new class gives us flexibility to introduce new scopes wherever we want:
```ruby
class LocalVarScope
 def initialize locals, next_scope
 @next = next_scope
 @locals = locals
 end

 def get_arg a
 a = a.to_sym
 return [:lvar,@locals[a]] if @locals.include?(a)
 return @next.get_arg(a) if @next
 return [:addr,a] # Shouldn't get here normally 
 end
end
```
This LocalVarScope class follows the same interface as the Scope class for the #get_arg method which is the only thing we really cares about. It returns a new type :lvar to indicate a local variable, which needs to be given different treatment during code generation. If it can't find a local variable, it tries passing the #get_arg request on to the next scope. The last line is a fallback - at that point we could either make it an error, or do as we've done in the past and just assume it refers to a global variable defined elsewhere. It would only get that far if the LocalVarScope occurs outside a function, which is a major WTF, so perhaps I should make it a variable. That's one for later.

Our next step is to handle :lvar. The following is added to #compile_eval_arg immediately before the return:
```ruby
 @e.load_local_var(aparam) if atype == :lvar   
```
This calls a new function in the emitter to load a local variable into %eax. So lets add the new functions to the emitter:
```ruby
 def local_var(aparam)
 # FIXME: The +2 instead of +1 is ecause %ecx is pushed onto the stack in "main". It's really only needed there. 
 "-#{PTR_SIZE*(aparam+2)}(%ebp)"
 end

 def load_local_var(aparam)
 movl(local_var(aparam),:eax)
 end

 def save_to_local_var(arg,aparam)
 movl(arg,local_var(aparam))
 end

 def with_local(args)
 # FIXME: The "+1" is a hack because main contains a pushl %ecx 
 with_stack(args+1) { yield }
 end
```
`#local_var` is a helper to return the correct offset off (%ebp). The FIXME relates to the fact that we save %ecx in the main function, and for now I won't treat the main function differently, which means wasting 4 bytes for every strack fame instead. [Yeah, I know - it'll get fixed]

`#with_local` is a helper equivalent to #with_stack. It's introduced for two reasons: a) it's possible we'll want to implement them differently on different architectures, and more importantly for now, b) it hides the "hack" mentioned above squarely in the Emitter.

`#load_local_var` and `#save_to_local` var does the obvious things - they load and store the local variable from/to a register where we can manipulate the value more easily. Which brings us to a problem...

#### Interlude: Register allocation (or lack thereof)

So far we've done nothing to handle register allocation. We've just assumed "default" registers for various expressions, and it's worked. But it's worked because almost all we've done has been function calls which involve saving stuff to the stack and popping it back off.

However, that's a very inefficient solution, and eventually it needs to be fixed. There are two approaches to solving this:

1) Allocate registers "properly"; and there's a hell of a lot of literature on doing that well - you want to avoid unnecessary memory accesses by keeping the most frequently used variables in memory, but you also need to be careful as it can cause a real mess when it comes to thread synchronization etc.

2) Keep pushing things on the stack, and then rely on a "smart" peephole analyzer to "fix" things by removing stack manipulation and doing the register allocation properly at that stage. Essentially you're not getting away from #1 - to get a high performance compiler, sooner or later you need to worry about proper register allocation.

Unsurprisingly, I've opted for later. Functionality first. So to make sure things continues to work for now we make some small changes to use the stack. Here's a diff of #compile_assign:
```ruby
 def compile_assign scope, left, right
 source = compile_eval_arg(scope, right)
+ @e.pushl(source)
 atype, aparam = get_arg(scope,left)
 if atype == :indirect
+ @e.popl(:eax)
 @e.emit(:movl,source,"(%#{aparam})")
+ elsif atype == :lvar
+ @e.popl(:eax)
+ @e.save_to_local_var(source,aparam)
 elsif atype == :arg
+ @e.popl(:eax)
 @e.save_to_arg(source,aparam)
 else
 raise "Expected an argument on left hand side of assignment"
 end
 return [:subexpr]
 end
```
In addition to the call to the previously defined #save_to_local_var to handle the case where a local variable is the left argument to an assign, we've littered the function with a pushl (push to the stack) and popl (pop something off the stack) to avoid clobbering intermediate results.

#### Introducing "let"

"let" is a name with long traditions - having been used in LISP, numerous BASIC's and a variety of modern functional languages. We use "let" as a construct to list a bunch of local variables, and make the expressions contained in a "let" define the scope. Example:
```lisp
(let (a b c) expr1 expr2 .. expr-n)
```
This allows us to introduce new scopes wherever we like. Later, when I put a more advanced parser on top, we may end up hiding this functionality and have the parser or an intermediate stage introduce let's where it fits to introduce new scopes.

First we augment #compile_exp:
```ruby
 return compile_let(scope,*exp[1..-1]) if (exp[0] == :let)
```
Simple enough, so let's move on to #compile_let:
```ruby
 def compile_let(scope,varlist,*args)
 vars = {}
 varlist.each_with_index {|v,i| vars[v]=i}
 ls = LocalVarScope.new(vars,scope)
 @e.with_local(vars.size) { compile_do(ls,*args) }
 end
```
As you can see it quite straightforwardly uses the new LocalVarScope class, and it then compiles the expressions enclosed in the let expression.
Here's a rewrite of the "testarray" example from part 12 that demonstrates the use of "let" (allowing us to drop the "lambda"):
```ruby
prog = [:let, [:i],
 [:assign, :i, [:array, 2]],
 [:printf, "i=%p\n",:i],
 [:assign, [:index, :i, 0], 2],
 [:assign, [:index, :i, 1], 42],
 [:printf, "i[0]=%ld, i[1]=%ld\n",[:index,:i,0],[:index,:i,1]]
]
```
As usual the full source code is available for download here.