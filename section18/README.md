# 插入运算符优先级分析器

### Plugging in an operator precedence parser

The code as at the end of this part is available here.

A while back I wrote a post about writing a simple operator precedence parser. I did that with this in mind. Instead of writing, and maintaining a set of grammar rules for each type of expression involving operators, and implement them, I've plugged in an operator precedence parser instead. You may want to refer back to that post - I won't go into detail about this parser component itself since most of the inner workings is covered there, so this will be another short part and will rely heavily on you having read that post.
To illustrate why I prefer an operator precedence parser, lets look at a small grammar snippet that gives an idea of how it would be implemented in a recursive descent parser. 
```
expr    ::= mulexpr (("+"|"-") mulexpr)*
mulexpr ::= primary (("*"|"/") primary)*
primary ::= number | "(" expr ")"
```
Now this looks simple, and it is: The looser an operator binds, the closer to the "top" of the expression it is, so the parser will look for tighter binding operators first, by following the rules down. 

But it gets tedious, especially when you convert it to code by hand. More importantly you need to add this to the grammar for each and every priority level in the grammar. With an operator precedence parser, you instead get a table of priorities - changing how the operator binds is just a matter of changing that table. Lets look at the table for my parser as it currently stands:
```ruby
    Oper = Struct.new(:pri,:sym,:type)
    
    Operators = {
      ","  => Oper.new(2,  :comma,  :infix),
    
      "="  => Oper.new(6,  :assign, :infix),
    
      "&lt;"  => Oper.new(9,  :lt,     :infix),
      ">"  => Oper.new(9,  :gt,     :infix),
      "==" => Oper.new(9,  :eq,     :infix),
      "!=" => Oper.new(9,  :ne,     :infix),
    
      "+"  => Oper.new(10, :add,    :infix),
      "-"  => Oper.new(10, :sub,    :infix),
      "!"  => Oper.new(10, :not,    :prefix),
    
      "*"  => Oper.new(20, :mul,    :infix),
      "/"  => Oper.new(20, :div,    :infix),
      
      "["  => Oper.new(99, :index,  :infix),
      "]"  => Oper.new(99, nil,     :rp),
      
      "("  => Oper.new(99, nil,     :lp),
      ")"  => Oper.new(99, nil,     :rp)
    }
```
Notice how "," is an operator - this is to handle function calls without any special handling in the shunting yard class (instead handling it by flattening the tree in the TreeOutput class) - adding special handling and avoiding any special checks in the TreeOutput class would also work. I didn't do any real analysis to find out which is best.

If you wanted to, you could mechanically convert this table to grammar fragments like above fairly easily, and automatically generating the parser code wouldn't be hard either. But there's not really much point - as my earlier post showed, implementing the Shunting Yard algorithm for an operator precedence parser is fairly simple, and in this case it drops straight, and adding a new operator to the parser is more or less a matter of adding to this table.

The next big chunk that needed to be added was a tokenizer, as the shunting yard class presented in the earlier mentioned post requires a stream of tokens rather than characters. The tokenizer needs to recognize the type of token, and then attempt to read it from the scanner. Here's the tokenizer as it stands at the moment:
```ruby
  class Tokenizer
    def initialize scanner
      @s = scanner
    end

    def each
      while t = get
        yield t
      end
    end

    def get
      @s.nolfws
      case @s.peek
      when ?"  
        return @s.expect(Quoted)
      when ?0 .. ?9
        return @s.expect(Int)
      when ?a .. ?z , ?A .. ?Z
        buf = @s.expect(Atom)
        if (buf == :end || buf == :def) # FIXME: Make this a keyword lookup                                                                                
          @s.unget(buf.to_s)
          return nil
        end
        return buf
      # Special cases - two character operators:                                                                                                           
      when ?=
        @s.get
        return "==" if @s.peek == ?=
        return "="
      when ?!
        @s.get
        return "!=" if @s.peek == ?=
        return "!"
      when nil
        return nil
      else
        return @s.get if Operators[@s.peek.chr]
        return nil
      end
    end
  end
```
Overall it's pretty simple. The one thing worth really paying attention to is how "end" and "def" are treated as terminating an expression. This is an important issue with operator precedence parser intermingled with something else: You need to take care to identify the tokens that means you've reached the end of an expression, and make sure they're not swallowed up by the tokenizer.

The main change from the shunting yard implementation in the earlier post is a slight change to TreeOutput to specifically handle the :call syntax used in the compiler:
```ruby
   def oper o
      rightv = @vstack.pop
      raise "Missing value in expression" if !rightv
      if (o.sym == :comma) && rightv.is_a?(Array) && rightv[0] == :comma
        # This is a way to flatten the tree by removing all the :comma operators
        @vstack << [o.sym,@vstack.pop] + rightv[1..-1]
      elsif (o.sym == :call) && rightv.is_a?(Array) && rightv[0] == :comma
        # This is a way to flatten the tree by removing all the :comma operators
        @vstack << [o.sym,@vstack.pop,rightv[1..-1]]
      else
        if o.type == :infix
          leftv = @vstack.pop
          raise "Missing value in expression" if !leftv
          @vstack << [o.sym, leftv, rightv]
        else
          @vstack <<  [o.sym,rightv]
        end
      end
    end
```
Specifically the "elsif" arm which is added to make sure the node comes out as [:call,function name,[arg1,arg2,...]]. We could have output [function name, arg1, arg2] instead, but that has a distinct disadvantage: You could have a function name that would collide with a primitive built into the compiler, and there'd be no end of confusing bugs. Forcing it into the [:call, function name, [args]] style prevents that from occurring.

As for the s-expression parser, this parser component is tied into the main parser by instantiating it and referencing it from a member variable by adding this to Parser#initialize: " @shunting = OpPrec::parser(s)". As an example of how the main parser is changed, consider #parse_condition:
```ruby
  # condition ::= sexp | opprecexpr                                                                                                                                     
  def parse_condition
    @sexp.parse || @shunting.parse
  end
```
At the end of this, the "testargs" example now looks almost like Ruby, apart from the "numargs" call, though only superficially so (remember - the language is still untyped; there are no objects or even type information at all; we'll get to a type system and object orientation soon, though we'll start with something much simpler than Ruby):
```ruby
def f test, *arr
  i = 0
  while i < (numargs - 1)
    printf("test=%ld, i=%ld, numargs=%ld, arr[i]=%ld\n",test,i,numargs,arr[i]))
    i = i + 1
  end
end

def g i, j
  k = 42
  printf("numargs=%ld, i=%ld,j=%ld,k=%ld\n",numargs,i,j,k)
end

f(123,42,43,45)
g(23,67)
```
Note that the operator table above is woefully incomplete - it doesn't even cover what our "runtime library" include. I'll add to it gradually.