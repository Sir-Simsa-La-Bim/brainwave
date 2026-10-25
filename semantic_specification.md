# Brainwave
## Semantic Specification

**Draft version:** 0.1

**Source file extension:** `.brnw`

This document defines the normative meaning of programs after successful lexical and syntactic analysis. The EBNF grammar defines form; this document defines behavior, validation, and compiler obligations.

## 1. Execution model

The target machine is standard eight-command Brainfuck with commands `+ - < > [ ] , .`.

- Memory is a linear tape of cells.
- Every cell contains one unsigned byte.
- `+` and `-` wrap modulo 256.
- The language introduces no additional runtime instructions. `goto`, repetition suffixes, and layout declarations are compile-time notation that lowers to standard Brainfuck or compiler metadata.

Tape bounds, end-of-input behavior, and external I/O encoding are properties of the parent Brainfuck runtime unless separately constrained by the parent system.

## 2. Function interface

A function signature declares logical data flow.

```text
def example(byte[4] input, byte mode) -> (byte[2] result)
```

- `byte` denotes one logical byte.
- `byte[N]` denotes an ordered sequence of `N` logical bytes.
- `N` must evaluate to an integer greater than zero.
- Signature types do not define physical tape addresses, adjacency, or calling layout.
- Inputs may be unused.
- Every individual output byte must be assigned exactly once in the `exit` section.

Inputs, outputs, and blocks share one function-local namespace. Duplicate names are semantic errors.

## 3. Section contract

A function contains `entry`, `code`, and `exit`, in that order.

- `entry` declares all named physical blocks and their required initial contents.
- `code` defines the algorithm and pointer-control assertions.
- `exit` identifies outputs and asserts the final state of every declared block.

Declarations and specifications from different sections cannot be interleaved.

## 4. Blocks and layout expansion

Each entry statement declares one named physical block:

```text
work <= (input[1:4], 0, ?, u16(0x1234));
```

After expansion, the items occupy consecutive cells in listed order. The example expands to seven cells:

```text
input[1], input[2], input[3], 0, ?, 0x34, 0x12
```

Rules:

1. Every block is nonempty.
2. Blocks do not overlap.
3. The relative order and distance between different blocks are unspecified.
4. Required adjacency between values must be expressed within one block.
5. Blocks may contain input references, literals, and `?`, but never references to other blocks.
6. One logical input byte may be absent, used once, or replicated into several physical cells.
7. The compiler or parent system must realize the requested entry layout, including any required copying.

A block's size is inferred from its fully expanded entry layout.

## 5. References, indexing, and slicing

A bare logical reference expands to the complete declared value. Indexing is zero-based. Negative indices count from the end.

```text
value       // all bytes
value[0]    // first byte
value[-1]   // last byte
value[1:4]  // indices 1, 2, 3
value[:]
```

Slice starts are inclusive and ends exclusive. Missing bounds mean the corresponding sequence boundary. Slice steps are unsupported.

Unlike Python's clipping behavior, out-of-range indices and slice boundaries are semantic errors when detectable. A slice that expands to zero bytes is also an error.

The optional namespaces `in.` and `out.` must agree with the declaration kind. Entry layouts may reference inputs only. Exit layouts may reference inputs and outputs, but not blocks.

## 6. Boundary values

### 6.1 Unknown cells

`?` declares an ordinary writable cell whose value is unconstrained at that boundary.

- In `entry`, any initial byte is permitted.
- In `exit`, no assertion is made about the final byte.
- `?` does not mean unallocated or inaccessible.

### 6.2 One-cell integer literals

An unwrapped integer occupies one cell and is reduced modulo 256.

```text
256  // 0
-1   // 255
```

### 6.3 Multi-cell integer constructors

`uN(value)` and `iN(value)` require a positive width divisible by eight. The width need not be a power of two.

- The value is reduced modulo `2^N`.
- Exactly `N / 8` cells are emitted.
- Bytes are emitted least significant first.
- `u` and `i` document interpretation but produce the same selected bit pattern.

### 6.4 Floating-point constructors

`float16`, `float32`, and `float64` emit IEEE binary16, binary32, and binary64 bit patterns in little-endian order.

- Decimal conversion uses round-to-nearest, ties-to-even.
- Overflow, underflow, subnormal values, infinities, and signed zero follow IEEE behavior.
- `nan` emits a canonical positive quiet NaN: `0x7E00`, `0x7FC00000`, or `0x7FF8000000000000`.
- Other NaN encodings may be produced with an integer constructor.

### 6.5 String constructors

`ascii` and `utf8` expand their string argument into bytes without an implicit terminator.

