# 一个单独的发射器类输出汇编语言

It's been a long time, and but I finally have a little bit of time again, and it's time to continue. At this point the next step is some re-factoring. As it stands the compiler is very tightly tied to x86, and before it gets bigger it's worth starting to redress that. The obvious solution is to start splitting out the code generation into a separate class. First of all I'll add a basic "emitter" - a class that just is responsible for spitting out the assembler instructions. I'll try to make it less x86 specific than what it replaces, but I won't put in a lot of effort to isolate all the x86 code straight away. Let's walk through some of it: First we require the emitter (the full source is linked below) and add this to the Compiler constructor:
```ruby
  def initialize
    @e = Emitter.new
    ...
  end
```
Then we rip the assembler prolog and epilog out of compile_main, and replace it with this:
```ruby
    @e.main do
      @main = Function.new([],[])
      compile_exp(Scope.new(self,@main),exp)
    end
```
The Emitter#main method looks like this:
```ruby
  def main
    puts ".text"
    export(:main,:function)
    label(:main)
    leal("4(%esp)",:ecx)
    andl(-16,:esp)
    pushl("-4(%ecx)")
    pushl(:ebp)
    movl(:esp,:ebp)
    pushl(:ecx)

    yield

    popl(:ecx)
    popl(:ebp)
    leal("-4(%ecx)",:esp)
    ret()
    emit(".size","main",".-main")
  end
```
I won't go into this in a lot of detail - it's a rehash of the old prolog and epilog, and then yield back to the compiler. You can read through the full source, but this depends on Emitter#method_missing and a few utility methods to make it more pleasant to write the emitter code without having to use puts all over the place. Gradually pushing the puts's down also makes it far easier to change this emitter to do other things later, such as write to file or do just in time code generation in memory. Here's the guts of it the utility functions - for the rest, see the download:
```ruby
  def method_missing(sym,*args)
    emit(sym,*args)
  end

  def emit op,*args
    puts "\t#{op}\t"+args.collect{|a|to_operand_value(a)}.join(',')
  end

  def int_value param
    return "$#{param.to_i}"
  end

  def to_operand_value src
    return int_value(src) if src.is_a?(Fixnum)
    return "%#{src.to_s}" if src.is_a?(Symbol)
    return src.to_s
  end
```
Let's look at some of the other ones. #compile_while:
```ruby
  def compile_while(scope, cond, body)
    @e.loop do |br|
      var = compile_eval_arg(scope,cond)
      @e.jmp_on_false(br)
      compile_exp(scope,body)
    end
    return [:subexpr]
  end
```
Nice. While this is not entirely architecture independent - other CPU's might have other constructs that are as suitable for doing loops - it is generic enough that given implementing Emitter#loop and #Emitter#jump_on_false it will work on any CPU, and implementing them should be fairly simple and doable on any CPU. For x86, to be equivalent to the old version, this is the code required:
```ruby
  def loop
    br = get_local
    l = local
    yield(br)
    jmp(l)
    local(br)
  end

 def jmp_on_false(label,op=:eax)
    testl(op,op)
    je(label)
  end
```
The #loop method depends on a couple of utility functions that handles allocation of branch labels:
```ruby
  def label(l)
    puts "#{l.to_s}:"
    l
  end

  def local(l=nil)
    l = get_local if !l
    label(l)
  end

  def get_local
    @seq +=1
    ".L#{@seq-1}"
  end
```
`#local` may be a bit "evil" - it has the side effect of outputting the label AND returning it, allocating a local label if none was given on input. Not sure combining all of that is a great idea, but it makes for compact code. Most of the code in the compiler changes in very similar ways, so I won't go through that much more, but here are a few more examples. #output_functions changes to this:
```ruby
  def output_functions
    @global_functions.each do |name,func|
      @e.func(name) { compile_exp(Scope.new(self,func),func.body) }
    end
  end
```
With the appropriate emitter function looking like this:
```ruby
  def func name
    export(name,:function) if name.to_s[0] != ?.
    label(name)
    pushl(:ebp)
    movl(:esp,:ebp)
    yield
    leave
    ret
    emit(".size",name.to_s, ".-#{name}")
  end
```
A better example of how this isolates architecture specific changes is #compile_eval_arg:
```ruby
  def compile_eval_arg scope,arg, call = false
    prefix = call ? "*" : ""
    atype, aparam = get_arg(scope,arg)
    return "$.LC#{aparam}" if atype == :strconst
    return "$#{aparam}" if atype == :int
    return aparam.to_s if atype == :atom
    if atype == :arg
      puts "\tmovl\t#{PTR_SIZE*(aparam+2)}(%ebp),%eax"
    end
    return "#{prefix}%eax"
  end
```
The new version:
```ruby
  def compile_eval_arg scope,arg
    atype, aparam = get_arg(scope,arg)
    return aparam if atype == :int
    return @e.addr_value(aparam) if atype == :strconst
    @e.load_address(aparam) if atype == :addr
    @e.load_arg(aparam) if atype == :arg
    return @e.result_value
  end
```
There isn't all that much more to say about the emitter. The full updated version is can be found here. Next time we'll be back to something more directly useful: Support for arrays.
