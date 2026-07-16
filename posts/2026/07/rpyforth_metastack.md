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
and carries values across _word_, which is a function definition in these
languages, boundaries. Values are passed implicitly through the data stack
across word invocations, rather than through explicitly named parameters and
return values.
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

## Acknowledgements

This project is a collaboration with Kota Hakamada, a master's student at [Tokyo
Metropolitan University](https://www.tmu.ac.jp/english/index.html).
