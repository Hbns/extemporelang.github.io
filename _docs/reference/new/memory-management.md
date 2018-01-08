**IGNORE Not ready for prime time**

---
title: Memory Management
---

The two languages hosted by the Extempore compiler, xtlang and Scheme, have
different approaches to dealing with memory allocation and management. While
Scheme manages memory for you, in xtlang you have to do it yourself. The tradeoff is that while this takes more work, it means that we can write highly performant code that can process real-time audio and video.

In xtlang whenever you need memory you ask the compiler for it, and when you're finished with it you tell the compiler to free the memory. This results in highly efficient code, but the downside is that if you forget to tell the compiler to free memory you can end up running out of memory pretty quickly. Fortunately xtlang has some good tools that can help you remember.


## Types of Memory

Xtlang has three types of memory available to it:
1. The Stack
2. The Heap
3. Zone Memory

We'll discuss each of these in turn, and when you should use them.

### The Stack

Whenever a function is called it pushes any variables passed in, along with local variables, onto the stack. When the function returns it 'pops' any memory allocated during the function call except for the return value. Consequently any

The **stack** is for dealing with function arguments and local
    variables. Each function call 'pushes' some new data onto the stack,
    and when the function returns it 'pops' off any local variables and
    leaves its return value. Memory allocated on the stack lasts for as long as you're in scope of the function that allocated it. 

There are two advantages to using the stack:
1. Access and allocation are typically very quick.
2. Memory deallocation is taken care of for you when the function returns.

So how do we use the stack? Let's look at some examples:

~~~~ sourceCode
(bind-func simple_stack_alloc
  (lambda ()
    (let ((a 2)
          (b 3.5))
      (printf "a x b = %f\n"
              (* (i64tod a) b)))))

(simple_stack_alloc) ;; prints "a x b = 7.000000"
~~~~

When this function is called, the two variables `a` and `b` that were bound in the `let` call are allocated on the stack. This is the default for all variables (except Strings [fn:: Strings are discussed in 'Chapter']). 

And this is true for anything else that you bind in a `let` inside a `lambda`, unless you explicitly request that they be stored somewhere else.

However you can also allocate larger memory blocks. For example you might need a buffer for audio:

~~~~ sourceCode
(bind-func buffer_stack_alloc
  (lambda (size:i64)
    (let ((buf:float* (salloc size)))
      (let ((i 0))
        (dotimes (i size)
          (pset! buf i 0.0)))))) ;; 0 out the audio buffer
~~~~

In the function above we used `salloc` to allocate a memory block on the stack. This block can store `size` float values.[fn:: For C programmers note that Extempore allocates memory using the size of the type, rather than in bytes like C and C++]. When this memory block was created a pointer to the start of the block was bound to `buf`, which is a pointer of type `float*`.

What happens if we try to return our buffer:

~~~~ sourceCode
(bind-func buffer_set
  (lambda (val:float)
    (let ((buf:float* (salloc 32)))
      (pfill! buf 32 val)
      buf)))
~~~~

Note also that this time we used `pfill!` to fill the buffer with `val`.

What happens if we try and dereference this returned pointer?

~~~~ sourceCode
(bind-func double_tuple_test
  (lambda ()
    (let ((tup (double_tuple 6)))
      (printf "tup* = <%lld,%lld>\n"
              (tref tup 0)
              (tref tup 1)))))

(double_tuple_test)

;; prints:

;; input: 6, output: <6,12>
;; tup* = <6,12>
~~~~

Well, that seems to work OK. What about if we call `double_tuple` again in the
body of the `let`, ignoring its return value?

~~~~ sourceCode
(bind-func double_tuple_test2
  (lambda ()
    (let ((tup (double_tuple 6)))
      (double_tuple 2)
      (printf "tup* = <%lld,%lld>\n"
              (tref tup 0)
              (tref tup 1)))))

(double_tuple_test2)

;; prints:

;; input: 6, output: <6,12> (in the 1st call to double_tuple)
;; input: 2, output: <2,4>  (in the 2nd call to double_tuple)
;; tup* = <2,4>
~~~~

This isn't right: `tup*` should still be the original tuple `<6,12>`, because
we've bound it the `let`. But somewhere in the process of calling `double_tuple`
again (with a different argument: `2`), the values in our original tuple (which
we have a pointer to in `tup`) have been overwritten.

Finally, consider this example:

~~~~ sourceCode
(bind-func double_tuple_test3
  (lambda ()
    (let ((tup (double_tuple 6))
          (test_closure
           (lambda ()
             (printf "tup* = <%lld,%lld>\n"
                     (tref tup 0)
                     (tref tup 1)))))
      (test_closure))))

(double_tuple_test3)

;; prints:

;; input: 6, output: <6,12>
;; tup* = <0,4508736416>
~~~~

Wow. That's not just wrong, that's *super wrong*. What's going on is that the
call to `salloc` inside the closure `double_tuple` doesn't keep the memory after
the closure returns, because at this point all the local variables get popped
off the stack. Subsequent calls to *any* closure will push new arguments and
local variables *onto* the stack and overwrite the memory that `tup` points to.

That's what deallocating memory *means*: it doesn't mean that the memory gets
set to zero, or that new values will be written in straight away, but it means
that the memory *might* be overwritten at any stage. Which, from a programming
perspective, is just as bad as having new data written into it, because if you
can't trust that your pointer still points to the value(s) you think it does
then it's pretty useless.

