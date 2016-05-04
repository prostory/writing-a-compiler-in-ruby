# 扩展语法分析器

From now on I'm not going to cover every little change - too much of it is repetitive, but I'll keep posting parts of this series to cover major changes. If you want to keep track of the changes "blow by blow", follow the project on GitHub. If you find something I've skipped over that you'd like to know more about, I'd be happy to expand on it in the comments, or perhaps cover it in the next part.
### Extending the parser: The first bits of "real" syntax.

The code as at the end of this post can be found at the 'step-16b' tag of the GitHub repository

We'll keep the s-expression grammar separate from the grammar for the rest of the language, so in the BNF grammar below all productions refers to other parts of this grammar other than "sexp" which refer to the start production of the s-expression grammar. This grammar covers the first set of extension for this part:
```
 program ::= exp* ws*
 exp   ::= ws* (def | sexp)
 def   ::= "def" ws* name args? ws* defexp* "end"
 defexp ::= sexp
 args  ::= ws* sexp
 name  ::= atom
```
This simple initial grammar will let us change some of the examples. Here's a snippet from testargs.l: 
```ruby
def g %s(i j)
%s(let (k)
 (assign k 42)
 (printf "numargs=%ld, i=%ld,j=%ld,k=%ld\n" numargs i j k)
 )
end 
```
Not exactly a tremendous difference, but it's a start, and it puts the basics in place to chip away at the parser bit by bit, so lets look at some code. It's fairly straightforward, so I'll just go through the "def" production - you can see the rest of this version here.
```ruby
 # def ::= "def" ws* name args? ws* defexp* "end"
 def parse_def
  return nil if !@s.expect("def")
  @s.ws
  raise "Expected function name" if !(name = parse_name)
  args = parse_args
  @s.ws
  exps = [:do]
  while e = parse_defexp; exps << e; end
  raise "Expected expression of 'end'" if !@s.expect("end")
  return [:defun, name, args, exps]
 end
```
As for the s-expression parser it's a fairly straightforward translation of the BNF into a recursive descent parser, with some minor additional error handling. 
Note how optional rules simply translate into not checking if it returns nil or not, and "zero or more" rules translate into parsing into an array.

This isn't terribly exciting (and you can probably see now why I won't be writing up every single commit), so lets expand the grammar a bit more:
```
 program ::= exp* ws*
 exp   ::= ws* (def | sexp)
 def   ::= "def" ws* name args? ws* defexp* "end"
 defexp ::= sexp
 args  ::= ws* sexp
 name  ::= atom
 args  ::= nolfws* ( "(" ws* arglist ws* ")" | arglist )
 arglist ::= ("*" ws*)? name nolfws* ("," ws* arglist)?
 nolfws ::= [ \t]+ ; This rule is defined in the scanner source code, like ws
```
The new thing in the updated grammar is redefining "args" and adding "arglist". Those rules effectively says:

A set of args consist of optional whitespace excluding line feed (and carriage return) followed by either an arglist, or an arglist in parentheses. An arglist consists of of an optional "*" ("splat",equivalent to our :rest flag) followed by a name, whitespace excluding lf, and optionally ",", whitespace and arglist. In other words we use recursion allow a list of arguments (and just while writing this I realized that grammar is stupid, as it allows "*" in front of all arguments, which makes no sense - to be fixed...)

So the example from above can be changed to this now:
```ruby
def g(i,j)
%s(let (k)
 (assign k 42)
 (printf "numargs=%ld, i=%ld,j=%ld,k=%ld\n" numargs i j k)
 )
end 
```
One step closer to a nice syntax...

One more thing is new: Pass the "--parsetree" option to the compiler to dump the parsetree (using PP) together with some crappy error reporting:
```sh
[vidarh@dev writing-a-compiler-in-ruby]$ cat testargs.l | ruby compiler.rb --parsetree
[:do,
 [:defun,
 :f,
 [:test, [:arr, :rest]],
 [:do,
  [:let,
  [:i],
  [:assign, :i, 0],
  [:while,
   [:lt, :i, [:sub, :numargs, 1]],
   [:do,
   [:printf,
    "test=%ld, i=%ld, numargs=%ld, arr[i]=%ld\n",
    :test,
    :i,
    :numargs,
    [:index, :arr, :i]],
   [:assign, :i, [:add, :i, 1]]]]]]],
 [:defun,
 :g,
 [:i, :j],
 [:do,
  [:let,
  [:k],
  [:assign, :k, 42],
  [:printf, "numargs=%ld, i=%ld,j=%ld,k=%ld\n", :numargs, :i, :j, :k]]]],
 [:f, 123, 42, 43, 45],
 [:g, 23, 67]]
[vidarh@dev writing-a-compiler-in-ruby]$ 
```
And that's where we'll leave it for this time.