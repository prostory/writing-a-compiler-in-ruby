# 真正的字符串：将字符串转换为对象

### Continuing down the rabbit hole: String

A couple of parts ago we established some of the problems with supporting even the seemingly simple attr_reader, attr_writer and attr_accessor. To reiterate, a naive implementation looks like this:
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
We resolved the issue of having a working sym.to_s, but that still leaves a number of things to address. For example, we need at least the start of a "proper" String class that we can add #to_sym to. That's what we'll look at this time.

The first bit is very similar to what we did for Symbol: We won't type tag, and we'll create String objects for literal strings by calling a low level function.

### Quick sidebar: Hiding the low level implementation

MRI "hides" the gory details of String and other basic classes by not making them real Ruby objects per-se. There's data there that is simply not accessible without dipping into the C API. We may eventually want to take a similar tack - add a mechanism for allocating space for, setting and retrieving low level variables via the s-expression syntax that are completely invisible for the higher level API.

Currently we don't. The "raw" string used for Symbol and for String objects is a normal instance variable, and it's perfectly possible to expose it, and accessing it thinking it's a real object will trivially easily cause nasty crashes.

In general, we would eventually benefit from creating a mechanism to "hide" the low level plumbing from accidental access by normal Ruby. For now we'll retain the ability to shoot ourselves in the foot easily.

### The basics

The String class will for now contain just a pointer to a C-style string. It will be immutable in it's first incarnation. This gets us a decent step further very easily - we can just keep the address to a C-style string constant.

As it turns out, this will be an easy foundation to build on:

MRI uses a number of flags for strings, for example to do "copy on write". Once we need to make strings mutable, we'll do the same. Strings created from constants will start out marked as "copy on write", and when doing an update, the string buffer will be copied into freshly allocated space first. The hope is that many string constants will never be updated.

We also need to implement String#to_sym. This is thankfully also reasonably easy: We need to access the low level function we added that will retrieve a symbol based on a "raw" c-style string.

### The code

As the last couple of times, this was unfortunately committed a bit piecemeal, so the commits aren't easy to follow. But lets go through this step by step.

First of all, we're going to rewrite the tree, to alter a literal string into
```ruby
%s(call __get_string ConstantReferringToTheRawString)
```
So, we alter compile to add a call to rewrite_strconst:
```ruby
#
# Re-write string constants outside %s() t
# %s(call __get_string [original string constant])
def rewrite_strconst(exp)
  exp.depth_first do |e|
    next :skip if e[0] == :sexp
    is_call = e[0] == :call
    e.each_with_index do |s,i|
      if s.is_a?(String)
        lab = @string_constants[s]
        if !lab
          lab = @e.get_local
          @string_constants[s] = lab
        end
        e[i] = [:sexp, [:call, :__get_string, lab.to_sym]]
        # FIXME: This is a horrible workaround to deal
        # with a parser inconsistency that leaves calls
        # with a single argument with the argument "bare"
        # if it's not an array, which breaks with this rewrite.
        e[i] = [e[i]] if is_call && i > 1 
      end
    end
  end
end
```
Effectively this visits all nodes that are not s-expressions, and if it finds a string, it will add a string constant. It will then use the label that is allocated to rewrite the expression like this:
```ruby
[:sexp, [:call, :__get_string, lab.to_sym]]
```
As the following line says, it's a nasty workaround, and it will be torn out as soon as I get a parser, so just ignore that... pretend it's not even there ;)

Now, the reason we wrap the call to __get_string is that otherwise it will be rewritten to a method call because there really is no such thing as function calls in Ruby, but for some of our low-level plumbing we need actual C-style function calls (such as for calling out to C code).

We're also changing strconst, the method used to get the "value" of a string constant as follows:
```ruby
-  def strconst str
-    lab = @string_constants[str]
-    return lab if lab
-    lab = @e.get_local
-    @string_constants[str] = lab
-    return lab
+  def strconst(a)
+    lab = @string_constants[a]
+    if !lab # For any constants in s-expressions
+      lab = @e.get_local
+      @string_constants[a] = lab
+    end
+    return [:addr,lab]
   end
```
What's left? Not much, really. You can see the basi c String class

Here's the important bit:
```ruby
class String
   def __set_raw(str)
     @buffer = str
   end
end

def __get_string(str)
   s = String.new
   %s(callm s __set_raw (str))
   s
end
```
It's worthwhile reading the comments in at the link above if you want to see more about where this is heading.

We'll flesh out the String class in coming parts, but this groundwork now means we're working on a "real" object instead of on a raw pointer to a c-style string, which means it's mostly a runtime library issue instead of requiring much in terms of compiler changes.

One of the things worth mentioning that is covered in one of the comments in lib/core/string.rb is that the whole `__get_string` and `__get_symbol` thing is a bit ugly. As I mentioned earlier in this part, hiding the low level implementation would be nice, and that includes preventing direct calls to `__get_string` and `_get_symbol`, and that can be done trivially by simply ensuring the Ruby code can't directly do function calls (as opposed to method calls). But that doesn't solve `String#__set_raw`. To fix that, I'll need to either remove the need for the method, or by making it possible to "hide" certain methods entirely from the Ruby code...

It's not a priority now, though - first we get things working, then we make it pretty.

What about String#to_sym? Well, this ought to do the trick:
```ruby
class String
  def to_sym
    __get_symbol(@buffer)
  end
end
```
So, that leaves us with a working define_method and %s(lambda) with support for variables to get attr_reader,attr_writer and attr_accessor working... We're heading down that road soon.