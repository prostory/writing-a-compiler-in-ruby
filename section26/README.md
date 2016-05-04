# 尝试在STABS支持（debugging/gdb）

I published the last article in this series in June 2010, after a long break. At the time of finally making a serious start to this part, it was April 2013, and as of the time I am editing and putting the final touches on this, it is July 2013, and I assume most of you have probably given up waiting by now.

I've kept meaning to publish more parts, and other parts of life have kept stopping it.

Incidentally, my son doesn't drool on the laptop anymore (see part 25) - now he hogs it to watch Youtube videos, particularly of anything Lego (which he then kindly requests that I build, often suggesting I build 2000-piece buildings with a pile of 50 or so bricks), since it's a great way of ensuring I give him my full attention when I can't use the laptop.

So my hacking is still limited in time, but I've finally managed to claw back enough to restart this series.

Anyway. I actually started writing several parts, before I finally got started on this one. As of publishing this article, I have 6 finished parts, and one halfway complete, in order to give me a buffer to prevent further gaps as long as this.

My intention is to post new parts every 5-6 weeks. I may adjust that up or down a bit as I see how regularly I manage to write additional parts to the series.

This article also represents a bit of a step to the side from the define_method stuff I was in the middle of doing last time. Partly because I want to start with a bit of a clean slate, partly because better debugging support is long overdue, and partly because in retrospect, define_method was a big bite to chew over too soon - it requires a surprising amount of scaffolding unless you want to do it in exceedingly hacky ways.

In fact, arguably going for define_method was a mistake, especially given this series original focus on "bottom up" development. We need a number of lower level features in place, and I got too excited about going straight for a full Ruby compiler. Pretty much exactly the type of complexity I wanted to avoid when I first set out by starting at the bottom.

I'll be back to define_method and its pre-requisites later, but my current plan is to seriously deal with it again somewhere around 8-10 parts out... Yeah.

### Taking a stab at STABS support

If you've toyed with what has been covered in previous parts, you might at one point or other have despaired over how to debug the compiler output. Some debugger support would be nice. ANY debugger support.

Especially given that it's easy to make it output programs that will crash...

STABS is a debugging information format supported by GDB.

It's fairly simple to generate, and flexible enough to allow us to add enough information to generate reasonable stack backtraces.

Unfortunately, it's not easy to decipher by looking at gcc output. So we'll take a stab at adding some basic stabs information to an intentionally crashing C program, with the aid of some GDB documentation
```cpp
    #include <stdio.h>
    
    int foo() {
      int v;
      printf ("bar");
        
      v = *(long *)0;
    }
          
    int main() {
      foo();
    }
```
Compiling this with gcc -gstabs -S -o test1.s test.c on a 32 bit Linux system gives a reasonable baseline. We'll strip it down as far as we can, and then add support for the few bits remaining to the Ruby compiler.

We want to do the minimum possible to make it possible to at least get a decent GDB backtrace, but not (for now) bother with all the complexities of proper type handling etc. that is needed for full gdb integration.

this is an example output from a GDB session with the above program:
```sh
$ gdb ./test
nGNU gdb 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http:>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu"...
(gdb) run
Starting program: /home/vidarh/src/compiler-experiments/test 

Program received signal SIGSEGV, Segmentation fault.
0x080483bb in foo () at test.c:8
v = *(long *)0;
(gdb) bt
#0  0x080483bb in foo () at test.c:8
#1  0x080483d8 in main () at test.c:12
(gdb) 
```
This is pretty much the extent of what we'll aim for in the first round.

So lets start by looking at bits and pieces of the output of gcc with -gstabs:
```asm
    .file   "test.c"
    .stabs  "test.c",100,0,2,.Ltext0
    .text
    .Ltext0:
```
It is pretty obvious that this tells us that the source file is test.c

But what does the 100,0,2,.Ltext0 mean? A lot can be said about Stabs, but "user friendly" does not spring to mind. Thankfully the documentation is quite decent, and some experimentation supplies the rest.

The .Ltext0 is simply a reference to the first address in the binary that stems from "test.c".

100 is the stabs symbol "N_SO" which contains the name and path of the source file.

The "2" is N_SO_C - it designates the source language. C in this case. I don't think there is one for Ruby, but we can either specify 0 for no language, or, as we will adhere to C calling conventions for the most part, we can use 2 in case it buys us anything extra from gcc (I honestly don't know - I'm new to stabs too).
```asm
    .stabs  "int:t(0,1)=r(0,1);-2147483648;2147483647;",128,0,0,0
```
The next interesting entry is a bunch of these. They define types to allow the debugger to more precisely manipulate data from the program.

