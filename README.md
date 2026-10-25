# Brainwave

**Draft language version:** 0.1

**Source file extension:** `.brnw`

This package contains the current language design, grammar, semantics, implementation guidance, examples, and conformance tests for a descriptive language for standard Brainfuck algorithms.

## Package contents

- `user_guide.md`  
  Brief, human-readable introduction to the language and its syntax.

- `grammar.ebnf`  
  Full lexical and syntactic EBNF, with semantic constraints and implementation notes.

- `semantic_specification.md`  
  Normative meaning of layouts, references, literals, pointer tracking, exit contracts, assertions, and diagnostics.

- `examples_cookbook.md`  
  Commented examples showing common language features and algorithm patterns.

- `compiler_integration_guide.md`  
  Non-normative guidance for parser construction, IR design, block allocation, lowering, verification, and parent-system integration.

- `examples/`  
  Standalone example source files using the `.brnw` extension.

- `tests/`  
  Conformance tests grouped into valid, invalid, and valid-but-unverified cases. `tests/manifest.json` provides machine-readable expectations.

- `VERSIONING.md`  
  Draft compatibility and versioning policy.

- `CHANGELOG.md`  
  Initial language-design change record.

- `FILE_INVENTORY.txt`  
  Generated list of files included in this package.

## Recommended reading order

1. `user_guide.md`
2. `examples_cookbook.md`
3. `semantic_specification.md`
4. `grammar.ebnf`
5. `compiler_integration_guide.md`
6. `tests/README.md` and `tests/manifest.json`

## Core model

The language targets standard eight-command Brainfuck with modulo-256 byte cells. Function signatures describe logical data flow, while `entry` and `exit` sections describe physical block contracts. The `code` section contains Brainfuck plus compile-time `goto` and `hint` directives.

This is a draft specification. Implementation experience and conformance testing should drive the next revision.