So, what we need in this case is to allocate some memory which will still hang
around after the closure returns. `salloc` isn't up to the task, but `zalloc`
is.



### The Heap

Unfortunately the stack is a limited resource and if you run out of stack memory your program will probably crash.


I should also point out that the stack and heap aren't actually different types
of memory in the computer---they're just different areas in the computer's RAM.
The difference is in the way the program *uses* the different regions. Each
running process has its own stack(s) heap, and they are just regions of memory
given to the process by the OS.

So, that's the stack and the heap, but there's actually one other type of memory in Extempore: **zone** memory. A zone is a
[region](http://en.wikipedia.org/wiki/Region-based_memory_management) of memory
which can be easily deallocated all at once. So, if you have some data that you
need to hang around longer than a function call (so a stack allocation is no
good), but want to be able to conveniently deallocate all at once, then use a
zone. There can be multiple zones in existence at once, and they don't interfere
(or have anything to do with) each other.

## The three flavours of memory in Extempore {#the-three-flavours-of-memory-in-extempore}

So, in accordance with the three different memory 'types' (the stack, the heap,
and zones) there are three memory allocation functions in xtlang: `salloc`,
`halloc` and `zalloc`. They all return a pointer to some allocated memory, but
they differ in *where* that memory is allocated from, and there are no prizes in
guessing which function is paired with which type of memory :)

Also, `alloc` in xtlang is an alias for `zalloc`. So if you ever see an `alloc`
in xtlang code just remember that it's grabbing memory from a zone.

## Stack allocation with salloc {#stack-allocation-with-salloc}


## Zone allocation with zalloc {#zone-allocation-with-zalloc}

Zone allocation is kindof like stack allocation, except with user control over
when the memory is freed (as opposed it happening at the end of function
execution, as with memory on the stack). Essentially this means that we can push
and pop zones off of a stack of memory zones of user-defined size.

A memory zone can be created using the special `memzone` form. `memzone` takes
as a first argument a zone size in bytes, and then an arbitrary number of other
forms (s-expressions) which make up the body of the `memzone`. The *extent* of
the zone is defined by `memzone`'s s-expression. Anything within the body of the
`memzone` s-expression is *in scope*.

Say we want to fill a memory region with `i64` values which just count from `0`
up to the length of the region (`region_length`). We'll need to allocate the
memory for this region, and get a pointer to the start of the region. We can do
this using `zalloc` inside a `memzone`.

~~~~ sourceCode
(bind-func fill_buffer_memzone
  (lambda ()
    (memzone 100000  ;; size of memzone (in bytes)
             (let ((region_length 1000)
                   (int_buf:i64* (zalloc region_length))
                   (i:i64 0))
               (dotimes (i region_length)
                 (pset! int_buf i i))
               (printf "int_buf[366] = %lld\n"
                       (pref int_buf 366))))))

(fill_buffer_memzone) ;; prints "int_buf[366] = 366"
~~~~

The code works as it should: as confirmed by the print statement. Notice how the
call to `zalloc` took an argument (`region_length`). This tells `zalloc` how
much memory to allocate from the zone. If we hadn't passed this argument (and it
*is* optional), the default length is `1`, to allocate enough memory for *one*
`i64`. All of the alloc functions (`salloc`, `halloc` and `zalloc`) can take
this optional size argument, and they all default to `1` if no argument is
passed.

Let's try another version of this code `fill_buffer_memzone2`, but with a much
longer buffer of `i64` values.

~~~~ sourceCode
(bind-func fill_buffer_memzone2
  (lambda ()
    (memzone 100000  ;; size of memzone (in bytes)
             (let ((region_length 1000000)
                   (int_buf:i64* (zalloc region_length))
                   (i:i64 0))
               (dotimes (i region_length)
                 (pset! int_buf i i))
               (printf "int_buf[366] = %lld\n"
                       (pref int_buf 366))))))

(fill_buffer_memzone2) ;; prints "int_buf[366] = 366"
~~~~

This time, with a region length of one million, the code still works (at least,
the 367Th element is still correct), but the compiler also prints a warning
message to the log:

~~~~ sourceCode
Zone:0x7ff7ac99a100 size:100000 is full ... leaking 8000000 bytes
Leaving a leaky zone can be dangerous ... particularly for concurrency
~~~~

So what's wrong? Well, remember that the `memzone` has a size (in bytes) which
is specified by its first argument. We can calculate how much space `int_buf`
will need (`region_length` multiplied by 8, because there are 8 bytes per `i64`)
and therefore how much of the zone's memory will be allocated with the call to
`(zalloc region_length)`. If this number is *greater* than the memzone size,
then we'll get the "Zone is full, leaking *n* bytes" warning---as we did with
`fill_buffer_memzone2`.

When zones leak, the Extempore run-time will scramble to find extra memory for
you, but it will be from the heap---which is time-consuming and it will never be
deallocated. This is bad, so it's always worth making sure that the zones are
big enough to start with.

`memzone` calls can also be nested inside one another. When a new zone is
created (pushed) any calls to `zalloc` will be allocated from the new zone
(which is the **top** zone). When the extent of the zone is reached it is
**popped** and its memory is reclaimed. The new **current** zone is then the
next **top** zone. The zones are in a stack in the 'stack *data structure*'
sense of the term, but this is not the stack that I was talking about earlier
with `salloc`. Hopefully that's not too confusing. So we'll talk about pushing
and popping zones from the *zone stack*, but it's still all done with `memzone`
and `zalloc`.

