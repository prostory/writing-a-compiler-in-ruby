# 将数字转换成Numbers
（或至少把足够小的整数转换成Fixnum）

As a recap from last time, where we left off we can parse our new, completely useless and primitive Numeric, Integer and FixNum classes, but our numbers aren't actually members of those classes, but raw integers. So we need to do some magic like we do for constant strings:

So lets start by looking at an excerpt of how we do this in String (with the caveat that this is not an ideal solution - as with so many of our implementation choices, I refer back to the principles listed in part 1):
```ruby
    class String
      def initialize
        # @buffer contains the pointer to raw memory
        # used to contain the string.
        #
        # An s-expression is used rather than = because
        # 0 outside of the s-expression eventually will
        # be an FixNum instance instead of the actual
        # value 0.
        %s(assign @buffer 0)
      end
    
      def __set_raw(str)
        @buffer = str
      end
    
      def __get_raw
        @buffer
      end
      
      ...
    end
```
This first part takes care of setting aside a buffer for the string, and then assigning raw memory for it as needed, and allowing the compiler to get at it. I'm not happy with this method, but it works.

For our FixNum version, the instance variable will instead hold the actual value we want to operate on.

We go on to do this at the very end of lib/core/string.rb:
```ruby
    %s(defun __get_string (str) (let (s)
      (assign s (callm String new))
      (callm s __set_raw (str))
      s
    ))
```
This encapsulates the step needed to turn a string constant into an object for now (in reality we'll probably want to do copy on write and avoid creating duplicate objects for the same actual string constant). For FixNum this gets complicated, as Ruby explicitly considers numeric constants with the same value to be the same object.

Compare irb output for two string constants (or rather the objects created from them) vs to numeric constants:
```sh
    >> "foo".object_id
    => 70161706007580
    >> "foo".object_id
    => 70161706000000
    >> 1.object_id
    => 3
    >> 1.object_id
    => 3
```
(you might wonder why MRI returns 3 as the #object_id of 1 in this example - it might seem natural to try to use the actual integer values; this is an implementation detail that is presumably down to the use of the use of type tagging )

#### Creating actual FixNum instances

So lets apply this approach to create proper objects for now (we will revisit the type-tagging approach at some - likely far later - date).

The change is quite simple, though this violates the requirement that we return the same object for every instance of the same constant. We'll want to fix that later, or at least ensure the observable effect is the same.

In 2cccb7f:
```ruby
    diff --git a/lib/core/fixnum.rb b/lib/core/fixnum.rb
    index 46f7c0a..49073a7 100644
    --- a/lib/core/fixnum.rb
    +++ b/lib/core/fixnum.rb
    @@ -1,6 +1,18 @@
     
     class Fixnum < Integer
     
    +  def initialize
    +    %s(assign @value 0)
    +  end
    +
    +  def __set_raw(value)
    +    @value = value
    +  end
    +
    +  def __get_raw
    +    @value
    +  end
    +
       def + other
     
       end
    @@ -47,3 +59,10 @@ class Fixnum < Integer
       end
       
     end
    +
    +
    +%s(defun __get_fixnum (val) (let (num)
    +  (assign num (callm Fixnum new))
    +  (callm num __set_raw (val))
    +  num
    +))
```
##### A minor digression about efficiency

Let me make an observation here: There's a major inefficiency we can address relatively easily here, which we as usual will ignore for the time being. Namely, by actually writing real Ruby in the __get_raw and __set_raw methods, we're confined to real instance variables. This is bad, because we do not have direct control over which slot in the object it gets.

This forces us to actually use __get_raw and __set_raw rather than inline the indirect lookup. Now, we could still do various hacks, such as figure out the offset (which will be the same for all objects of this class) and store it somewhere, and inline code to look up that, but that is still wasteful.

Barring some asshat deciding to override basic operators (which does work in MRI at least, though it even makes irb fail horribly unless you're very careful), we could instead decide on a static instance variable slot, and inline code to refer to that slot. We can pretty much guarantee that @value will be the first instance variable in the object, and so we can pretty much know the offset, but it's brittle - it takes only incidental changes to the compiler code, or re-ordering of the code in lib/core/fixnum.rb to break this, so we won't go there for now as a proper variation of this will involve ensuring we can guarantee a specific vtable slot, which isn't hard but still adds complexity we don't need at this point.

Even better: If we didn't have that pesky class pointer, we wouldn't need to provide an offset at all - we could store it at the address the object points to. (And there we are at type-tags, though type-tags adds the cost of masking out and checking the tag)

##### Back to the action

We now need to change the compiler to rewrite all integer constants outside [:sexp] blocks with calls to __get_fixnum.

First we add a test case to catch this transformation (in c0acf92):
```
    Feature: Transformations
      In order to implement the correct semantics, the compiler post-processes
      the output from the Parser to modify the AST.
    
      Scenario Outline: Simple expressions
        Given the expression <expr>
        When I parse it with the full parser
        And I preprocess it with the compiler transformations
        Then the parse tree should become <tree>
    
        Examples:
        | expr    | tree                                                                               | notes |
        | "1 + 2" | [:do,[:add, [:sexp,[:call, :__get_fixnum, 1]], [:sexp,[:call, :__get_fixnum, 2]]]] |       |
```
This test case should initially fail badly:
```
        Examples: 
          | expr    | tree                                                                               | notes |
          | "1 + 2" | [:do,[:add, [:sexp,[:call, :__get_fixnum, 1]], [:sexp,[:call, :__get_fixnum, 2]]]] |       |
          expected: [:do, [:add, [:sexp, [:call, :__get_fixnum, 1]], [:sexp, [:call, :__get_fixnum, 2]]]]
               got: [:do, [:add, 1, 2]] (using ==) (RSpec::Expectations::ExpectationNotMetError)
          ./step_definitions/shunting_steps.rb:32:in `/^the parse tree should become (.*)$/'
          ./transform.feature:10:in `Then the parse tree should become <tree>'
```
So lets implement the rewrite. We'll do that by pretty much rewriting rewrite_strconst in transform.rb. This function tries to use a pool of string constant data. Here we don't need that, as the values are small (though we in the future will need to reconcile the issue of singleton objects, that is not addressed by the method used in rewrite_strconst anyway). What I ended up with for now is this (in 6c30e87):
```ruby
      # Rewrite a numeric constant outside %s() to
      # %s(call __get_fixnum val)
      def rewrite_fixnumconst(exp)
        exp.depth_first do |e|
          next :skip if e[0] == :sexp
          is_call = e[0] == :call
          e.each_with_index do |v,i|
            if v.is_a?(Integer)
              e[i] = E[:sexp, E[:call, :__get_fixnum, v]]
    
              # FIXME: This is a horrible workaround to deal with a parser
              # inconsistency that leaves calls with a single argument with
              # the argument "bare" if it's not an array, which breaks with
              # this rewrite.
              e[i] = E[e[v]] if is_call && i > 1
            end
          end
        end
      end
```
It's not very DRY, but we'll revisit that later as we will run into this same issue for Float as well. Note that Symbol has gotten special treatment - it is handled in Compiler#intern. Why? Symbol's have no "native" representation, and so we don't need to protect it within %s() blocks to be able to implement the core functionality, that is why. Though this lack of symmetry is not very pleasing. Some more thought is needed here.

So now we generate the calls, and we have a Fixnum class. But if we look at our test case above, this is still broken.

### Converting operators to use method calls

We really need it to look like is roughly this:
```lisp
    (callm (call __get_fixnum 1) + (sexp (call __get_fixnum 2))))
```
(And this is where it starts to become horribly obvious why making Ruby fast is hard)

This is what 1.+(2) should look like, and 1+2 is just syntactic sugar for 1.+(2).

Now there's reason to suspect trouble ahead - will our parser like this? (no, not at all, is the reasonable answer at this point...) So let's start by adding this test case to the parser.feature to fix this case at the same time (though we could ignore it and "just" handle the 1 + 2 form at this point (in 1a0988c):
```
        @operatorcalls
        Scenario Outline: Operator method calls
            Given the expression <expr>
            When I parse it with the full parser
            Then the parse tree should become <tree>
    
        Examples:
          | expr     | tree                      | notes |
          | "1.+(2)" | [:do, [:callm, 1, :+, 2]] |       | 
```
(Remember, this is before rewrites, so we can't expect to get __get_fixnum etc.)
```
        Examples:
          | expr     | tree                      | notes |
          | "1.+(2)" | [:do, [:callm, 1, :+, 2]] |       |
          Missing value in expression / op: #<Oper:0xb7136df4 @minarity=2, @sym=:callm, @pri=100, @assoc=:left, @arity=2, @type=:infix> / vstack: [] / rightv: 1 (RuntimeError)
          /home/vidarh/src/compiler-experiments/writing-a-compiler-in-ruby/treeoutput.rb:33:in `oper'
          /home/vidarh/src/compiler-experiments/writing-a-compiler-in-ruby/shunting.rb:28:in `reduce'
          /home/vidarh/src/compiler-experiments/writing-a-compiler-in-ruby/shunting.rb:80:in `shunt'
```
Uh-oh. Presumably it's assuming the + is an unary plus, not a method name. This is a tricky one, as actually, if there is no whitespace after '.', then names that are normally operators are actually method names. But we don't normally bubble whitespace up to the parser components.

We can special-case in the Tokenizer, or we will need more information in the operator precedence parser. For now we opt for doing naughty stuff to the Tokenizer.

This is the meat of the change (in git: b58da12):
```ruby
    diff --git a/tokens.rb b/tokens.rb
    index 805cdf2..0e355a1 100644
    --- a/tokens.rb
    +++ b/tokens.rb
    @@ -93,7 +93,16 @@ module Tokens
       # A methodname can be an atom followed by one of the method endings
       # defined in MethodEndings (see top).
       class Methodname
    +    # Operators that are allowed as method names
           +    OPER_METHOD = %w{=== []= [] == <=> <= >= ** << >> != ! = ! ~ + - * / % & | ^ < >}
    +
         def self.expect(s)
    +      # FIXME: This is horribly inefficient.
    +      name = nil
    +      OPER_METHOD.each do |op|
    +        return name.to_sym if name = s.expect(op)
    +      end
    +
           pre_name = s.expect(Atom)
           if pre_name
             suff_name = MethodEndings.select{ |me| s.expect(me) }.first
    @@ -359,11 +368,20 @@ module Tokens
     
         def get
           @lasttoken = @curtoken
    -      @lastop ? @s.ws : @s.nolfws
    -      @lastop = false
    -      @lastpos = @s.position
    -      res = get_raw
    +
    +      if @last.is_a?(Array) && @last[1].is_a?(Oper) && @last[1].sym == :callm
    +        @lastop = false
    +        @lastpos = @s.position
    +        res = Methodname.expect(@s)
    +        res = [res,nil] if res
    +      else
    +        @lastop ? @s.ws : @s.nolfws
    +        @lastop = false
    +        @lastpos = @s.position
    +        res = get_raw
    +      end
           # The is_a? weeds out hashes, which we assume don't contain :rp operators
    +      @last = res
           @lastop = res[1] && (!res[1].is_a?(Oper) || res[1].type != :rp)
           @curtoken = res
           return res
```
The first part you see is rewriting the Methodname tokenizer to actually get closer to tokenizing valid method names. We should probably just bite the bullet on this one soon and add test cases to make it match the full Ruby grammar. As commented, the added method is rather inefficient, but it will check for the list of "operators-allowed-as-methods". Previously the parser actually tried to parse the full set of operators for def, which includes some operators like = that can't be overridden (because it doesn't make any sense - = is not an operation on the object, but on the variable that stores the reference to the object).

The latter part modifies Tokenizer#get to keep track of the last token parsed, and expect a Methodname after a '.' operator. I actually haven't stopped to check if there are any important exceptions to this.. We'll find out in due time.

This also slightly simplifies the parser, where we've replaced calls to parse_fname with simply expect(Methodname).

Because of this change a minor adjustment to the tokenizer test cases was also necessary.

This means we can now handle the "un-sugared" operator method calls. But what about 1 + 2?

Let's amend the test case from earlier to expect a :callm node:
```ruby
        | "1 + 2" | [:do,[:callm, [:sexp,[:call, :__get_fixnum, 1]], :+ , [:sexp,[:call, :__get_fixnum, 2]]]] |       |
```
This will of course fail, since we got the previous version working.

We will handle this partially via tree re-writes, partially by updating the operator precedence parser to return the operators themselves rather than a sanitized name. The tree-rewrite then is just a simple change from [:some-oper, arg1, arg2] to [:callm, arg1, :some-oper, arg2].

The first step is to clean up a number of the tests, which I won't include here as they're fairly trivial replacements of :add with :+ etc. You can find it in 93d98a8

Secondly, lets update operators.rb accordingly. E.g.:
```ruby
     "+"         => Oper.new( 10, :+,      :infix),
```
See 07cab35 for the full details of this change.

We then change our test case in features/transform.feature to account for this and the planned tree rewrite:
```ruby
    -    | expr    | tree                                                                               | notes |
    -    | "1 + 2" | [:do,[:add, [:sexp,[:call, :__get_fixnum, 1]], [:sexp,[:call, :__get_fixnum, 2]]]] |       |
    +    | expr    | tree                                                                                      | notes |
    +    | "1 + 2" | [:do,[:callm, [:sexp,[:call, :__get_fixnum, 1]], :+ , [:sexp,[:call, :__get_fixnum, 2]]]] |       |
```
This change will now fail, so lets sort out that tree rewrite (in 1c198e4):
```ruby
    +  # Rewrite operators that should be treated as method calls
    +  # so that e.g. (+ 1 2) is turned into (callm 1 + 2)
    +  #
    +  def rewrite_operators(exp)
    +    exp.depth_first do |e|
    +      next :skip if e[0] == :sexp
    +
    +      if OPER_METHOD.member?(e[0].to_s)
    +        e[3] = e[2]
    +        e[2] = e[0]
    +        e[0] = :callm
    +      end
    +    end
    +  end
```
We here check whether the first element in the expression is one of the operator methods. We then move the second operand to its new position, then the operator, we don't need to do anything with the first operator, and then last we set the first element to :callm. It might be slightly cleaner-looking to pull the values out into temporary variables an overwrite the whole thing, but we're only "juggling" three values here... (though do be aware of the order of assignments)

### Next time.

This is where we leave it for this time. We can now parse and transform the source so that we remove the syntactic sugar added for operators. The next part will wrap up the operator handling by showing the code we put in place to actually implement a couple of the operators.