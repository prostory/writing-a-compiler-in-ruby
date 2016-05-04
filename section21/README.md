# 对符号类的基本支持；开始工作在attr_reader / attr_writer / attr_accessor上

I've been lazy lately... Well, not really, I've been extremely busy, but I ought to have fit this in earlier. It's gotten harder and harder to get done too, since it's now more work since I had to go back and figure out a lot of the reasons for what I'd done.

Anyway, finally a new part, though short.

### Down the rabbit hole: attr_(reader|writer|accessor)

Adding attr_reader / "attr_writer" / "attr_accessor" Should be easy, right? After all, all they do is allow read/write or both of member variables.

Trouble is you can't know that in advance.
```ruby
    class Class
      def attr_reader foo
        puts "Hah!"
      end
    end

    class Foo
      attr_reader :bar
    end

    foo = Foo.new
    p foo.bar
```
Ouch. This is part of what makes Ruby exceptionally painful to compile.

It doesn't mean we can't make some assumptions, though, as long as we can handle the worst case where someone does something stupid (later we may want to add an option to make it assume you're not being stupid, and enable additional optimizations).

So how do we do this then?

Well, the obvious answer is to implement it in Ruby. Here are naive initial implementations (that only handle a single symbol, not an array like the real thing):
```ruby
    def attr_accessor sym
      attr_reader sym
      attr_writer sym
    end

    def attr_reader sym
      define_method sym do
         %s(ivar self sym)
      end
    end

    def attr_writer sym
      define_method "#{sym.to_s}=".to_sym do |val|
        %s(assign (ivar self sym) val)
      end
    end
```
Note the s-expressions that rely on a new "ivar" primitive (that's not been added yet). That part is simple enough to add, but what else does the above require to be added to the compiler?

That's the ugly part. Here's a (possibly incomplete) list:

* define_method
* Real Symbol class
* Symbol#to_s
* Real String class
* String.to_sym
* Indirectly the :lambda s-expression construct needs to have support for variables

This is part of the reason I picked attr_* as the next thing to implement: It's an important, frequently used piece of functionality that snowballs and help drive implementation of functionality that is actually likely to be used.

### Baby steps

First take a look at this commit

This is one of our assumptions (in between some other cruft): If you use attr_accessor etc., you are likely to need a vtable entry for that method.

Remember, we still don't implement a proper "fallback" in the form of a hash table for cases where the vtable gets "too big" (whatever we decide too big is) so for now we need this, but even later it'll be a useful optimization for many cases.

Worst case? We waste a vtable slot.

Next is where we actually add the basic implementations shown above, plus some debug statements, and stub for "define_method".

The comments bring us to an important question:

### To type-tag or not to type-tag?

In MRI Symbol objects are "type tagged" integers. That is, they are not real objects at all, rather each symbol is represented by a specific 32 bit value, and those values can be identified as symbols by looking for a specific bit-pattern in the least significant byte.

This has the advantage of saving space - no actual instances need to be constructed. In this instance, however, it creates a lot of complication, by requiring the type tags to be checked on each and every method call.

For this reason we will, at least for now, avoid it.

(For Fixnum we will run into this problem again, and it will be even more tricky - for Symbol the number of objects can be expected to be reasonably small, but what about Fixnum? Ugh... Lets think about that later)

Instead we will keep a hash table of allocated symbols, which we will use to return the same object for the same symbol literal

I'm going to skip over most of the commits here, and show you the current state of the Symbol class and its associated compiler changes, since I must admit my commits have been quite messy and all over the place while putting in place these changes, and it's not very condusive to explaining the actual changes.
```ruby
    class Symbol
      # Using class instance var instead of class var
      # because the latter is not properly implemented yet,
      # though in this case it may not make a difference
      #  @symbols = {} # FIXME: Adding values to a class ivar like this is broken
      
      # FIXME: Should be private, but we don't support that yet
      def initialize(name)
        @name = name
      end
    
      def to_s
        @name
      end

      # FIXME
      # The compiler should turn ":foo" into Symbol.__get_symbol("foo").
      # Alternatively, the compiler can do this _once_ at the start for 
      # any symbol encountered in the source text, and store the result.
    #  def self.__get_symbol(name)
    #    Symbol.new(name)       
    #    sym = @symbols[name]  
    #    if !sym               
    #      sym = Symbol.new(name)
    #    end
    #    sym
    #  end
    end

    def __get_symbol(name)
      Symbol.new(name)
    end
```
Uh, yes. I started out with actually having it turn symbols from text into an object at runtime, but I quickly realized this makes no sense for the case where a literal symbol is present in the program.

Note that we still need a proper Symbol#__get_symbol as commented out here later, because it is necessary to handle things like String#to_sym. However for now I've skipped it for a simple reason:

It requires implementing a hash table. It's not that hash tables are hard. But currently implementing Hash in pure Ruby likely will require features that are not in place yet... So, lets sort out the literal Symbols first, and then work our way up.

The trivial version above is well and good, but we need to make the compiler call __get_symbol..

So we replace Compiler#intern with this:
```ruby
    # Allocate a symbol
    def intern(scope,sym)
      # FIXME: Do this once, and add an :assign to a global var, and use that for any
      # later static occurrences of symbols.
      args = get_arg(scope,sym.to_s.rest)
      get_arg(scope,[:sexp,[:call,:__get_symbol, sym.to_s]])
    end
```
As you can see from my comment, it really makes no sense to call __get_symbol over and over - it's inefficient. But it works for now. We'll go back and fix that later.

Also note that this doesn't exactly match the above commit, but also incorporate a change from a later commit: We do [:sexp ..] there because :sexp nodes don't get rewritten to a :callm, and __get_symbol here is indeed a function not a method (this distinction doesn't really exist in Ruby, but it exist at the implementation level in our compiler, because it eases interoperability with C - this may change whenever I get to the point where it makes sense to implement FFI support)

The other changes are simply there to reflect the fact that intern now returns the result of a get_arg instead of some arbitrary integer:
```ruby
    -      return [:int,intern(name.rest)] if name[0] == ?:
    +      return intern(scope,name.rest) if name[0] == ?:
```
### That's it for now, folks

I promise it won't be nearly as long to the next part. I need to untangle my changes for splat operator support, method_missing debug improvements, and work on the Array and String classes. I'll likely give each one of them a short part each before I get fully "back on track" - I intend to write the future parts in parallel with actually making the code changes.
