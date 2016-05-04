# 外部作用域：清理并增加“main”对象

Apologies for the delay... This part was ready before christmas, but for various reasons I never got around to posting it. And to make matters worse I managed to post the wrong part yesterday. Sigh.
### The Outer Scope

Stepping back from the attr_* debacle for a bit... I wanted to look at something else, mostly because it affects all my ad-hoc testing.

One of the downsides of starting this without a specific design in mind is that after I decided to go down the Ruby path we've been dragging along quite a bit of cruft.

But that is the cost of experimentation.

In particular, the compiler has a weird distinction between functions and methods: Define something with def outside of a class and it becomes a function, inside it becomes a method.

We need support for functions to make implementation reasonably easy: It makes bootstrapping the object model far easier, as well as integration with the outside C-based world.

But Ruby doesn't have functions.

So what to do?

Recently I wrote about hiding the "low level plumbing" and the change I'm about to show you gets us a bit closer to that while at the same time starting to clean up the function vs. method mess.

In Ruby, self outside of a class is the main object - an instance of Object. Logically then, for a def outside of a class, the method will be defined on `Object. So, we'll create that object.

### What about function calls?

Defining and calling functions will still be possible, but only using the s-expression syntax. That reasonably cleanly "hides" the plumbing we use functions for (in fact, it means we could stop the annoying convention I started of prefixing the names with "__" since they effectively now live in their own namespace, though I haven't changed that yet).

### Some preparations

First we must rewrite a few functions fully as s-expressions. This is straightforward. For example:
```ruby
    -def __get_string(str)
    -  s = String.new
    -  %s(callm s __set_raw (str))
    +%s(defun __get_string (str) (let (s)
    +  (assign s (callm String new))
    +  (callm s __set_raw (str))
       s
    -end
    +))
```
Otherwise the upcoming changes will turn them into methods, which is not what we want.

I'm not going to present all of them. The biggest one is probably __new_class_object, but all of these are straight forward translations.

These changes were done on the master branch prior to starting this feature proper.

### Splitting up "defun"

You can follow these remaining changes on the main-object branch

So far defun has been made up by two mostly different branches: If occurring inside a class, it's compiled as a method, if not it'd compile a function.

This is both messy and not very logical.

Going forward we'll separate it into defm that defines a Ruby style method, and defun, which, as before, defines a function. defun will be completely hidden from normal Ruby code - you'll have to dip down into the s-expression syntax to access it.

First our list of compile_* methods in compiler.rb:
```ruby
    -                   :do, :class, :defun, :if, :lambda,
    +                   :do, :class, :defun, :defm, :if, :lambda,
```
The new, simplified compile_defun:
```ruby
    # Compiles a function definition.
    # Takes the current scope, in which the function is defined,
    # the name of the function, its arguments as well as the body-expression
    # that holds the actual code for the function's body.
    #
    # Note that compile_defun is now only accessed via s-expressions
    def compile_defun(scope, name, args, body)
      f = Function.new(args, body,scope)
      name = clean_method_name(name)
    
      # add function to the global list of functions defined so far
      @global_functions[name] = f
    
       # a function is referenced by its name (in assembly this is a label).
       # wherever we encounter that name, we really need the adress of the label.
       # so we mark the function with an adress type.
       return [:addr, clean_method_name(name)]
    end
```
This is really more or less just tearing out the method part, and brings it roughly back to how it was before we added the method calling bit.

But compile_defm is also somewhat simpler than the old method part:
```ruby
    # Compiles a method definition and updates the
    # class vtable.
    def compile_defm(scope, name, args, body)
      scope = scope.class_scope
    
      f = Function.new([:self]+args, body, scope) # "self" is "faked" as an argument to class methods.
    
      @e.comment("method #{name}")
    
      body.depth_first do |exp|
        exp.each do |n| 
          scope.add_ivar(n) if n.is_a?(Symbol) and n.to_s[0] == ?@ && n.to_s[1] != ?@
        end
      end
    
      cleaned = clean_method_name(name)
      fname = "__method_#{scope.name}_#{cleaned}"
      scope.set_vtable_entry(name, fname, f)
    
      # Save to the vtable.
      v = scope.vtable[name]
      compile_eval_arg(scope,[:sexp, [:call, :__set_vtable, [:self,v.offset, fname.to_sym]]])
        
      # add the method to the global list of functions defined so far
      # with its "munged" name.
      @global_functions[fname] = f
        
      # This is taken from compile_defun - it does not necessarily make sense for defm
      return [:addr, clean_method_name(fname)]
    end
```
The most important changes?

The call to scope.class_scope at the top.
The change to call __set_vtable near the bottom instead of hardcoding the asm to set the vtable entry.
So, why those changes?

If you look at GlobalScope, we've added a @class_scope instance variable that is initialized to this:
```ruby
    @class_scope = ClassScope.new(self,"Object",@vtableoffsets)
```
This means that if you have a :defm occurring in the global scope (we'll get to that) it will be compiled in that class scope, as a method of class Object.

Poof and our functions are (almost) gone.

(In ClassScope we just return self from the new class_scope method.)

For the second change, adding __set_vtable is just the purist in me wanting to expel as much assembler as possible. Here it is (from lib/core/class.rb):
```ruby
    # FIXME: This only works correctly for the initial
    # class definition. On subsequent re-opens of the class
    # it will fail to correctly propagate vtable changes 
    # downwards in the class hierarchy if the class has
    # since been overloaded.
    %s(defun __set_vtable (vtable off ptr)
      (assign (index vtable off) ptr)
    )
```
Ok, so not just the purist in me. In fact, this makes re-opening classes "cleaner" in that this function will actually get reasonably complex as we fix this issue, and also later handle define_method and otherwise deal with cases where no vtable slot is actually available.

Anything else? A few pieces of cleanups to deal with the scope changes, and this little gem / turd depending on your point of view, in lib/core/core.rb:
```ruby
    +
    +# OK, so perhaps this is a bit ugly...
    +self = Object.new
    +
```
Ehm. We probably should put that in the compiler, or at least prevent arbitrary assigning to self in user code. But it works for now, and we've done worse in the past. To reiterate: Get things working first. Make them fast/pretty later.

The last vital change is this, in Parser#parse_def:
```ruby
    -    return E[pos, :defun, name, args, exps]
    +    return E[pos, :defm, name, args, exps]
       end
```
It does what it looks like - make the parser for the Ruby code spit out only :defm nodes instead of :defun nodes. Our code is now free of functions anywhere outside of s-expressions.

### What's the effect?

These changes make this work as expectd:
```ruby
    class Foo
      def hello
        puts "hello"
      end
    end
    
    def hello_world
      puts "hello world!"
    end
    
    %s(puts "foo") # Calls the C function
    Foo.new.hello  # Calls Foo#hello which calls Object#puts
    puts "world"   # Calls Object#puts (via the "main" instance)
    hello_world    # Calls Object#hello_world (via the "main" instance)
```
Adding this on the end on the other hand still fails (but it works in MRI) because __set_vtable doesn't propagate the vtable update downwards to subclasses of Object:
```ruby
    Foo.new.hello_world
```
When re-opening a class, new methods needs to be added "properly" by processing all child classes and updating their vtables.

The alternative (which moves the cost from the point of defining a method to the point of calling the method), is to implement a proper default method missing that ascends the class hierarchy to look for an implementation before giving an error.

We really should do both, since we eventually need to handle methods without vtable entries.

But that's for a future installment