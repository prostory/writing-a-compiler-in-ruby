# 调度场解析组件的详细攻略

You may or may not have seen my recent post where I admitted to more or less having decided to make my compiler project a Ruby compiler. On the downside this means a lot of complexity that may make it harder to follow. On the upside... Well, you get to read about a compiler for a "real" language as opposed to a toy, and hopefully the result will be usable and I'll manage to contain myself and not go off on wild tangents (I have a long list of experiments I want to do)

Since the last part things have moved quite far, and I'm not going to do a "patch by patch" overview of what's happened since. Instead I will this time give a rough overview of the state of the compiler as of starting to write this - April 25th (slightly derailed by having a child). Parts of the compiler with more extensive changes will get more "coverage".

Let me again point out that all of the code is on Github. Specifically the state of the code when writing this can be found at tag "step-20c".

Furthermore, I'd like to extend an invitation to anyone who's interested in contributing. Go check out the project from Github, fork it if you like, and send me pull requests when you have something you'd like me to consider for the main tree, or just let me know if it's ok for me to pull things in whenever I see something interesting. There's now two of us - Christopher Bertels started contributing a while back, and have added lots of rdoc documentation and a range of other improvements, such as part of the instance variable support.

We've so far decided to go with the same license as Matz' version of Ruby, but I'm open to discussion about that. Specifically I would prefer the core library to be licensed under terms sufficiently loose to allow closed source apps to be compiled as well.

Anyway, lets see what things look like now.

### The Parser

If you've followed my blog, you'll probably have seen the announcement that the compiler can now parse itself. I'm sure it still has bugs (it certainly did when I posted the announcement, as I've found out since), and it most certainly can't compile the AST it generates, but it's still a fairly big milestone. It also means that it can parse a small but relatively central subset of Ruby (there are many, many gaps).

The parser is split in three components:
The low level s-expression style parser. This part has hardly changed.
The operator precedence parser used for expressions. This has changed a lot, and needs a cleanup, but I'll attempt a description of the current state
The "high level" parser used for control structures etc.. This is a simple recursive descent parser. It also would benefit from a cleanup, but it's not hard to understand (I think).

First of all, let us take a look at something else: Test cases. Parsing Ruby is nasty. As much as I love to program in Ruby, the grammar makes me feel downright dirty. It is full of context sensitive parts, exceptions and weird rules that make it painful to write a small and readable parser for it. All the more reason to write extensive sets of test cases.

#### Testing the parser components

I've been using Cucumber to add test cases, mainly because so far for the parser the test cases are very repetitive "here's the input, it's meant to generate this tree" kind of tests, and Cucumber makes that look nice and readable:
```
Feature: Shunting Yard
    In order to parse expressions, the compiler uses a parser component that uses the shunting yard
    algorithm to parse expressions based on a table.

    Scenario Outline: Basic expressions
        Given the expression expr
        When I parse it with the shunting yard parser
        Then the parse tree should become tree

    Examples:
      | expr                 | tree                                 |
      | "__FILE__"           | :__FILE__                            |
      | "1 + 2"              | [:add,1,2]                           |
      | "1 - 2"              | [:sub,1,2]                           |
      | "1 + 2 * 3"          | [:add,1,[:mul,2,3]]                  |
      | "1 * 2 + 3"          | [:add,[:mul,1,2],3]                  |
      | "(1+2)*3"            | [:mul,[:add,1,2],3]                  |
      | "1 , 2"              | [:comma,1,2]                         |
      | "a << b"             | [:shiftleft,:a,:b]                   |
      | "1 .. 2"             | [:range,1,2]                         |
      | "a = 1 or foo + bar" | [:or,[:assign,:a,1],[:add,:foo,:bar]]|
      | "foo and !bar"       | [:and,:foo,[:not,:bar]]              |
      | "return 1"           | [:return,1]                          |
      | "return"             | [:return]                            |
      | "5"                  | 5                                    |
      | "?A"                 | 65                                   |
      | "foo +\nbar"         | [:add,:foo,:bar]                     |
      | ":sym"               | :":sym"                              |
      | ":[]"                | :":[]"                               |
```
As of writing, the parser components have 106 scenarios (each entry in the example tables counts as one scenario) including a few failing ones. Whenever we find anything broken, a new test case goes in. 106 scenarios is nowhere near complete, but it helps tremendously with debugging, particularly with the operator precedence parser, as it's been fairly tedious to adjust and adapt it to handle the peculiarities of Ruby. It also serves as documentation of sorts of what the parser is expected to deliver
#### The operator precedence parser

