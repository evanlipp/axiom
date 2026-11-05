# Axiom Standard Library v0.1

Status: design specification.

The standard library provides typed, portable foundations. It must not hide
effects or turn platform-specific behavior into apparently pure functions.

## 1. Core packages

```text
axiom.core          primitive helpers, Option, Result, diagnostics
axiom.collections   List, Map, Set and immutable operations
axiom.text          Unicode strings and text processing
axiom.math          numeric and mathematical operations
axiom.time          timestamps, durations and calendar operations
axiom.serialization typed encoding and decoding
axiom.testing       test and property-test interfaces
```

## 2. Effectful packages

```text
axiom.io            streams and console I/O
axiom.filesystem    files and directories
axiom.network       network clients and protocols
axiom.process       process interaction
axiom.crypto        cryptographic primitives and secure randomness
axiom.database      database interfaces
axiom.browser       DOM and browser capability bindings
```

Every effectful package exposes effect declarations and capability requirements
in its public interface.

## 3. Library rules

Standard-library functions must use typed errors, document determinism and
avoid hidden global state. Security-sensitive primitives must delegate to
trusted platform implementations rather than reimplementing cryptography in
ordinary Axiom code.

## 4. Portability

Each package declares supported targets and unavailable operations. A package
must not compile successfully for a target when an advertised operation has no
valid implementation; it must report an explicit target diagnostic.

## 5. Browser support

`axiom.browser` exposes browser APIs through capabilities. A web backend may
compile logic to WASM and generate a JavaScript bridge for DOM and Web APIs.
