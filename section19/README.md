# 对象模型

If you've been following the commits to the Github repository, you've already seen this go in... Specifically, this was the state as at the end of this post. Here's finally some explanation.

### The Object Model

Every object oriented language implements it's own "brand" of object model. For this compiler I want to eventually approximate the Ruby object model. The Ruby model is, however, extremely dynamic, and extremely dynamic translates to hard to compile efficiently (a post on the problems facing compilation of Ruby is upcoming).

For most of this series I've ignored performance issues, but only because they've not been structural or really significant. The code we generate is messy and ugly and inefficient, but because of lack of optimization more than because the concepts are unsound.

I will leave the details of the problems with compiling the Ruby model for my later post, but lets boil it down to something very simple. Two criteria for making something easy to compile efficiently:
Low cost of determining the type of an expression at runtime.
Low cost of determining the method to call for a specific (type,method name) pair.
A naive implementation of the Ruby model falls down on both of these. So lets take a step back and see if we can approximate the dynamic features by layering some relatively simple approaches. This is not going to happen in one post, but by the end of this post we will have the infrastructure in place and be able to call methods.

First, lets take a look at some C code. C code?!? Every C-programmer that's written a huge project is likely to have come across or implemented an object system in C. It's easy thanks to ease of bit-fiddling and function pointers, so it makes it easy to illustrate the concepts. The following two sections are pretty basic if you already understand how object orientation in static languages is usually implemented.


#### A basic "static" object model


If you work in a statically typed language, efficient object orientation is pretty much trivial. The simplest example is a "non-virtual" member function - A function that is ALWAYS the one being called when you call a method of it's name on an object stored in a variable of a specific type. In C++ this would be:
```cpp
    class Foo {
       void bar() {
           puts("Hello world");
       }
    };

    int main() {
        Foo test;
        test.bar();
    }
```


Well, this isn't really anything but a function. In C we can do the same thing easily:
```cpp
    struct Foo {
    };
    
    void Foo_bar(Foo * this);
    
    int main() {
       struct Foo test;
       Foo_bar(&test);
    }
```
There's no real point in having the "this" pointer, but implicitly it's there in the C++ code, and in the C code we'd need it the second we want to add instance variables.

A non-virtual member function in a statically typed language is nothing more than a function where "this" (or "self" in other languages) is passed as an argument.


#### A static model with virtual methods

The next step up is allowing "virtual methods". While non-virtual methods can't be overriden - the method that gets called is tied to the type of the pointer, not to the type of the object - virtual methods can. If you've ever had a programming class with any language like C++ or Java you should be familiar with this.

But how is it implemented? Enter the "vtable". A vtable is a table of pointers to virtual methods. The compiler will decide on the layout at compile time, implementing something like this (details omitted):
```cpp
    struct Foo;
    
    struct Foo_vtable {
       void (* bar) (struct Foo * this);
    };
    
    struct Foo {
       struct Foo_vtable * vtable;
    };
    
    /* ... later after setup: */
    struct Foo * test = new_Foo();
    test->vtable->bar();
```
Foo_vtable's "bar" function pointer would be filled in with the address of Foo_bar from earlier, and methods can be called indirectly via the vtable. In Ruby the equivalent - sort of - is the "klass" pointer each object has (reachable inside Ruby - sort of - with #class). The indirection adds a small cost: You need to load the address of the method indirectly from an offset into the vtable, but with it comes great flexibility.

Typically a purely C based OO system would hide the "vtable" bit by offering wrappers, so you could do Foo_bar(test) just like before (the actual implementation would then be named something else), but under the hood it'd do the same.

#### The problem of the static model in a dynamic world

The approach above works great if all type information is known at compile time. It produces efficient code. But already in the case of virtual methods there's an overhead: The extra lookup. Theoretically this lookup can be omitted if the compiler can know at compile time which implementation can be called - if it can determine statically which specific class an object belongs to, rather than a shared ancestor. In practice few if any compilers do this.

