# Ruby的编译问题
> Overview of some of the problems with compiling Ruby

Compiler technology is one of my hobbies, most recently satisfied with my project to write a series of post of how to write a (Ruby) compiler in Ruby. Since I really like Ruby, it's natural that I've done a fair amount of thinking about compiling Ruby, and what the problems with that are, and I decided it was time I wrote some of them down along with some thoughts on how they can be solved -- ideally without changing Ruby. Even more so since I decided I DO want to take the leap towards making my compiler project focus on actually compiling Ruby, and not just something somewhat like Ruby. 
All of the issues here affect a traditional "ahead of time" compiler - one that takes source in and spits out a binary, but many also affects JIT's.
I'll start with the problems, in no particular order:

### Problem #1: No delineation of compile time and runtime

Many scripting languages fall in this category, but Ruby does it one better, by making the class definitions executable. Where a compiler for a language like C++ can do a lot of work to analyze and optimize code at compile time, and can build most structures related to classes at compile time, in Ruby it may in some cases be hard to avoid executing code at runtime to determine what the class will look like.

A potential Ruby compiler also needs to solve the issue of what files are compiled once at compile time and linked into the application, and what files are evaluated (and possibly JIT compiled) at runtime. This is a possibly tricky tradeoff - reloading code has become a common solution for Ruby web applications to avoid shutdown and restarts, for example, but there are no formal hints in Ruby code that tells the application what may or may not be attempted reloaded later. 

Reloading in Ruby tends to depend on the ability of the Ruby object model to replace classes and methods, so the most natural solution for handling reloading may simply be to compile the classes in statically but retain this ability. That still doesn't solve the first part of the problem, though: If my app starts with "require 'foo'", do I expect "foo" to be loaded when compiling or each time? You probably wouldn't be surprised if the compiler loaded it once. But what if it starts with code that reads the contents of a directory, and require's each file in turn? That one is trickier - some scripts may use that method as a crude "plugin" mechanism, for example.


My compiler project has reached the point where dealing with something supposedly simple like "require" is now actually a stumbling block. Even something "simple" like Mspec from Rubyspec  actually depends on Ruby code being executed to determine what files to require, and I have to decide whether to for now use "hacks" to get around it (Mspec and a lot of other Ruby code use a small set of common idiomatic ways of modifying the load path for the code being required - a tiny interpreter subset could take care of the basic cases) or do it "properly" (compiling to shared objects and use dynamic loading at runtime, but this kills a lot of optimizations; or JIT compilation of files that get required this way; or even JIT compilation as a means of executing the code to determine the files to require and then requiring them statically). Or I could just skip the problem for now, and just handle the specific case where files are required using a static string and/or add a hack that will use a substitution table or regexp to rewrite the require's (eww...)

Anyway, the point is that this is a big hurdle for anyone hoping to write an ahead of time compiler for Ruby without resorting to JIT compiling most of the program anyway. Part of the appeal of ahead of time compilation is to avoid JIT compilation (and avoid lugging around a ton of source files), so while supporting JIT compilation for pathological cases is good and/or necessary depending on how you look at it, for an ahead of time compiler to make sense you want to make that a "last resort" if there is no sensible alternative.

### Problem #2: A very costly method dispatch


Let me count the number of ways Ruby makes method dispatch expensive in a naive implementation (see also my post on the Ruby Object Model as implemented in MRI)
A Ruby method call involves first identifying the class of an object. This either requires following the "class" pointer of an object, OR a "decoding scheme" (as used in MRI) to allow small objects like Fixnum, True, False, Symbol and Nil to be encoded without a pointer.
Then we must follow the "super" chain from class to class, potentially all the  way to Object, to determine which class can satisfy the method call. Then if that fails, it needs to do the same for #method_missing.
Because the type of the object stored in any variable is unknown, a compiler can not assume anything about the type of an object based on where it is stored. Unlike, say, C++, where the compiler will happily assume that a Foo * will hold a pointer to an object of class Foo or a subclass, and treat it accordingly, a Ruby compiler could not. This also largely affect inlining of methods as a viable optimization.
.. and anyway, since Ruby classes are open, users can add, alias and remove methods at will (with some minor restrictions), so a Ruby compiler largely can't assume a method stays the same through the lifetime of the object.
... even worse, thanks to meta-classes in Ruby, there may conceptually be a new "almost" superclass inserted for an object.
Luckily, there are a number of solutions that can vastly improve on the naive implementation of Ruby method dispatch. Unfortunately most (though not all) of them make the compiler and runtime more complex.
### Problem #3: Meta-programming

