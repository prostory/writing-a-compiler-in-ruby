# 真正的语法分析器

### Starting on the parser

The time has come. All the previous parts to this series were mostly written a long time ago, but went through various cleanups. I originally said I had 20 lines up, but as I went through them many were consolidated into larger ones. This is the first new one, and I've decided to start with the parser.

True to the series so far we're going to do things different to the norm. If you want a classic approach, there's a ton of resources out there. If you want a pragmatic "let's se results as quickly as possible" approach, you may like this.

Previously I've used a LISP like primitive text mangling script to demonstrate the bits of the language we have so far. We'll throw that away. But what we will keep is the approach: LISP has one huge thing going for it, and it's the fact that code is data. So far, that's more or less been the case for us too, in that we've used Ruby Array's to represent the code in a tree structure, because it was simple and convenient.

Now, we face a choice. What to do about the syntax? Personally I intensely dislike the LISP syntax, and I love the Ruby syntax, so the parser you'll start to see is Ruby-like. Ruby like because a full Ruby grammar is damn complex, and full of warts. We'll get to that. At the same time, the LISP style syntax is very convenient because it's so simple, and it's prettier than the Ruby Array syntax combined with symbols.

So we'll steal a trick from Ruby: Ever seen things like %w(this is an array of words) in Ruby? We'll use %s() to encompass an "s-expression" just like with our example text mangler. Whenever %s(something) appears, we'll just turn that into a tree of arrays and insert it directly into our syntax tree. Then we'll add a more Ruby-ish syntax around that, making the s-expressions less and less necessary.

### The s-expression grammar

A grammar for a programming language is pretty much the same as a grammar for a natural language, only a LOT simpler, and heavily formalized because we after all want it to be predictable what the programs will (absent bugs and bad design).

Choosing to do "s-expressions" is a bit of a cheat to start with, as the grammar we'll be using is exceedingly simple. I'll present it here in a BNF (Backus Naur Formalism) variant:
```
start  ::= "%s" sexp
sexp  ::= "(" ws* exp* ")"
exp   ::= (atom | int | quoted | sexp) ws*
atom  ::= [a-zA-Z][a-zA-Z0-9_]*
int   ::= "-"?[0-9]+
quoted ::= "\"" escaped* "\""
escaped ::= "\" . | [~"]
ws   ::= [ \t\r\n]
```

Translated into English in case you haven't come across BNF before, this becomes roughly:
* It starts with "%s" followed by an s-expression.
* An s-expression starts with "(" followed by optional whitespace, followed by zero or more expressions, followed by ")"
* An expression consists of any one of an atom, an int, a quoted string, or an s-expression (note the recursion) followed by optional whitespace.
* An atom starts with an alphabetic character, and is followed by zero or more alphabetic characters, digits or underscores.
* An int consists of an optional "-" followed by one or more digits
* A quoted string starts with a double-quote character, is followed by zero or more escaped characters and ends with a double-quote
* An escaped character is either the character '\' followed by ANY character, or any character other than a double-quote.
* A whitespace character is any of space, tab, carriage return or linefeed.

You may have noticed that the "start" rule is only needed to provide a way of escaping out of the grammar we're going to wrap around it. Also the "escaped" rule is only to make the "quoted" rule more readable.

This precisely defines the grammar in enough detail to implement it. So lets.
### A simple scanner


A traditional compiler usually separates the parsing step in a lexical scanner / analyzer and the parser proper. During implementation a separation is sometimes made between a "scanner" that is just taking care of low level character handling and a "lexer" that creates tokens from a stream of characters too (another name for a lexer is "tokenizer" for this reason), in effect splitting the lexical analysis step in too.

To confuse matters further, sometimes in the literature you'll find references to "scanner-less" parsers. In that case "scanner" refers mainly to the lexer/tokenizer, and "scanner less" implies that the parser works directly on characters instead of tokens. A lot of my parsers follows that approach, but in this case I've decided to do some level of tokenization, though the parser will work directly on the character stream whenever it's looking for a specific string, just because it's simple.

Confusing you with terminology above is mainly setting the stage, so the beginner among you have some links for additional reading... But all you really need to know is the class I present below provides a simple basis for categorizing the input into "tokens". To take a simple example, given this:
```lisp
(defun foo (a b c) (puts "this is a test"))
```
.. we have the following token stream:
```
'(', 'defun', 'foo', '(', 'a','b','c',')','(','puts','"this is a test"',')',')'
```
If what we want is so simple, then there's one more question worth asking:

Why not Ruby's StringScanner class? Well, it's a Ruby C-extension, and I want to get the compiler self-hosted as soon as possible,  so I'm sticking to something simple. The code below is sufficient to write recursive descent parsers in a pretty concise style in Ruby    

Let's start with the basics:
```ruby
class Scanner
 def initialize io
  @io = io
  @buf = ""
 end

 def fill
  if @buf.empty?
   c = @io.getc
   c = c.chr if c
   @buf = c ? c.to_s : ""
  end
 end

 def peek
  fill
  return @buf[-1]
 end

 def get
  fill
  return @buf.slice!(-1,1)
 end

 def unget(c)
  c = c.reverse if c.is_a?(String)
  @buf += c
 end
```
This is pretty much the basic of most of the scanners I do. #initialize takes any object that supports #getc, which means any IO object (STDIN, File's etc.), but also anything you might decide to write that can implement a #getc method. Writing an adapter for practically anything that can be represented like a sequence of characters is in other words trivial.