But languages like Ruby takes this to a whole other level. In Ruby you can add methods, rename methods, remove them, import modules and more. Each time this happens the actual method that gets executed when you run foo.bar can change.

But more importantly, in C++, if I have the variable "foo" that I know contains a pointer to an object, I can know enough about the type of that object by lookup back at the type declarations. Specifically I can know the vtable layout, and so I can turn it into a simple array lookup to find the address of the method.

In Ruby that's simply not possible in most cases ("most" because static analysis could tell you the initial type of many objects, however even then things could quickly change at runtime) since a variable "foo" can hold an object of ANY type, and they may not really be related other than both being instances of subclasses of Object.

The original Ruby implementation solved this by checking each class in turn, starting with the objects class, and then the superclass of that class and so on, and using a hash table to match a method name to a method body for each class. This is slow.

#### Minimizing the overhead

The first stab at the object model for my compiler will use a vtable. In fact, the version you're about to see doesn't support inheritance yet, so a vtable is trivial. However, let me describe how we'll address inheritance and working towards a model more or less as dynamic as Ruby's.

First of all we should realize that most objects share a lot of methods in any languages like Ruby where all objects inherit from a common base class. All of these methods are in almost all classes because they're defined in Object (the exception being if a subclass specifically remove a method):
```sh
    irb(main):003:0> Object.new.methods.sort
    => ["==", "===", "=~", "__id__", "__send__", "class", "clone", "display", "dup", "eql?", "equal?", "extend", "freeze",
    "frozen?", "hash", "id", "inspect", "instance_eval", "instance_of?", "instance_variable_defined?", "instance_variable_get",
    "instance_variable_set", "instance_variables", "is_a?", "kind_of?",  "method", "methods", "nil?", "object_id",
    "private_methods", "protected_methods", "public_methods", "respond_to?", "send", "singleton_methods", "taint",
    "tainted?", "to_a", "to_s", "type", "untaint"]
```
Since we know this is the case, there's no reason to do a complex lookup if we're willing to a little extra work when creating classes and overloading methods:

* When creating a new subclass, we copy the vtable of the parent.
* When overloading a method, then the vtable entry for that method for the class is modified, and so are the vtable entries for all subclasses that haven't provided an overloaded method themselves.

This is actually pretty simple logic to implement - we just need to know what subclasses a class has.

One problem though: Adding new methods, and what about handling a larger set of methods?

When we add a new method we need extra vtable slots, and that's obviously not going to scale, since every class in the same class hierarchy need to have a slot for every method in the hierarchy. In a language like Ruby where ALL classes inherit from a single root - Object in Ruby's case - that means ALL classes will have a vtable slot for every method in the system. We face the problem of what happens if you have a 100 classes, all adding it's own methods, and all using different names.

Well... Oops. Actually it's not so bad: Since we can't know of all the methods at compile time we set a maximum size of the vtable. Within the allowed space we pack the methods that are shared across the largest set of classes. Then we just need a method for handling "spill". Worst case? We do like Ruby and reserve space for a hash table for each class - we get the best of both worlds: Very common methods can be looked up extremely cheaply, while the rest needs to resort to more work (but even for that there are lots of techniques we can use, such as various types of method caches, and something called "trace trees" you might have heard about if you're following what's going on with javascript implementations for modern browsers)

If a method is ... missing from a class, whether deleted or just not there, we simply add the address for the method_missing for that class (or the global one).

For now we'll just add the vtable. Then we'll worry about handling "spill" and inheritance.


### Implementation

#### An example, and bootstrap code


