# Versioning and Compatibility

## Current status

The language is currently a **draft** with version **0.1**.

Until version 1.0, syntax and semantics may change incompatibly as implementation experience exposes design issues.

## Recommended version format

Use semantic-style versions:

```text
MAJOR.MINOR.PATCH
```

- **MAJOR**: incompatible language or contract change.
- **MINOR**: backward-compatible syntax, literal, or semantic addition.
- **PATCH**: clarification or correction that does not intentionally change valid-program behavior.

## Source version declaration

Version syntax inside source files is not yet defined. The parent system should associate a language version with each source unit through configuration, package metadata, or an external file.

A future source-level declaration should be introduced only after its syntax and compatibility behavior are specified.

## Compatibility principles

- A compiler should report the language versions it accepts.
- A compiler should reject a source unit requiring an unsupported version.
- Extensions should not silently reinterpret existing valid programs.
- Implementation-defined flags and special function names are outside language-version compatibility unless a parent-system profile makes them normative.

## Draft change policy

During the 0.x series:

1. Record every accepted design change in `CHANGELOG.md`.
2. Mark unresolved behavior explicitly rather than relying on implementation accident.
3. Add or update conformance tests whenever semantics change.
4. Update the user guide, grammar, and semantic specification together.
