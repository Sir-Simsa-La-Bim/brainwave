# Conformance Tests

The suite is grouped by expected diagnostic class:

- `valid/`: must be accepted.
- `invalid-lexical/`: must be rejected during lexical analysis.
- `invalid-syntax/`: must be rejected by the grammar.
- `invalid-semantic/`: parses successfully but violates a semantic rule.
- `valid-unverified/`: must be accepted, although correctness depends on trusted author assertions.

`manifest.json` is the machine-readable index. Test runners should resolve every `path` relative to this directory.

A compiler may emit additional warnings for accepted tests. It must not reject `valid-unverified` tests merely because it does not prove their assertions. It may reject one only if it actually proves an assertion false.