`#fill` takes care of ensuring we always have at least one character waiting before `#peek` and `#get` does their jobs.

`#peek` gives us a single character lookahead without doing any additional buffering, and you'll see how that simplifies things later. `#get` returns a single character. Notice a difference in how they act that may be a bit annoying - I'm not sure whether to keep it that way or not: `#peek` returns the integer value that represents a specific character but `#get` returns a single character string. It makes some things simpler in the parser (through the savings are pretty trivial), but you need to keep an eye on it or risk confusion.

`#unget` puts a string or character back onto the buffer. As the name says, it "ungets" something you've previously used `#get` on. `#unget` combined with #get provides longer lookahead for the cases where that is convenient. Many languages are fine with just one character lookahead, but if you want to write that parser in a "scanner less" (i.e. without tokenization) style, it's often easier to use more lookahead even if the grammar otherwise can be parsed with one character lookahead. An example:

If the parser expects "defun" but the user has written "define", a tokenizing parser will see the "d", decide that this is a token of type "word" (or whatever the compiler writer has called it), parse the whole thing, and return a "Token" object or somesuch with the type "word" and the value "define", and the parser will raise an error at that level. This works because the grammar is designed so that when "d" occurs, it's always a word, and then the parser above it relies on a single token of lookahead, not a character. Without the tokenizer, you're stuck either writing the parser so that it will explicitly look for common prefixes first, and then the rest of the words, OR you use more lookahead.

Imagine if an alternative grammar rule for the parser above that expects "defun" is that "dsomethingelse" is also allowed at the same spot. Now the parser writer can either look for "d", and then look for "e" or "s", or he can use a scanner like the above that can use more than one character lookahead, and look directly for "defun", and if that fails look for "dsomethingelse". For handwritten parsers without a tokenizer the latter is simpler, and a lot easier to read, and it only results in more lookahead actually being used in cases where there are multiple rules that are valid at the same point, and they have common prefixes, which isn't too bad as it's something we'll generally want to avoid.

Note that I won't avoid more generic tokenization everywhere in the parser. Let's move on:
```ruby
 def expect(str)
  return true if str == ""
  return str.expect(self) if str.is_a?(Class)
  buf = ""
  str.each_byte do |s|
   c = peek
   if !c || c.to_i != s
    unget(buf)
    return false
   end
   buf += get
  end
  return true
 end
end
```
This allows us to do things like myScanner.expect("defun"), and it will obediently recognize the string, and then unget the whole thing if it doesn't get to the end. So we can do myScanner.expect("defun"), and then myScanner.expect("dsomethingelse"), and it will handle the lookahead for us.

To make it easier to get it self hosted, it doesn't support regexps like the StringScanner class does. However it does have one concession: "return str.expects(self) if str.respond_to?(:expect)". If you pass it something that responds to "expect", it'll call expect and pass itself, and let that object handle the recognition itself instead. That'll let us do things like myScanner.expect(Token::Word) in the future, so we can happily mix a tokenizer style with a character-by-character style.

### A simple s-expression parser

Now that we have the scanner, lets move on and go through the important bits:

First thing we do is implement a function to skip whitespace where it is allowed. I won't dwell on it, as it should be simple enough.
```ruby
 def ws
  while (c = @s.peek) && [9,10,13,32].member?(c) do @s.get; end
 end
Then we start at the top, literally, with our "start" rule:

 def parse
  ws
  return nil if !@s.expect("%s")
  return parse_sexp || raise("Expected s-expression")
 end
```
The one thing worth noting here is that we return nil if we can't find "%s" but we raise an exception (and we will switch to using a custom exception class rather than just passing a string to raise later on, don't worry) if parse_sexp doesn't return something. This is a general pattern: Return nil for as long as you leave the stream unchanged. When you are sure that you have satisfied the start of the rule (and you know that no other rule that is valid at this point starts the same way), you know that failure to satisfy the rest of the rule is an error.

next up, #parse_sexp needs to handle a list of expressions:
```ruby
 def parse_sexp
  return nil if !@s.expect("(")
  ws
  exprs = []
  while exp = parse_exp; do exprs << exp; end
  raise "Expected ')'" if !@s.expect(")")
  return exprs
 end
```
`#parse_exp` is mostly a much of alternatives, and here you'll see the custom classes passed to expect:
```ruby
 def parse_exp
  (ret = @s.expect(Atom) || @s.expect(Int) || @s.expect(Quoted) || parse_sexp) && ws
  return ret
 end
```
... and, that's the whole s-expression parser. Of course, it won't really be complete without the custom tokenization classes, so let's quickly take a look at one of them - the most complex one:
```ruby
 class Atom
  def self.expect s
   tmp = ""
   if (c = s.peek) && ((?a .. ?z).member?(c) || (?A .. ?Z).member?(c))
    tmp += s.get

    while (c = s.peek) && ((?a .. ?z).member?(c) || 
       (?A .. ?Z).member?(c) ||
       (?0 .. ?9).member?(c) || ?_ == c)
     tmp += s.get
    end
   end
   return nil if tmp == ""
   return tmp.to_sym
  end
 end
```
As you can see it's really just a container for a single function, and really nothing is stopping you from doing Tokens::Atom.expect(s) instead of @s.expect(Token::Atom), but I feel the latter reads better.

You can download an archive of the source as at this step here, or follow the individual commits for this step (here is the last one for this part), or the project as a whole on Github 