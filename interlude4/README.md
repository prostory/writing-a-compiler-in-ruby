# 怎样实现闭包

This is a sort-of interlude to my regular compiler series.

The goal is to give a brief overview of some techniques for implementing closures in a programming language. I will use C for my examples, mostly because it's low level enough that a further translation to assembler etc. is straight forward, and many compilers target C directly anyway.

Before we start, most of the code examples in this post are available in this Gist so you don't need to cut and paste bits and pieces (which won't work anyway, as the text below will omit details such as header includes)

A closure) is in it's simplest form a block of code that can be passed around as a value, and that can reference variables in the scope it was created in even after exiting from that scope.

(For a more formal description look at the Wikipedia page linked above)

A Ruby block forms a closure, for example:
```ruby
    def foo x
      printf "x is %d\n",x
      lambda do
        x += 1
        printf "block: x is %d\n",x
      end
    end
    
    c = foo(5)
    c.call
    c.call
```
Every time foo is called it will return a closure that has access to the x that was passed in to foo. In the example, x will start out at 5 and get incremented to 7.

The most important implication of this is that any variables used in the closure must be guaranteed to live as long as the closure does.

We'll get back to this example throughout this article.

There are a number of ways of handling this, for example:

* Instead of a traditional stack, put activation frames (arguments and local variables) for function/method calls on the heap, as a linked list. When creating a closure, you just keep a reference to it. When returning from a function you unlink the frame from the ones below. Presto, the gc handles the remaining dirty details. Some variations of this is called a spaghetti stack
* Rewriting. You can rewrite any function that may create a closure to create a separate heap allocated environment, and to copy/locate all arguments and variables that may be reused into it. The environment can safely be returned with the closure.
* Copy. You can copy the current activation frame into a separate closure environment when returning the closure, ensure the closure refers to the variables via a reference, that is replaced with a references to the environment copy.
* More? Probably...

Each of these have advantages and disadvantages.

We're going to look at the rewriting method mainly, though most of what you find below also apply to the copying method.

These methods are pretty similar - they both involve returning a pointer to the function implementing the code in the closure combined with a pointer to the data in the closure. The difference lies in how that data is accessed.

The rewriting approach creates the closure environment as early as possible, and changes the surrounding function to refer to that environment. It incures the closure creation cost whenever the closure generating function is created, but since local variables are initialized straight into the environment it avoids later copying.

The copy approach delays the creation of the closure environment as much as possible, and then copies the data. It can avoid unnecessary creation cost if there are paths that don't lead to creation of a closure, but when the creation happens, it needs to handle full copying of any objects or object references involved, which may be more expensive.

For both of these methods, care must be taken to create a single environment for any set of closures created during the same execution of the function that creates the closures.

### "Fat pointers", objects and thunks

When returning a closure we also need to be able to pass along the environment.

As it turns out, there are a number of ways of handling this:

* We can create a "fat pointer". I.e. instead of passing around only the address to the code, we also pass around a pointer to the environment, and it is the callers responsibility to load that address onto the stack or into a register so the code can get at it.
* We can turn the closure into an object, like Ruby's Proc, and simply treat the variables used as instance variables of that object.
* We can create our own half-assed almost object by storing the function pointer in the environment.
* We can create a "thunk" on the fly - a small piece of code that will load the address of the environment and jump straight into the real code - and return that instead of a pointer to the real code of the closure. The major downside here is that it requires turning off protection against executing code from the heap.

The "fat pointer" approach is simple but involves more work at every call site, and doubles the size of the data that is passed around.

Turning it into an object is simple, and for my Ruby compiler it's even necessary a lot of the time since most of the time when you handle blocks in Ruby, you'll actually get a Proc object. But it has the full overhead of method dispatch.

The "half-assed object" approach still requires each call site to do a little bit more work, but less than the fat pointer approach. It also doesn't require additional data to be passed around (the function pointer is stored in the environment instead of copied around)

Creating a "thunk" also has overhead, but isn't as scary as it may sound - the code to create is very simple, and really it consists of copying a few bytes around.

We'll start with a "sort of" object, and then take a look at the thunk approach.

### Simulating the rewriting method in C

closures-basic.c

C lacks pretty much everything that could make this convenient and easy, so it really lays the implementation bare, for better or worse.

First, let's create a structure to hold the function pointer and environment:
```cpp
    struct closure {
      void (* call)(struct closure *);
      int x;
    };
```
If we had more local variables in foo, we'd add them to this structure.

