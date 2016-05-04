# 测试编译器：原始的”解析器“

Uh, yeah. So much for posting the next part in a few days. I think I'll stop trying to second guess when I'll next have time (but sine I'm going off to Norway for vacation for a week, it's a safe bet the next part won't show up until later than that). For those who care (anyone? anyone at all? didn't think so), I've been very busy lately, both with a big new project at work that'll see us relaunching the websites for three large restaurant chains in the UK. The exciting part of it isn't the code - that's pretty straight forward - but the infrastructure we're building, which is giving me a great opportunity to set up a very resilient OpenVz based setup with every single component virtualized. My spare time has all been taken up by other projects which I might say more about some other time... Anyone, back to the subject at hand...
### What can we do so far?

Each step so far has included minor bits and pieces to test specific features. But it's hard to get an idea for how complete a language is becoming without writing something a little bit more substantial. In this case I've chosen to write a "text-mangler" that will "parse" an s-expression like (Lisp like) syntax and turn it into a Ruby script that will run the compiler with the equivalent program. It will not be a proper parser, but merely a very simple case of rewriting pieces of text to turn something like this:
```lisp
(foo "bar" 1 2 3)
```
into something like this:
```ruby
[:foo, "bar", 1,2,3]
```
Simple enough to do, but it'll test quite a few things.
### Preliminaries

First of all I'll separate the compiler from the "driver" - the little snippet of code containing the actual program, and instantiating and running the compiler. Essentially the entire Compiler class gets moved into a separate file. Here's a really stripped down starting point that illustrates how to use it, and how we'll start producing the "parser":
```ruby
require 'compiler'

prog = [:do,
  [:puts, "require 'compiler'\\n"],
  [:puts, "prog = [:puts,'Program goes here']"],
  [:puts, ""],
  [:puts, "Compiler.new.compile(prog)"]
]

Compiler.new.compile(prog)
```
Notice that the program included here reproduces another program with the same structure. If run, it'll output another driver program containing a basic program itself. Our goal is to expand on this program so that it will read a Lisp like program and output something almost exactly like above. The program above, in fact, should be exactly what we'd expect to get if "parsing"
```lisp
   (puts "Program goes here")
```
### The rules

What exactly will we "parse"? We'll turn
[ ... ] into ( .... )
foo into :foo
123 into 123
a, b into :a :b
"foo" into "foo"
There are lots of copouts here:
We won't tokenize properly, so lots of invalid input will create even more broken output
It won't handle escaped double quotes at all.
No error checking.
Did I mention no error checking?
This is throwaway code - I'm not intending to write a Lisp compiler - so I'm even less inclined than usual to spend time writing something robust. Consider this a test of the capabilities of the compiler so far, and a way of planning out what to add next. Nothing more.
### So lets write the damn thing