By default each process has an initial **top** zone with 1M of memory. If no
user defined zones are created (i.e. no uses of `memzone`) then any and all
calls to zalloc will slowly (or quickly) use up this 1M of memory---you'll know
when it runs out as you'll get about a gazillion memory leak messages.

In general this is the zone story. But to complicate things slightly there are
two special zones.

1.  The **audio zone**: there is a zone allocated for each audio frame
    processed, be that sample by sample, or buffer by buffer. The zones extent
    is for the duration of the audio frame (i.e. is deallocated at the end of
    the frame).

2.  **Closure zones**: all 'top level' closures (any closure created using
    `bind-func`) has an associated zone created at compile time (not at
    run-time, although this distinction is quite blurry in Extempore). The
    `bind-func` zone default size is 8KB, however, `bind-func` has an optional
    argument to specify any arbitrary `bind-func` zone size.

To allocate memory from a closure's zone, we need a `let` outside the `lambda`.
Anything `zalloc`'ed from there will come from the closure's zone. Anything
`zalloc`'ed from *inside* the closure will come from whatever the top zone is at
the time---usually the default zone (unless you're in an enclosing `memzone`).

As an example, let's revisit our 'fill buffer' examples from earlier. With a
region length of one thousand:

~~~~ sourceCode
(bind-func fill_buffer_closure_zone
  (let ((region_length 1000)
        (int_buf:i64* (zalloc region_length))
        (i:i64 0))
    (lambda ()
      (dotimes (i region_length)
        (pset! int_buf i i))
      (printf "int_buf[366] = %lld\n"
              (pref int_buf 366)))))
~~~~

The `let` where `int_buf` is allocated is outside the `lambda` form, so the
memory will be coming from the zone associated with the closure
`fill_buffer_closure_zone`. When we try and compile that, we get the warning:

~~~~ sourceCode
Zone:0x7fb8b3a4a610 size:8192 is full ... leaking 32 bytes
Leaving a leaky zone can be dangerous ... particularly for concurrency
~~~~

Let's try it again, but with a 'zone size' argument to `bind-func`

~~~~ sourceCode
(bind-func fill_buffer_closure_zone2 10000 ;; zone size: 10KB
  (let ((region_length 1000)
        (int_buf:i64* (zalloc region_length))
        (i:i64 0))
    (lambda ()
      (dotimes (i region_length)
        (pset! int_buf i i))
      (printf "int_buf[366] = %lld\n"
              (pref int_buf 366)))))

(fill_buffer_closure_zone2) ;; prints "int_buf[366] = 366"
~~~~

Sweet---no more warnings, and the buffer seems to be getting filled nicely.

This type of thing is very useful for holding data closed over by the top level
closure. For example, an audio delay closure might specify a large `bind-func`
zone size and then allocate an audio buffer to be closed over. The example file
`examples/core/audio-dsp.xtm` has lots of examples of this.

The `bind-func` zone will live for the extent of the top level closure, and will
be refreshed if the closure is rebuilt (i.e. the old zone will be destroyed and
a new zone allocated).

## Heap allocation with halloc {#heap-allocation-with-halloc}

Finally, we meet `halloc`, the Extempore function for allocating memory from the
heap. The heap is for long-lived memory, such as data that you want to keep
hanging around for the life of the program.

You can use `halloc` anywhere you would use `salloc` or `zalloc` and it will
give you a pointer to some memory on the heap. So, let's revisit the
`double_tuple_test3` example from earlier, which didn't work because the memory
for `tup` on the stack went out of scope when the closure returned. If we
replace the `salloc` with a `halloc`:

~~~~ sourceCode
(bind-func double_tuple_halloc
  (lambda (a:i64)
    (let ((tup:<i64,i64>* (halloc))) ;; halloc instead of salloc
      (tfill! tup a (* 2 a))
      tup)))

(bind-func double_tuple_halloc_test
  (lambda ()
    (let ((tup (double_tuple_halloc 4))
          (test_closure
           (lambda ()
             (printf "tup* = <%lld,%lld>\n"
                     (tref tup 0)
                     (tref tup 1)))))
      (test_closure))))

(double_tuple_halloc_test) ;; prints "tup* = <4,8>"
~~~~

Now, the returned tuple pointer `tup` is a heap pointer, so we can refer to it
from *anywhere* without any issues. In fact, the only way to deallocate memory
which has been `halloc`'ed and free it up for re-use is to use the xtlang
function `free` (which is the same as calling `free` in C).

In practice, a lot of the times where you want long-lived memory you'll want it
to be associated with a closure anyway, so the closure's zone is a better option
than the heap for memory allocation, as in the `fill_buffer_closure_zone2`
example above. This has the added advantage that if you re-compile the closure,
because you've changed the functionality or whatever, all the memory in the zone
is freed and re-bound, which is often what you want.

Where you *may* want to use `halloc` to allocate memory on the heap, is in
binding global data structures which you want to have accessible from anywhere
in your xtlang code. Binding global xtlang variables is the job of `bind-val`.

## Choosing the right memory for the job {#choosing-the-right-memory-for-the-job}

Each different alloc function is good for different things, and the general idea
to keep in mind is that you want your memory to hang around for as long as you
need it to---and *no longer*. Sometimes you only need data in the body of a
closure---then `salloc` is the way to go. Other times you want it to be around
for as long as the closure remains unchanged, then `zalloc` is the right choice.
Also, if you're going to be alloc'ing a whole lot of objects for a specific
algorithmic task and want to be able to conveniently let go of them all when
you're done, then creating a new zone with `memzone` and using `zalloc` is a
good way to go. Finally, if you know that a particular buffer of data is going
to hang around for the life of the program, then use `halloc`.

It's worth acknowledging that memory management in xtlang is a 'training wheels
off' scenario. It's a joy to have the low level control and performance of
direct memory access, but there are also opportunities to really mess things up
in a way that's trickier to do in higher-level languages. Remember that memory
is a finite resource. Don't try and allocate a memory region of 10<sup>15</sup>
8-byte `i64`:

~~~~ sourceCode
(bind-func fill_massive_buffer
  (lambda ()
    (let ((region_length 1000000000000000)
          (int_buf:i64* (zalloc region_length))
          (i:i64 0))
      (dotimes (i region_length)
        (pset! int_buf i i))
      (printf "int_buf[366] = %lld\n"
              (pref int_buf 366)))))

(fill_massive_buffer)
~~~~

When I call `(fill_massive_buffer)` on my computer (with 8GB of RAM), disaster
strikes.

~~~~ sourceCode
Zone:0x7fc5cbc268c0 size:100000 is full ... leaking 8000000000000000 bytes
Leaving a leaky zone can be dangerous ... particularly for concurrency
extempore(21386,0x11833d000) malloc: *** mmap(size=8000000000000000) failed (error code=12)
error: can't allocate region
set a breakpoint in malloc_error_break to debug
Segmentation fault: 11
~~~~

If you're not used to working directly with memory, you'll almost certainly
crash (segfault) Extempore when you start out. In fact, be prepared to crash
things *a lot* at first. Don't be discouraged: once you get your head around the
three-fold memory model and where each allocation function is getting its memory
from, it's much easier to write clean and performant code in xtlang. And from
there, the performance and control of working with 'bare metal' types opens up
lots of cool possibilities.

## Pointers {#pointers}

xtlang's pointer types may cause some confusion for those who aren't used to
(explicitly) working with reference types. That's nothing to be ashamed of---the
whole [pass by
value](http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value) / [pass
by
reference](http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_reference)
thing can take a bit to get your head around.

So what does it mean to say that xtlang supports pointer types? Simply put, this
means that we can use variables in our program to store not just values, but the
*addresses* of values in memory. A few examples might help to clarify things.

The `let` form in xtlang (as in Scheme) is a way of binding or assigning
variables: giving a name to a particular value. If we want to keep track of the
number of cats you have, then we can create a variable `num_cats`

~~~~ sourceCode
(bind-func print_num_cats
  (lambda ()
    (let ((num_cats:i64 4))
      ;; the i64 printf format specifier is %lld
      (printf "You have %lld cats!\n" num_cats))))

(print_num_cats) ;; prints "You have 4 cats!"
~~~~

What's happening here is that the `let` assigns the value `4` to the variable
`num_cats`, so that whenever the program sees the variable `num_cats` it'll look
in the `num_cats` 'place' in memory and use whatever value is stored there. The
computer's memory is laid out like a row of little boxes, and each box has an
address (the location of the box) and also a value (what's *in* the box).

![image](/images/pointer-tut-1.png)

In this image the computer's memory is represented by the blue boxes. Each box
has an address (the number below the box), an in this picture you can see that
this is only a subset of the total number of memory boxes (in a modern computer
there are millions of memory boxes).

The variable `num_cats` keeps track of the value that we're interested in. In
this case the address of that value is 'memory location 26', but it could easily
be any other location (and indeed will almost certainly be different if the
closure `print_num_cats` is called again).

Once a variable exists, we can change its value with `set!`:

~~~~ sourceCode
(bind-func print_num_cats2
  (lambda ()
    (let ((num_cats:i64 4))
      (printf "You have %lld cats... " num_cats)
      (set! num_cats 13)
      (printf "and now you have %lld cats!\n" num_cats))))

(print_num_cats2)
;; prints "You have 4 cats... and now you have 13 cats!"
~~~~

The `set!` function changes the value of `num_cats`: it sets a new value into
the memory location that `num_cats` refers to. In `print_num_cats2` the value of
`num_cats` starts out as `4`, so the first `printf` call prints "You have 4
cats…". The memory at this point might look like this:

![image](/images/pointer-tut-2a.png)

But then a new value (`13`) is set into `num_cats` with the call to `set!`, so
the second call to `printf` prints "and now you have 13 cats!". After the call
to `set!`, this is what the memory looks like:

![image](/images/pointer-tut-2b.png)

Notice how this time the memory address for `num_cats` is different to what it
was the previous time (28 rather than 26). This is because the `let` rebinds all
its variable-value pairs each time it is entered, and then forgets them when it
is exited (that is, when the paren matching the opening paren is reached).

## Pointers: storing memory addresses as values {#pointers-storing-memory-addresses-as-values}

What we've done so far is store the value (how many cats we have) into the
variable `num_cats`. The value has an address in memory, but as a programmer we
don't necessarily know what that address is, just that we can refer to the value
using the name `num_cats`. It's important to note that the *compiler* knows what
the address is---in fact as far as the compiler is concerned every variable is
just an address. But the compiler allows us to give these variables names, which
makes the code much easier to write and understand.

Pointer types in xtlang are indicated with an asterisk (`*`), for example the
type `i64*` represents a pointer to a 64-bit integer (sometimes called an
`i64`-pointer). With pointers, we actually assign the *address itself* in a
variable. That's the reason it's called a pointer: because it points to (is a
reference to) the value.

Let's update our code for printing the number of cats to use a pointer to the
value, rather than the value itself. Notice how the type of `num_cats_ptr` is
`i64*` (a pointer to an `i64`) rather than just an `i64` like it was before.

~~~~ sourceCode
(bind-func print_num_cats3
  (lambda ()
    (let ((num_cats_ptr:i64* (zalloc)))
      (printf "You have %lld cats!\n" num_cats_ptr))))

(print_num_cats3) ;; prints "You have 4555984976 cats!"
~~~~

There are a couple of other changes to the code. Firstly, we no longer bind the
value straight away (as we were doing with `(num_cats:i64 4)`), but instead we
make a call to `zalloc`. This is the way to get pointers in xtlang: through a
call to an 'alloc' function. `zalloc` is a function which 'allocates' and
returns the *address* (i.e. a pointer) of some memory which can be used to store
the value in. This address is the assigned to the variable `num_cats_ptr`, just
like the number `4` was assigned to `num_cats` in the earlier examples. The
orange bar on the variable name indicates that it's a pointer.

So why does `print_num_cats3` print such a weird (on my machine: 4555984976
cats!) answer? Well, it's because we're trying to print it as an `i64` *value*
(using `%lld` in the `printf` format string), but it's not an `i64` value---it's
the *address* of a memory location where an `i64` value is located. On a 64-bit
system (such as the laptop I'm writing this blog post on) the pointers *are*
actually 64-bit integers, because an integer is a pretty sensible way to store
an address.

Incidentally, this is one of the key benefits (and driving forces behind) the
switch from 32 to 64 bit architectures---the need for more memory addresses. If
a pointer is a 32 bit integer, then you can only 'address' about 4.3 billion
(2<sup>32</sup>) different memory locations. This might seem like a lot, but as
more and more computers came with more than 4.3Gb of RAM installed, so the need
for 64-bit pointers became more pressing. There are workarounds, but having a
larger addressable space is a key benefit of 64-bit architectures. And it helps
to remember that pointers *are* just integers, but they're not like the int
types that we use to store and manipulate data.

In `print_num_cats3` we don't set any value into that location, we only deal
with the address. In fact, the memory this address points to is referred to as
*uninitialised*, which is a name for memory that has been allocated but hasn't
had any values set into it. In Extempore, uninitialised memory will be 'zeroed
out', meaning all of the bits will be set to `0`. So for an `i64` this will be
the integer value `0`.

After the call to `zalloc`, the memory therefore will look like this (the value
is now shown in a different coloured box, to indicate it's an `i64*` pointer
type and not an `i64` value type)

![image](/images/pointer-tut-3.png)

This is cool, we can see that the value in memory location 27 is actually the
address 29, and the value of 29 is `0` because we haven't initialised it yet.
So, remember how in `print_num_cats2` we used `set!` to set a value into the
variable `num_cats`? Well, we can do a similar thing with the pointer
`num_cats_ptr` using the function `pset!`. `pset!` takes three arguments: a
pointer, an index (which is zero in this next example, but I'll get to what the
index means in the next section) and a value. The value must be of the right
type: e.g. if the pointer is a pointer to a double (a `double*`) then the value
must be a `double`.

~~~~ sourceCode
(bind-func print_num_cats4
  (lambda ()
    (let ((num_cats_ptr:i64* (zalloc)))
      (pset! num_cats_ptr 0 5)
      (printf "You have %lld cats!\n" (pref num_cats_ptr 0)))))

(print_num_cats4) ;; prints "You have 5 cats!"
~~~~

Great---the function now prints the right number of cats (in this case `5`), so
things are working properly again. After the `pset!` call, the memory will look
like this (the only difference from last time is that the value 5 is stored in
address 29, just as it should be).

![image](/images/pointer-tut-4.png)

Notice also that in `print_num_cats4` we don't pass `num_cats_ptr` directly to
`printf`, we do it through a call to `pref`. Whereas `pset!` is for writing
values into memory locations, `pref` is for reading them out. Like `pset!`, pref
takes a pointer as the first argument and an offset for the second argument. In
this way, we can read *and* write `i64` values to the memory location without
actually having a variable of type `i64` (which we did with `num_cats` in the
`print_num_cats` and `print_num_cats2`). All this is possible because we have a
pointer variable (`num_cats_ptr`) which gives us a place to load and store the
data.

## Buffers and pointer arithmetic {#buffers-and-pointer-arithmetic}

In all the examples so far, we've only used a pointer to a single value. This
has worked fine, but you might have been wondering why we bothered, because
assigning values directly to variables (as we did in the first couple of
examples) seemed to work just fine.

One thing that pointers and alloc'ing allows us to do is work with whole regions
in memory, in which we can store *lots* of values. Say we want to be able to
determine the mean (average) of 3 numbers. One way to do this is to store each
of the different numbers with its own name.

~~~~ sourceCode
(bind-func mean1
  (lambda ()
    (let ((num1:double 4.5)
          (num2:double 3.3)
          (num3:double 7.9))
      (/ (+ num1 num2 num3)
         3.0))))

;; call the function
(mean1) ;; returns 5.233333
~~~~

The `let` form binds the (`double`) values `4.5`, `3.3` and `7.9` to the names
`num1`, `num2` and `num3`. Then, all three values are added together (with `+`)
and then divided by `3.0` (with `/`). Now, this code does give the right answer,
but it's easy to see how things would get out of hand if we wanted to find the
mean of 5, 20 or one million values. What we really want is a way to give *one*
name to all the values we're interested in, rather than having to refer to all
the values by name individually. And to do that, we can use a pointer.

~~~~ sourceCode
(bind-func mean2
  (lambda ()
    (let ((num_ptr:double* (zalloc 3)))
      ;; set the values into memory
      (pset! num_ptr 0 4.5)
      (pset! num_ptr 1 3.3)
      (pset! num_ptr 2 7.9)
      ;; read the values back out, add them
      ;; together, and then divide  by 3
      (/ (+ (pref num_ptr 0)
            (pref num_ptr 1)
            (pref num_ptr 2))
         3.0))))

(mean2) ;; returns 5.233333
~~~~

In `mean2`, we pass an integer argument (in this case `3`) to `zalloc`.
`zalloc` then allocates enough memory to fit 3 `double` values. The
pointer that gets returned is still only a pointer to the first of these
memory slots. And this is where the second 'offset' argument to `pref`
and `pset!` come in.

![image](/images/pointer-tut-5.png)

See how the repeated calls to `pset!` and `pref` above have different offset
values? Well, that's because the offset argument allows you to get and set
values 'further into' the memory returned by `(zalloc 3)`. This isn't anything
magical, they just add the offset to the memory address.

There is a helpful function called `pfill!` for filling multiple values into
memory (multiple calls to `pset!`) as we did in the above example. Rewriting
`mean2` to use `pfill!`:

~~~~ sourceCode
(bind-func mean3
  (lambda ()
    (let ((num_ptr:double* (zalloc 3)))
      ;; set the values into memory
      (pfill! num_ptr 4.5 3.3 7.9)
      ;; read the values back out, add them
      ;; together, and then divide  by 3
      (/ (+ (pref num_ptr 0)
            (pref num_ptr 1)
            (pref num_ptr 2))
         3.0))))

(mean3) ;; returns 5.233333
~~~~

Finally, one more useful way to fill values into a chunk of memory is using a
`dotimes` loop. To do this, we need to bind a helper value `i` to use as an
index for the loop. This function allocates enough memory for 5 `i64` values,
and just fills it with ascending numbers:

~~~~ sourceCode
(bind-func ptr_loop
  (lambda ()
    (let ((num_ptr:i64* (zalloc 5))
          (i:i64 0))
      ;; loop from i = 0 to i = 4
      (dotimes (i 5)
        (pset! num_ptr i i))
     (pref num_ptr 3))))

(ptr_loop) ;; returns 3
~~~~

After the `dotimes` the memory will look like this:

![image](/images/pointer-tut-6.png)

There's one more useful function for working with pointers: `pref-ptr`. Where
`(pref num_ptr 3)` returns the *value* of the 4th element of the chunk of memory
pointed to by `num_ptr`, `(pref-ptr num_ptr 3)` returns the address of that
value (a pointer to that value). So, in the example above, `num_ptr` points to
memory address 27, so `(pref num_ptr 2)` would point to memory address 29.
`(pref (pref-ptr num_ptr n) 0)` is the same as `(pref (pref-ptr num_ptr 0) n)`
for any integer *n*.

## Pointers to higher-order types {#pointers-to-higher-order-types}

The <span role="doc">xtlang type system &lt;types&gt;</span> has both primitive
types (floats and ints) and higher-order types like tuples, arrays and closures.
Higher-order in this instance just means that they are made up of other types,
although these component types may be themselves higher-order types.

As an example of an aggregate type, consider a 2 element tuple. Tuples are
(fixed-length) n-element structures, and are declared with angle brackes (`<>`).
So a tuple with an `i64` as the first element and a double as the second element
would have the type signature `<i64,double>`. Getting and setting tuple elements
is done with `tref` and `tset!` respectively, which both work exactly like
`pref=/=pset!` except the first argument has to be a pointer to a tuple.

~~~~ sourceCode
(bind-func print_tuples
  (lambda ()
    ;; step 1: allocate memory for 2 tuples
    (let ((tup_ptr:<i64,double>* (zalloc 2)))
      ;; step 2: initialise tuples
      (tset! (pref-ptr tup_ptr 0) 0 2)         ; tuple 1, element 1
      (tset! (pref-ptr tup_ptr 0) 1 2.0)       ; tuple 1, element 2
      (tset! (pref-ptr tup_ptr 1) 0 6)         ; tuple 2, element 1
      (tset! (pref-ptr tup_ptr 1) 1 6.0)       ; tuple 2, element 2
      ;; step 3: read & print tuple values
      (printf "tup_ptr[0] = <%lld,%f>\n"
              (tref (pref-ptr tup_ptr 0) 0)    ; tuple 1, element 1
              (tref (pref-ptr tup_ptr 0) 1))   ; tuple 1, element 2
      (printf "tup_ptr[1] = <%lld,%f>\n"
              (tref (pref-ptr tup_ptr 1) 0)    ; tuple 2, element 1
              (tref (pref-ptr tup_ptr 1) 1))))); tuple 2, element 2

(print_tuples) ;; prints
;; tup_ptr[0] = <2,2.000000>
;; tup_ptr[1] = <6,6.000000>
~~~~

This `print_tuples` example works in 3 basic steps:

1.  **Allocate memory** for two (uninitialised) `<i64,double>` tuples, bind
    pointer to this memory to `tup_ptr`.
2.  **Initialise tuples with values** (in this case `2` and `2.0` for the first
    tuple and `6` and `6.0` for the second one). Notice the nested `tset!` and
    `pref-ptr` calls: `pref-ptr` returns a pointer to the tuple at offset 0 (for
    the first) and 1 (for the second). This pointer is then passed as the first
    argument to `tset!`, which fills it with a value at the appropriate element.
3.  **Read (& print) values** back out of the tuples. These should be the values
    we just set in step 2---and they are.

Let's have a look at what the memory will look like during the execution of
`print_tuples`. After the call to `(zalloc)` (step 1), we have a pointer to a
chunk of memory, but the tuples in this memory are uninitialised (indicated by
u).

![image](/images/pointer-tut-7.png)

After using `pref` and `tset!` in step 2, the values get set into the tuples.
Step 3 simply reads these values back out---it doesn't change the memory.

![image](/images/pointer-tut-8.png)

There are a couple of other things worth discussing about this example.

-   We used `pref_ptr` rather than `pref` in both step 2 and step 3. That's
    because `tset!` and `tref` need a *pointer to* a tuple as their first
    argument, and if we had used regular `pref` we would have got the tuple
    itself. This means that we could have just used `tup_ptr` directly instead
    of `(pref-ptr tup_ptr 0)` in a couple of places, because these two pointers
    will always be equal (have a think about why this is true).
-   There are a few bits of repeated code, for example `(pref-ptr tup_ptr 1)`
    gets called 4 times. We could have stored this pointer in a temporary
    variable to prevent these multiple dereferences, how could we have done that
    (hint: create the new 'tmp' pointer in the `let`---make sure it's of the
    right type).

There's one final thing worth saying about pointers in xtlang. Why do pointers
even *have* types? Isn't the address the same whether it's an int, a float, a
tuple, or some complex custom type stored at that memory address? The reason is
to do with something all this talk of memory locations as 'boxes' has glossed
over: that different types require different amounts of memory to store.

A more accurate (though still simplified) picture of the computer's memory is to
think of the boxes as 8-bit bytes. One bit (a binary digit) is just a `0` or a
`1`, and a byte is made up of 8 bits, for example `11001011`. These are just
[base-2 numerals](http://en.wikipedia.org/wiki/Binary_numeral_system), so `5` in
decimal is `101`, and although they are difficult for humans to read (unless
you're used to them), computers *live and breathe* binary digits.

This is why the integer types all have numbers associated with them---the number
represents the number of bytes used to store the integer. So `i64` requires 64
bits, while an `i8` only requires 8. The reason for having different sizes is
that larger sizes take up more room (more bytes) in memory, but can also store
larger values (n bits can store 2<sup>n</sup> different numbers). All the other
types have sizes, too: a `float` is 32 bits for instance, and the number of bits
required to represent an aggregate type like a tuple or an array is (at least)
the sum of the sizes of their components.

So, reconsidering our very first example, where we stored an `i64` value of `4`
to represent how many cats we had, a more accurate diagram of the actual memory
layout in this situation is:

![image](/images/pointer-tut-9.png)

See how each `i64` value takes up 8 bytes? Also, each byte has a memory
addresses, so the start of each `i64` in memory is actually 8 bytes along from
the previous one.

Now, consider the layout of an aggregate type like a tuple:

![image](/images/pointer-tut-10.png)

Each tuple contains (and therefore takes up the space of) an `i64` and a
`double`. So the actual memory address offset between the beginning of
consecutive tuples is 16 bytes. But `pref` still works the same as in the `i64*`
case. `(pref tup_ptr 1)` gets the second tuple---it doesn't try and read a tuple
from 'half way in'.

This is one reason why pointers have types: the type of the pointer tells `pref`
how far to jump to get between consecutive elements (this value is called the
stride). This becomes increasingly helpful when working with pointers to
compound types: no-one wants figure out (and keep track of) the size of a tuple
like `<i32,i8,|17,double|*,double>` and calculate the stride manually.

## Other benefits of using pointers {#other-benefits-of-using-pointers}

There are a few other situations where being able to pass pointers around is
really handy.

-   When the chunks of memory we're dealing with are large, copying them around
    in memory becomes expensive (in the 'time taken' sense). So, if lots of
    different functions need to work on the same data, instead of copying it
    around so that each function has its own copy of the data, they can just
    pass around pointers to the same chunk of data. This means that each
    function needs to be a good citizen and not stuff up things for the others,
    but if you're careful this can be a huge performance benefit.
-   You can programatically determine the amount of memory to allocate, which is
    something you can't to with xtlang's array types.





### Pointer types {#pointer-types}

![image](/images/pointer-examples.png)

xtlang supports
[pointers](http://en.wikipedia.org/wiki/Pointer_(computer_programming)) to any
type, in fact some types (such as closures) are almost always handled by
reference (that is, through pointers). Pointers in xtlang are indicated by the
usual `*` syntax.

Examples:

-   `double*`: a pointer to a double
-   `i64**`: a pointer to a pointer to a 64-bit integer

Pointers represent *memory addresses*, and the use of pointers in xtlang is one
of the key differences between xtlang and Scheme (and indeed between xtlang and
any high-level language). C programmers will be (intimately) familiar with the
concept of pointers, and xtlang's pointers are the same (you can `printf` them
with `%p`) For others, though, a more in-depth explanation of the concept of
pointers can be found <span role="ref">here &lt;pointer-doc&gt;</span>. If
you've never encountered pointers before then I suggest you check it out before
continuing.

The way to allocate (and store a pointer to) memory is through a call to one of
xtlang's 'alloc' functions. Extempore has 3 different alloc functions: `salloc`,
`halloc` and `zalloc`. They all allocate a chunk of memory and return a pointer
type, but they differ in *where* that memory is allocated from. In order of how
'long-lived' the memory will be: `salloc` allocates memory on the stack
(shortest-lived), `zalloc` allocates memory from the current zone, and `halloc`
allocates memory from the heap (longest-lived). Finally, `alloc` is an alias for
`zalloc`.

More detail about the Extempore's memory architecture (and the difference
between `salloc`, `halloc` and `zalloc`) can be found in <span
role="doc">memory</span>. For now, though, we'll just use `zalloc`, which
allocates memory from the current zone, which for these examples will work fine.

~~~~ sourceCode
(bind-func ptr_test
  (lambda ()
    (let ((a:double* (zalloc)))
      (printf "address = %p\n" a))))

(ptr_test) ;; prints "address = 0x1163bc030"
~~~~

In this example, the function closure `ptr_test` takes no arguments, binds a
pointer to a `double` (`a`) in the `let`, and then prints the memory address
that `a` points to.

Pointers aren't very interesting, though, if you can't read and write to the
values they point to. That's where the xtlang functions `pref`, `pset!` and
`pref-ptr` come in.

{:.note-box}

The semantics of the `*ref` functions are in the process of being changed---see
[this thread](https://groups.google.com/forum/#!topic/extemporelang/HiYKEstuM_w)
on the mailing list for more details. We'll update these docs as soon as things
settle down, but for now accept my humble apology that some of this stuff is out
of date. Sorry!

Unlike in C, `*` is not a dereference *operator*, it's just the syntax for the
specifying pointer types. Instead, there's a function `pref` for *dereferencing*
a pointer (getting the value the pointer 'points' to). `pref` takes two
arguments: the pointer, and an (integer) offset. So if `a` is a pointer to a
chunk of 10 `double` in memory (such as returned by `zalloc`, for instance),
then `(pref a 2)` in xtlang is the value of the third (`pref` uses 0-based
indexing) of those `double` (equivalent to `a[2]` in C).

To *set* the value associated with a pointer, there's `pset!`. Like `pref`,
`pset!` takes a pointer as the first argument, and offset as the second
argument, but it also takes an additional third argument---the value to set into
that memory location. This must be of the appropriate type: so if the pointer is
to a double, then the value passed to `pset!` must also be a double.

~~~~ sourceCode
(bind-func ptr_test2
  (lambda ()
    (let ((a:double* (zalloc))) ; allocate some memory for a double, bind
                                        ; the pointer to the symbol a
      (pset! a 0 2.4)          ; set the value at index 0 (of a) to 2.4
      (pref a 0))))            ; read the value at index 0 of a

(ptr_test2) ;; returns 2.400000
~~~~

In this example the closure `ptr_test2` takes no arguments, allocates some
memory, sets a value into that memory location, then reads it back out. Notice
that for both `pref` and `pset!` the index argument was zero---this means that
we were storing and reading the value directly into the pointer (memory
location) bound to `a`.

This is important (and useful) because the call to `zalloc` can (optionally)
take an integer argument. So, if we know we're going to store 4 doubles, we can
do this:

~~~~ sourceCode
(bind-func ptr_test3
  (lambda ()
    (let ((a:double* (zalloc 4)))
      (pfill! a 1.2 3.4 4.2 1.1) ; fill the pointer a with values
      (pref a 2))))              ; read the value at index 2 of a

(ptr_test3) ;; returns 4.200000
~~~~

`(zalloc 4)` will allocate enough memory for `4` doubles (4 doubles with 64
bytes/double means 256 bytes all up).

There's one new function in this example: `pfill!`, which is helpful for filling
multiple values into a byte array. Using `pfill!` is exactly the same as calling
`pset!` 4 times with an index of 0, 1, 2, and 3, but it's a bit more concise.

Finally, one more useful way to fill values into a chunk of memory is using a
loop.

~~~~ sourceCode
(bind-func ptr_test4
  (lambda ()
    (let ((a:double* (zalloc 10))
          (i:i64 0))
      (dotimes (i 10)
        (pset! a i (i64tod i)))
     (pref a 6))))

(ptr_test4) ;; returns 6.000000
~~~~

There's one more useful function for working with pointers: `pref-ptr`. Where
`(pref a 3)` returns the *value* of the 4th element of the chunk of memory
pointed to by `a`, `(pref-ptr a 3)` returns a *pointer* to that value. This also
implies that `(pref (pref-ptr a n))` is the same as `(pref (pref-ptr a 0) n)`
for any integer *n*.

One final note for C programmers: there is no `void*` in xtlang, use an `i8*`
instead.