It's alluded to above. Ruby allows extensive modification of classes and even objects at runtime. Defining new methods, inserting new modules or otherwise messing with the structure makes static analysis to optimize other problematic aspects of compiling Ruby very hard. 

Take even something simple like integer arithmetic. Never mind that Ruby automatically handles overflow and turns Fixnum's into Bignum's, which means that even if you do try to make it cheaper than a method call, you first have to check whether you deal with a Fixnum (which is not an ordinary object) or a Bignum (which is an ordinary object), but if both values ARE Fixnum's you still face the uncertainty of whether or not a method of Fixnum has been replaced by unscrupulous monkey patchers...

### Problem #4: No statically defined set of instance variables

In a language like C++, object size is kept down because the compiler knows at compile time what instance variables exist for any given object. As a result, it can pack them tightly together. In Ruby, theoretically an object can have new instance variables show up at any time, through mechanisms such as #instance_variable_set, #instance_eval etc.. MRI solved this by making the instance variables stored in a hash table, which is incredibly wasteful for otherwise small objects. Ruby 1.9 has reduced this impact somewhat by storing up to 3 instance variables in the object itself (see the Ruby1.9 section), and then falling back to an external structure.

Luckily, in practice most objects will have a small and mostly statically determinable instance variable set, and some of the same methods that can speed up method dispatch can be used to handle this more effectively as well.


### Problem #5: Method visibility, default arguments and method arity


In C++ for example, you can easily know at compile time if a method is private or protected. In Ruby, since you don't know the class of the object you will be passed, you can't know that until runtime. This means this needs to be checked at runtime as well. We could imagine checking this only in private or protected methods, since use of them is relatively rare in Ruby code, but that's not easy either: 

Private methods in Ruby require self as an explicit receiver, which means they are only allowed to be called from within a method, and on the same object as the method is operating on. Slight little problem... How will the executed method know that this is the case? And then there's the ways of bypassing the check. An alternative for handling this is to pass along a flag saying "I promise I was called with self., honest!" (or used #__send__ etc. to bypass the check), but handling that without imposing overhead on calls that don't need it is non-trivial as well.

Default arguments, and more generally method arity, suffers from the same problem. In C++ the caller can take responsibility for initializing default arguments when not providing values for all of the arguments. In Ruby, the compiler won't know when you are calling a method that has default arguments vs. one that just have fewer arguments (for that matter, you don't know for sure if the number of arguments is right at all), and so this has to be handled by the callee. That means the callee needs to get an argument count, or have another method of determining how many arguments were passed. That's not to bad. 

But the callee then also have to contain the logic to initialize missing arguments. That's messier as it means either manipulating the stack frame, handling multiple ways of accessing an argument, making the arguments a "real" Ruby array and push the default arguments onto it, or otherwise moving arguments around to get them in a consistent location. There are simple ways of doing this (shove it all into a "real" Ruby Array object for example and convert the default argument initializers into the equivalent of ||= calls), and there are faster ways (keep things on the stack as much as possible, possible mess with the stack frame, and only convert to a Ruby Array object for methods where it's actually treated as one).

## Solutions


Below are some of my thoughts on how to address the bigger ones of these problems.

### Solution #1: Speeding up method dispatch with a multilevel approach to dispatch

In "Protocol Extension: A Technique for Structuring Large Extensible Software Systems" (1994; original Postscript file; PDF from Citeseer X), Michael Franz, presents a method for "protocol extension" in Oberon - a way of relatively effectively adding methods at runtime. The basic idea is that method changes (addition, overriding or removal) are rare, and method dispatch is frequent, so it makes sense to do more work when modifying the methods than it does when calling them. 

Franz suggested that the "vtable" for each subclass is made to contain pointers to all the methods for the entire hierarchy. A method that is overridden in a subclass has its method pointer overwritten in that subclass and all descendants of that subclass that has not overridden it themselves. 

The problem he noted is that in a big class hierarchy the vtables for each class may grow prohibitively large, while remaining mostly unchanged. This can be counteracted in a number of ways - one of them being splitting the vtable into chunks for "interfaces", and making each vtable a set of pointers to vtables for interfaces.

Another way is this: 

