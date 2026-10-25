# Compiler Integration Guide

This guide describes one practical architecture for embedding the language in a parent compiler. It is non-normative except where it restates language requirements.

## 1. Recommended pipeline

Use explicit stages:

1. **Lexing**
   - Remove whitespace and comments between tokens.
   - Preserve source spans for diagnostics.
   - Recognize compound tokens such as `->`, `<=`, and `=>` before their individual characters.
   - Recognize repetition suffixes only when decimal digits are directly attached to `+`, `-`, `<`, or `>`.

2. **Parsing**
   - Build an abstract syntax tree from `grammar.ebnf`.
   - Keep raw code elements in source order.
   - Match Brainfuck loop brackets or represent them as nested loop nodes.

3. **Name and type analysis**
   - Register inputs and outputs first.
   - Add block names while analyzing `entry`.
   - Enforce the shared namespace.
   - Resolve optional `in.` and `out.` qualifiers.

4. **Layout expansion**
   - Expand slices and typed literals to cell-level items.
   - Infer each block size.
   - Build symbolic addresses such as `(block_id, offset)`.

5. **Contract validation**
   - Match exactly one exit specification to each block.
   - Check expanded entry and exit sizes.
   - Count every output-byte assignment.
   - Classify exit assertions.

6. **Pointer analysis and code lowering**
   - Track symbolic or allocated pointer addresses.
   - Lower repetition suffixes and `goto`.
   - Treat `hint` as a zero-length assertion node.
   - Mark pointer state unknown after unbalanced loops.

7. **Allocation and composition**
   - Assign physical tape ranges to blocks.
   - Insert input-copy/setup code.
   - Insert the lowered algorithm.
   - Collect outputs or expose their final cell locations.

8. **Optional verification and optimization**
   - Verify hints and exit assertions where practical.
   - Simplify repeated arithmetic modulo 256.
   - Remove redundant pointer movement.

## 2. Suggested intermediate representation

A useful function representation contains:

```text
Function
  name
  inputs[]
  outputs[]
  blocks[]
  codeTree
  assertions[]
  derivedProperties
```

A logical byte reference can be represented as:

```text
LogicalByte(kind, declarationId, byteIndex)
```

A physical block cell can be represented as:

```text
BlockCell(blockId, offset)
```

Keep logical values and physical cells distinct. One logical input byte may map to several block cells.

## 3. Layout item expansion

Normalize every layout into a flat list before allocation.

Examples:

```text
input[1:4]
```

becomes:

```text
InputByte(input, 1)
InputByte(input, 2)
InputByte(input, 3)
```

```text
u24(0x123456)
```

becomes:

```text
KnownByte(0x56)
KnownByte(0x34)
KnownByte(0x12)
```

An entry `?` becomes an unconstrained initial cell. An exit `?` becomes an unconstrained final-state classification. Preserve these as different IR roles even if they share source syntax.

## 4. Block allocation

Each block needs one contiguous, non-overlapping tape range. The language does not constrain relative block order.

An allocator may optimize order based on:

- Frequency of `goto` transitions
- Setup and output-copy costs
- Parent-system reserved regions
- Alignment or guard-cell policies

Do not expose chosen adjacency as a language guarantee. Recompilation may reorder blocks.

## 5. Entry realization

The parent system is responsible for supplying logical inputs. For every expanded entry cell:

- Input reference: ensure the referenced logical byte is present, copying if necessary.
- Literal: initialize the cell to the required byte.
- `?`: provide a cell without making an initial-value guarantee.

Because an input can appear multiple times, entry realization is a data-placement problem rather than a one-to-one binding.

The compiler may reuse an already suitable cell if doing so does not violate block contiguity, non-overlap, or caller ownership constraints.

## 6. The initial `goto`

Source semantics say the author's entry pointer position is unknown, while compilation still requires a concrete calling state.

The parent system should provide one of:

- The actual pointer address at function entry
- A symbolic caller pointer resolved during whole-program layout
- A calling convention that places the pointer at a known anchor

The compiler then lowers the first `goto` from that calling state to the target cell. A standalone fragment cannot navigate from a genuinely unknowable runtime position without an external convention.