First of all: This is an exercise in bootstrapping. First we write the "parser" in our current syntax. Then we'll manually translate it into the lisp like syntax it translate. Then we'll finally use itself to regenerate it's own Ruby syntax. This roundtrip will also serve as a test:
Make it parse itself to generate Ruby.
Run the Ruby to generate assembler.
Assemble the parser.
Run the result on itself again.
Compare the Ruby code generated in the first iteration with the one generated in the second. If they're not identical something is broken.
This is an IMPORTANT series of steps to keep in mind. Whenever you bootstrap a compiler for a new language or a part of one in the language it compiles so it can process it's own source, you want to make VERY sure that each step can process the next version of itself correctly, and that you archive every single version, since it's very easy to get yourself into a situation where you suddenly have a lot of uncompilable broken code when iterating the syntax and semantics of a new language. First we spit out a driver as described above:
```ruby
require 'compiler'

prog = [:do,
  [:defun, :parse, [:c,:sep], [] ],

  [:puts, "require 'compiler'\\n"],
  [:puts, "prog = [:do,"],
  [:parse, 0,0],
  [:puts, "]\\n"],
  [:puts, "Compiler.new.compile(prog)"]
]

Compiler.new.compile(prog)
```
The code above defines a "parse" function for future use. It then outputs a prolog, calls the parse function, and outputs a tiny epilog, just like the first part of writing the compiler itself. Next we need to start filling in the parse function:
```ruby
 [:defun, :parse, [:c,:sep],
    [:while, [:and, [:ne, [:assign, :c, [:getchar]], -1], [:ne, :c, 41]], [:do,
         ...
    ]]
 ]
```
Hard to read? What about if I rewrite it like this:
```ruby
defun parse c,sep
  while (c=getchar) != -1 and c != 41
    ...
  end
end
```
Character 41 (decimal) is incidentally right paranthesis. It's a shortcut to make it trivial to use this function recursively to parse expressions. So "c" is obviously the current character. What about "sep"? Sep will be use later to indicate whether or not to output "," to separate sub-expressions. So why are they passed as arguments? Well, there's the first reminder for this step: We lack local variables. They'll be added shortly. Next piece. Inside the while loop we add this:
```ruby
     [:if, [:eq,:c, 40], [:do,
            [:printf, "["],
            [:parse,0,0],
            [:printf, "]"],
            [:assign, :sep, 1]
          ], ... (else bit goes here)
     ]
```
In other words: If we see a '(' (ASCII 40), we output '[', call parse() recursively, output ']' and indicate that a comma separator is needed on the next sub expression. Then:
```ruby
      [:if, [:eq, :c, 34], [:do,
              [:putchar,34],
              [:parse_quoted,0],
              [:putchar,34],
              [:assign, :sep, 1]
            ], (... else goes here)
```
If we see a double quote (ASCII 34), output double-quotes, call a new function "parse_quoted" to deal with it, output double-quotes again, and indicate we need a separator. And here's the last chunk of "parse":
```ruby
           [:do,
              [:if, [:and, [:isspace, :c], :sep], [:do,
                  [:printf, ","],
                  [:assign, :sep, 0]
                ]
              ],
              [:if, [:and, [:isalnum, :c], [:not, :sep]], [:do,
                    [:assign, :sep, 1],
                    [:if, [:not, [:isdigit, :c]],[:printf,":"]]                 
                ]
              ],
              [:putchar, :c]
            ]
```
If "c" is whitespace (calling isspace() from the c library) and "sep" is not 0, we output the separator, and clear the flag. Note that this relies on Ruby ignoring trailing "," in array declarations, so [1,2,3,] works the same way as [1,2,3]. Then, if "c" is alphanumeric (isalnum(c)) and "sep" isn't set, we'll indicate we need a separator and specifically if it's not a digit we'll assume it needs to be turned into a Ruby symbol and output ':', then we'll finally output the character. The reason we check to see that "sep" isn't set is to avoid outputing ":" in front of every letter. That leaves parse_quotes, which looks like this:
```ruby
  [:defun, :parse_quoted, [:c],
    [:while, [:and, [:ne, [:assign, :c, [:getchar]], -1], [:ne, :c, 34]], [:do,
        [:putchar, :c]
      ]
    ]
  ],
```
or:
```ruby
defun parse_quoted c
  while (c=getchar) != -1 and (c != 34)
      putchar c
  end
end
```
Put it all together, and this is the result:
```ruby
require 'compiler'

prog = [:do,
  [:defun, :parse_quoted, [:c],
    [:while, [:and, [:ne, [:assign, :c, [:getchar]], -1], [:ne, :c, 34]], [:do,
        [:putchar, :c]
      ]
    ]
  ],
  [:defun, :parse, [:c,:sep],
    [:while, [:and, [:ne, [:assign, :c, [:getchar]], -1], [:ne, :c, 41]], [:do,
        [:if, [:eq,:c, 40], [:do,
            [:printf, "["],
            [:parse,0,0],
            [:printf, "]"],
            [:assign, :sep, 1]
          ],
          [:if, [:eq, :c, 34], [:do,
              [:putchar,34],
              [:parse_quoted,0],
              [:putchar,34],
              [:assign, :sep, 1]
            ],
            [:do,
              [:if, [:and, [:isspace, :c], :sep], [:do,
                  [:printf, ","],
                  [:assign, :sep, 0]
                ]
              ],
              [:if, [:and, [:isalnum, :c], [:not, :sep]], [:do,
                    [:assign, :sep, 1],
                    [:if, [:not, [:isdigit, :c]],[:printf,":"]]
                ]
              ],
              [:putchar, :c]
            ]
          ]
        ]
      ]
    ]
  ],
  [:puts, "require 'compiler'\\n"],
  [:puts, "prog = [:do,"],
  [:parse, 0,0],
  [:puts, "]\\n"],
  [:puts, "Compiler.new.compile(prog)"]
]

Compiler.new.compile(prog)
```
Completely unreadable and a nightmare to debug. So lets translate it into our Lisp like syntax and see if it can process itself. Some regexps and manual massage later:
```lisp
(do
 (defun parse_quoted (c)
   (while (and
           (ne (assign c (getchar)) -1)
           (ne c 34))
     (do (putchar c))
     )
   )

 (defun parse (c sep)
   (while (and
           (ne (assign c (getchar)) -1)
           (ne c 41))
     (do
      (if (eq c 40)
          (do
           (printf "[")
           (parse 0 0)
           (printf "]")
           (assign sep 1)
           )
        (if (eq c 34)
            (do
             (putchar 34)
             (parse_quoted 0)
             (putchar 34)
             (assign sep 1)
             )
          (do
           (if (and (isspace c) sep)
               (do
                (printf ",")
                (assign sep 0)
                )
             )
           (if (and (isalnum c) (not sep))
               (do
                (assign sep 1)
                (if (not (isdigit c)) (printf ":"))
                )
             )
           (putchar c)
           )
          )
        )
      )
     )
   )

  (puts "require 'compiler'\\n")
  (puts "prog = [:do,")
  (parse 0 0)
  (puts "]\\n\\n")
  (puts "Compiler.new.compile(prog)")
)
```
Yikes. More parenthesis than in actual Lisp. I wouldn't call it readable (particularly the lack of an "else" keyword or similar makes it hard to see where an else block begins, but it's an improvement. The real goal was to write a slightly larger test, anyway. Lets test it by running it on "itself", then compiling the result, and run that on itself, and check that they match:
```sh
$ make parser
ruby parser.rb >parser.s
as   -o parser.o parser.s
cc    -c -o runtime.o runtime.c
cc   parser.o runtime.o   -o parser
$ ./parser parser2.rb
$ make parser2
ruby parser2.rb >parser2.s
as   -o parser2.o parser2.s
parser2.s: Assembler messages:
parser2.s:238: Warning: unterminated string; newline inserted
parser2.s:239: Warning: unterminated string; newline inserted
cc   parser2.o runtime.o   -o parser2
$ ./parser2 parser3.rb
$ diff -B parser2.rb parser3.rb
$
```
Nice. Next time we'll go back to the compiler and do some refactoring, before adding some functionality (such as local variables) that would make the above example quite a bit cleaner. Once that's done it's time for a real parser for a much nicer syntax. In the meantime the archive of the files for this round can be found here
