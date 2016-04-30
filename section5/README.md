#数字字面量；if .. then .. else
Last time I promised to post these more frequently, and blatantly failed to find the time... In return, I've taken the time to combine what would have been parts 5, 6 and 7, that by themselves would've been annoyingly short. Here goes:

###Handle number literals

So far our program have only handled numbers as an accidental by-product of not doing any type checking of any kind, and even then only if you have an external function that produces them.

So lets take a look at how gcc handles integers up to long long for C. Again we're restricting it to 32bit x86:
```cpp
    void foo1(unsigned short a) {}
    void foo2(signed short a) { }
    void foo3(unsigned int a) {}
    void foo4(signed int a) { }
    void foo5(unsigned long a) {}
    void foo6(signed long a) {}

    void foo7(unsigned long long a) { }
    void foo8(signed long long  a) {}

    int main()
    {
      foo1(1);
      foo2(2);
      foo3(3);
      foo4(4);
      foo5(5);
      foo6(6);
      foo7(7);
    }
```
I'm omitting most of the gcc output, you can run gcc -S yourself if you care. The interesting bit is the calls to the various functions, which end up like this:
```cpp
        movl    $1, (%esp)
        call    foo1
        movl    $2, (%esp)
        call    foo2
        movl    $3, (%esp)
        call    foo3
        movl    $4, (%esp)
        call    foo4
        movl    $5, (%esp)
        call    foo5
        movl    $6, (%esp)
        call    foo6
        movl    $7, (%esp)
        movl    $0, 4(%esp)
        call    foo7
```
In other words, for the purposes of a function call at least, gcc threats all types except long long as a single 32 bit value. We'll conveniently forget about long long and stick to values up to 32 bit then, since that means we can keep ignoring types completely.

Laziness is a virtue.

We'll also blatantly ignore floats? Why? Because you can easily write a compiler with only integer math, and so adding float support now would only be a waste of time, when things are bound to change down the line anyway.

Besides, when I was young we didn't have those fancy FPU's, sonny, and we did just fine anyway with fixed point calculations using integers.

So what do we actually change?

In #get_arg, before we handle string constants, we add this:
```ruby
      return [:int, a] if (a.is_a?(Fixnum))
```
In #compile_exp, we add this to use the return values from #get_arg:
```ruby
      elsif atype == :int then param = "$#{aparam}"
```
Aaand, we're done. That was almost to easy.

Here's a test:
```ruby
    prog = [:do,
      [:printf,"'hello world' takes %ld bytes\\n",[:strlen, "hello world"]],
      [:printf,"The above should show _%ld_ bytes\\n",11]
    ]
```
###Intermission: Some thoughts on primitive data types

"Pure" object oriented languages are great, but not at the relatively low level of the code generator, in my opinion. Whether or not you want to implement a pure OO language, I strongly believe that it's worth implementing primitives that can work on primitive data types first. Then you can decide whether or not to hide those primitives completely from the user at a later stage, or find ways to make them look like objects, or quietly convert them back and forth as needed behind the users back.

Note that the Matz Ruby Interpreter (MRI) does this: Numbers for example are quietly treated differently than "real" objects, but the interpreter does it best to hide this fact from the user. Personally I don't think MRI goes far enough.

###If ... then ... else

You don't get very far without some kind of conditional logic. some form of if .. then .. else ... exists in most languages that wants to be useful. For now we will implement [:if, condition, if-arm, else-arm], and do it the C way, in other words both 0 and null pointers evaluate to false, everything else to true.

Again a simple test:
```cpp
    void foo() {}
    void bar() {}

    int main()
    {
      if (baz()) {
        foo();
      } else {
        bar();
      }
    }
```
Giving the relevant part:
```cpp
        call    baz
        testl   %eax, %eax
        je  .L6
        call    foo
        jmp .L10
    .L6:
        call    bar
    .L10:
```
This is a pretty much universal pattern for compiling if .. then ... else, across a huge number of languages and architectures:

Calculate the expression that occurs in the condition
Test it in some way (here with "testl" - other variations that are common for various architectures is a "cmp" [compare] instruction or to subtract the register from itself). testl does a comparision of the left and right operands, and set various flags.
Next, do a conditional branch to the else arm. In other words, we're checking if the condition is NOT true. In this case it's done with "je" which translates to "jump on equal", in other words if the preceding comparison marked the result as equal (note that on most CPU's, lots of instructions set the condition codes, not just explicit tests).
Then execute the "if-arm".
Jump over the else-arm to after the end of the entire if..then..else construct.
Place a label for the else-arm, and the code to execute it.
Place a label marking the end.
There are lots of variations, such as reordering the if/else arms depending on estimates on which condition is most likely and knowledge of whether a missed or taken branch is cheapest on a particular architecture, but the above is all that's needed for now.

Again the compilation is deceptively simple - it's really just a direct translation of the above:
```ruby
      def ifelse cond, if_arm,else_arm
        compile_exp(cond)
         puts "\ttestl   %eax, %eax"
         @seq += 2
         else_arm_seq = @seq - 1
         end_if_arm_seq = @seq
         puts "\tje  .L#{else_arm_seq}"
         compile_exp(if_arm)
         puts "\tjmp .L#{end_if_arm_seq}"
         puts ".L#{else_arm_seq}:"
         compile_exp(else_arm)
         puts ".L#{end_if_arm_seq}:"
      end
```
It should be pretty self explanatory - it's basically calling compile_exp for each of the condition, if-arm and else-arm and weaving that into the above code, using @seq to generate unique labels.

To hook it in, we add this right after "return defun ...." in #compile_exp:
```ruby
      return ifelse(*exp[1..-1]) if (exp[0] == :if)
```
And here's a simple test:
```ruby
    prog = [:do,
      [:if, [:strlen,""],
        [:puts, "IF: The string was not empty"],
        [:puts, "ELSE: The string was empty"]
      ],
      [:if, [:strlen,"Test"],
        [:puts, "Second IF: The string was not empty"],
        [:puts, "Second IF: The string was empty"]
      ]
    ]
```
Here is the result

As usual you run it like this:
```sh
    $ ruby step5.rb >step5.s
    $ make step5
    cc    step5.s   -o step5
    $ ./step5
    ELSE: The string was empty
    Second IF: The string was not empty
    $
```
Some thoughts on loops

Unconditional loops are trivial to add to the language, but do we need to? The answer is strictly speaking no. We can already do loops with recursion, so why pollute the language?

For that to be a good solution, though, we need tail recursion elimination, and I'm not prepared to get into that right now. Tail recursion elimination, or it's more general form - tail call elimination - essentially means to identify cases where the exit out of a function consists of a call with the same or fewer arguments, followed by returning it's return value. If that is the case, you can reuse the current functions call frame for the arguments of the function you are calling, followed by a "jmp" instruction instead of "call". "jmp" doesn't push a return address onto the stack, and so when the function that you jump to return, it will return to the place the current function was called from, rather than to the current function.

This achieves a couple of things: First and foremost, the stack doesn't grow. Secondly, you save some precious cycles. Combined with appropriate other optimizations, with tail call elimination, you can implement a loop like this without being worried about running out of stack:
```ruby
    [:defun, :loop, [], [:do,
      [:puts, "I am all loopy"],
      [:loop]
    ],

    [:loop]
```
In other words, tail call elimination means that for any function on the form "(defun foo () (do bar foo))", stack usage goes from being proportional to the depth of the tail calls to being 1 frame deep at that point...

The above will run with the current compiler, but it'll quickly overrun the stack and crash. Not good.

I sense a disturbance in the force. All two geeks reading this shuddering at the thought of the stack growing with each iteration..

For now, we'll just ignore this, and not do anything to abuse the stack too much. Instead we'll later implement a proper loop construct as a primitive. At least for now - if/when I add tail call elimination we can consider ripping it out again and make it part of the runtime library instead.

So we can do non-terminating loops... That doesn't do us much good now does it?.

Well, we can already do while loops too:
```lisp
    (defun some-while-loop () (if condition (some-while-loop) ()))
```
Not very satisfying, but it works. It's too ugly to be an acceptable workaround, though, so a "while" primitive is on the list.

(As the realization that I've used Lisp like syntax sinks in, there's another disturbance in the force as the same two geeks as before shriek in agony and bolt for the door...)

I'm not a Lisp'er. I can't handle the parentheses overload... But Lisp syntax is convenient when the language doesn't really have a syntax of it's own yet - that'll come after bootstrapping when it's time to write a proper parser. Incidentally, most of what I've implemented and will implement will bear some kind of resemblance to badly broken LISP - as it turns out, Lisp constructs are very powerful if you can stand the syntax, and even if you don't want to program in it, learning more about Lisp and/or Scheme is worthwhile.

Personally I don't heed my own advice, and most of the Lisp-like ideas I'll be using are probably based on incomplete understanding from the very limited time I've spent looking at Lisp.

Next time: Anonymous functions, maybe more.