We've covered this component before of sorts (go revisit it if you're unsure about it), but it's grown significantly in complexity since, mostly to account for weirdness in the Ruby grammar, some to account for missing functionality.  A word of warning before we start: It's gotten messy, and it needs to be cleaned up and re-factored. But it's not that hard to figure out.

I'm going to assume you're familiar with the previous parts, and have a rough idea of Dijkstra's Shunting Yard algorithm which this parser component is based on. We'll examine it top-down the way it's being used:
```ruby
  def self.parser(scanner, parser)
     ShuntingYard.new(TreeOutput.new,Tokens::Tokenizer.new(scanner), parser)
  end
```
We start by instatiating the parser and passing  the most frequently used components to it. By passing an alternative to TreeOutput you an turn the output into reverse polish notation or anything you like. But we want a parse tree. Tokens::Tokenizer is used to retrieve a stream of tokens instead of working on individual characters. Last but not least we're passing it a reference to the recursive descent parser. Yuck. We do this because of Ruby's block syntax, which because of the way blocks can look like literal hashes and/or can be chained and used as part of an expression means we need to be able to call bak out of the shunting yard parser. 
It might be that it'd be just as well to combine the two, but I don't like tight coupling and I'm still living in hope of getting the time to figure out a way to reduce the coupling between these two components, and make the shunting yard parser generic enough to be reusable. One of the promises of an operator precedence parser is to be configured with a simple table of operators, and it's possible more of the exceptions in this current model can be handled by adding a few flags.

On to the next bit:
```ruby
    def parse(inhibit=[])
      out = @out.dup
      out.reset
      tmp = self.class.new(out, @tokenizer,@parser)
      res = tmp.shunt(@tokenizer,[],inhibit)
      res ? res.result : nil
    end
```
A couple of oddities here. I've introduced an argument to allow specifying a set of tokens to inhibit and terminate the expression. This is needed because of peculiarities in how Ruby handle commas. Some places it's ok as it separates function arguments or array elements, but you also need to be able to put expressions  IN argument lists as default assignments, and there commas are not allowed without parantheses, as they're needed to separate arguments, and the argument list itself is parsed by the recursive descent parser. It would be possible to change this, but I'm not sure I want to try to make the argument list be parsed by the same component until I'm more sure of the consequences. Besides the current arrangement works.
You can also see we do some annoying initialization - copying and resetting the tree output object. This is done because the parser recursively calls itself, and the state of the output tree builder need to be clean when it gets called for a subtree in parantheses for example. 

The next bit is in a fairly shameful state at the moment, but rather than wait until it's cleaned up I'd rather show you the intermediate state, and we'll revisit again later once I have it all figured out - the shunting parser probably deserve it's own follow up article as it can get a bit hard to follow. 