Then we need to create a function with the code for the lambda block:
```cpp
    void block(struct closure * env) {
      env->x += 1;
      printf ("block: x is %d\n", env->x);
    }
```
Finally we can implement foo:
```cpp
    struct closure * foo(int x)
    {
      struct closure * closure = (struct closure *)malloc(sizeof(struct closure *));
      closure->x = x;
      printf ("x is %d\n",closure->x);
      closure->call = &block;
      return closure;
    }
```
Ewww..

A couple of observations: If we want to be able to return multiple closures (say, an array of them), the variables needs to be acessed via one more indirection, which makes this even more disgustingly convoluted.

It's also annoying, because it means lots of extra overhead in order to handle a situation that might very well never arise.

If the language requires supporting multiple closures (like Ruby), a compiler could support both approaches to optimize - adding the cost of the extra redirection only:

when more than one closure can potentially be returned at the same time.
when those closures need access to the same variables.
For an example of these restrictions, consider:
```ruby
     def foo x
       a = 1

       b = 2
       if x
         return lambda { a += 1 }

       else
         return lambda { b+= 1 }

       end
     end
```
First of all, this function is guaranteed to only return one lambda at the time, so applying that rule, we can just assing the appropriate function pointer to the call member variable, and do away with the extra indirection.

Secondly, even if we decide to return both of them at the same time, they are guaranteed to never access the same variables, so we could instead create two distinct closure environments, and still avoid the extra indirection.

But let's take a look at the complete example with the extra indirection anyway, as a worst case scenario:

### Indirection hell

closures-indirection.c

First we create an environment for the free variables we wish to "capture":
```cpp
    struct env {
      int x;
    };
```
Our modified closure structure holds a pointer to it, instead of holding the variables:
```cpp
    struct closure {
      void (* call)(struct env *);
      struct env * env;
    };
```
The block takes a pointer to the environment:
```cpp
    void block(struct env * env) {
      env->x += 1;
      printf ("block: x is %d\n", env->x);
    }
```
And the closure function itself needs to first allocate the environment, and then the closure:
```cpp
    struct closure * foo(int x) {
      struct env * env = (struct env *)malloc(sizeof(struct env));
      env->x = x;
    
      printf ("x is %d\n",env->x);
      
      struct closure * closure = (struct closure *)malloc(sizeof(struct closure *));
      closure->env = env;
      closure->call = block;
    
      return closure;
    }
```
Finally we call it with the env:
```cpp
    int main() {
      struct closure * c = foo(5);
    
      c->call(c->env);
      c->call(c->env);
   }
```
### Fat pointers

The example above is actually really simple to use to illustrate the fat pointer approach. Instead of returning a pointer to the closure object, we simply return the object itself:

closures-fatptr.c
```cpp
    struct closure foo(int x)
    {
      struct env * env = (struct env *)malloc(sizeof(struct env));
      env->x = x;
    
      printf ("x is %d\n",env->x);
      
      struct closure closure;
      closure.env = env;
      closure.call = block;
    
      return closure;
    }
    
    int main() {
      struct closure c = foo(5);
    
      c.call(c.env);
      c.call(c.env);
    }
```
As you can see it makes some things simpler (no need for the second memory allocation step).

### Variable substitution

An optimization worth keeping in mind is variable substitution. In cases where it can be guaranteed that the variables in question keeps the same value, the need for a separate environment may go away if variables are substituted for their values in the closures body.

Furthermore, if the variables can be guaranteed not to change in any closure returned from the function, then whether or not the variable changes in the function outside the closures, the closures may keep separate environments (and hence do the optimization to avoid indirection) with copies of the variables in question.

Of course this would require the function to carry out any updates once for each generated closure.

(you might have spotted here one of the advantages for functional languages with little or no mutation of variables - they have a lot fewer issues to worry about with respect to sharing of viarable state)

### Moving on to thunks

closures-thunks.c

Now that we have the basics down, how do we return just a "plain" function pointer, so we can simplify the call sites?

What we want to create is something like the code below.

Note that we take a shortcut and don't create a proper stack frame - this kind of thing can easily confuse gdb and other debuggers, which is not necessarily very nice.

The thunk below also does not directly allow passing any arguments to the block - if we wanted to do that it gets hairier, since we'd need to manipulate the parameters passed, while the caller doesn't know we've altered the size of the parameter space set aside. In a compiler this would likely be solved by making either the caller or callee aware that it's a closure call, and adjusting the stack accordingly separate from the thunk.