We're going to ignore these for now, since our initial focus is just getting the backtrace so we at least know where we're crashing.

Then this:
```asm
    .stabs  "/usr/include/stdio.h",130,0,0,0
```
130 is N_BINCL. It marks the start of an include file. N_BINCL is paired with N_EINCL:
```asm
    .stabn  162,0,0,0
```
N_BINCL and N_EINCL can be nested. These are interesting for us since we'll (for now at least) not entertain the idea of handling separate compilation units, but rather generate a single static executable. As such, all "require"'d files that we resolve at compile time (sigh, there we have an ugly beast rearing it's head - figuring out what to load at compile time vs. runtime, but that's for some later time) will be treated as include files.

In this case, all that gets included is type definitions.

Finally, after lots more type definitions and the string "bar", we get the function foo(), which looks like this full of stabs:
```asm
        .stabs   "foo:F(0,1)",36,0,0,foo
    .globl foo
        .type    foo, @function
    foo:
        .stabn   68,0,3,.LM0-.LFBB1
    .LM0:
    .LFBB1:
        pushl %ebp
        movl  %esp, %ebp
        subl  $24, %esp
        .stabn   68,0,5,.LM1-.LFBB1
    .LM1:
        movl  $.LC0, (%esp)
        call  printf
        .stabn   68,0,7,.LM2-.LFBB1
    .LM2:
        movl  $0, %eax
        movl  (%eax), %eax
        movl  %eax, -4(%ebp)
        .stabn   68,0,8,.LM3-.LFBB1
    .LM3:
        leave
        ret
        .size    foo, .-foo
        .stabs   "v:(0,1)",128,0,0,-4
        .stabn   192,0,0,.LFBB1-.LFBB1
```
gdb will happily(?) rely on the "basic" function prolog we're already using, so we can ignore most of the new stubs, though it means gdb won't have information about the number of arguments.

The critical bit for us to get a basic backtrace is this:
```asm
    foo:
      .stabn  68,0,3,.LM0-.LFBB1
    .LM0:
    .LFBB1:
```
What this does is indicate line numbers. In this case the line number is 3.

The .LM0 and .LFBB1 nonsense is there to generate a relative file number. .LFBB1 is a marker for the start of the function, and .LM0 in this case refers to the start of the line the stabs entry refers to.

This is really all we care about.

### Making the compiler generate stabs

You will find all of these changes and the remaining bits and pieces on Github

The first step we take is to add a new node in the AST: required. This node is used to allow us to easily identify the inclusion of files later, during processing of the AST, so that we can indicate the correct source filename in the stabs:
```ruby
      def require q
        return true if @@requires[q]
        STDERR.puts "NOTICE: Statically requiring '#{q}'"
        # FIXME: Handle include path                                                                                                                                  
        paths = rel_include_paths(q)
        f = nil
        paths.detect { |path| f = File.open(path) rescue nil }
        error("Unable to load '#{q}'") if !f
        s = Scanner.new(f)
    -    @@requires[q] = Parser.new(s, @opts).parse(false)
    +    pos = position
    +    expr = Parser.new(s, @opts).parse(false)
    +    @@requires[q] = E[pos,:required, expr]
      end
```
We also add the "required" node to the @@keywords set of the Compiler class, and add a method to process it:
```ruby
    def compile_required(scope,exp)
      @e.include(exp.position.filename) do
        compile_exp(scope,exp)
      end
    end
```
We add the include method to the Emitter class to indicate just that:
```ruby
      def include(filename)
        @section += 1
        @out.emit(".stabs  \"#{filename}\",130,0,0,0")
        ret = yield
        @out.emit(".stabn  162,0,0,0")
        @section -= 1
        comment ("End include \"#{filename}\"")
        ret
      end
```
We also call this from output_functions in the Compiler class, which is where we spit out all the functions we generate from various methods etc:
```ruby
          # FIXME: Would it be better to output these grouped by source file?
    
        if func.body.is_a?(AST::Expr)
            @e.include(func.body.position.filename) do
                @e.func(name, func.rest?, func.body.position) { compile_eval_arg(FuncScope.new(func), func.body) }
            end
          else
            @e.func(name, func.rest?, nil) { compile_eval_arg(FuncScope.new(func), func.body) }
          end
        end
```
This bit ensures that we output the suitable BINCL and EINCL tags around function definitions too.

