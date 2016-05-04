# 转移到method_missing

### A diversion into Method missing

So far the method_missing implementation has just printed a notice and quit.

During the trip down the rabbit hole that is attr_accessor and friends that became a major annoyance.

The problem is that this notice has not included a stack backtrace or any way to figure out where it occurred. It's also been impossible to override it and actually figure out what method was being called because we currently get to method_missing by jumping straight into the vtable.

We get method_missing when we hit the pointer that's installed there as default when nothing has overridden it.

So how do we get better debug output? And how do we support users overriding method_missing and actually getting a symbol to use?

### Thunks

A "thunk" in terms of a object oriented languages is generally a small piece of compiler generated code that gets inserted to "adjust" a function or method call.

In this specific case, we will generate a separate thunk for each vtable entry.

Instead of inserting a pointer to method_missing directly, we will insert the address of a small thunk. The thunk will not even create a full stack frame, but simply add the address of the Symbol corresponding to the vtable slot as the first argument on the stack, and then jump straight into method_missing, thereby simulating a "direct" call to method_missing with the symbol as the first argument.

It's actually very simple - we just need to pop the real return address off the stack, push the symbol onto the stack, and then push the real return address back on.

Ok, so we still cheat a bit. Eventually we need to make our method_missing into a real method, but for now it's a function. Here's the code I've added to create a "base" vtable that is used to initialize the vtable slots of Object:
```ruby
    def output_vtable_thunks
      @vtableoffsets.vtable.each do |name,_|
        @e.label("__vtable_missing_thunk_#{clean_method_name(name)}")
        # FIXME: Call get_symbol for these during initalization 
        # and then load them from a table instead.  
        compile_eval_arg(GlobalScope.new, ":#{name.to_s}".to_sym)
        @e.popl(:edx) # The return address 
        @e.pushl(:eax)
        @e.pushl(:edx)
        @e.jmp("__method_missing")
      end
      @e.label("__base_vtable")
      # For ease of implementation of __new_class_object we
      # pad this with the number of class ivar slots so that the
      # vtable layout is identical as for a normal class 
      ClassScope::CLASS_IVAR_NUM.times { @e.long(0) }
      @vtableoffsets.vtable.to_a.sort_by {|e| e[1] }.each do |e|
        @e.long("__vtable_missing_thunk_#{clean_method_name(e[0])}")
      end
    end
```
I hope it's reasonably easy to follow. First it generates a number of functions that will look like this:
```asm
    __vtable_missing_thunk_to_yaml:
        subl    $4, %esp
        movl    $1, %ebx
        movl    $.L110, (%esp)
        movl    $__get_symbol, %eax
        call    *%eax
        addl    $4, %esp
        popl    %edx
        pushl   %eax
        pushl   %edx
        jmp __method_missing
```
This can be optimized a lot, but if you've followed this series, you know getting something working is higher priority. In this case we generate a call to __get_symbol, which was introduced in the last part, and we pass the string corresponding to the name:
```ruby
    .L110:
        .string "to_yaml"
```
Then we adjust the stack as mentioned above.

The next step is to create the __base_vtable. Here's an excerpt:
```asm
    __base_vtable:
        .long 0
        .long 0
        .long __vtable_missing_thunk_new
        .long __vtable_missing_thunk___send__
        .long __vtable_missing_thunk___get_symbol
        .long __vtable_missing_thunk___method_missing
        .long __vtable_missing_thunk_array
        .long __vtable_missing_thunk___new_class_object
        .long __vtable_missing_thunk_define_method
        ...
```
Then we need to modify __new_class_object to assign entries from __base_vtable instead of just blindly assigning a pointer to __method_missing:
```ruby
    # size 
    def __new_class_object(size,superclass,ssize)
      ob = 0
      %s(assign ob (malloc (mul size 4))) # Assumes 32 bit                                                                                                                  
      i = 1
      %s(while (lt i ssize) (do
          (assign (index ob i) (index superclass i))
          (assign i (add i 1))
      ))
      %s(while (lt i size) (do
           # Installing a pointer to a thunk to method_missing                                                                                                              
           # that adds a symbol matching the vtable entry as the                                                                                                            
           # first argument and then jumps straight into __method_missing                                                                                                   
           (assign (index ob i) (index __base_vtable i))
           (assign i (add i 1))
      ))
      %s(assign (index ob 0) Class)
      ob
    end
```
Finally we make __method_missing output the symbol, instead of just spitting out "Method missing":
```ruby
    def __method_missing sym
      %s(printf "Method missing: %s\n" (callm sym to_s))
      %s(exit 1)
      0
    end
```