- `ascii` accepts ASCII characters and raw bytes expressed with `\xNN`.
- `utf8` encodes Unicode scalar values as UTF-8.
- Invalid escapes, surrogate code points, and values above `U+10FFFF` are errors.
- Adjacent literals are not concatenated.

All literal forms have identical expansion semantics in `entry` and `exit`.

## 7. Code lowering

The `code` section contains raw Brainfuck commands, repetition suffixes, `goto`, and `hint`.

### 7.1 Repetition suffixes

A directly attached decimal count repeats `+`, `-`, `<`, or `>`.

```text
+3  // +++
>2  // >>
```

The count must be greater than zero. Leading zeros are allowed. Ignored text may not appear between the command and count. `+-3` is one `+` command followed by `-3`.

A compiler may reduce repeated `+` and `-` modulo 256. It must not reduce pointer movement modulo 256.

### 7.2 Initial pointer state

The function author cannot assume a data-pointer location at entry. Therefore, the first executable code element must be `goto`.

The parent system must provide enough placement information for the compiler to lower the initial `goto` to pointer movement from the actual calling state.

### 7.3 `goto`

`goto(block[index]);` lowers to the required sequence of `<` or `>` commands. A bare block name means index zero.

- The target must be exactly one cell of a declared block.
- Negative indices are allowed.
- Slices are not allowed.
- Except for the initial `goto`, the compiler must know the current pointer position.
- `goto` may occur inside a Brainfuck loop.

### 7.4 `hint`

`hint(block[index]);` emits no Brainfuck command. It asserts that the pointer is currently at the named cell and resumes pointer tracking from there.

- `hint` may occur inside loops.
- Redundant hints are allowed.
- A compiler is not required to verify a hint.
- A compiler may reject a hint that it can prove false.
- A false accepted hint makes the algorithm contract incorrect.

## 8. Pointer analysis

The compiler tracks pointer displacement through straight-line code and nested pointer-balanced loops.

A loop is pointer-balanced when the compiler can establish that every completed iteration returns the pointer to the loop-entry position. For example:

```text
[->+<]
```

After an unbalanced loop such as `[>]`, the pointer position is unknown. A truthful `hint` is required before a later `goto` and before the end of `code`.

No particular final location is required, but the location must be known to the compiler when `code` ends.

## 9. Exit semantics

Every declared block has exactly one exit statement.

An ordinary exit layout expands cell-for-cell and must have the same size as the entry layout.

```text
work => (result[:], in.input[0], 0, ?);
```

Each position means:

- Output reference: the physical cell supplies that output byte.
- Input reference: the physical cell contains the original function-entry value of that input byte.
- Literal: the physical cell contains the expanded known value.
- `?`: the cell's final value is unconstrained.

A physical cell supplies at most one output byte. Every declared output byte must appear exactly once across all ordinary exit layouts.

### 9.1 Preserved blocks

```text
work => preserved;
```

This asserts that every physical cell contains exactly its own entry value at exit. For an entry `?`, the same concrete unknown byte must remain, not merely another unconstrained value.

`preserved` cannot simultaneously assign outputs. Use an ordinary exit layout when an unchanged cell must also supply an output.

## 10. Assertions and verification

Entry layouts are preconditions supplied by the compiler or parent system. The following are author assertions:

- `hint`
- Exit literals
- Original input references in exit layouts
- `preserved`

A conforming compiler may trust these assertions. It may additionally verify them and reject provably false claims. Assertions do not generate runtime checks or cleanup code.

## 11. Access outside declared blocks

Raw Brainfuck may move to and modify cells outside named blocks.

- Their initial values are unspecified.
- They cannot be named in layouts, `goto`, `hint`, or `exit`.
- The programmer cannot assume that a separate block is adjacent.
- The compiler may expose external-cell access through implementation-defined metadata.

## 12. I/O and derived properties

The commands `,` and `.` remain valid. Their use is not represented by function parameters or results.

A compiler or parent system may derive implementation-specific properties such as I/O use, hints, unbalanced loops, or external-cell access. The language does not prescribe a fixed flag set.

A parent system may assign special meaning to selected function names, such as `addUint`, including required signatures or substitution behavior. Such conventions are outside the language semantics.

## 13. Diagnostics

Implementations should distinguish:

- Lexical errors: malformed tokens, strings, or escapes.
- Syntax errors: token sequences that violate the grammar.
- Semantic errors: duplicate names, invalid references, layout-size mismatches, missing outputs, unknown pointer state, and similar contract violations.
- Unverified assertions: accepted claims whose truth was not proven.

Compile-time detection is required where this specification declares a semantic error and the relevant values are statically available.