Here's what we'll make the compiler handle:
```ruby
    def __new_class_object(size)
      ob = malloc(size)
      %s(assign (index ob 0) Class)
      ob
    end
     
    class Class
      def new
        ob = malloc(4)
        %s(assign (index ob 0) self)
        ob
      end
    end
     
    class Foo
     
      def bar
        puts("test")
        self.hello
      end
     
      def hello
        puts("Hello World!")
      end
    end
     
    %s(let (f) 
      (assign f (callm Foo new))
      (callm f bar)
    )
```
The s-expressions at the end is a stupid workaround - as it stands the compiler doesn't handle nested local variable scopes, and so we can't blindly wrap one around the entire program, and we don't have parser support for adding explicit let's (I don't want it - I like Ruby's approach to inferring new variables).
The other s-expressions are workarounds too - we don't have anything equivalent to "index" that can be used on the left hand side of an assign. Anyway, the first part here shows part of the bootstrapping of the object model. "__new_class_object" is used to return raw slap of memory with a pointer to the class "Class" as the first thing.

Then we actually define the class Class, and implement a rudimentary "new" method. As you can see, for now it just allocates a 4-byte "object" and assigns "self" to the first 4 bytes, and then returns the new object. No call to "initialize" like in Ruby etc.

Apart from some very basic initialization which you'll see soon, I'd like to implement as much as possible this way, without adding runtime libraries written in a different language.

The rest is just a simple test. Note that for now I'm requiring an explicit receiver (hence "self.hello" instead of "hello").


#### Updating the parser


Here's the Github commit for the first parser change.

We're not doing much. Just making the parser handle this grammar fragment:
```
class ::= "class" ws* name ws* exp* "end"
```
Simple and straightforward (and nothing near as expressive as Ruby - note the absence of subclassing for starters).

The commit really does speak for itself, so I won't go into much more detail, other than to say I just noticed something I didn't like - namely that I patched "class" into "exp", and thus the grammar allows classes within classes - ignore that... If there's anything you don't get, feel free to ask in the comments.

The one other thing worth noticing is that I've added a "." operator:
```ruby
 "."  => Oper.new(90, :callm,  :infix),
```
So now we need to add the :class and :callm constructs to the compiler proper, and that's a bit more involved.

#### Adding global constants

First things first. To have a way of storing the pointers to the class objects we need some form of global storage, and Ruby style global "constants" seem like an ok way of doing it. So take a look at scope.rb, which is where the scope classes have been relocated, and which holds this:
```ruby
    # Holds globals, and (for now at least), global constants.
    # Note that Ruby-like "constants" aren't really - they are "assign-once"
    # variables. As such, some of them can be treated as true constants
    # (because their value is known at compile time), but some of them are
    # not. For now, we'll treat all of them as global variables.
    class GlobalScope
      attr_accessor :globals
     
      def initialize
        @globals = Set.new
      end
     
      def get_arg a
        return [:global,a] if @globals.member?(a)
        return [:addr,a]
      end
    end
```
output_functions is then updated to do this:
```ruby
       def output_constants
         @e.rodata { @string_constants.each { |c,l| @e.string(l,c) } }
    +    @e.bss    { @global_constants.each { |c|   @e.bsslong(c) }}
       end
```
Also take a look at "emitter.rb" for #bss and #bsslong, and search compiler.rb for ":global" to take a look at how the space is allocated (#bss/#bsslong) and how data is moved into and out of the constant references.

#### The juicy bits - :callm and :class

If you looked through scope.rb you might have already noticed ClassScope and VTableEntry. If not, do that now - they're pretty simple, and for now mainly ensure we have a place to stuff the vtable information. Lets start with :class, which is used to actually define the class.
```ruby
     def compile_class(scope,name,*exps)
        @e.comment("=== class #{name} ===")
        cscope = ClassScope.new(scope,name,@vtableoffsets)
        # FIXME: (If this class has a superclass, copy the vtable
        # from the superclass as a starting point)
        # FIXME: Fill in all unused vtable slots with __method_missing
        # FIXME: Fill in slot 0 with the Class vtable.
        exps.each do |l2|
          l2.each do |e|
            if e.is_a?(Array) &amp;&amp; e[0] == :defun
              cscope.add_vtable_entry(e[1])
            end
          end
        end
        @classes[name] = cscope
        @global_scope.globals &lt;&lt; name
        compile_exp(scope,[:assign,name.to_sym,
              [:call,:__new_class_object,[cscope.klass_size]]])
        @global_constants &lt;&lt; name
        exps.each do |e|
          addr = compile_do(cscope,*e)
        end
        @e.comment("=== end class #{name} ===")
      end
```
This is hooked into #compile_exp in the usual way. It's quite straightforward. The goal is to compile the class definition straight into wherever it is found in the form of the code needed to allocate and initialize the class object with the vtable. First we create a new class scope, and we'll look at that later. Then we loop through the expressions inside the [:class, ...] array to see which of them are :defun's. For each of them we add a vtable entry to the class scope.