#### Line numbers

However, we need more. This handles inclusion. But we also want line numbers.

We handle that in two different places. First in compile_eval_arg
```ruby
      def compile_eval_arg(scope, arg)
        if arg.respond_to?(:position) && arg.position != nil
          pos = arg.position.inspect
          if pos != @lastpos
            @e.lineno(arg.position)
```
Notice that we only call this if a cached @lastpos doesn't change. I'm not sure if I'm happy with this. It intermingles the language handling with debugging.

There are a few alternatives here: * Move it into the Emitter. I see that as a bad idea because the Emitter will change per CPU target if I ever choose to retarget it. * Add a policy object that gets called every time, and that can be swapped out * "Just" move the logic to a separate method. * Add an intermediary between the lower level code generation of the Emitter, and higher level concerns, such as debug output etc.

I'm not yet decided, so I leave it at this for now.

We also add a similar line to compile_exp:
```ruby
    @e.lineno(exp.position) if exp.respond_to?(:position) && exp.position
```
Note in this case we always send the line number if we know if from the parser.
And there's where most of the mess you'll find in this parts commit comes from:

We want to ensure that the AST nodes store the position wherever possible. That means moving more of the parser code from standard Array instances to AST::Expr, or E as it is aliases most places in the compiler.

### Line numbers in the Emitter

This took some experimentation to get right, since it was by no means clear to me from the documentation exactly how it was meant to work, but gcc seems happy enough with this:
```ruby
      def lineno(position)
        @lineno ||= nil
        @linelabel ||= 0
        if position.lineno != @lineno  
          # Annoyingly, the linenumber stabs use relative addresses inside include sections
          # and absolute addresses outside of them.
          #                                   
          if @section == 0
            @out.emit(".stabn  68,0,#{position.lineno},.LM#{@linelabel}")
          else
            @out.emit(".stabn  68,0,#{position.lineno},.LM#{@linelabel} -.LFBB#{@curfunc}")
          end
          @out.label(".LM#{@linelabel}")
          @linelabel += 1
          @lineno = position.lineno
        end
      end
```
### Function boundaries and file headers

To wrap everything up, we add some bookkeeping to the func method of the Emitter, indicating the line numbers of the function, as mentioned earlier:
```ruby
    +  def func(name, save_numargs = false, position = nil)
    +    @out.emit(".stabs  \"#{name}:F(0,0)\",36,0,0,#{name}")
        export(name, :function) if name.to_s[0] != ?.
        label(name)
    +
    +    @funcnum ||= 1
    +    @curfunc = @funcnum
    +    @funcnum += 1
    +
    +    lineno(position) if position
    +    @out.label(".LFBB#{@curfunc}")
    +
        pushl(:ebp)
        movl(:esp, :ebp)
        pushl(:ebx) if save_numargs
        yield
        leave
        ret
        emit(".size", name.to_s, ".-#{name}")
    +    @scopenum ||= 0
    +    @scopenum += 1
    +    label(".Lscope#{@scopenum}")
    +    @out.emit(".stabs  \"\",36,0,0,.Lscope#{@scopenum}-.LFBB#{@curfunc}")
    +    @curfunc = nil
      end
```
And to Emitter#main to indicate the source file:
```ruby
    -  def main
    +  def main(filename)
    +    @funcnum = 1
    +    @curfunc = 0
         if @basic_main
           return yield
         end
     
    +    @out.emit(".file \"#{filename}\"")
    +    @out.emit(".stabs \"#{File.dirname(filename)}/\",100,0,2,.Ltext0")
    +    @out.emit(".stabs \"#{File.basename(filename)}\",100,0,2,.Ltext0")
         @out.emit(".text")
```
#### Some notes, and what's up next

There are a bunch of additional things in the commit, and unfortunately I've not committed this piecemeal, as this change was made over a period of a year (!) while I was trying to get time to continue this serious, and repeatedly shelved it.

I'm happy to answer questions in the comments if there are things that are unclear. Thank you for the kind encouragement over my hiatus...

As for what is next, the next four parts will cover replacing runtime.c with code generation and method implementations, partly because I want to get rid of runtime.c, partly because functionally, the more complete method dispatch and the more we try to write code in Ruby, the current handling of it fails horribly because we try to intermingle code in the low level s-expression based intermediate representation and higher level Ruby code.

Replacing it should suddenly make quite a bit more code work, and make further extensions easier to make without constantly dipping down into the murky land of %s()...