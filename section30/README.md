# 比较操作符

Where we left off last time, we had everything but comparison operators and &amp;&amp;, ! sorted out. When I started replacing runtime.c, I expected to do it in one go, as I've mentioned, but as you have seen there was quite a bit to cover.

This is it, though. runtime.c will finally go, at any cost.

I'll try to keep this short since we've covered most of the steps in the previous parts, and only address differences affecting the remaining operators.

(Note that I'll reduce my "buffer" which is currently about 5 months, as I've been able to consistently write a part ever month since April; that means I'll aim to post the next part in the next week or two, and will probably post two in January too)

### Compiling comparisons

Let us dive right in and take a look at the first of the comparison operators first, as they are all very similar:
```asm
    ne:
        pushl   %ebp
        movl    %esp, %ebp
        movl    8(%ebp), %eax
        cmpl    12(%ebp), %eax
        setne   %al
        movzbl  %al, %eax
        popl    %ebp
        ret
```
This isn't too bad. After the normal setup, one operand is copied into %eax and cmpl is used to compare the second one with %eax.

Then setne is used to set its operand to 1 if the condition flags set by cmpl match the ne (not equal) condition, and there are equivalent variants for a long range of other conditions that matches our comparison operators neatly. %al is part of the ugliness of x86 - it is a register descriptor that maps to the least significant byte of %eax.

movzbl is then used to clear the remaining bytes.

All of the comparisons follow this patterns, with le, ge, ne mapping one to one to set(something) opcodes, lt mapping to setl, gt to setg and eq to sete.

First lets write a simple test program:
```ruby
    %s(do
       (if (lt 2 5) (call puts ("2 < 5")))
       (if (lt 1 2) (call puts ("1 < 2")))
       (if (le 2 2) (call puts ("2 <= 2")))
       (if (ge 2 2) (call puts ("2 >= 2")))
       (if (gt 3 2) (call puts ("3 > 2")))
       (if (eq 2 2) (call puts ("2 == 2")))
       (if (ne 3 2) (call puts ("3 != 2")))
    
       (puts "false: (no output expected)")
    
       (if (lt 5 2) (call puts ("5 < 2")))
       (if (lt 2 2) (call puts ("2 < 2")))
       (if (le 3 2) (call puts ("3 <= 2")))
       (if (ge 1 2) (call puts ("1 >= 2")))
       (if (gt 2 2) (call puts ("2 > 2")))
       (if (eq 4 2) (call puts ("4 == 2")))
       (if (ne 2 2) (call puts ("2 != 2")))
    
       (puts "done")
    )
```
The expected output is:
```
$ /tmp/testcmp 
2 = 2
3 &gt; 2
2 == 2
3 != 2
false: (no output expected)
done
```
The actual compilation is easy. We make use of #compile_2 that we
added in the previous part to compile both operands in turn and preserve
the result, then compare it, and set and extend %eax accordingly
(in bba2c93)
```ruby
      def compile_comparison(scope, op, left, right)
        compile_2(scope,left,right) do |reg|
          @e.cmpl(:eax,reg)
          @e.emit("set#{op.to_s}".to_sym, :al)
          @e.movzbl(:al,:eax)
        end
      end
    
      def compile_eq(scope,left,right)
        compile_comparison(scope, :e, left,right)
      end
```
I've only given #compile_eq as an example here, but the code is pretty much identical for the rest - only substituting the op argument according to the "set[op]" variant we want to use.

The rest is a trivial commit to strip these operators out of runtime.c, and register them wth the Compiler class, that you can see in ce6deb9

#### A first stab of Ruby level comparisons

Our first problem is that most of Ruby's comparisons are mixins in the Comparable module. We'll ignore that for now, and simply implement integer to integer comparisons directly in Fixnum and then generalize. We'll want specializations of the operators for integers anyway, as we don't want to fall back on the `` operator for integer to integer comparisons.

First a Ruby version of our tests from above:
```ruby
    if 2 < 5; puts "2 < 5"; end
    if 1 < 2; puts "1 < 2"; end
    if 2 <= 2; puts "2 <= 2"; end
    if 2 >= 2; puts "2 >= 2"; end
    if 3 > 2; puts "3 > 2"; end
    if 2 == 2; puts "2 == 2"; end
    if 3 != 2; puts "3 != 2"; end
    
    puts "false: (no output expected)"
    
    if 5 < 2; puts "5 < 2"; end
    if 2 < 2; puts "2 < 2"; end
    if 3 <=2; puts "3 <= 2"; end
    if 1 >=2; puts "1 >= 2"; end
    if 2 > 2; puts "2 > 2"; end
    if 4 == 2; puts "4 == 2"; end
    if 2 != 2; puts "2 != 2"; end
    
    puts "done"
```
The we add the operators to Fixnum. It's similar to what we did for Fixnum.+ etc, except we can't turn it back into a Fixnum object, since 0 would never get interpreted as false. Since we don't have the proper true and false yet, we just do this (in a844694):
```ruby
      def == other
        %s(eq @value (callm other __get_raw))
      end
```
Note that this will break hopelessly in a number of situations, such as if you try to compare, say, a string with a Fixnum. In Ruby this should result in an ArgumentError exception, but here it'll cause a crash or nasty silent bugs. We'll quash those issues later.

#### Not and and

The choice above to not treat true and false properly at the Ruby level leaves us in a bit of a pickle. The input to these will often be the outcome of another comparison. If the input is another expression the effect will potentially be entirely different than if checking against variables set to true or false directly. Many results will be totally bogus.

As a result I finally decided on writing this to punt entirely on !. &amp;&amp; is in a similar situation, and furthermore it is easy to "emulate" with if, and so we'll punt on that too.

So we'll defer these until we can get a handle on true, false and nil.

We'll however still finally get rid of runtime.c (in 28d246c). I won't cover the specifics - look at the commit.