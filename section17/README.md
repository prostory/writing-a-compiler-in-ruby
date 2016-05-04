# 推断"let"表达式

The code as at the end of this part can be found here

I won't promise to keep up the posting frequency I've kept over the last week or so, but here's a tiny little part showing the next step. This time we're getting rid of "let". Well, sort of. I won't remove it from the syntax tree, but I will hide it, by making the compiler identify all :assign nodes in a function and creating a let for the variable referenced. Note that the current version is simplistic: It will ignore any :let nodes, and so may introduce redundant variables if you embed s-expressions with :let nodes, and it also ignores expressions on the left hand of an assign - %s(index arr 0) for example is a valid expression on the left hand of an assign. But it's a start.
If you're following my Github repository, you may already have seen the code, but here's a quick explanation.

First we make this change to the parsers #parse_defexp method:
```ruby
    -    exps = [:do] + zero_or_more(:defexp)
    +    exps = zero_or_more(:defexp)
    +    vars = deep_collect(exps,Array) {|node| node[0] == :assign ? node[1] : nil}
    +    exps = [:let,vars] + exps 
```
It leaves the method looking like this:

`# def ::= "def" ws*` name args? ws* defexp* "end" def parse_def return nil if !@s.expect("def") @s.ws raise "Expected function name" if !(name = parse_name) args = parse_args @s.ws exps = zero_or_more(:defexp) vars = deep_collect(exps,Array) {|node| node[0] == :assign ? node[1] : nil} exps = [:let,vars] + exps raise "Expected expression of 'end'" if !@s.expect("end") return [:defun, name, args, exps] end As you can see the new thing is a call to a new function - deep_collect - that is used to collect the variable names. Lets take a look at deep collect:
```ruby
    # Visit all objects in an array recursively.
    # yield any object that #is_a?(c)
    def deep_collect node, c = Array, &block
      ret = []
      if node.is_a?(c)
        ret << yield(node)
      end
      if node.is_a?(Array)
        node.each do |n| 
          if n.is_a?(Array) || n.is_a?(c)
            ret << deep_collect(n,c,&block) 
          end
        end
      end
      ret.flatten.uniq.compact
    end
```
I am sure there's cleaner way of doing this, but the purpose is to walk through the tree of arrays we use as a syntax tree, and yield any object that #is_a?(c). #parse_defexp use that to check if one of the Array nodes in the tree starts with :assign, and assumes that the first argument to :assign will be a variable name (in the future we'll need to expand on that to walk through any trees there instead of just expecting a variable name), and return an array of the identified variable names.

Then #parse_defexp will create a :let node instead of a :do node like it used to.

That's all there's too it. The next part will look at something more interesting: I'm plugging in an adaptation of my operator precedence parser. Also note that code is already on Github to handle "while" - I won't be going through those changes as they're pretty straightforward, and uses exactly the same approach as I did to handle "def".