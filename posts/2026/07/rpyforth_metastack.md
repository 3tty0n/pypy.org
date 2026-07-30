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

[Forth](https://www.forth.com/forth/) treats its stack differently. In Forth
and other stack-oriented languages, the _data stack_ is part of the programming
model. A word consumes values from it and leaves results for the next word.
There are no named parameters or return values at these boundaries. The
position of a value on the stack carries the data flow.

A straightforward interpreter stores this stack in a mutable memory region and
keeps a pointer to its top. Each word updates the pointer and reads or writes
stack slots. These operations are harder to remove than temporary stack access
in a bytecode VM because the data stack remains live between words, and a
word's stack effect is implicit in its behavior rather than declared in a
function signature.

Why revisit Forth now? Interpreter performance is not a settled problem.
[Recent work by Ertl and Paysan](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.ECOOP.2024.14)
shows that VM instruction-pointer updates, which had little performance effect
on older hardware, can become a critical dependency chain on recent
out-of-order processors. Old implementation trade-offs can look different on
current hardware. RPyForth examines another interpreter cost, the data stack:
if a meta-tracing JIT compiler observes the program as it runs, how much of the
stack machinery can it remove across word calls?

We built RPyForth to test that question. It is an ANS Forth interpreter written
in RPython, from which the PyPy toolchain generates a meta-tracing JIT compiler.
The aim is to expose the otherwise implicit data flow between words to the
compiler while preserving the data stack seen by Forth programs.

## Background: Forth's data stack

To see why this is difficult for a compiler, consider how Forth expresses the
same data flow as a language with named arguments and return values. In Python,
we might square a value and then add one like this:

```python
def square(x):
    return x * x

def add_one(x):
    return x + 1

result = add_one(square(5))
```

The Forth version uses the data stack to pass the intermediate value:

```forth
: square   ( n -- n^2 ) dup * ;
: add-one  ( n -- n+1 ) 1 + ;
: compute  ( n -- n^2+1 ) square add-one ;

5 compute
```

The colon introduces a word. `square` duplicates the top value and multiplies
the two copies; `add-one` pushes `1` and adds it. The text in parentheses is a
[*stack-effect comment*](https://www.complang.tuwien.ac.at/forth/gforth/Docs-html/Stack_002dEffect-Comments-Tutorial.html):
values on the left of `--` are consumed, and values on the right remain.

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
data stack, where `add-one` finds it. In a straightforward interpreter,
`square` writes an array slot and `add-one` reads it back. RPyForth tries to
make that intermediate value visible to the meta-tracing JIT compiler instead.

## RPyForth and the stack-fragmented layout

RPyForth's interpreter uses indirect threaded code: a colon definition is
a sequence of word references dispatched one at a time. The PyPy toolchain
builds a tracing JIT compiler from this interpreter.

To expose values across those dispatches, RPyForth keeps a small part of the
data stack in a _stack-fragmented layout_ that the tracing JIT compiler can
track. We call the complete three-area representation a _metastack_.

## One stack, three storage areas

The metastack is easiest to think of as a small working area in front of a
larger backing store. The top two cells live in scalar fields named `t0` and
`t1`. The next eight cells live in a small array called `frame`. Deeper values
go into `spill`, one preallocated array shared by the VM.

![Fragmented stack layout](/images/2026-08-rpyforth-metastack-layout.svg){: width="580" }

To a Forth program, these areas are still one stack. Ordinary stack operations
route each depth to the right area. A push moves the old `t1` into `frame`,
moves `t0` to `t1`, and puts the new value in `t0`. Pop follows the same path
backwards. Values cross into `spill` only when the small cache fills or empties.

The two scalar fields and the default eight-cell frame give the active cache
ten cells in total. The shared `spill` array has 16,384 cells.
`RPYFORTH_FRAME_SIZE` can set the frame to between one and 64 cells; a normal
build uses eight. The older `FRAGMENT_SIZE = 256` constant in
`rpyforth/metastack.py` is not used by this stack path.

## What happens at a word call

A colon-defined word starts with a clean two-cell window. Just before the call,
RPyForth keeps `t0` and `t1` in place and moves any live `frame` cells to
`spill`. The top two values are often the callee's arguments, so the common
case crosses the word boundary without copying them.

![Aligning the stack fragment at a Forth word call](/images/2026-08-rpyforth-word-call.svg)

The move does not split the data stack. A callee can still reach a third or
deeper argument through ordinary stack operations. It can also reuse the now
empty `frame` for temporary values.

There is no matching restore step on return. Forth words consume their inputs
and leave their results on the same logical stack, so the caller simply
continues from that state. A fragment is a window over reusable storage, not a
heap object. Calls and recursive calls do not allocate fragment objects or
build a linked list. A spill slot remains in use while its value is live and
can be reused after that value is popped.

## What the meta-tracing JIT compiler gets from this layout

RPyForth marks the scalar fields, the small frame, and their counters so the
tracing JIT compiler can treat them as part of the trace state. During tracing,
the compiler can carry these stack values through inlined Forth words instead
of repeatedly loading and storing them. If a hot loop's working stack fits in
the active cache, much of its stack traffic can become register operations.

The large `spill` array stays in memory. Deep or irregular stack access still
uses it, but the tracing JIT compiler does not have to track the whole data
stack. This is why the active cache is deliberately small.

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
before the word first runs, so execution does not trigger compilation or
warm-up.

On the other hand, RPyForth performs meta-tracing compilation:
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
failures and bridge activity, although we still need logs from the meta-tracing
JIT compiler to count them. The effect is smaller in `wordfreq`, at 1.8x
slower, where each word is copied from Forth's byte-addressed memory into an
allocated RPython string before lookup. Keeping similar parsing and formatting
paths on the byte buffer already produced large gains in `sumcol`, `hash`, and
`moments`, so this boundary is the next likely target, though `wordfreq` also
includes dictionary and file operations. Finally, `reversefile` and
`spellcheck` finish in single-digit microseconds, too quickly to amortize fixed
runtime costs; `reversefile` does not produce a compiled trace at all, so these
two results say little about compiled-code quality.

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

RPyForth began with a simple question: can a meta-tracing JIT compiler optimize
Forth's data stack across word calls? The metastack is our answer. It keeps the
active part of the stack in two scalar fields and a small frame that the
compiler can track, while a shared spill array handles deeper values. Forth
programs still see one data stack, and word calls do not allocate fragment
objects.

The preliminary results are encouraging. RPyForth is faster on average than
the three systems in the shootout. However, these comparisons do
not tell us how much of the speedup comes from the metastack itself yet.
To measure that effect, we plan to compare RPyForth with and without the
fragmented layout and inspect the optimized traces for stack loads and stores.

There are also clear places to improve. In `hash2`, 200 different words pass
through the same polymorphic `EXECUTE` site. We need to inspect logs from the
meta-tracing JIT compiler to see how much time is lost to guard failures and
bridges. `wordfreq` has a different problem: it copies words from Forth's
byte-addressed memory into RPython strings before lookup. Keeping that path on
the byte buffer should remove those allocations.

After improving these paths, we plan to evaluate RPyForth with larger programs,
including the [Forth appbench suite (zip)](https://www.complang.tuwien.ac.at/forth/appbench-1.4.zip).
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