## 7. Pointer-state lattice

A minimal analysis state is:

```text
Known(address)
Unknown
```

For pre-allocation analysis, `address` may be symbolic:

```text
BlockCell(work, 2) + displacement
```

After allocation, it may be a concrete tape offset.

Command transfer rules:

- `+`, `-`, `.`, `,`: no pointer change.
- `>` and `<`: adjust a known address; unknown remains unknown.
- Balanced loop: preserve the loop-entry pointer state.
- Unbalanced loop: produce `Unknown` after the loop.
- `goto`: require known state, except for the initial special case; set state to target.
- `hint`: set state to asserted target without emitted code.

Nested loops are easiest to analyze with a structured code tree rather than a flat bracket stream.

## 8. Lowering `goto`

Once source and target addresses are known:

```text
delta = target - current
```

- Positive delta emits `>` repeated `delta` times.
- Negative delta emits `<` repeated `-delta` times.
- Zero emits nothing.

The compiler may retain compact repetition notation internally, but final output must use standard Brainfuck commands unless its downstream interface accepts counts.

## 9. Exit handling

Treat ordinary exit layouts as a cell-level contract.

- Output references identify result locations.
- Input references and literals are assertions.
- `?` requires no action.
- `preserved` expands conceptually to entry-value identity assertions for all block cells.

The language does not require outputs to be copied into a separate calling area. A parent system may instead treat their current cells as the returned locations. Copy only when required by the caller's layout.

## 10. Verification strategy

Verification can be incremental:

### Required static checks

Implement all deterministic semantic checks, including names, bounds, block matching, layout sizes, output completeness, bracket matching, and pointer-known requirements.

### Optional assertion checks

A compiler may attempt to prove:

- A `hint` target is correct
- A final cell equals a literal
- A final cell retains an input value
- A block is preserved

Failure to prove is not itself an error. A proof of contradiction may be reported as an error.

## 11. Diagnostics

Diagnostics should include:

- Source span
- Error category
- Short primary message
- Relevant declaration or earlier assignment location
- Optional remediation hint

Examples:

```text
semantic error: output byte 'result[1]' is not assigned
semantic error: block 'work' expands to 4 cells at entry but 3 at exit
semantic error: pointer position is unknown before goto(other)
```

For unverified assertions, use warnings or metadata rather than rejection unless the assertion is proven false.

## 12. Suggested derived properties

The language does not prescribe flags. Potentially useful properties include:

```text
usesInputCommand
usesOutputCommand
usesHints
usesUnbalancedLoops
accessesExternalCells
containsUnverifiedAssertions
maximumNamedBlockSize
```

Keep these out of syntax unless a later language revision deliberately standardizes them.

## 13. Special function names

A parent system may designate names such as:

```text
addUint
copyBytes
compareUint
clearBytes
```

It should define, outside this language specification:

- Required signature
- Value interpretation
- Permitted side effects
- Selection and fallback rules
- Whether several implementations may coexist

The parser should treat these as ordinary identifiers. Special treatment belongs in the integration layer.

## 14. I/O integration

`,` and `.` operate on the Brainfuck runtime's streams. The parent profile should define:

- End-of-input behavior
- Byte or character interpretation
- Whether I/O is permitted in a given context
- Ordering with surrounding generated code

A useful policy is to infer I/O use and reject function placement only when the calling context forbids it.

## 15. External-cell access

Named block contracts do not cover cells reached by raw pointer movement outside allocated blocks.

A conservative compiler can detect possible external access by calculating the minimum and maximum known displacement from each named range. Unbalanced loops may force the result to unknown.

The parent system must ensure that permitted external access cannot corrupt protected memory. Possible policies include guard regions, sandbox ranges, warnings, or contextual rejection.

## 16. Conformance testing

Use `tests/manifest.json` as the test index. A compiler should:

1. Accept every `valid` test.
2. Reject every invalid test in the indicated broad category.
3. Accept `valid-unverified` tests unless it proves an assertion false.
4. Preserve diagnostics as implementation-specific text; exact wording is not standardized.

Add a test whenever parser behavior, expansion, pointer analysis, or contracts change.