Of course, the downside of using asm here is that it's architecture specific, and the code to generate the thunk will need to be modified accordingly to port the code, but then if you do this in a compiler that is outputting native assembly, you have to do that anyway.
```asm
    my_closure_instance:
        pushl $my_environment ; Push the environment onto the stack as the first arg
        call  $the_block      ; Go to the real code
        addl  $4,%esp         ; Throw the environment pointer away
        ret                   ; Return to the caller
```  
Putting some arbitrary values in there, and assembling it with gcc/gas, and then doing objdump -D to the resulting binary gives this (and other bits and pieces I've cut - use the -nostdlib option to avoid dealing with a bunch of initialization code):
```asm
    08048055 :
    8048055:       68 00 08 af 2f          push   $0x2faf0800
    804805a:       e8 00 00 00 00          call   804805f 
    804805f:       83 c4 04                add    $0x4,%esp
    8048062:       c3                      ret  
```
This shows us the values to put in our thunk. Couple of observations: The push is just followed by the address itself, but the call uses an offset from the first byte of the following instruction.

This approach of using structs to generate the thunk, I've blatantly stolen from Joe Damato because I like how it makes the code that manipulates the thunk more readable:
```cpp
    struct __attribute__((packed)) thunk {
      unsigned char push_op;
      void * env_addr;
      unsigned char call_op;
      signed long call_offset;
      unsigned char add_esp_ops[3];
      unsigned char ret_op;
    };
    
    struct thunk default_thunk = {0x68, 0, 0xe8, 0, {0x83, 0xc4, 0x04}, 0xc3};
```
(the __attribute__ stuff is the gcc specific way of avoiding padding for alignement)

We change the rest like this:
```cpp
    typedef void (* cfunc)();
    
    cfunc foo (int x) {
      struct env * env = (struct env *)malloc(sizeof(struct env));
      env->x = x;
    
      printf ("x is %d\n",env->x);
    
      struct thunk * thunk = (struct thunk *)mmap(0,sizeof(struct thunk), PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
      *thunk = default_thunk;
      thunk->env_addr = env;
      thunk->call_offset = (void *)&block - (void *)&thunk->add_esp[0]; // Pretty!
      mprotect(thunk,sizeof(struct thunk), PROT_EXEC);
      return (cfunc)thunk;
    }
```
The typedef is a workaround for C's absolutely atrocious ptr-to-function declarations...

The interesting bit is at the end where we allocate the thunk, and fill in the addresses

Then we cast the thunk data structure to a function pointer. That ought to make you feel dirty, and a bit queasy. It's ok, though.

We use mmap to avoid problems on systems with executable heaps turned off (execshield etc. that wil cause a segmentation fault if you try to execute code in malloc()'d memory or on the stack), and the mprotect() turns off write access to the page after we're done. For a production approach you may want a dedicated allocation function to properly manage this and perhaps avoid doing a separate mmap for every thunk created.

All of this could really be wrapped up into a nice generic function. Something like this:
```cpp
    struct thunk * make_thunk(struct env * env, void * code) {
      struct thunk * thunk = (struct thunk *)mmap(0,sizeof(struct thunk), PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
      *thunk = default_thunk;
      thunk->env_addr = env;
      thunk->call_offset = code - (void *)&thunk->add_esp[0]; // Pretty!
      mprotect(thunk,sizeof(struct thunk), PROT_EXEC);
      return thunk;
    }
```
Finally, this makes our main function look like this:
```cpp
    int main() {
      cfunc c = foo(5);
      
      c();
      c();
    }
```
Now that's nicer...

Since this is already gcc/Linux/x86-32 specific, it can be made even nicer. gcc supports inner functions, and with a tiny bit of restructuring and a couple of macros I did this, mostly for fun to see how close to a "natural" syntax for closures I could get in C without changing the compiler:
```cpp
    #define initenv(__vars__) struct env { __vars__ ; } * env = (struct env *)malloc(sizeof(struct env));
    #define new_closure(__block__) (closure)make_thunk(env,&__block__)
    
    closure foo (int x)
    {
      initenv(int x)
      env->x = x;
        
      printf ("x is %d\n",env->x);
    
      void block (struct env * env) {  
         env->x += 1;
         printf ("block: x is %d\n", env->x);
      } 
    
      return new_closure(block);
    }
```
I'll still stick to Ruby...

### Some parting comments

I wrote this while exploring various approaches to add closure support to my Ruby compiler. I haven't quite made up my mind yet. The object approach is tempting because Ruby already has the Proc class, but I'll probably go for an environment + fat pointer approach that will be converted into a Proc object if assigned to anything (as opposed to just used for yield)

The thunk approach is somewhat appealing too, but if turned into a Proc object it may be as much or more overhead than the fat pointer approach (and there will only be one call site: In Proc#call) and this approach would mean the fat pointer wouldn't be passed around much - typical usage would be passing a block to a method, and then yield'ing to it, where the yield would be the only call site.