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
.. has_math: true
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
slots. These operations are harder to remove than the temporary stack access
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

Forth programmers usually document words with
[*stack-effect comments*](https://www.complang.tuwien.ac.at/forth/gforth/Docs-html/Stack_002dEffect-Comments-Tutorial.html):

```forth
: square   ( n -- n^2 ) dup * ;
: add-one  ( n -- n+1 ) 1 + ;
: compute  ( n -- n^2+1 ) square add-one ;
```

The values to the left of `--` are consumed, and the values to the right remain
after the word finishes. These annotations are ordinary Forth comments.

Here is the stack while `5 compute` runs. The rightmost value is on top:

```text
[]          # initial state
[5]         # push 5

[5, 5]      # dup
[25]        # *

[25, 1]     # push 1
[26]        # +
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

We built RPyForth to test whether meta-tracing can remove this stack access.
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
3. all deeper cells, stored in a preallocated shared array named `spill`.

The default frame has eight cells, so the active cache can hold ten values in
total: two scalar tops and eight frame slots. Anything deeper goes to `spill`.

After pushing the values `0` through `13`, the layout looks like this. Depth
zero is the top of the stack.

<div class="math">
$$
\begin{array}{r@{\qquad}l@{\;}l}
\text{depth} & \textit{top of stack} \\[0.6ex]
\begin{array}{r} 0 \\ 1 \end{array} &
\left.\begin{array}{@{}l@{}}
  \rlap{t_0 = 13}\hphantom{\mathit{frame}[7] = 11} \\
  \rlap{t_1 = 12}\hphantom{\mathit{frame}[7] = 11}
\end{array}\right\} & \text{cached fields} \\[3.2ex]
\begin{array}{r} 2 \\ \vdots \\ 9 \end{array} &
\left.\begin{array}{@{}l@{}}
  \mathit{frame}[7] = 11 \\
  \vdots \\
  \mathit{frame}[0] = \phantom{0}4
\end{array}\right\} & \text{cached frame} \\[4.5ex]
\begin{array}{r} 10 \\ \vdots \\ 13 \end{array} &
\left.\begin{array}{@{}l@{}}
  \rlap{\mathit{spill}[3] = \phantom{0}3}\hphantom{\mathit{frame}[7] = 11} \\
  \vdots \\
  \rlap{\mathit{spill}[0] = \phantom{0}0}\hphantom{\mathit{frame}[7] = 11}
\end{array}\right\} & \text{shared spill} \\[3.2ex]
 & \textit{bottom of stack}
\end{array}
$$
</div>

The indexes in both arrays increase toward the top of their area. In the
active cache, `frame[0]` is the deepest cached value, while the highest live
frame index sits immediately below `t1`. Likewise, `spill[0]` is the deepest
spilled value and `spill[spill_ptr - 1]` sits immediately below the cache. The
lookup code uses these index calculations:

<div class="math">
$$
\begin{aligned}
\mathit{frame\_index} &= \mathit{cache\_depth} - 1 - \mathit{depth} \\
\mathit{spill\_index} &= \mathit{spill\_ptr} - 1 - (\mathit{depth} - \mathit{cache\_depth})
\end{aligned}
$$
</div>

The first formula applies to cached depths of two or greater; depths zero and
one map to `t0` and `t1`.

Two counters tie the areas together. `cache_depth` is the number of live cells
in the active cache, including `t0` and `t1`, and `spill_ptr` is the number in
`spill`. The logical depth is therefore

<div class="math">
$$
\mathit{depth}_{\text{logical}} = \mathit{cache\_depth} + \mathit{spill\_ptr}
$$
</div>

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

<div class="math">
$$
\begin{array}{c@{\qquad}c@{\qquad}c}
\text{before the call} & & \text{after normalization} \\[1.4ex]
\begin{array}{@{}l@{\;}l@{}}
  t_0 &= e \quad (\text{top}) \\
  t_1 &= d \\
  \mathit{frame}[2] &= c \\
  \mathit{frame}[1] &= b \\
  \mathit{frame}[0] &= a \\[0.8ex]
  \hline \\[-1.8ex]
  \mathit{cache\_depth} &= 5 \\
  \mathit{spill\_ptr} &= 0
\end{array}
& \Longrightarrow &
\begin{array}{@{}l@{\;}l@{}}
  t_0 &= e \quad (\text{callee argument}) \\
  t_1 &= d \quad (\text{callee argument}) \\
  \mathit{spill}[2] &= c \\
  \mathit{spill}[1] &= b \\
  \mathit{spill}[0] &= a \\[0.8ex]
  \hline \\[-1.8ex]
  \mathit{cache\_depth} &= 2 \\
  \mathit{spill\_ptr} &= 3
\end{array}
\end{array}
$$
</div>

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

<div class="math">
$$
t_0 = 4,\quad t_1 = 3,\quad \mathit{frame}[0] = 9,\quad
\mathit{cache\_depth} = 3,\quad \mathit{spill\_ptr} = 0
$$
</div>

On entry, `SUMSQ` keeps `4` and `3` in the scalar tops. The unrelated `9` moves
to `spill[0]`:

<div class="math">
$$
t_0 = 4,\quad t_1 = 3,\quad \mathit{frame} = \varnothing,\quad
\mathit{spill}[0] = 9,\quad
\mathit{cache\_depth} = 2,\quad \mathit{spill\_ptr} = 1
$$
</div>

The table records the important word boundaries. A dash marks an unused logical
slot. Physical slots may still contain old bits, but values outside
`cache_depth` are not part of the stack.

| event | `t0` | `t1` | `frame` | `spill` | `cache_depth` | `spill_ptr` |
|---|---:|---:|---:|---:|---:|---:|
| push `9 3 4`             |  4 |  3 | 9 | - | 3 | 0 |
| enter `SUMSQ`            |  4 |  3 | - | 9 | 2 | 1 |
| return from first `SQR`  | 16 |  3 | - | 9 | 2 | 1 |
| `SWAP`                   |  3 | 16 | - | 9 | 2 | 1 |
| return from second `SQR` |  9 | 16 | - | 9 | 2 | 1 |
| `SUMSQ`'s `+`            | 25 |  - | - | 9 | 1 | 1 |
| return from `SUMSQ`      | 25 |  - | - | 9 | 1 | 1 |
| caller's `+`             | 34 |  - | - | - | 1 | 0 |

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
stack fits in the cached frame, its stack access can compile down to register
operations even though the Forth program still sees a shared data stack.

Deep or irregular accesses still reach `spill` and remain real memory
operations. That is the trade-off: the JIT sees a small cache that it can track,
while ordinary memory preserves the rest of the Forth stack.

## Preliminary evaluation

We ran the shootout benchmarks on RPyForth and three existing Forth systems:
[Gforth](https://gforth.org/) 0.7.9, running as `gforth-fast`,
[SwiftForth](https://www.forth.com/swiftforth/) x64-Linux 4.1.8, and
[VFX Forth](https://www.mpeforth.com/) 64 5.43. The last two are commercial
systems. This is a comparison of these workloads, not a general ranking of
Forth implementations.

The workloads are the Forth ports of the [shootout](https://dada.perl.it/shootout/)
suite. We use the median time for each benchmark and the geometric mean of the
per-benchmark ratios. Ratios are easier to read here because the absolute times
span four orders of magnitude.

We measured the stable and warmup performance by iterating 50 times for each
benchmark. We use median for reporting the benchmarking data.

Next, we see the differences in terms of code generation/compilation on the
four systems.

### How the four systems produce machine code

All four systems generate some machine code while the Forth process is running,
but they do it at different times and with different information.
`gforth-fast` uses
[dynamic superinstructions with replication](https://gforth.org/manual/Dynamic-Superinstructions.html):
it copies machine-code fragments for Forth primitives and joins adjacent
fragments where possible.

The
[VFX code generator](https://www.mpeforth.com/software/pc-systems/vfx-forth-common-features/)
and the
[SwiftForth optimizing compiler](https://www.forth.com/swiftforth/)
generate optimized native code as a definition is compiled. They use inlining
and other compile-time rules, but no execution profile. The code is ready
before the word first runs, so there is no JIT warm-up.

On the other hard, RPyForth performs meta-tracing compilation:
recording an executed path, compiling a  specialized trace, and adding guards
for the assumptions in that trace. It means that compilation
costs time, and a failed guard may lead back to the interpreter or to a bridge.

| System | Code generation | Execution feedback | Speculative guards | Execution-triggered warm-up |
|---|---|---|---|---|
| `gforth-fast` | threaded code with replicated primitive fragments and dynamic superinstructions | no | no | no |
| VFX Forth, SwiftForth | native code at definition time | no | no | no |
| RPyForth | native code from recorded execution traces | yes | yes | yes |

Next, we see the result of the stable performance of three interesting warmup patterns.

### Result: Stable performance

![Figure 1](/images/2026-08-rpyforth-shootout.png)

Figure 1 shows RPyForth time divided by baseline time, so values below 1 mean
that RPyForth is faster. In our setup, RPyForth is faster than the three targets:
1.42x faster than VFX Forth, 1.62x faster than `gforth-fast`, and 2.74x faster than
SwiftForth.

The slower results show the current boundary of RPyForth's optimization: work
that cannot stay as simple operations in a stable trace still falls onto paths
we have not optimized well. The effect is largest in `hash2`, which is 5.4x
slower than `gforth-fast`: it calls 200 different words through one highly
polymorphic `EXECUTE` site, while a trace specializes to the execution tokens
it observes. Its timing and warm-up curve are consistent with repeated guard
failures and bridge activity, although we still need the JIT log to count
them. The effect is smaller in `wordfreq`, at 1.8x slower, where each word is
copied from Forth's byte-addressed memory into an allocated RPython string
before lookup. Keeping similar parsing and formatting paths on the byte buffer
already produced large gains in `sumcol`, `hash`, and `moments`, so this
boundary is the next likely target, though `wordfreq` also includes dictionary
and file operations. Finally, `reversefile` and `spellcheck` finish in
single-digit microseconds, too quickly to amortize fixed runtime costs;
`reversefile` does not produce a JIT trace at all, so these two results say
little about compiled-code quality.

### Result: Warm-up depends on the trace

A steady-state median tells us where a benchmark ends up, but not how it gets there.
So, we should observe the warmup behavior next.

We took 50 iterations for each benchmark and plotted an elapsed time in each iteration.
Figure 2 shows three of these curves. The vertical axis is logarithmic. RPyForth is red,
`gforth-fast` is blue, VFX Forth is purple, and SwiftForth is orange. The
dotted lines show the steady-state summaries.

![Figure 2: Selected warm-up curves for callheavy, sumcol, and hash2](/images/2026-08-rpyforth-warmup-selected.png)

`callheavy` pays most of the cost on its first iteration and then settles. `sumcol`
appears to settle, but improves again around iteration 17. `hash2` keeps moving
between two levels and occasionally spikes. In other words, RPyForth may warm up at once,
in stages, or not reach one stable level during the measured run.

The largest wins against `gforth-fast` are `callheavy` at 23.8x, `ack` at 3.4x,
`heap` at 3.3x, and `matrix` at 2.7x. These are the workloads where the fragment
layout should help: hot stack values remain visible to the trace optimizer
instead of being written to the shared stack array. The result is consistent
with that explanation, although an endpoint comparison does not isolate the
layout from the rest of RPyForth.

## Conclusion and future work

RPyForth began with a simple question: can a meta-tracing JIT optimize Forth's
data stack across word calls? The metastack is our answer. It keeps the active
part of the stack in two scalar fields and a small frame that the JIT can
track, while a shared spill array handles deeper values. Forth programs still
see one data stack, and word calls do not allocate fragment objects.

The preliminary results are encouraging. RPyForth is faster on average than
the three systems in the shootout. However, these comparisons do
not tell us how much of the speedup comes from the metastack itself yet.
To measure that effect, we plan to compare RPyForth with and without the
fragmented layout and inspect the optimized traces for stack loads and stores.

There are also clear places to improve. In `hash2`, 200 different words pass
through the same polymorphic `EXECUTE` site. We need to inspect the JIT logs to
see how much time is lost to guard failures and bridges. `wordfreq` has a
different problem: it copies words from Forth's byte-addressed memory into
RPython strings before lookup. Keeping that path on the byte buffer should
remove those allocations.

After improving these paths, we plan to evaluate RPyForth with larger programs,
including the [Forth appbench suite](https://www.complang.tuwien.ac.at/forth/appbench-1.4.zip).
Ertl and Paysan used appbench in their
[ECOOP 2024 paper](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.ECOOP.2024.14)
because it contains substantial programs that are closer to idiomatic Forth
applications than small benchmarks. We will use it to compare steady-state
performance with Gforth, SwiftForth, and VFX Forth, and to see whether the
warm-up patterns observed in the shootout suite still appear in larger
programs.

## Acknowledgements

This project is a collaboration with Kota Hakamada, a master's student at [Tokyo
Metropolitan University](https://www.tmu.ac.jp/english/index.html).

