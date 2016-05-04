# 解析；开始删除'runtime.c'

### The Operators - Part 1: Parsing

As mentioned last time, over the previous parts I at some point brutally broke our built in operators. I did so by actually implementing proper method calls.

It also made the compiler largely unusuable, and went unnoticed because, well, the compiler is still largely unusable.

So this time we'll add the built in operators back in. At the same time, we'll start the process of getting rid of that obnoxious vestige of C:

runtime.c

I originally hoped to cover this in one part, but it grew and grew, and you're going to have to endure 4 parts of this.

On the upside it represents a return to the low level / code generation focus, along with "threading" the changes through to the Ruby end, and actually adding tests (!).

There are a few things we need to do:

* Implement the minimum support in our s-expression syntax for basic integer arithmetic and comparison operators.
* Implement the very basics of the Ruby integer type hierarchy
* Implement the very basics of comparisons in Object
* Implement support for boolean operators (and,or) in the compiler

Originally, when we started out, anything the compiler didn't recognize was turned into a call to a function expected to exist and follow C calling convention.

This was convenient, as it allowed adding stuff like simple addition like this:
```ruby
    signed int add(signed int a, signed int b)
    { 
      return a + b;
    }
```
rather than having to figure out the code required to inline the addition directly.

However, Ruby has strange expectations of numbers: They are objects (well, this is not a strange expectation per se - everything is an object in Ruby), and you can add methods to their classes (but really, please, don't do that).

MRI ("Matz Ruby Interpreter" - the disambiguating nickname for the original Ruby, though if you've followed this series this far, you probably knew that) handles integers in part by type tagging, rather than turning them into "proper" objects.

That might be a good route for us to take too, eventually, but there's a ton of caveats to consider (not least that it means you can never just dereference an object pointer).

But for now, lets do the bare minimum (you haven't forgot I'm lazy, have you? defer elegance and finesse in favour of getting something working as quickly as possible).

What classes matter? Let's break out IRB:
```sh
    $ irb
    >> 1.class
    => Fixnum
    >> Fixnum.superclass
    => Integer
    >> Integer.superclass
    => Numeric
    >> Numeric.superclass
    => Object
```
Essentially we want to implement Numeric,Integer and FixNum and not much more. Eventually that will burn us as our integers will overflow and act weirdly, or someone tries to use a Float, but it's a distinct step up from what we have now.

So we will implement small bits of Numeric and Integer and FixNum in terms of a small number of s-expression operators, which we will then need to figure out how to compile.

We will blatantly disregard overflow issues and expect everyone to behave nicely and stay within the confines of a 32 bit signed integer for now... We will also implement just the bare minimum set of operators.

And we will add some simple test programs.

### The bare minimum

Our current runtime.c implements add,sub,div,mul, ne,eq,not,and,gt,ge,lt,le.

(Note that the "and" and "not" above are logical, not bitwise)

Numeric, Integer and Fixnum have quite a lot of methods, but we will for now implement only those which aids us in removing these.

Lets stub it out:
```ruby
    class Numeric
    end
    
    class Integer < Numeric
    end
    
    class Fixnum < Integer
      def + other
        ...
      end
      
      def - other
        ...
      end
      
      def <= other
        ...
      end
      
      def == other
        ...
      end
      
      def < other
        ...
      end
      
      def > other
        ...
      end
      
      def >= other
        ...
      end
      
      def div other
      end
      
      def mul other
      end
      
      # These two definitions are only acceptable temporarily,
      # because we will for now only deal with integers
      
      def * other
        mul(other)
      end
      
      def / other
        div(other)
      end
    end
```
We will expand on these with others that can be implemented in terms of the basics later, as well as consider which others might be worthwhile implementing as primitives.

So to recap: For now, we will

* Ignore the possibility of other being anything but a Fixnum
* Ignore overflows
* Generally take shortcuts as always, as long as we move forwards...

### Can we even parse this, and compile the result?

That's the first thing to verify, as we need to ensure the method names get suitably mangled to 1) pass through the C compiler, and 2) not clash with other potentially legal method names.

The first problem I ran into was that the test suite has gotten quite broken due to some cucumber change in the last couple of years, so that's fixed in 679ab6a, and I won't go over that change as it's simple to follow.

Secondly, while updating the tests I ran into a minor parser bug in (lack of) handling ';' as a expression terminator. That is fixed (at least to the extent of passing the currently relevant tests) in 0a8b33a.

And the answer is still 'no', so lets take a detour into debugging the parser.
```sh
    # ruby compiler.rb --norequire lib/core/fixnum.rb 
    reading from file: lib/core/fixnum.rb
    Missing value in expression / op: #<Oper:0xb74e4334 @pri=6, @assoc=:right, @arity=2, @minarity=2, @type=:infix, @sym=:assign> / vstack: [] / rightv: :other
    Failed at line 16 / col -1  before:
    
      end
      
      def < other
        
      end
```
The error messages can clearly need some love, but here's line 16 in lib/core/fixnum.rb at this stage:
```ruby
      def == other
```
### Tracking down a tokenizer bug

The output from the operator precedence part of the parser indicates it's processing an assignment operator, which it obviously isn't, so this looks superficially like a tokenizer bug. As good a time as any to update our test suite.

To faciliate this I first updated the Rakefile to include a new target 'failing' that only outputs the failing features (in 48d1382)

I then added a basic test or `==` in e347ce4, but this passed so the problem does not appear to be in the operator precedence parser.

In 840262f I then added a basic test for `==` as a method name, and it fails as expected. Instrumenting parse_fname which is used to parse the name of a definition, reveals that the offender actually appears to be the operator tokenizer, that actually does not recognize `==`

Before going on to add tests, looking at the code reveals a fairly obvious problem:
```ruby
      # fname ::= name | "[]"
      
      def parse_fname
        name = expect("[]") || expect(Methodname) || expect(Oper)
        name.is_a?(String) ? name.to_sym : name
      end
```
expect(Oper) leads us to this code in operators.rb:
```ruby
      def self.expect(s)
        # expect any of the defined operators
        # if operator found, return it's symbol (e.g. "*" -> :*)
        # otherwise simply return nil,
        # as no operator was found by scanner
    
        Operators.keys.each do |op|
          if s.expect(op)
            return op.to_sym
          end
        end
        return nil
      end
```
Spotted the problem yet? How about with this excerpt from the Operators hash:
```ruby
      "="         => Oper.new(  6, :assign,   :infix),
    ...
      "=="        => Oper.new(  9, :eq,       :infix),
```
Oops. Oper.expect scans the operators linearly, without taking into account length. It is both horribly inefficient, and flat out broken since any cases where a short token gets inserted prior to a longer token with the shorter token as a prefix will fail.

This is one of those instances where laziness and lack of tests does come back to bite us later.

So lets add some tests (added in 6198c0a):
```
    Feature: Tokenizers
        In order to tokenize programs, a series of tokenizer classes provides
        components that can tokenize various subsets of Ruby tokens.
    
        @operators
        Scenario Outline: Operators
            Given the expression <expr>
            When I tokenize it with the Oper tokenizer
            Then the result should be <result>
    
        Examples:
          | expr                   | result                     |
          | "="                    | :"="                       |
          | "=="                   | :"=="                      |
```
You know what? This test passes... Lets check something manually:
```ruby
    # ruby -roperators -e 'p Operators.keys'
    ["do", "..", "::", "+", "==", "<<", "||", "&&", ",", "!", "<=", "=>", "and", "#index#", "#call#", "-", ".", "or", "/", ":", "#,#", "#flatten#", "{", "[", "-=", "!=", "<", "&", "}", "]", "<=>", "+=", "||=", "=", "#block#", "(", ">", ")", "#hash#", ">=", "?", "return", "*"]
```
So the code in Oper.expect is almost certainly broken, but it's not actually the problem tripping us up in this case. Lets add some more cases to the new tokenizer.feature anyway. In c2360cf I've added a couple more test cases, including one for "+=" which illustrates the problem with Oper.expect perfectly. We'll leave actually fixing it for later, since it's not what's breaking our other test case.

Some more instrumenting to print out the result of @scanner.get in parse_fname reveals the real source of the problem: Methodname.expect does not correctly rewind the scanner state, and/or reads the wrong thing. (Arguably it would also be cleaner if Methodname did what it says on the tin and actually tokenized method names fully rather than only handle parts of them...)

So lets add some more test cases (added in 767de38 and 95addac)
```
        @methodname
        Scenario Outline: Method names
            Given the expression <expr>
            When I tokenize it with the Tokens::Methodname tokenizer
            Then the result should be <result>
    
        Examples:
          | expr                   | result   |
          | "="                    | nil      |
          | "=="                   | nil      |
```
We expect nil because despite the comment above we don't handle operators in this tokenizer class. So why does it return :"=" in both cases?

Turns out MethodName.expect checks for valid method name suffixes, including = without checking that they occur after a valid Atom:
```ruby
         pre_name = s.expect(Atom)
         suff_name = MethodEndings.select{ |me| s.expect(me) }.first
```
This is fixed in 6906426.

And that concludes our detour into bug fixing for now, you might think. Not so. Rerunning the parser:
```
    Missing value in expression / op: #<Oper:0xb7481388 @pri=6, @assoc=:right, @arity=2, @minarity=2, @type=:infix, @sym=:assign> / vstack: [] / rightv: :other
    Failed at line 28 / col -1  before:
```
This time, though, we have an easier time. It is &gt;= it chokes on this time, so we add it to the tokenizer tests in 053052b. Unsurprisingly, this is now Oper.expect coming back to bite us. So it's good news, bad news: We've already tracked down the problem, but we actually have to fix it.

The naive/trivial fix is to simply sort the keys in Oper.expect:
```ruby
    --- a/operators.rb
    +++ b/operators.rb
    @@ -36,7 +36,7 @@ class Oper
         # if operator found, return it's symbol (e.g. "*" -> :*)
         # otherwise simply return nil,
         # as no operator was found by scanner
    -    Operators.keys.each do |op|
    +    Operators.keys.sort_by {|op| -op.to_s.length}.each do |op|
           if s.expect(op)
             return op.to_sym
           end
```
This works and passes the tests, and gets us to a state where we can parse the Fixnum stub. It's obviously not very efficient, but then again linearly checking the operators hash was never efficient to begin with. So let's go with that for now, but add a suitable "FIXME" to explain the rationale (in 4fdd8d4)

### Preventing method name breakage

While the code now parses, we face a second problem: The method names causes the output to become invalid. E.g:
```asm
        __method_Fixnum_==:
                    .stabn  68,0,16,.LM20 -.LFBB8
            .LM20:
            .LFBB8:
                    pushl   %ebp
                    movl    %esp, %ebp
                    subl    $20, %esp
                    addl    $20, %esp
                    leave
                    ret
                    .size   __method_Fixnum_==, .-__method_Fixnum_==
```
Let's just say that gas and most other sane assemblers would take issue with labels that include "==" and other operators. Even my syntax highlighting chokes.

For robustness we should do some general escaping of method names with symbols. So far we've gotten away with just these very specific fixups:
```asm
      # Need to clean up the name to be able to use it in the assembler. 
      # Strictly speaking we don't *need* to use a sensible name at all,
      # but it makes me a lot happier when debugging the asm.
      def clean_method_name(name)
        cleaned = name.to_s.gsub("?", "__Q") # FIXME: Needs to do more.
        cleaned = cleaned.to_s.gsub("[]", "__NDX")
        cleaned = cleaned.to_s.gsub("!", "__X")
        return cleaned
      end
```
Lets make this more generic:

First we formulate a test (in df6ae1a - oops, forgot to commit it until after I actually fixed the method, but I did write it first, I promise). You will find it in test/compiler.rb. I like Cucumber a lot for the tests that require long tables of examples, or complex scenarios of reusable steps, but for tests like these that are very specialized and one-off, I prefer rspec:
```ruby
    require 'compiler'
    
    describe Compiler do
    
      describe "#clean_method_name" do
    
        it "should escape all characters outside [a-zA-Z0-9_]" do
          input = (0..255).collect.pack("c*")
          output = Compiler.new.clean_method_name(input)
          expect(output.match("([0-9a-zA-Z_])+")[0]).to eq(output)
        end
      end
    end
```
This test simply state that we expect the output string matched against [0-9a-zA-Z_]+ to be equal to the input string. The regexp capture will exclude any characters outside our desired range, so if any nasty characters are left, they'll fail. We don't nail down the actual specific format beyond that.

This simple change will accommodate that:
```ruby
      # Need to clean up the name to be able to use it in the assembler.
      # Strictly speaking we don't *need* to use a sensible name at all,
      # but it makes me a lot happier when debugging the asm.
      def clean_method_name(name)
        cleaned = name.to_s.gsub("?", "__Q") # FIXME: Needs to do more.
        cleaned = cleaned.to_s.gsub("[]", "__NDX")
        cleaned = cleaned.to_s.gsub("!", "__X")
        cleaned = cleaned.split(//).collect do |c|
          if c.match(/[a-zA-Z0-9_]/)
            c
          else
            "__#{c[0].to_s(16)}"
          end
        end.join
        return cleaned
      end
```
Though this is suboptimal for debugging, so I'm adding a small array of exceptions (in 656921a):
```ruby
      def clean_method_name(name)
        dict = {
          "?" => "__Q",     "!"  => "__X",
          "[]" => "__NDX",  "==" => "__eq",
          ">=" => "__ge",   "<=" => "__le",
          "<"  => "__lt",   ">"  => "__gt",
          "/"  => "__div",  "*"  => "__mul",
          "+"  => "__plus", "-"  => "__minus"}
    
        cleaned = name.to_s.gsub />=|<=|==|[\?!=<>+\-\/\*]/ do |match|
          dict[match.to_s]
        end
    
        cleaned = cleaned.split(//).collect do |c|
          if c.match(/[a-zA-Z0-9_]/)
            c
          else
            "__#{c[0].to_s(16)}"
          end
        end.join
        return cleaned
      end
```
(This could be done more cleanly with Ruby 1.9+ #gsub, but I'm being a luddite and still support 1.8.x since I have it installed all over the place, so sue me)

This finally gives us acceptable labels.

### Using the new Fixnum

The next problem we run across is that numeric constants in our compiler so far have not been objects, but actual numbers. There are two problems here:

* Creating Fixnum objects similar to how we create String's or Symbol's.
* Ensuring we extract the "real" value when trying to use the numbers.

Like with String, rather than try to copy MRI's type-tagging, we will create an action class

We'll tackle that in the next part.