Leading up to the main loop, which I'll break up in several chunks:
```ruby
    def shunt(src, ostack = [], inhibit = [])
      possible_func = false     # was the last token a possible function name?
      opstate = :prefix         # IF we get a single arity operator right now, it is a prefix operator
                                # "opstate" is used to handle things like pre-increment and post-increment that
                                # share the same token.
      lp_on_entry = ostack.first && ostack.first.type == :lp
      opcall  = Operators["#call#"]
      opcallm = Operators["."]
      lastlp = true
```
The commented local vars are hopefully reasonably understandable. "lp_on_entry" is set to true if the method is passed an :lp operator in - that means '(','[', '{' etc.. We'll see how that's used later. opcall and opcallm are just convenience shortcuts, though the hardcoded references to Operators[] are ugly (we'll meet more of them, and they'll go as soon as I find a clean way of decoupling the logic in this method). "lastlp" is set to indicate whether or not the last token was an :lp type operator.
```ruby
      src.each do |token,op|
        if inhibit.include?(token)
          src.unget(token)
          break
        end
```
Ok, so this is the start of the main loop. We get the token as a string in in "token", and if it's an operator we get the operator object in "op". We check if the token is member of a set of "inhibited" tokens that we don't allow to be part of an expression. This is used to handle cases where the recursive descent parser wants an expression that stops on encountering something that is normally a legal part of the operator. We'll see it used when we look at the recursive descent parser.
```ruby
        if op
          op = op[opstate] if op.is_a?(Hash)
```
We then start handling operator tokens - most of the loop is split between handling operators and handling non-operator tokens, with very little shared logic. In some cases there will be two operators that share the same token. Currently we handle :infix_or_postfix vs. :prefix operators, and the only case it's currently used for is "*" as multiplication operator vs. as the "splat" operator (which expands arrays).
```ruby
          # This makes me feel dirty, but it reflects the grammar:
          # - Inside a literal hash, or function call arguments "," outside of any type of parentheses binds looser than a function call,
          #   while outside of it, it binds tighter... Yay for context sensitive precedence rules.
          # This whole module needs a cleanup
          op = Operators["#,#"] if op == Operators[","] and lp_on_entry
```
The comment above speaks for itself, no? And we see a use for lp_on_entry to help decide the precedence rule to use for the comma, by switching objects around (Note that this also shows an anti-pattern that needs to be fixed: When you have hashes, try to avoid string keys. Strings are expensive objects where symbols get mapped to an integer - for things like this using symbols as keys tends to be more efficient).
```ruby
          if op.sym == :hash_or_block || op.sym == :block
            if possible_func || ostack.last == opcall || ostack.last == opcallm
              @out.value([]) if ostack.last != opcall
              @out.value(parse_block(token))
              @out.oper(Operators["#flatten#"])
              ostack << opcall if ostack.last != opcall
            elsif op.sym == :hash_or_block
              op = Operators["#hash#"]
              shunt(src, [op])
              opstate = :infix_or_postfix
            else
              raise "Block not allowed here"
            end
```
The whole block above gets executed if the current operator is a '{' (:hash_or_block) or :block ("do"), and it's one of the hairy bits... If we've seen what is possibly the start of a block (possibly, because '{' could also start a hash, hence the symbol), we first check if we're possibly looking at a function call (a pre-requisite for a block) based on the previous token, OR if the last operator to have been pushed on the operator stack is the function or method call operators (note: We currently still differentiate between function and method calls, though that is not a distinction Ruby makes; it is helpful for some aspects of the parser, and it is helpful for the low level s-expression syntax, but this distinction will normally become hidden from the "end-user").  

If there's a function call being handled, then we know we're dealing with a block. We output an empty argument array if needed, and then call back up into the recursive descent parser via "parse_block" (token is passed so that the parser can determine what symbol the block should end on - '}' vs. 'end'). The "fake" operator "#flatten#" is output to aid in the tree building as the tree builder doesn't directly handle cases where a node has more than two children/arguments. This is a bit of a hack, and there is actually no real reason why a shunting yard parser can't be adapted to directly handle operators that are prefixes to multiple operands, so that may be part of the cleanup later.  

