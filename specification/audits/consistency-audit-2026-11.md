# Axiom Specification Consistency Audit

Date: 2026-11-05

Scope: all documents under `specification/`.

## Result

The architecture is coherent after the corrections recorded below. The
documents still describe a design draft, not a frozen language standard.

## Corrections applied

### 1. Core grammar was incomplete

The grammar referenced `parameter_list`, `argument_list`, `type_expr_list`,
`identifier_list` and `binary_expr` without defining them. These productions
are now defined in `core-v0.1.md`.

### 2. Sum-type syntax was ambiguous

An alias and a sum type could both begin with a bare identifier. Core sum types
now use braces, for example:

```axiom
type PaymentError = { InsufficientFunds | AccountBlocked }
```

### 3. The IR could not represent record construction

The type system and semantics supported records, but CIR only had a variant
constructor. `RecordConstruct` has been added to the IR expression operations.

### 4. Core incorrectly implied all functions were pure

The scope now says functions have explicit effect sets. Functions without an
effect declaration remain pure.

### 5. Totality was too strong

The Core specification now qualifies totality by preconditions and available
capabilities. A function can still fail at a contract or host boundary.

### 6. An effect example violated the type system

The clock example attempted `String + Integer`, while Core defines no string
concatenation operator. It now returns the integer and defers formatting to the
standard library.

### 7. Runtime determinism needed a host qualification

Different capability policies can legitimately produce different boundary
failures. Runtime equivalence is now defined for identical inputs and effect
handler traces.

## Cross-layer invariants

The following relationships are now the authoritative design rules:

```text
Source syntax → CIR
CIR validity → type checking + effect checking
Function call → transitive effect requirement
Contract → pure Boolean predicate
External operation → declared effect + runtime capability
Agent mutation → candidate CIR → validation → atomic commit
Test result → Checked evidence, not automatically Proven
```

## Remaining issues, not yet contradictions

These are open design questions rather than known conflicts:

1. The JSON Schema covers the CIR v0.1 interchange model, but cross-node
   reference existence is still checked semantically rather than by JSON Schema.
2. Property-test generator semantics and shrinking remain platform-defined.
3. The Agent API has a JSON envelope, while complete payload schemas remain a
   follow-up tooling task.
4. Runtime memory, concurrency and cancellation are specified as a platform
   layer and remain outside Core v0.1 language semantics.

## Boundary decisions

The following are intentional and should not be treated as conflicts:

- module imports/exports are designed but excluded from executable Core v0.1;
- Agent API is platform infrastructure, not language semantics;
- standard library effects are outside the pure Core kernel;
- FFI declarations are trusted boundary contracts, not ordinary pure functions;
- `Proven`, `Checked`, `Assumed` and `Unknown` are evidence states, not types.

## Recommendation

The architecture is ready for implementation of the Core checker and runtime
prototype. Cross-node reference validation, refinement evidence propagation and
the advanced runtime policies must be covered by the first implementation tests.
