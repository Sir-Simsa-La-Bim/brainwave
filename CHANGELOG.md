# Changelog

All notable language and documentation changes should be recorded here.

## 0.1 - Draft

Initial documented design, including:

- Language name **Brainwave** and source extension `.brnw`.

- Standard eight-command Brainfuck with modulo-256 byte cells.
- C-style function declarations with logical `byte` and `byte[N]` values.
- Mandatory `entry`, `code`, and `exit` sections.
- Named contiguous physical blocks with entry and exit contracts.
- Python-style indexing and slicing without slice steps.
- Unknown cells, preserved blocks, and original-input exit assertions.
- Little-endian integer, floating-point, ASCII, and UTF-8 literal expansion.
- `goto` and `hint` pointer directives.
- Decimal repetition suffixes for `+`, `-`, `<`, and `>`.
- C-style and Python-style comments.
- Optional compiler metadata and special function-name integration.