Then we add the class to a global hash, as well as add a global constant for the class. Then we call #compile_exp with an expression to call one of the bootstrap functions above - __new_class_object, with the size of the class object (calculated based on the vtable) and assigning the result to the freshly created global constant.

We then go through the expressions again, and compile the code contained in the class. We'll look at the modified #compile_defun next:
```ruby
      def compile_defun scope,name, args, body
        # Ugly. Create a default "register_function" or something. 
        # Have it return the global name
        if scope.is_a?(ClassScope) 
          f = Function.new([:self]+args,body)
          @e.comment("method #{name}")
          fname = @e.get_local
          scope.set_vtable_entry(name,fname,f)
          @e.load_address(fname)
          @e.movl(scope.name.to_s,:edx)
          v = scope.vtable[name]
          @e.addl(v.offset*Emitter::PTR_SIZE,:edx) if v.offset > 0
          @e.movl(@e.result_value,"(%edx)")
          name = fname
        else
          f = Function.new(args,body)
        end
        @global_functions[name] = f
        return [:addr,name]
      end
```
As you can see, when given a class scope, this function will do more work: It'll create the function with a "local" name, just like we do with lambda's. Then the local name is added to the vtable entry so we can refer to it later.

Then we emit code to load the address of the function into %eac, and the address of the class object to %edx. We add the offset of where in the vtable the address of the method should be stored to %edx, and finally we save the address of the method (held in %eax) into the address pointed to by %edx. This is all very ugly and should be refactored so that the compiler code is less dependent on the architecture - we'll do that later.

Then we finish up just as before.

So, that's what it takes to define the class. Now we need to be able to compile method calls. Enter :callm:
```ruby
      def compile_callm scope,ob,method, args
        @e.comment("callm #{ob.to_s}.#{method.to_s}")
        args ||= []
        @e.with_stack(args.length+1,true) do
          ret = compile_eval_arg(scope,ob)
          @e.movl(ret,:eax) if ret != :eax
          @e.save_to_stack(:eax,0)
          args.each_with_index do |a,i| 
            param = compile_eval_arg(scope,a)
            @e.save_to_stack(param,i+1)
          end
          @e.movl("(%esp)",:eax)
          @e.movl("(%eax)",:edx)
          off = @vtableoffsets.get_offset(method)
          raise "No offset for #{method}, and we don't yet implement send" if !off
          @e.movl("#{off*Emitter::PTR_SIZE}(%edx)",:eax)
          @e.call(:eax)
        end
        @e.comment("callm #{ob.to_s}.#{method.to_s} END")
        return [:subexpr]
      end
```
Eww.. Compare to #compile_call and it looks pretty nasty, but it's not that horrible. First we compile the expression provided for the object we call the method on, and save that to the allocated stack space. In fact what we're doing is allocating "self" (notice the "+1" added to args.length in the #with_stack call). Remember we've already added "self" to the list of arguments for the function in #compile_defun

Then we compile the arguments as normal.

After that we get hold of "self" again, and move the address held at the first 4 bytes of the object (which is our vtable pointer) to %edx. Then we retrieve the vtableoffset for this method, and copy the address of the method from the vtable into %eax, and finally we call it, just as we would for #compile_call.

This isn't a very efficient implementation - we should avoid the extra stack access to get the method address, but we can optimize that later.