# Brainwave

Brainwave source files use the `.brnw` extension.

## Purpose

This language describes reusable algorithms written in standard Brainfuck while separating three concerns:

1. **Logical data flow** through typed function inputs and outputs.
2. **Physical tape layout** through named, contiguous cell blocks.
3. **Algorithm implementation** through Brainfuck code plus pointer directives.

The target is standard eight-command Brainfuck: `+ - < > [ ] , .`. Every tape cell is one byte and arithmetic wraps modulo 256.

## Function structure

```text
def add(byte a, byte b) -> (byte sum)
{
    entry {
        work <= (a, b, 0);
    }

    code {
        goto(work);
        [->2+<2]
        >[->+<]
    }

    exit {
        work => (0, 0, sum);
    }
}
```

Every function contains exactly these sections, in this order:

- `entry`: declares physical tape blocks and their initial contents.
- `code`: contains Brainfuck commands, `goto`, and `hint`.
- `exit`: describes the final contents and output mapping of every block.

Whitespace and comments are insignificant between tokens. Descriptive statements end with `;`.

## Function signatures

Declarations use C-style type-first syntax:

```text
def transform(byte[4] input, byte mode) -> (byte[4] output, byte status)
```

- `byte` is one logical byte.
- `byte[N]` is an ordered collection of `N` logical bytes.
- A signature describes data flow only. It does not prescribe tape placement or adjacency.
- Input, output, and block names share one namespace and must be unique.
- Empty input and output lists are allowed.

## Entry layouts

An entry statement creates a named contiguous block:

```text
entry {
    work <= (input[1:4], mode, 0, ?, u16(0x1234));
}
```

The layout expands from left to right into physical cells. Blocks cannot overlap, and their relative order on the tape is unspecified. A layout may contain:

- Input references such as `input`, `input[2]`, or `input[1:4]`
- Integer and typed literals
- `?` for a cell with an unknown initial value

An input byte may appear zero, one, or multiple times. The compiler provides any required copies.

## References and slices

References use zero-based, Python-style indexing:

```text
value       // all bytes
value[2]    // one byte
value[-1]   // last byte
value[1:4]  // indices 1, 2, 3
value[:4]
value[2:]
value[:]
```

Slice steps are unsupported. Empty and out-of-range selections are errors when detectable. Optional `in.` and `out.` qualifiers may improve readability:

```text
in.input[0]
out.result[:]
```

## Code and pointer control

The `code` section freely interleaves Brainfuck with pointer directives:

```text
code {
    goto(work);

    +3[
        >2+
        <2-
    ]

    [>]
    hint(work[-1]);
}
```

### `goto`

```text
goto(block);
goto(block[index]);
```

`goto` emits the `<` or `>` commands needed to reach one physical block cell. A bare block name means index zero. The first executable element must be `goto`, because the function-entry pointer position is unspecified.

### `hint`

```text
hint(block[index]);
```

`hint` emits no Brainfuck code. It asserts the current pointer location after code whose final pointer position cannot be inferred, typically an unbalanced loop. A compiler may verify hints, but is not required to do so.

The compiler tracks pointer movement through nested pointer-balanced loops. At the end of `code`, the pointer position must be known, though no particular final cell is required.

### Command repetition

`+`, `-`, `<`, and `>` accept an attached decimal repetition count:

```text
+3    // +++
-4    // ----
>2    // >>
<5    // <<<<<
```

The count must be positive; leading zeros are allowed. `+-3` means one `+` followed by three `-` commands.

## Exit layouts

Every block has exactly one matching exit statement:

```text
exit {
    work => (result[:], in.input[0], 0, ?);
}
```

An exit item means:

- Output reference: the cell supplies that result byte.
- Input reference: the cell contains the original function-entry value.
- Literal: the cell contains that known final value.
- `?`: the final value is unconstrained.

Every declared output byte must be assigned exactly once. Entry and exit layouts must expand to the same block size. Exit descriptions are assertions and do not generate cleanup code.

A completely unchanged block may use:

```text
exit {
    constants => preserved;
}
```

`preserved` means every cell retains its specific entry value, including cells whose entry value was unknown.

## Literals

### One-cell integers

```text
42
-1
0x2A
0b101010
```

An unwrapped integer occupies one cell and is reduced modulo 256.

### Multi-cell integers

```text
u16(0x1234)   // bytes: 34 12
i24(-2)       // three-byte two's-complement bit pattern
u40(7)        // five cells
```

- `uN` and `iN` use a positive width divisible by 8.
- Widths need not be powers of two.
- Values are truncated to their lowest `N` bits.
- All multi-cell values are little-endian.
- Signedness documents intent; both forms emit the selected bit pattern.

### Floating point

```text
float16(1.5)
float32(-2.5e3)
float64(inf)
float32(-inf)
float16(nan)
```

Floating-point literals use IEEE binary16, binary32, or binary64 in little-endian order, with round-to-nearest, ties-to-even. A single canonical positive quiet NaN is used for each width. Other NaN bit patterns can be written with integer constructors.

### Strings

```text
ascii("Hello")
ascii('OK')
utf8("Grüezi")
```

Strings add no implicit null terminator and are not implicitly concatenated. Supported escapes include:

```text
\\  \'  \"  \0  \n  \r  \t  \xNN  \uNNNN  \UNNNNNNNN
```

Invalid escapes, invalid Unicode scalar values, and characters unsupported by the selected encoding are errors. `ascii` also permits arbitrary raw bytes through `\xNN`.

## Comments

```text
// C-style line comment
# Python-style line comment
/* Non-nested block comment */
```

Comments behave as whitespace. Brainfuck characters inside comments are ignored.

## Memory outside declared blocks

Raw Brainfuck may access cells outside named blocks. Their initial values are unspecified, they cannot be referenced in `exit`, and code must not assume that another declared block is adjacent. A compiler may record such access as implementation-defined metadata.

## Contracts and verification

Entry layouts are preconditions supplied by the compiler or parent system. Exit literals, preserved input values, `preserved`, and `hint` are author assertions. A compiler may verify and reject provably false assertions, but does not have to prove them.

Compilers may infer implementation-specific properties such as I/O usage, unbalanced loops, hints, or external-cell access. Parent systems may also assign special meaning to function names such as `addUint`; neither flags nor special names are normative parts of the language.

## Compact grammar outline

```ebnf
function =
    "def", identifier,
    "(", [ declaration-list ], ")",
    "->", "(", [ declaration-list ], ")",
    "{", entry-section, code-section, exit-section, "}";

declaration-list = declaration, { ",", declaration }, [ "," ];
declaration = type, identifier;
type = "byte", [ "[", integer-literal, "]" ];

entry-section = "entry", "{", { entry-statement }, "}";
entry-statement = identifier, "<=", layout, ";";

code-section = "code", "{", { code-element }, "}";
code-element = brainfuck-command | repeated-command
             | goto-statement | hint-statement;

exit-section = "exit", "{", { exit-statement }, "}";
exit-statement = identifier, "=>", (layout | "preserved"), ";";

layout = "(", layout-item, { ",", layout-item }, [ "," ], ")";
layout-item = reference | integer-literal | typed-literal | "?";

reference = [ ("in" | "out"), "." ], identifier, [ selector ];
selector = "[", integer-literal, "]"
         | "[", [ integer-literal ], ":", [ integer-literal ], "]";

goto-statement = "goto", "(", block-location, ")", ";";
hint-statement = "hint", "(", block-location, ")", ";";
block-location = identifier, [ "[", integer-literal, "]" ];
```
