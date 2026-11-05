# Axiom Tooling v0.1

Status: design specification.

Tooling turns the language and platform specifications into a usable
development workflow.

## 1. CLI commands

The initial CLI should provide:

```text
axiom init
axiom check
axiom run
axiom test
axiom build
axiom inspect
axiom diff
axiom format
axiom package
axiom agent
```

Commands must return structured diagnostics in machine-readable mode and clear
human-readable output by default.

## 2. Compiler pipeline

```text
source / Agent API
  → parse or apply IR operation
  → resolve
  → type check
  → effect check
  → contract and verification checks
  → lower to interpreter, WASM or native target
```

Each stage must preserve node identities and diagnostics where possible.

## 3. Language server

The language server should expose symbol lookup, type information, effect
requirements, contract status, impact analysis and structured code actions. It
should operate on CIR so text edits and agent edits share one semantic model.

## 4. Formatter and source views

Formatting changes must not change CIR identity or semantics. The formatter is
therefore a view operation, not a semantic modification.

## 5. Debugger

The debugger should support node-level breakpoints, call traces, effect
requests, capability decisions, contract checks and deterministic replay when
the runtime has a replay log.

## 6. Agent integration

Agent tooling is an adapter over the Agent API. It may provide planning,
context selection and explanations, but only the protocol can commit program
changes.

## 7. Diagnostics

All tools share the diagnostic schema from the language and Agent API. A tool
may add presentation fields but must preserve stable error codes, node IDs and
related-node references.
