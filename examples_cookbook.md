# Examples Cookbook

These examples illustrate language features and common implementation patterns. They are examples of contracts and syntax, not claims that each algorithm is optimal.

## 1. Identity

```text
def identity(byte value) -> (byte result)
{
    entry { work <= (value); }
    code { goto(work); }
    exit { work => (result); }
}
```

The compiler binds the input to one physical cell. No Brainfuck operation is needed; the cell becomes the result.

## 2. Copy one byte

```text
def copyByte(byte value) -> (byte original, byte copy)
{
    entry { work <= (value, 0, 0); }

    code {
        goto(work);
        [->+>+<2]
        >2[<2+>2-]
    }

    exit { work => (original, copy, 0); }
}
```

The first loop distributes the input to two cells while clearing the source. The second loop restores one copy into the source cell. Pointer-balanced loops keep location tracking intact.

## 3. Add two bytes modulo 256

```text
def addByte(byte a, byte b) -> (byte sum)
{
    entry { work <= (a, b, 0); }

    code {
        goto(work);
        [->2+<2]
        >[->+<]
    }

    exit { work => (0, 0, sum); }
}
```

Both inputs are consumed into the result cell. Byte wrapping makes the result modulo 256.

## 4. Preserve an input while producing an output

```text
def incrementPreserving(byte value) -> (byte result)
{
    entry {
        work   <= (value, 0, 0);
        backup <= (value);
    }

    code {
        goto(work);
        [->+>+<2]
        >2[<2+>2-]
        goto(work[1]);
        +
    }

    exit {
        work   => (in.value, result, 0);
        backup => preserved;
    }
}
```

This demonstrates input replication at entry and an original-input assertion at exit. `backup => preserved;` asserts exact block identity.

## 5. Literal expansion

```text
def literalLayout() -> ()
{
    entry {
        data <= (
            257,
            u24(0x123456),
            i16(-2),
            float16(nan),
            float32(1.5),
            float64(-inf),
            ascii('OK'),
            utf8("Gr\u00FCezi"),
            0,
        );
    }

    code { goto(data); }
    exit { data => preserved; }
}
```

Expansion is little-endian. Strings have no implicit terminator, so the final zero is explicit.

## 6. Brainfuck input and output

```text
def echoOne() -> ()
{
    entry { io <= (0); }

    code {
        goto(io);
        ,.
    }

    exit { io => (?); }
}
```

`,` and `.` are ordinary Brainfuck commands. A parent system may classify the function as using input and output.

## 7. Balanced nested loops

```text
def nested(byte count) -> ()
{
    entry { work <= (count, 0, 0); }

    code {
        goto(work);
        [
            -
            >
            [->+<]
            <
        ]
    }

    exit { work => (?, ?, ?); }
}
```

Each loop body returns to its own starting pointer position, so no hint is needed.

## 8. Unbalanced scan with a hint

```text
def scanToZero(byte first) -> ()
{
    entry { work <= (first, ?, ?, 0); }

    code {
        goto(work);
        [>]
        hint(work[-1]);
    }

    exit { work => (?, ?, ?, 0); }
}
```

`[>]` has an unknown iteration count and moves the pointer. The author asserts that the first zero encountered is the final block cell. The compiler may trust this claim without proving it.

## 9. Python-style slices and replication

```text
def arrange(byte[5] input) -> (byte[3] middle)
{
    entry {
        work <= (
            input[1:4],
            input[-1],
            input[0],
            input[0],
        );
    }

    code { goto(work); }

    exit {
        work => (
            middle[:],
            in.input[-1],
            in.input[0],
            in.input[0],
        );
    }
}
```

The slice expands to three cells. The final input byte and first input byte are preserved in additional physical copies.

## 10. Unknown scratch cells

```text
def consumeScratch(byte value) -> (byte result)
{
    entry { work <= (value, ?, 0); }

    code {
        goto(work[1]);
        [-]
        goto(work);
        [->+<]
    }

    exit { work => (0, result, 0); }
}
```

The unknown cell is first cleared with `[-]`, after which it can safely receive the result.

## 11. Access outside a block

```text
def externalScratch(byte value) -> (byte result)
{
    entry { work <= (value, 0); }

    code {
        goto(work);
        <
        [-]
        >
        [<+>-]
        <[->+<]
        >
    }

    exit { work => (result, 0); }
}
```

The code temporarily uses one unnamed cell to the left of `work`, then returns the value to the declared block. The unnamed cell cannot appear in `exit`, and the code cannot assume it belongs to any other block. A compiler may flag the external access.

## 12. Readable qualification

```text
def qualify(byte[2] source) -> (byte[2] destination)
{
    entry { work <= (in.source[:]); }
    code { goto(work); }
    exit { work => (out.destination[:]); }
}
```

`in.` and `out.` are optional, but useful when a contract contains many references.

## 13. Formatting and comments

```text
def formatted(
    byte a,
    byte b,
) -> (
    byte sum,
)
{
    entry {
        work <= (
            a,
            b,
            0,
        );
    }

    code {
        goto(work); // Establish pointer location.
        [->2+<2]
        >[->+<]     # Add b into the result.
        /* Brainfuck symbols in comments are ignored: [>+<-] */
    }

    exit {
        work => (
            0,
            0,
            sum,
        );
    }
}
```

Whitespace is insignificant between tokens. Trailing commas are allowed in declarations and layouts.
