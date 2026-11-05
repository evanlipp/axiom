# Axiom

AI-native general-purpose programming language.

Axiom is designed around a simple idea: a program is a structured, typed and
verifiable model of computation. Text is a human-readable view of that model;
agents can also inspect and modify it through a structural API.

## Status

This repository contains the initial language documents. The current target is
`Axiom Core v0.1`: a deliberately small kernel with types, functions, results,
contracts, effects and deterministic evaluation.

The specification is normative. The manifesto describes design intent and is
not itself a compiler contract.

## Documents

- [Manifesto](specification/manifesto.md) — goals and design principles.
- [Core concepts](specification/concepts.md) — the internal model of programs,
  values, effects, contracts and agent changes.
- [Canonical IR](specification/ir/canonical-ir-v0.1.md) — the structured
  semantic representation of an Axiom program.
- [CIR JSON Schema](specification/ir/axiom-cir-v0.1.schema.json) — machine-
  readable validation schema for the IR interchange format.
- [Core semantics](specification/semantics/core-semantics-v0.1.md) — how valid
  IR evaluates and how contracts, effects and failures behave.
- [Core type system](specification/type-system/core-types-v0.1.md) — typing
  rules, refined types and constructor compatibility.
- [Refined construction](specification/type-system/refinement-construction-v0.1.md)
  — safe conversion of external values into refined types.
- [Core effects](specification/effects/core-effects-v0.1.md) — effect
  propagation, capability boundaries and effect diagnostics.
- [Module system](specification/modules/module-system-v0.1.md) — module
  interfaces, imports, visibility, dependencies and compatibility.
- [Agent API](specification/agent-api/agent-protocol-v0.1.md) — structural
  inspection, changes, validation, diffs, commits and rollback.
- [Verification model](specification/verification/verification-model-v0.1.md) —
  tests, claims, evidence statuses, impact-based checking and commit policy.
- [Test syntax](specification/verification/test-syntax-v0.1.md) — initial
  platform-level test and property-test notation.
- [Conformance fixtures](specification/conformance/core-v0.1.md) — shared
  valid and invalid Core examples with expected results.
- [Runtime model](specification/runtime/runtime-model-v0.1.md) — execution,
  effects, replay and resource limits.
- [Advanced runtime](specification/runtime/advanced-runtime-v0.1.md) — memory,
  structured concurrency, tasks and cancellation.
- [Security model](specification/security/security-model-v0.1.md) — capabilities,
  isolation, secrets and trust boundaries.
- [Standard library](specification/stdlib/standard-library-v0.1.md) — package
  responsibilities and platform APIs.
- [Package system](specification/packages/package-system-v0.1.md) — versions,
  dependencies, integrity and reproducible builds.
- [FFI](specification/ffi/foreign-function-interface-v0.1.md) — safe boundaries
  to C, WASM, JavaScript and host runtimes.
- [Tooling](specification/tooling/tooling-v0.1.md) — CLI, compiler pipeline,
  language server, debugger and agent integration.
- [Consistency audit](specification/audits/consistency-audit-2026-11.md) —
  cross-layer contradictions, corrections and remaining open decisions.
- [Core v0.1 specification](specification/core-v0.1.md) — normative language rules.

## Research

- [AI-native language research](research/ai-native-languages-2026.md) — review
  of Edict, Promise, Astra, Mog, NTNT, LMQL, DSPy, Guidance and PydanticAI.

## Development order

1. Keep the core concepts and specification precise and internally consistent.
2. Define structural Agent API operations over the canonical IR.
3. Implement a parser and checker against the specification.
4. Implement a deterministic reference interpreter.

## Non-goals of Core v0.1

Native code generation, concurrency, ownership, databases, networking,
packages and full formal theorem proving are intentionally outside the first
kernel. They can be layered on top once the core semantics are executable.
