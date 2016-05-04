# 一个简单的运算符优先级分析器

Consider this an "interlude" of sorts in my series on writing a compiler, since the parser that will be presented in one of the future parts of that series depends significantly on an operator precedence parser to cut down on the amount of code.

### Prologue

When I was 12 or 13 I was rifling through discounted books in some book store, and I found a book on Prolog. I was deeply fascinated by different programming languages at the time, so I picked it up. It gave a lot of interesting examples, including how to do symbolic differentiation. I found it interesting how much manipulation you could do that way very easily in Prolog, but I didn't have a Prolog handy.

Instead I decided to try to replicate the example in Pascal (I was using a Amiga at the time, and the C compilers were dog slow, but I had HiSoft Pascal, which was fast and with an IDE). So I had to write an expression parser to get things into tree form first.

If you studied computer science, you've probably done something similar several times - it's commonly done in introductory algorithms courses. It's not hard, and there's a large number of different ways of doing it, one of the most commonly presented ones being Dijkstra's Shunting Yard algorithm to turn an infix expression into reverse Polish notation.

Not having any textbooks handy, however, meant I had to figure it out myself. What I stumbled upon was a simple operator precedence parser, quite close to the Shunting Yard algorithm, as it's a quite natural approach to take if you sit down and just go walk the problem with a few examples. Originally it was a recursive beast - the inspiration I was drawing from the Prolog book didn't help. The code got real messy. Eventually I sat down, drew a flowchart and figured out how to clean it up. I was very happy about it. Especially when I got the symbolic differentiation working (I fully believe that writing the code to do it taught me far more about it than practicing on examples did at school; though it also made me think it was pointless to memorize the rules).

As it always does when I start a hobby project, it took off in all kinds of directions, and before I knew it it had started morphing into my first compiler - a bizarre mix of inline assembler and a "Pascal with braces" where variables in expressions could be substituted directly for M68000 registers. In some ways it was reminiscent of Randall Hyde's High Level Assembly though it certainly never got as polished as HLA.

But that operator precedence parser was the start of my compiler interest. For all intents and purposes this was a variation of Dijkstra's algorithm with the only real difference being that I build a tree instead of outputting in RPN, which I finally realized when I about 5-6 years later started university and it showed up in my algorithm course first semester. D`oh.

It was a learning experience worth having, though - I know that algorithm a lot better than a textbook would've taught it to me.

Below is a walk through of building a fairly generic Shunting Yard implementation for Ruby with pluggable tokenizers and output generators. The difference between generating RPN vs. a tree structure is minimal.

### What is an operator precedence parser and why should I care?

First of all, consider what operator precedence is:
```ruby
1 + 2 * 3
```
Does this mean (1 + 2) * 3 or 1 + (2 * 3)?

Most would agree it means the latter. "*" has higher precedence than "+".