The compiler (or compiler-writer) can make educated guesses about which methods are most likely to exist in all classes, and which methods are less likely to get called:
Methods in classes high up in the hierarchy are more likely to remain present. In Ruby, methods on Object will exist in almost all classes (the only exception will be cases where the methods have been explicitly removed or aliased away).
Methods that are referenced in inner loops may be more important to make fast to call than others.
Analysis of number of call-sites can also give an indication of how frequent a method will be called.
Methods that are present in more than a certain threshold of classes, or that are judged to be particularly performance sensitive can be allocated an offset in a per-class vtable. Methods that are slightly less likely can be grouped into "interfaces", and require a one level indirection. As a last resort the implementation can be forced to fall back on a #send call that does lookups the way MRI does now. But see solution #2

### Solution #2: Polymorphic Inline Caches

Otherwise expensive method lookups can be cached. This caching can even be done "inline" in he code path by dynamically inlining the code for the classes that are seen at a call site in practice. Polymorphic inline caches were introduced in Optimizing Dynamically-Typed Object-Oriented Programming Languages with Polymorphic Inline Caches by Urs Hlzle, Craig Chambers, and David Ungar.
This does not remove the problem of potentially expensive lookups, but it drastically reduces the impact in cases where the type used is relatively stable (which in practice is the common case - the same variable is rarely assigned more than a handful different types).

The key to PIC for languages like Ruby is that you still need to check the type, and you either need to invalidate the old type when a class is modified, or you need to invalidate the caches. Maintaining that logic can be complicated (compare to the approach in solution #1, where all that is required is updating the vtables). PIC's can be combined with the approach above: Solution #1 provides cheap lookups, but #2 can still allow inlining of whole methods where appropriate, in a way that is safe.

### Solution #3: Trace Trees

Dr. Michael Franz and Andreas Gal have been working on a technique called Trace Trees. Possibly the best introduction is in a blog post by Andreas Gal, but the papers are also fairly accessible. Trace trees is the "hot new thing" - you'll find it used in the new JS JIT for Mozilla for example ("Tracemonkey"). The short description is that trace trees consists of tracking the execution of bits of an application - typically in a bytecode interpreter, and when a certain part is executed often enough, you "trace" the code execution  and create a "tree" of code fragments. You use this to identify loops or frequently executed code paths that are optimized (in the bytecode interpreter case, this would involve JIT compilation), and protected by a "guard" that verify whether to keep executing the new native generated code path.

While the approach is intended  for a bytecode interpreter, it has scope for being used to handle dynamic runtime optimization and inlining of specific code paths generated by an "ahead of time" compiler as well. The compiler can generate the best code it can, but inject timing and tracing code where appropriate to allow it to use information gathered at runtime to inline specific method calls into inner loops etc.. An in-between alternative is to let the AOT compiler use profiling data gathered from past runs to built trees on subsequent compiles (this approach is also suggested in the papers on polymorphic inline caches)

### Solution #4: Dynamic object packing (for instance variables)

We've already explored the building blocks for handling the troublesome issue of dynamic instance variables above. There are two parts to that problem: Quick access, which is hindered by having to resort to schemes that requires expensive instance variable lookups, and space which is hindered by a potentially dynamically changing set of instance variables. First of all it is possible to apply a similar analysis to that suggested for vtables: Identify the most likely set of instance variables for objects of a class. Assign specific offsets for those instance variables, and compile code using static offsets for code internal to the class.

For instance variables that are not guaranteed to be ever used, there's a value judgement: If information is available to determine a rough likelihood you can use a cutoff to decide which to always include in the object. This has the potential for huge space overhead if the guess is wrong. The alternative is to fall back on a hash table referenced from the object to handle additional instance variables. This is costly in space and time if the number of objects with extra instance variables is high.

However the latter method can be combined with the earlier solutions to reduce the time overhead.

To reduce space usage further, we can dynamically pack the object:

We can potentially cooperate with the garbage collector to identify pointers to an object, and then reallocate the object elsewhere and change the layout, and then use tracing similar to the one mentioned above to created optimized code paths that rely on the new static layout. If we don't want to move all objects, we can handle this by inserting "proxy classes" and making specific subsets of these objects instances of specific proxy classes. In fact, Ruby already sort-of does this by inserting proxies for Module's that are "hidden" for normal Ruby code when walking the inheritance chain.

This is a fairly complex solution, but one that can potentially allow very tight packing of objects, and avoiding a significant percentage of hash table lookups for instance variables. Combined with trace trees and PIC's, a significant part of the code overhead for method accessors used from outside the class can also be removed.