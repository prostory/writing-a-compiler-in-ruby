# 一个内建的"while"语句

> UPDATE: Fixed a bug in compile_while

Ok, so I know it's been far too long. Lots going on at the moment... This is a very short part, but I'll try to get the next part cleaned up in the next few days (teaser at the end...)

### Implementing 'while' as a primitive

I like small language cores, and it really appeals to me to implement control structures as methods rather than hardcoding them into the language. As we've seen, 'while' can be implemented fairly easily in terms of a quite basic language. We could streamline it even more by reducing the verbiage needed to pass anonymous functions or by adding a Lisp style macro facility.

There's an added complication: Using lambda's that are not proper closures, which is the case for us so far, means there's no way for the body to access external variables. That makes the while loop pretty useless. The "proper" way of fixing that would be to actually implement proper closures, and I promise we'll get there, but not right now - there are other pieces to get in place first.

So I'm biting the bullet and adding a built in "while" construct for now. It also serves the purpose of showing how it's usually done - it's quite simple really.

As usual we start with a simple C test to see how to do this:
```cpp
int main() {
  while (foo()) {
    bar();
  }
}
```
gcc -S, and this is the relevant part:
```cpp
   jmp .L2
.L3:
    call    bar
.L2:
    call    foo
    testl   %eax, %eax
    jne .L3
```
Notice how the condition is placed at the end, and we start by jumping to the end? The alternative is to place the conditional check at the start, reversing the test and making it jump to after the end of the loop, and then placing an unconditional jump after the body, like the example below, but that results in one extra branch instruction per iteration of the loop:
```cpp
.L3:
    call    foo
    testl   %eax, %eax
    je .L2
    call    bar
    jmp .L3
.L2:
```
This is not a big deal since performance isn't a concern right now, but the distinction here doesn't add any complexity to speak of. So here's what we do:
```ruby
      def compile_while(scope, cond, body)
        start_while_seq = @seq
        cond_seq = @seq + 1
        @seq += 2
        puts "\\tjmp\\t.L#{cond_seq}"
        puts ".L#{start_while_seq}:"
        compile_exp(scope,body)
        puts ".L#{cond_seq}:"
        var = compile_eval_arg(scope,cond)
        puts "\\ttestl\\t#{var}, #{var}"
        puts "\\tjne\\t.L#{start_while_seq}"
        return [:subexpr]
      end
```
Then we add it to #compile_exp:
```ruby
       return compile_while(scope,*exp[1..-1]) if (exp[0] == :while)
```
And a test:
```ruby
    prog = [:do,
      [:call, [:lambda, [:i],
          [:while,
            :i,
            [:do,
              [:printf, "Countdown: %ld\\n", :i],
              [:assign, :i, [:sub, :i, 1]]
            ]
          ]
        ], [10] ]
      ]
```
### Wrapping it up

Like last time, I've tar'ed up the files for step 9 together with a Makefile here. If you have Ruby, make and gcc installed you should be able to just untar it and run make to get the "step9" test binary:
```sh
[vidarh@dev step9]$ make
ruby step9.rb &gt;step9.s
as   -o step9.o step9.s
cc    -c -o runtime.o runtime.c
cc   step9.o runtime.o   -o step9
[vidarh@dev step9]$ ./step9 
Countdown: 10
Countdown: 9
Countdown: 8
Countdown: 7
Countdown: 6
Countdown: 5
Countdown: 4
Countdown: 3
Countdown: 2
Countdown: 1
[vidarh@dev step9]$ 
```
### Next time

Next time we'll start putting together and going through a simple "text mangler" that'll let us "parse" a lisp like syntax and generate a Ruby script that runs the compiler on the equivalent program...

It will turn this (not the complete program):
```lisp
    (defun parse () (let (c sep expr) 
      (assign sep 0)
      (assign expr 0)
      (while (and (ne (assign c (getchar)) -1) (ne c 41))
            (if (eq c 40)
              (do (copy_expr) (assign sep 1))
              (if (eq c 59)
                    (parse_comment)
                    (if (eq c 34) 
                      (do (parse_quoted c) (assign sep 1))
                      (do
                       (if (and (isspace c) sep) (do (printf ",") (assign sep 0)))
                       (if (and (isalnum c) (not sep)) 
                               (do (assign sep 1) (if (not (isdigit c)) (printf ":")))
                             )
                       (putchar c)
                       )
                      )
                    )
              )
            )
      ))
```
into this plus driver code to run the compiler:
```ruby
    [:defun, :parse, [], [:let, [:c, :sep, :expr], 
      [:assign, :sep, 0],                             
      [:assign, :expr, 0],
      [:while, [:and, [:ne, [:assign, :c, [:getchar]], -1], [:ne, :c, 41]],
            [:if, [:eq, :c, 40],
              [:do, [:copy_expr], [:assign, :sep, 1]],
              [:if, [:eq, :c, 59],
                    [:parse_comment],
                    [:if, [:eq, :c, 34], 
                      [:do, [:parse_quoted, :c], [:assign, :sep, 1]],
                      [:do,
                       [:if, [:and, [:isspace, :c], :sep], [:do, [:printf, ","], [:assign, :sep, 0]]],
                       [:if, [:and, [:isalnum, :c], [:not, :sep]], 
                               [:do, [:assign, :sep, 1], [:if, [:not, [:isdigit, :c]], [:printf, ":"]]],
                             ],
                       [:putchar, :c],
                       ],
                      ],
                    ],
              ],
            ],
      ]],
```
By the end of it it will process itself and spit out the equivalent parse-tree encoded in Ruby. Note how I've put "parse" in quotes etc.. It hardly deserves to be described as a parser - we'll get to a real parser a few steps further out.

Note that I am not going to go down towards writing a Lisp, but I figured since I've used a Lisp like syntax so far for examples, it would be a good test to see whether we're far enough along to do something remotely useful yet. It did need a couple of helper functions beyond what's present so far to get a decent result, but not much. The "real" parser will be for a much more Ruby-ish syntax.