An Operator Precedence Parser (hereafter I'll refer to it as an OPP) is a parser that uses operator precedence rules to parse an expression instead of lists of valid productions. More specifically, it is most often used about a parser that is not hardwired in code for a specific set of operator precedences - it is typical to write an OPP that runs entirely of a small table with information about the operators.

The latter is what makes them interesting. Manually writing parsers is tedious. Especially when the patterns repeat over and over again, as they do for expressions that follows a very regular grammar. So if you do want to use a manually written parser for whatever reason, an OPP is often a good way of making the parser compact and easy to extend or modify, as simple modifications to the grammar can be done just by amending a table of operator precedences and types.

Fairly large languages can use OPP's. My first compiler was eventually, after starting out with a mix of a standard recursive descent parser and an OPP, driven entirely by an operator precedence parser with some unusual extensions (support for prefix and postfix operators with multiple operands, for example). Statement delimiters were operators. Start / end of blocks (using curly baces) were treated as special types of parentheses that translated into a special "block" operator, and square brackets were treated the same way for array index. "."'s in method calls were operators. Method names were operators that took an argument list as a parameter. The argument list was a list of expressions combined by the "," operator. Even if/while etc. where operators.

I wouldn't recommend going that far usually - unless the language is very regular and very limited in number of constructs it can get quite hard to pick the correct priorities and other flags. That said, it did mean the printout of the parser excluding the tokenization fit on less than two sheets of paper. It also worked because the language was designed to make it work - today I don't like having to separate statements with ';' any more, for example, and like keeping the use of parentheses and braces down after having gotten used to Ruby.

### Problems with Operator Precedence Parsers

OPP's in themselves work on context free grammars. "*" has the same properties no matter where it's found.

In many languages that is not sufficient. A typical example is "a * b" in C, which can be a multiplication or a pointer declaration.

This can sometimes be worked around, but not without additional information provided by the parser. In the C example, a symbol table lookup is required on "a" (which also means the symbol table must be build during parsing). For an OPP, the specific type of the following operator must then be adjusted based on the type of the preceding operand. That gets ugly pretty quickly.

For parsers where context is required many places, an OPP is often not an appropriate choice, as a lot of the simplicity that makes them attractive gets lost in working around context issues. A "workaround" may be to restrict the use of the OPP to subsets of the language that are context free, but this creates issues of it's own; specifically the issue of identifying the boundary between the part of the source text that should be parsed by the OPP and the rest. In some languages this is easy (an expression might always be terminated by a line-feed, a semicolon or the token 'end' for example), but that's not a general rule.

### Bring on the code

It is hard to write a totally generic shunting yard implementation for the simple reason that so many rules will depend on the specifics of your grammar. The code is however small enough that cut and past and adapting to a specific solution is perhaps more appropriate for many types of adaptation than complicating the code with a lot of knobs to tune behavior with. Here's a reasonably feature full version:
```ruby
    Oper = Struct.new(:pri,:sym,:type)
    
    class ShuntingYard
      def initialize opers,output
        @ostack,@opers,@out = [],opers,output
      end
    
      def reduce op = nil
        pri = op ? op.pri : 0
        # We check for :postfix to handle cases where a postfix operator has
        # been given a lower precedence than an infix operator, yet it needs
        # to bind tighter to tokens preceeding it than a following infix operator
        # regardless, because the alternative gives a malformed expression.                                                                                      
        while  !@ostack.empty? && (@ostack[-1].pri > pri || @ostack[-1].type == :postfix)
          o = @ostack.pop
          return if o.type == :lp
          @out.oper(o)
        end
      end
    
      def shunt src
        possible_func = false     # was the last token a possible function name?                                                                    
        opstate = :prefix         # IF we get a single arity operator right now, it is a prefix operator                                            
                                  # "opstate" is used to handle things like pre-increment and post-increment that                                   
                                  # share the same token.                                                                                           
        src.each do |token|
          if op = @opers[token]
            op = op[opstate] if op.is_a?(Hash)
            if op.type == :rp then reduce
            else
              opstate = :prefix
              reduce op # For handling the postfix operators                                                                                        
              @ostack << (op.type == :lp && possible_func ? Oper.new(1, :call, :infix) : op)
            end
          else
            @out.value(token)
            opstate = :infix_or_postfix # After a non-operator value, any single arity operator would be either postfix,                            
                                        # so when seeing the next operator we will assume it is either infix or postfix.                            
          end
          possible_func = !op && !token.is_a?(Numeric)
        end
       reduce
    
        return @out if  @ostack.empty?
        raise "Syntax error. #{@ostack.inspect}"
      end
    end
```
Let's walk through it bit by bit:

#### The #initialize method

@ostack is our operator stack. We'll push any operators we come across onto it, and then remove them from it again as and when the top operator has the appropriate priority.

@opers gets assigned a Hash of operators provided by the caller.

@output gets assigned a class that handle the tokens that are ready to be output.
```ruby
  def initialize opers,output
    @ostack,@opers,@out = [],opers,output
  end
```
We'll skip the #reduce method for now.

#### The #shunt method

The #shunt method is called with an object we can iterate over to get tokens. I'll show a simple implementation that satisfies the requirements for this later.

Lets take a look at the guts of #shunt:
```ruby
    src.each do |token|
      if op = @opers[token]
        op = op[opstate] if op.is_a?(Hash)
```
First we've now looked up the current token, and if it's in the operator table we now have an object representing the operator in "op". The last line above handles the case where there's more than one operator with the same name. This is the case for things like prefix vs postfix increment ("++") in many languages, for example. "opstate" holds a symbol used to pick the right one depending on whether or not the current state means that a prefix or postfix/infix operator is allowed.
```ruby
        if op.type == :rp then reduce
```
If the operator is a closing parenthesis, we need to output the entire parenthesized sub-expression. Any operands have already been output, and the call to #reduce will take care of the operators.
```ruby
        else
          opstate = :prefix
          reduce op # For handling the postfix operators                                                                                        
          @ostack 
```
Above is what happens when any normal operator is found. After an operator, a prefix operator would be valid, but another infix/postfix one is not, so we set opstate accordingly. Then we reduce the operator stack, passing in the current operator to take into account it's priority. Then lastly we push either a synthesized ":call" operator to handle a function call on the form f(arg1,arg2 ...), or just plain the operator from the lookup.
```ruby
      else
        @out.value(token)
        opstate = :infix_or_postfix # After a non-operator value, any single arity operator would be either postfix,                            
                                    # so when seeing the next operator we will assume it is either infix or postfix.                            
      end
      possible_func = !op && !token.is_a?(Numeric)
    end
```

... and if it's an operand it's just fed to the output object.

Last but not least, we set the flag that indicate if this is the possible start of a function call. In this case we simply assume that any non-numeric, non-operator token is a function if it's later followed by a left parenthesis token, which is likely to be far too aggressive for a real language.

#### The #reduce method



The #reduce method loops over the operator stack as long as the top of the stack contains an element with a higher priority value (which means lower priority) OR the top operator is a postfix operator:
```ruby
    while  !@ostack.empty? && (@ostack[-1].pri > pri || @ostack[-1].type == :postfix)
```

If so, the operator is removed. If it's a left parenthesis it means we've reduced / output the entire parenthesized sub-expression, so we just return. If not we output the operator, and continue:
```ruby
      o = @ostack.pop
      return if o.type == :lp
      @out.oper(o)
    end
  end
```

### Outputting RPN



The original shunting yard algorithm outputs reverse polish notation, and this is trivial with the above class. The appropriate output class is as follows:
```ruby
class RPNOutput
  def initialize; @rpn = []; end
  def oper o;     @rpn 
```
### Outputting a tree structure



RPN output is not very interesting to me, so I also hacked together this example class to do tree output (this was the reason I split out the output handling in the first place - doing it for RPN would've been kind of overkill):
```ruby
class TreeOutput
  def initialize
    @vstack = []
  end

  def oper o
    rightv = @vstack.pop
    raise "Missing value in expression" if !rightv
    if (o.sym == :comma || o.sym == :call) && rightv.is_a?(Array) && rightv[0] == :comma
      # This is a way to flatten the tree by removing all the :comma operators                                                                  
      @vstack 
```
It should be fairly self-explanatory. It uses the "type" accessor to determine how to put the tree together. Apart from that the only weirdness is the little bit of code to "flatten" a function call because I rely on treating "," as an operator separating the arguments, and for multi-argument functions it'd quickly create fairly deep, messy trees.

### Pulling it all together



How to use it? First of all we need a simple lexer to produce tokens. This just uses a regex to split things up and convert streams of digits into integers. For "real world" use you'd probably want something more advanced:
```ruby
class SimpleLexer
  def initialize s; @s = s; end

  def each
    @s.scan(/[ \r\n]*([0-9]+|[A-Za-z]+|\+\+|[\(\)+\-*\/\!,])[ \r\n]*/).each do |token|
      token = token[0]
      yield((?0 .. ?9).member?(token[0]) ? token.to_i : token)
    end
  end
end
```

Then we create a Hash of operators and get going:
```ruby
def shunt a
  opers = {
    "+" => Oper.new(10, :plus,  :infix),
    "++" => {:infix_or_postfix => Oper.new(30, :postincr,  :postfix),
             :prefix => Oper.new(30,:preincr, :prefix)},
    "-" => Oper.new(10, :minus, :infix),
    "*" => Oper.new(20, :mul,   :infix),
    "/" => Oper.new(20, :div,   :infix),
    "!" => Oper.new(30, :not,   :prefix),
    "," => Oper.new(2,  :comma,   :infix),
    "(" => Oper.new(99, nil,   :lp),
    ")" => Oper.new(99, nil,   :rp)
  }

  ShuntingYard.new(opers,TreeOutput.new).shunt(SimpleLexer.new(a)).result
end
```

Some examples:
```ruby
PP.pp shunt("1 + !5")
PP.pp shunt("1 + 5")
PP.pp shunt("1 + 2 * 3")
PP.pp shunt("1 * 2 + 3 / 5")
PP.pp shunt("(1 + 2) * 3")
PP.pp shunt("3 * (1 + 2)")
PP.pp shunt("3 * (1 + (2 * 4))")
PP.pp shunt("f(1)")
PP.pp shunt("f(1,2)")
PP.pp shunt("f(1,2,3)")
PP.pp shunt("1 * f ++ + 5")
PP.pp shunt("++f")
PP.pp shunt("1 + ++f")
PP.pp shunt("1 + f ++ - f")
begin
  PP.pp shunt("f + + 5") # Makes no sense. Should give an error.                                                                                
rescue
  puts "Failed, as it should"
end
```

... and it's output:
```sh
$ ruby shunting.rb
[:plus, 1, [:not, 5]]
[:plus, 1, 5]
[:plus, 1, [:mul, 2, 3]]
[:plus, [:mul, 1, 2], [:div, 3, 5]]
[:mul, [:plus, 1, 2], 3]
[:mul, 3, [:plus, 1, 2]]
[:mul, 3, [:plus, 1, [:mul, 2, 4]]]
[:call, "f", 1]
[:call, "f", 1, 2]
[:call, "f", 1, 2, 3]
[:plus, [:mul, 1, [:postincr, "f"]], 5]
[:preincr, "f"]
[:plus, 1, [:preincr, "f"]]
[:plus, 1, [:minus, [:postincr, "f"], "f"]]
[:plus, 1, [:mul, [:not, 5], 2]]
```
Failed, as it should


(Yeah, I know, should do proper unit tests)

You can download the full thing here