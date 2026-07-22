<!--
.. title: RPyForth: An ANS Forth Interpreter written in RPython
.. slug: benchmarker2-for-pypy
.. date: 2026-07-16 17:01:09 UTC
.. tags:
.. category:
.. link:
.. description:
.. type: text
.. author: Yusuke Izawa
-->

In most stack-based virtual machines (VMs), the operand stack is an
implementation detail. Once a compiler understands the data dependencies, it
can replace stack operations with single static assignment (SSA) values or
registers. The operand stack then disappears from the optimized machine code.

Forth treats its stack differently. In stack-oriented languages such as
[Forth](https://www.forth.com/forth/),
[PostScript](https://www.adobe.com/products/postscript.html), and
[Factor](https://factorcode.org), the _data stack_ is part of the programming
model. Words take values from it and leave results for the next word. There are
no named parameters or return values at these boundaries.

A typical interpreter stores this stack in a mutable memory region and keeps a
pointer to its top. Each word updates the pointer and reads or writes stack
slots. These operations are harder to remove than the temporary stack traffic
in a bytecode VM because the stack remains live between words, and a word's
stack effect is implicit in its behavior rather than declared in a function
signature.

RPyForth asks whether a meta-tracing JIT can remove those loads, stores, and
pointer updates without changing the data stack that Forth programs see. It is
an ANS Forth interpreter written in RPython.

## Stack-oriented languages and Forth

The easiest way to see the difference is with a small example. In Python, we
might square a value and then add one like this:

```python
def square(x):
    return x * x

def add_one(x):
    return x + 1

result = add_one(square(5))
```

The Forth version is:

```forth
: square   dup * ;
: add-one  1 + ;
: compute  square add-one ;

5 compute
```

A colon definition introduces a word. Here, `square` duplicates the top stack
value and multiplies the two copies. Then `add-one` pushes `1` and adds it.

Forth programmers usually document words with *stack-effect notation*:

```forth
: square   ( n -- n^2 ) dup * ;
: add-one  ( n -- n+1 ) 1 + ;
: compute  ( n -- n^2+1 ) square add-one ;
```

The values to the left of `--` are consumed, and the values to the right remain
after the word finishes. These annotations are ordinary Forth comments.

Here is the stack while `5 compute` runs. The rightmost value is on top:

```text
[]          initial state
[5]         push 5

[5, 5]      dup
[25]        *

[25, 1]     push 1
[26]        +
```

`square` does not return `25` in the usual sense. It leaves `25` on the shared
data stack, where `add-one` finds it.

The composition

```forth
square add-one
```

still describes the same data flow as

```text
add_one(square(x))
```

but the intermediate value has no name. Its location on the stack carries the
data flow.

A straightforward Forth interpreter turns the example into a series of array
accesses and stack-pointer updates. The awkward part for an optimizer is the
word boundary: `square` writes a real stack slot, and `add-one` reads that slot
later. The stack cannot simply disappear as a temporary bytecode stack often
does.

## RPyForth and the stack-fragmented layout

We built RPyForth to test whether meta-tracing can remove this stack traffic.
Its inner interpreter uses indirect threaded code. A colon definition is a
sequence of word references, and the interpreter dispatches them one at a time.
The PyPy toolchain generates a tracing JIT from that interpreter.

RPyForth also has a _stack-fragmented layout_ that lines up the hot part of the
data stack with word calls. We call the full three-area representation a
_metastack_.

## The three-area metastack

In the default fragment layout, the integer data stack has three parts:

1. the top two cells, stored in the cached fields `t0` and `t1`;
2. the next few cells, stored in a small fixed-size array named cached frame (or
   simply `frame`); and
3. all deeper cells, stored in a preallocated array named `spill`.

The default frame has eight cells, so the active cache can hold ten values in
total: two scalar tops and eight frame slots. Anything deeper goes to `spill`.
After pushing the values `0` through `13`, the layout looks like this. Depth
zero is the top of the stack.

```text
                  top of stack
                       |
                       |
  depth 0        t0       = 13     cached field
  depth 1        t1       = 12     cached field

  depth 2        frame[7] = 11
      ...             ...
  depth 9        frame[0] =  4     cached frame

  depth 10       spill[3] =  3
      ...             ...
  depth 13       spill[0] =  0     shared spill
                       |
                       |
                 bottom of stack
```

The indexes in both arrays increase toward the top of their area. In the
active cache, `frame[0]` is the deepest cached value, while the highest live
frame index sits immediately below `t1`. Likewise, `spill[0]` is the deepest
spilled value and `spill[spill_ptr - 1]` sits immediately below the cache. The
lookup code uses these index calculations:

```text
frame_index = cache_depth - 1 - depth
spill_index = spill_ptr - 1 - (depth - cache_depth)
```

The first formula applies to cached depths of two or greater; depths zero and
one map to `t0` and `t1`.

Two counters tie the areas together. `cache_depth` is the number of live cells
in the active cache, including `t0` and `t1`, and `spill_ptr` is the number in
`spill`. The logical depth is therefore

```text
cache_depth + spill_ptr
```

To a Forth program, this is still one stack. `peek(depth)` and
`poke(depth, value)` work out which area contains the requested value. Depths
zero and one map straight to `t0` and `t1`, which covers most arithmetic and
basic stack words.

### Push and pop

An ordinary push moves the two tops and touches at most one frame slot.
When needed, the old `t1` goes into the frame, `t0` moves to `t1`, and the new
value becomes `t0`. Pop runs the same steps in reverse. If the cached frame is
full, RPyForth first moves its deepest cell to `spill`. If the cache is empty,
pop reads the top of `spill` instead.

The cache is small on purpose. A larger frame would catch more stack values, but
it would also give the JIT more slots to track. `spill` handles deep stacks
without pulling the whole data stack into the optimizer's virtual state.

## Aligning a fragment at a word call

Before RPyForth enters a colon-defined word, it trims the active cache to its
top two cells. `t0` and `t1` stay where they are, while any live frame cells are
appended to `spill`.

```text
before the call                         after normalization

t0       = e  (top)                     t0       = e  (callee argument)
t1       = d                            t1       = d  (callee argument)
frame[2] = c
frame[1] = b
frame[0] = a

                                        spill[2] = c
                                        spill[1] = b
                                        spill[0] = a

cache_depth = 5                         cache_depth = 2
spill_ptr   = 0                         spill_ptr   = 3
```

This ordering follows `push_fragment_on()` directly. If `ap` is the old
`spill_ptr`, the loop copies `frame[i]` to `spill[ap + i]` for every live frame
cell. It then advances `spill_ptr` and sets `cache_depth` to two. Thus the copy
preserves index order: `frame[0]` becomes the deepest newly parked value, and
the highest live frame index becomes the new `spill[spill_ptr - 1]`.

The top two cells become the callee's initial window. They often hold its
arguments, so common calls pass them across the boundary without a copy. The
rest of the caller's cached state waits below them in `spill`. This is not an
access barrier: a callee that needs a third argument can still read it through
the normal stack operations.

Return is simple. The callee has already left its results on the
same logical stack, with the saved caller values immediately below them, so
there is nothing to copy back.

Despite the name, a fragment is not a heap object allocated for each call. It
is an implicit window over the cached frame and the used prefix of one shared,
preallocated spill array. Recursive calls do not build a linked list of
fragment objects.

### Walking through a real call

An example makes the alignment easier to follow. `SUMSQ` computes the sum of two
squares while another value remains below its arguments:

```forth
: SQR    DUP * ;
: SUMSQ  SQR SWAP SQR + ;

9 3 4 SUMSQ + .   \ prints 34
```

Just before the call, the cached frame holds all three values:

```text
t0 = 4, t1 = 3, frame[0] = 9
cache_depth = 3, spill_ptr = 0
```

On entry, `SUMSQ` keeps `4` and `3` in the scalar tops. The unrelated `9` moves
to `spill[0]`:

```text
t0 = 4, t1 = 3, frame empty, spill[0] = 9
cache_depth = 2, spill_ptr = 1
```

The table records the important word boundaries. A dash marks an unused logical
slot. Physical slots may still contain old bits, but values outside
`cache_depth` are not part of the stack.

```text
event                  t0    t1    frame    spill    cache_depth  spill_ptr
---------------------------------------------------------------------------
push 9 3 4              4     3      9        -           3          0
enter SUMSQ             4     3      -        9           2          1
return from first SQR  16     3      -        9           2          1
SWAP                    3    16      -        9           2          1
return from second SQR  9    16      -        9           2          1
SUMSQ's +              25     -      -        9           1          1
return from SUMSQ      25     -      -        9           1          1
caller's +             34     -      -        -           1          0
```

Neither call to `SQR` parks anything because `cache_depth` is already two.
Inside `SQR`, `DUP` briefly reuses `frame[0]`, then `*` brings the cache back to
two cells. `SUMSQ` eventually leaves `25` in `t0` and returns without moving
`9`. The caller's final `+` is the first operation that reaches below the cache.
It pops `9` from `spill` and produces `34`.

### Does RPyForth reuse fragments?

Yes, but it reuses storage, not fragment objects. There is no allocation per
call, no free list, and no object pool.

The fixed-size `frame` is scratch space for the cached frame. As soon as its
contents move to `spill`, the callee may overwrite those slots. The first `DUP`
in the example writes to the newly vacant `frame[0]`.

The spill array is reusable too, but a live stack value keeps its slot. Popping
a spilled value decrements `spill_ptr`. `THROW` restores the value captured by
`CATCH`, and a stack reset sets it to zero. The next park may overwrite any
cells above the current pointer.

In the example, `spill[0]` remains occupied while `9` is live. The caller's
final `+` consumes it and changes `spill_ptr` from one to zero. The next saved
value can then use the same slot.

One detail matters here: return does not move `spill_ptr` backwards. A Forth
word may consume values below its initial window, or it may leave a different
number of results. Spill slots therefore follow the lifetime of values on the
data stack, not the lifetime of a call.

## What the meta-tracing JIT compiler gets from this layout

In the stack-fragmented build, the interpreter object is a PyPy
_virtualizable_. Its scalar tops, `cache_depth`, `spill_ptr`, and the small
frame are part of the virtualizable state. The large spill array is not.

While tracing, the JIT treats `t0` and `t1` as ordinary values. It can carry them
through inlined Forth words and remove the interpreter's field loads and stores.
It can also track the small frame one slot at a time. If a hot loop's working
stack fits in the cached frame, its stack traffic can compile down to register
operations even though the Forth program still sees a shared data stack.

Deep or irregular accesses still reach `spill` and remain real memory
operations. That is the trade-off: the JIT sees a small cache that it can track,
while ordinary memory preserves the rest of the Forth stack.

## Preliminary evaluation

We evaluate RPyForth against the dedicated Forth interpreter and JIT compilers:
Gforth (fast) version 0.7.9, SwiftForth x64-Linux 4.1.8, and VFXForth 64 5.43.
The latter two compilers are commercial grade. We only use them for this evaluation.

We run the [shootout](https://dada.perl.it/shootout/) benchmark suite and
measure the elapsed time of the four (RPyForth, GForth (fast), SwiftForth, and
VFXForth) targets.

## Acknowledgements

This project is a collaboration with Kota Hakamada, a master's student at [Tokyo
Metropolitan University](https://www.tmu.ac.jp/english/index.html).
