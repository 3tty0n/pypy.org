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

In a conventional stack-based virtual machines (VMs), the operand stack is
mainly an intermediate execution representation. A compiler can analyze data
dependencies and replace stack operations with single static assignment (SSA)
values or registers. As a result, the concrete operand-stack representation
often disappears from optimized machine code.

On the other hand, in stack-oriented languages, including
[Forth](https://www.forth.com/forth/),
[PostScript](https://www.adobe.com/products/postscript.html), and
[Factor](https://factorcode.org),
the operand stack called _data stack_ is part of the programming model
and carries values across _word_ boundaries. Values are passed implicitly
through the data stack across word invocations, rather than through explicitly
named parameters and return values.
A typical interpreter represents this stack as a mutable memory region together
with a stack pointer.
Each word therefore performs concrete stack-pointer updates and loads/stores to
stack slots, while successive words communicate through the same share stack
representation.

These stack accesses are structurally more difficult to be eliminated than
the operand-stack operations of a conventional bytecode VM. The stack state
remains live across word boundaries. Stack effects are encoded implicitly in
word behavior rather than in explicit function signatures.
The interpreter repreatedly materializes the shared stack through indexed memory
accesses and stack-pointer manipulation.

This raises a natural question: can a meta-tracing compiler eliminate the
interpreter's concrete stack loads, stores, and stack-pointer updates while
preserving the language-level semantics of the data stack? To answer this
question, we propose RPyForth: an ANS Forth interpreter written in RPython.

## What is a Stack-Oriented Language and Forth?

A stack-oriented language is a programming language in which an operand stack is
part of the programming model. Forth belongs to a broader family of
stack-oriented languages that includes Factor, Joy, and PostScript. In these
languages, the stack is not merely an internal execution representation of a
virtual machine. Instead, it is exposed to the programmer: words consume values
from the stack and leave their results for subsequent words.

In a conventional language, we might write a function that squares a value and
then adds one as follows:

```python
def square(x):
    return x * x

def add_one(x):
    return x + 1

result = add_one(square(5))
```

The same computation can be written in Forth as:

```forth
: square   dup * ;
: add-one  1 + ;
: compute  square add-one ;

5 compute
```

A colon definition introduces a new word. The word `square` duplicates the top
value of the data stack and multiplies the two values, while `add-one` pushes
`1` and performs addition.

Forth words are often documented using a *stack-effect notation*:

```forth
: square   ( n -- n^2 ) dup * ;
: add-one  ( n -- n+1 ) 1 + ;
: compute  ( n -- n^2+1 ) square add-one ;
```

The values to the left of `--` are consumed from the stack, and the values to
the right are left on the stack.

The execution of `5 compute` changes the data stack as follows. Here, the
rightmost element is the top of the stack:

```text
[]          initial state
[5]         push 5

[5, 5]      dup
[25]        *

[25, 1]     push 1
[26]        +
```

The same execution can be viewed at word boundaries:

```text
(figure here)
```

The value `25` is not returned from `square` through an explicit return value
and then passed to `add-one` as an explicit argument. Instead, `square` leaves
`25` on the shared data stack, and `add-one` consumes it from that same stack.

Conceptually, the composition

```forth
square add-one
```

corresponds to the nested expression

```text
add_one(square(x))
```

However, the data flow is represented implicitly through stack effects rather
than through named parameters and return values.

A straightforward Forth interpreter represents the data stack as a mutable array
together with a stack pointer. Consequently, the execution above is implemented
as a sequence of stack-slot reads and writes and stack-pointer updates. More
importantly, the concrete stack state remains live between successive word
invocations: the value written by `square` is later read by `add-one`. This
differs from an operand stack that exists only as a temporary execution
representation inside a conventional stack-based VM.

## Acknowledgements

This project is a collaboration with Kota Hakamada, a master's student at [Tokyo
Metropolitan University](https://www.tmu.ac.jp/english/index.html).