If we're NOT dealing with a function call, we check if we're looking at the :hash_or_block ('{') operator, and if so we know we're now dealing with a Hash, and we recursively call shunt to handle the interior of the Hash.
```ruby
          else
            if op.type == :rp
              @out.value(nil) if lastlp
              @out.value(nil) if src.lasttoken and src.lasttoken[1] == Operators[","]
              src.unget(token) if !lp_on_entry
            end
```
If we are not dealing with a potential block, we move on. First, above, we check for a :rp (right parenthesis, bracket or brace) operator. We "fake" a value if we've just seen an empty pair of parentheses, so we don't have to deal with operand-less operators elsewhere. We also fake value if the last token was a comma operator. This is to handle the convenience case in Ruby where arrays or hashes can have a "dangling" comma at the end like so: [1,2,3,] (useful for symmetry and machine generating Ruby source.
```ruby
            reduce(ostack, op)
```
We call reduce in order to fore output of any tighter binding operators. In the case of, say, 1 * 2 + 3, (* 1 2) will get output when encountering '+'.
```ruby
            if op.type == :lp
              shunt(src, [op])
              opstate = :infix_or_postfix
              # Handling function calls and a[1] vs [1]
              ostack << (op.sym == :array ? Operators["#index#"] : opcall) if possible_func
```
If it's an lp type operator, we do as we did for Hash ('{'), except we also want to selectively output "#index#" or the "#call#" operators if we're dealing with a function call. This occurs when we have an expression like "foo(1)", in which case "possible_func" will be true after "foo" has been encountered, and we then parse "(1)" and pass the result onto the output handler, and then output the "#call#" operator to tie "foo" and it's arguments together. As the slightly misleading comment says, "#index#" is used if we see '[...]' after something that might indicate a function call, as we need to differentiate a[1] (a call to the method "[]" on the object reference held in "a") vs [1] (constructing an Array object).
```ruby
            elsif op.type == :rp
              break
```
If we dealt with an :rp we want to exit from the loop.
```ruby
            else
              opstate = :prefix
              ostack << op
            end
          end
```
... if not we just push the operator on the operator stack.
```ruby
        else
          if possible_func
            reduce(ostack)
            ostack << opcall
          end
          @out.value(token)
          opstate = :infix_or_postfix # After a non-operator value, any single arity operator would be either postfix,
                                      # so when seeing the next operator we will assume it is either infix or postfix.
        end
        possible_func = op ? op.type == :lp :  !token.is_a?(Numeric)
        lastlp = false
        src.ws if lp_on_entry
      end
```
Ok, time to handle non-operator values. Not much to say about this - it's hopefully reasonably understandable. The last line about will skip whitespace including line feeds if we're inside a parenthesized sub-expression, while normally (inside the tokenizer) we only skip whitespace excluding linefeeds. This is because the general rule in Ruby is to allow linefeeds anywhere where it doesn't create an ambiguity. I'm sure there are more special cases we'll need to handle, but for now this approximates the rule closely enough.
```ruby
      if opstate == :prefix && ostack.size && ostack.last && ostack.last.type == :prefix
        # This is an error unless the top of the @ostack has minarity == 0,
        # which means it's ok for it to be provided with no argument
        if ostack.last.minarity == 0
          @out.value(nil)
        else
          raise "Missing value for prefix operator #{ostack[-1].sym.to_s}"
        end
      end

      reduce(ostack)
      return @out if ostack.empty?
      raise "Syntax error. #{ostack.inspect}"
    end
```
This is the last part we'll look at (for the "reduce" method, see the earlier post), and it's outside the look. It's down to handling errors and optional operands, and then reducing the operator stack completely before returning the output handler (which should at this point hopefully contain a complete expression).
#### The recursive descent parser

The operator precedence parser was the hard part of parsing. There's not really all that much to say about the changes to the recursive descent parser - it's all really formulaic, so I'll pick one of the larger functions and walk through that:
```ruby
  def parse_block(start = nil)
    pos = position
    return nil if start == nil and !(start = expect("{")  || expect("do"))
    close = (start.to_s == "{") ? "}" : "end"
    ws
    args = []
    if expect("|")
       ws
      begin
        ws
        if name = parse_name
          args << name
          ws
        end
      end while name and expect(",")
      ws
      expect("|")
    end
    exps = parse_block_exps
    ws
    expect(close) or expected("'#{close.to_s}' for '#{start.to_s}'-block")
    return E[pos,:block] if args.size == 0 and !exps[1] || exps[1].size == 0
    E[pos,:block, args, exps[1]]
  end
```
This is what the shunting yard parser calls back into. Most of this should be fairly understandable assuming you remember that "expect" tries to match a string, and "ws" skips past whitespace. 
The major new thing here is "position" and that "E[]" stuff. "position" is a method that returns what the scanner thinks is the current position. It isn't guaranteed to be 100% accurate all the time as the scanner doesn't itself keep track of the length of lines it has scanned past, so it depends on the token being "unget" to contain a postition for that (see scanner.rb - not going in depth into that). 

In reality it works well enough to give reasonable error reporting during parsing. But what about errors identified in later stages? That's where E[] comes in. It's a shortcut to create an instance of a subclass of Array: AST::Expr. We'll be using this class to carry additional annotation of the nodes in the parse tree in later stages. For now the additional element is the position, which it takes from the first argument that either is a position or has one. See ast.rb for the details. 
### Compiling the syntax tree

The compiler class itself hasn't changed all that much since last time apart from much better error reporting (see error()). The most significant worthwhile change to look at is the implementation of basic support for instance variables in objects. As for vtables for classes we're taking major shortcuts to start with.

The magic starts in "scope.rb".  Specifically we now keep an array of instance variables we've seen for this current class. This chunk in Scope#get_arg will return [:ivar, offset] where "offset" is the number of the "slot" of PTR_SIZE values starting from the beginning of an instance object that this instance variable belongs to.
```ruby
    # instance variables.
    # if it starts with a single "@", it's a instance variable.
    if a.to_s[0] == ?@ or @instance_vars.include?(a)
      offset = @instance_vars.index(a)
      add_ivar(a) if !offset
      offset = @instance_vars.index(a)
      return [:ivar, offset]
    end
```
At the moment the above contains a bug, as the instance variables will actually get added too late for the "new" method to correctly allocate the object of the right size. So in the next batch of changes will be one to "visit" the nodes of a class definition to pre-allocate the instance variable offsets.
This current approach also has another problem: In Ruby you can dynamically add instance variables to an object. There's no guarantee that all (or any) of the instance variables identified will actually be added to any specific instance. Here the compiler will need to make a trade-off:

We can allocate space for all the instance variables we see (we still need a way to handle dynamically allocated ones) or we can pick a subset that is likely to be present on most objects (if it's set in "initialiaze" for example). For dynamically allocated ones we can set aside space for pointers to a hash table to hold them. We can also use a level of indirection to "pack" the instance variables more to avoid having to resort to the hash tables as often as we otherwise might do. There's no guarantee any specific one of these strategies will be best for exactly your app, so there's room for switches here, or have the system dynamically optimize the choices at runtime (at the cost of more complexity). For now, though, we'll just allocate sufficient space for everything we see, and ignore dynamically allocated ones. Then later, we'll add support for handling dynamically allocated ones, and then we can start looking at optimizations. As usual we take the shortcut that bring us functionality first.

The compilation of the instance variables is pretty trivial - load and saves happens in compiler.rb, in compile_assign and compile_eval_arg respectively. Here's what we do in compile_assign:
```ruby
    if atype == :ivar
      ret = compile_eval_arg(scope,:self)
      @e.save_to_instance_var(source, ret, aparam)
      return [:subexpr]
    end
```
Emitter#save_to_instance_var just stores the value to the appropriate offset into the object.
### The support libraries

We've barely started on the support libraries, so this will be short, but it's worth taking a brief look at what's in the main tree so far, not least because it demonstrates how I intend to proceed - even core elements of the object model, such as the Object and Class classes will largely be written in Ruby, and make use of only very limited bootstrapping help from the compiler. Only a small number of elements will require us to "dip out" of pure Ruby, and in those cases we'll as much as possible rely on the s-expression syntax that allow direct access to the AST. Doing C etc. is a last resort or temporary workaround only - a goal of this project is minimal dependencies.

Here's what we use to bootstrap Class:
```ruby
def __new_class_object(size)
  ob = malloc(size)
  %s(assign (index ob 0) Class)
  ob
end

class Class
  def new
    # @instance_size is generated by the compiler. YES, it is meant to be                                              
    # an instance var, not a class var                                                                                 
    ob = malloc(@instance_size*4)
    %s(assign (index ob 0) self)
    ob
  end
end
```
Calls to __new_class_object is called by the bootstrap code generated by the compiler (in compile_class). It just uses the C-library's malloc() for now, and assigns a pointer to the class Class in the first slot.
Class#new is implemented so that it will take the instance size from each subclass (that's why we use an instance variable of the class instead of a class variable - class variables are shared across subtrees of the inheritance tree in Ruby), and then does more or less what __new_class_object does.

It's worth noting that we make this raw class reference available as the instance variable @__class__ in the objects. Why not as "class"? Well, the Ruby object model is finicky. Modules and meta-classes are inserted into the inheritance chain (for good reason - it makes things a lot easier), but the are "hidden" from you when you access self.class. So @__class__ is an implementation detail that should not be dependent on other than to help us implement the core classes. The same is true for "@instance_size" in class Class.

### Final words (for now)

As usual, if you have questions or comments, they are welcome. Get all of the code from Github. The state of the code when writing this can be found at tag "step-20c".

And feel free to get in touch to contribute, or just fork the code and hack away. 

There's already a ton of changes since I started writing this part. The next milestone is to get the compiler to compile itself - that's probably 2-3 articles away. The first step will be to get it to generate code that will link, but I fully expect it to have plenty of problems that will prevent it from running... An article on debugging the code generation is likely to be forthcoming at that stage...
Once it can compile itself, my next goal is to work on getting it to compile mspec, in order to get a measure of what is missing from Ruby. It will require quite a few fixes to the parser.
