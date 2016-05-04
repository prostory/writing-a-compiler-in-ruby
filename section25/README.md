# 通过闭包实现define_method

First of all, here's the biggest reason I've been so excrutiatingly slow with getting this part together (at least that's my story, I guess I've had other things on my plate too...):



Tristan is 13 months now, and a real menace to my laptop (pulling off keys and drooling on the screen) and anything else within reach, and a real attention seeker...

So, whenever I'm slow at posting a new part, I'll blame him. I have no part in the delays at all, of course. None. I'm completely faultless...

### Towards define_method via closures

For starters, here's the commit that covers most of this

To support define_method we need to support blocks with arguments. Really, full closures.

If you haven't already, you should read my post on closures

We want this:
```ruby
    define_method(:foo) do |a,b|
       # Do something with a,b
    end
```
to be mostly equivalent to:
```ruby
    def foo a,b
       # Do something with a,b
    end
```
For our attr_* implementation we actually want more:
```ruby
    def attr_reader sym
      define_method sym do
        %s(ivar self sym) # FIXME: Create the "ivar" s-exp directive.
      end
    end
```
(or, as it turns out, not really that, as I realize the above can be simplified - if we have a lookup function to lookup the symbol to instance var slot, we can just use our index primitive to get the address; we'll return to that, but the above is the current state of attr_reader)

What that calls for is actually proper closures. One step harder.

There's the other issue that the above will be really inefficient. As in, requiring a hashtable lookup on every call inefficient, when we can statically determine the offset for any instance variable we know about at compile time. We'll get back to that at some point too, but if you've been following this series you know I care about getting things working first, efficient second.

For now attr_reader and friends still serve as a useful test case for basic closure support.

### Baby steps

The first step towards closures is actually easy (we've already done it). We have our :lambda primitive that creates an anonymous function (which isn't really anonymous, it's just that the name is quitely created in the compiler and not accessible to the programmer).

### Adding an environment

But an anonymous function is only of relatively limited use if you can't bind variables to it - it's not much more than a function pointer in that case.

So how do we let "sym" hang around, and how do we take advantage of that for define_method?

First, let us consider how you can call a lambda in Ruby:

* yield
* Proc#call

We could implement Proc#call via yield by passing the block as an argument to a method, if we wanted to, but that would probably be more inefficient than implementing both in terms of a primitive.

But this means those are the only things that has to explicitly know how to deal with the environment.

That gives us an interesting option. We could turn:
```ruby
    c,d = somevalues
    do |a,b|
       ... uses c,d 
    end
```
into
```ruby
    class SomePrivateUniqueClassName
        def initialize c,d
           @c,@d = c,d
        end
        
        def call(a,b)
            ... code here with access to c,d rewritten
        end
    end
```
And we could have it inherit Proc. yield in that case is just rewritten to block.call

This does however have the troubling implication of messing with self, and as this irb session shows, we'd have to fix that:
```sh
>> lambda { p self }.call
main
=> nil
>> 
```
The other alternative is to create a lightweight environment binding of sorts. Oh wait, we already have that. It's called an object. The problem is not the class above as such, but creating a full class for each block. We could do something like this instead:
```ruby
    class Proc
      def initialize & block
         # So Proc#initialize takes a block, but we're using this
         # to create blocks... Uh oh, I sense endless recursion.
         # We need to make it work so we can initialize a Proc
         # both from a block and from a raw function pointer
      end
      
      def call *args
         %s(call @block_func (res args))
      end
    end
```
Our code needs to be rewritten, so that blocks get wrapped in code to create the appropriate Proc object. Additionally, if the method uses variables from the surrounding scope, we need to alter the surrounding method and the block to refer to instance variables in this object, and if any of those variables are arguments to the surrounding scope, we need to copy them.

#### An example

Here's a simple example using a closure. Note the use of s-expressions here because we're now starting to get bitten by the changes we've done to turn Ruby code by default into method calls (the way it should be) without having put in the pre-requisite work to implement the number classes and Kernel methods (such as print to replace the libc printf). Ignore that for now.
```ruby
    def mkcounter step
      cnt = 0
      lambda do
        %s(assign cnt (add cnt step))
        %s(printf "cnt: %d\n" cnt)
      end
    end
    
    cnt = mkcounter(5)
    cnt.call
    puts "Calling again..."
    cnt.call
    puts "Done"
```
And here's the expected output:
```
cnt: 5
Calling again...
cnt: 10
Done
```
And here's the syntax tree after we've done the required transformations (more about that in a second):
```ruby
    [:do,
     [:defm,
      :mkcounter,
      [:step],
      [:let,
       [:__env__, :__tmp_proc],
       [:sexp, [:assign, :__env__, [:call, :malloc, [8]]]],
       [:assign, [:index, :__env__, 1], :step],
       [:assign, [:index, :__env__, 0], 0],
       [:do,
        [:assign,
         :__tmp_proc,
         [:defun,
          ".L2",
          [:self, :__closure__, :__env__],
          [:let,
           [],
           [:sexp,
            [:assign,
             [:index, :__env__, 0],
             [:add, [:index, :__env__, 0], [:index, :__env__, 1]]]],
           [:sexp, [:printf, "cnt: %d\n", [:index, :__env__, 0]]]]]],
        [:sexp, [:call, :__new_proc, [:__tmp_proc, :__env__]]]]]],
     [:assign, :cnt, [:call, :mkcounter, 5]],
     [:callm, :cnt, :call],
     [:call, :puts, [[:sexp, [:call, :__get_string, :".L0"]]]],
     [:callm, :cnt, :call],
     [:call, :puts, [[:sexp, [:call, :__get_string, :".L1"]]]]]
```
That's a bit of a mouthful, so lets go through it the new bits
```ruby
    [:let,
       [:__env__, :__tmp_proc],
```
This is added to hold a pointer to the environment we create to hold cnt and step, and to hold a pointer to the function we use to create the Proc respectively.
```ruby
       [:sexp, [:assign, :__env__, [:call, :malloc, [8]]]],
       [:assign, [:index, :__env__, 1], :step],
       [:assign, [:index, :__env__, 0], 0],
```
Then we allocate the environment. We copy the argument, and from now on we use (index env 1) to refer to step. We then carry out the cnt = 0 statement. cnt and step are both moved into the environment because they are used inside the closure.

(You may have already picked up that we currently won't handle nested closures - lets leave some fun for later)
```ruby
        [:assign,
         :__tmp_proc,
         [:defun,
          ".L2",
          [:self, :__closure__, :__env__],
```
Then we create a function and assign the address of the function to __tmp_proc. As you can see there are tree synthetic arguments to it:

self,`__closure__` and `__env__`. The first is, as you'd expect, the object the method is called on. The second is to be used when passing blocks to a method, and the third is used to refer to the environment when inside.

(Something I just realized: We're begging for a name clash, if there's a closure defined inside that needs a separate environment. Obnoxious details; later)
```ruby
            [:assign,
             [:index, :__env__, 0],
             [:add, [:index, :__env__, 0], [:index, :__env__, 1]]]],
```
And this monstrosity is what cnt = cnt + step was turned into.

Now, to make this work, we obviously need this `__new_proc` function, and a `Proc#call` that's sensible. Here's our current Proc:
```ruby
    class Proc
      # FIXME: Add support for handling arguments (and blocks...)
      def call
        %s(call (index self 1) (self 0 (index self 2)))
      end
    end
    
    # We can't create a Proc from a raw function directly, so we get
    # nasty. The advantage of this rather ugly method is that we
    # don't in any way expose a constructor that takes raw functions
    # to normal Ruby
    #
    %s(defun __new_proc (addr env)
    (let (p)
     # Assuming 3 pointers for the instance size. Need a better way for this
       (assign p (malloc 12))
       (assign (index p 0) Proc)
       (assign (index p 1) addr)
       (assign (index p 2) env)
       p
    ))
```
Very simplistic and rather ugly, but it does the work. As you can see, the environment pointer is stored in the Proc object, and Proc#call hides the ugliness.

You may notice that this is inefficient - an extra level of indirection. Indeed it is - we can in theory make the environment part of the Proc object and save an indexed lookup, and there's a bunch of other tricks waiting. But again, that's optimizations we wont deal with now.

For now, lets move on to how to do the appropriate changes to get the example output above.

### Some refactoring, and a few rewrites

Almost all the changes for this is in the Compiler class, and the code in question is now in the transform.rb file. There are some other small changes, but I don't think they need to be covered in detail.

Specifically, transform.rb represents the start of a bit of refactoring.

Some constructs in the Compiler class can be built easily on top of more primitive operations. lambda was an example of that.

Overall, a lot of things can be done by rewriting the parse tree. Doing it that was has the distinct advantage that it's possible to switch the various rewrites on/off to debug their effects, and to see the result long before it ends up as machine code.

It also helps isolate the CPU architecture specific parts further.

As a first step, transform.rb contains methods that solely rewrite the syntax tree. They are still currently part of the Compiler class, and do make use of the Emitter (specifically Emitter.get_local to get a unique label), but this can be changed later.

Unfortunately not all of these rewrites are small and simple.

In this round, we're left with three:

* rewrite_strconst
* rewrite_let_env
* rewrite_lambda

rewrite_strconst used to be in compiler.rb and is mostly unchanged.

#### rewrite_lambda

rewrite_lambda replaces the lambda handling in compiler.rb with a rewriting step that looks like this:
```ruby
    def rewrite_lambda(exp)
      exp.depth_first do |e|
        next :skip if e[0] == :sexp
        if e[0] == :lambda
          args = e[1] || []
          body = e[2] || nil
          e.clear
          e[0] = :do
          e[1] = [:assign, :__tmp_proc, 
            [:defun, @e.get_local,
              [:self,:__closure__,:__env__]+args,
              body]
          ]
          e[2] = [:sexp, [:call, :__new_proc, [:__tmp_proc, :__env__]]]
        end
      end
    end
```
What this rewrite does, is use our Array#depth_first method from extensions.rb to descend down the parse tree looking for :lambda nodes. It will skip any :sexp nodes, as they are used as "guards" to mark areas that should not be touched by rewrites.

When it finds a :lambda, it will replace the :lamda with this:
```ruby
    %s(do
       (assign __tmp_proc
               (defun [result of @e.get_local
                 (self __closure__ __env__ [ + args])
                   [body]))
       (sexp call __new_proc (__tmp_proc __env__))
      )
```
Effectively defining the lambda (as we did before), but then calling __new_proc with the result. You'll note __tmp_proc is seemingly superfluous. It is in fact a workaround for a slight problem: The call needs to be within a sexp to not be rewritten to a callm (Ruby method call), but the defun can't be within a sexp, or other rewrites won't touch the body of the defun.

We'll clean that up later.

#### rewrite_let_env

Now for the hairy bit. Really hairy.

The goal of this one is to identify variables in the methods - this used to happen in the parser in a really simplistic way - combined with identifying :lambda nodes (and so this must run before rewrite_lambda) and lifting variables that are used inside the closure into an environment, and update the :let in the surrounding method definition to have a suitable environment.

It then also adds code to allocate the environment, as well as to shadow method arguments. Finally it rewrites accesses to variables that have been lifted into the environment, so that it uses %s(index __env__ offset) instead of the variable name.

Let's go through the code.
```ruby
    def rewrite_let_env(exp)
      exp.depth_first(:defm) do |e|
```
We're hunting for :defm nodes only. Everything else will be ignored.
```ruby
        vars,env= find_vars(e[3],[Set[*e[2]]],Set.new)
``` 
First step is to find a candidate set of variables. find_vars is another new method, and we'll go over that next. e[3] is the body of the method. find_vars is recursive, and we pass in the methods arguments (e[2]) as the starting "scope". Third we pass in an empty Set as the the starting point for the environment.
```ruby
        vars -= Set[*e[2]].to_a
```
We remove the method arguments, as we'll use vars to create a :let node later on.
```ruby
        if env.size > 0
```
If find_vars found any variables to make part of the environment, we need to:
```ruby
           body = e[3]
           rewrite_env_vars(body, env.to_a)
```    
Rewrite access to the members of the environment
```ruby
           notargs = env - Set[*e[2]]
           aenv = env.to_a
           extra_assigns = (env - notargs).to_a.collect do |a|
             [:assign, [:index, :__env__, aenv.index(a)], a]
           end
```    
Create assigns to copy all arguments that are used in the closures from the arguments on the stack into the environment 
```ruby
           e[3] = [[:sexp,[:assign, :__env__, [:call, :malloc,  [env.size * 4]]]]]
```    
Allocate the environment 
```ruby
           e[3].concat(extra_assigns)
           e[3].concat(body)
 ```
Tack on the extra assigns and the body.
```ruby
         end
         # Always adding __env__ here is a waste, but it saves us (for now)
         # to have to intelligently decide whether or not to reference __env__
         # in the rewrite_lambda method
         vars  :__env__
         vars  :__tmp_proc # Used in rewrite_lambda. Same caveats as for __env_
    
         e[3] = [:let, vars,*e[3]]
```
Then we create a :let node with the new variable set, including `__env__` and `__tmp_proc` which we've seen elsewhere.
```ruby
         :skip
       end
     end
```
Finally we skip any nodes below the :defm so the depth_first call does not waste time.

#### find_vars

The purpose of find_vars is to identify variables that should be put in a let node as well as that should be part of a closure environment. This one is hairy, and a clear candidate for looking for ways to clean up later

def find_vars(e, scopes, env, in_lambda = false, in_assign = false)
```ruby
    return [],env, false if !e
```
First of all, if we've been called with a nil expression, there are now variables to return, and the same environment passed in. 
```ruby
    e = [e] if !e.is_a?(Array)
    e.each do |n|
      if n.is_a?(Array)
```  
An expression? 
```ruby
        if n[0] == :assign
          vars1, env1 = find_vars(n[1],     scopes + [Set.new],env, in_lambda, true)
          vars2, env2 = find_vars(n[2..-1], scopes + [Set.new],env, in_lambda)
          env = env1 + env2
          vars = vars1+vars2
          vars.each {|v| push_var(scopes,env,v) }
```          
If it's an assignment, then we want to process both the left hand and right hand, but we need to mark the left hand since variables on the left hand introduce a new variable in the scope.
```ruby
        elsif n[0] == :lambda
          vars, env = find_vars(n[2], scopes + [Set.new],env, true)
          n[2] = [:let, vars, *n[2]]
```         
Special casing on :lambda because if we're handling a closure, and variables found inside the closure that were defined outside means that those variables needs to be lifted into the closure environment we're creating so that they're not allocated on the stack. 
```ruby
        else
          if    n[0] == :callm then sub = n[3..-1]
          elsif n[0] == :call  then sub = n[2..-1]
          else sub = n[1..-1]
          end
          vars, env = find_vars(sub, scopes, env, in_lambda)
          vars.each {|v| push_var(scopes,env,v) }
        end
```
Finally we special case on :callm and :call when handling the remainder, because they have arguments that are not sub-expressions to be considered when creating the closure environment. 
```ruby
      elsif n.is_a?(Symbol)
        sc = in_scopes(scopes,n)
        if sc.size == 0
          push_var(scopes,env,n) if in_assign
```
If n is a variable that does not exist in any of the scopes we keep track of, and it is not part of the environment, and it's on the left hand of an assign, we add it to the top scope.

NOTE: This current version is buggy. Not every variable potentially occuring on the left hand side should be added, just the one assigned to.
```ruby
        elsif in_lambda
          sc.first.delete(n)
          env  n
        end
```
If it is in a scope and we're in a lambda, then the variable needs to be moved (deleted) from that scope and added to the closure environment. 
```ruby
      end
    end
    return scopes[-1].to_a, env
  end
```
Finally we return the innermost scope from this level and the environment

#### rewrite_env_vars

Finally let's look at rewrite_env_vars. When we've gathered together all the variables for the closure environment, we have an array, and any accesses to those variables needs to be rewritten to (index `__env__` num). That's done like this, and I hope this one is simple enough not to go through line for line:
```ruby
    def rewrite_env_vars(exp, env)
      exp.depth_first do |e|
        STDERR.puts e.inspect
        e.each_with_index do |ex, i|
          num = env.index(ex)
          if num
            e[i] = [:index, :__env__, num]
          end
        end
      end
    end
```
### Next steps

We still haven't actually added support for define_method, but we now finally have most of the plumbing. That's the next step.

Then I want to start alternating between something more practical: Making the compiler compile early/simple versions of itself, and let it "eat its own tail" so to speak, and secondly to start refactoring and cleaning it up.