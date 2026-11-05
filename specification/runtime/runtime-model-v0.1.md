# Axiom Runtime Model v0.1

Status: design specification.

The runtime executes validated Axiom programs and mediates every interaction
with the host environment.

## 1. Runtime layers

```text
Program
  → execution engine
  → effect dispatcher
  → capability policy
  → host environment
```

The runtime MUST NOT infer permissions from an attempted operation. Every
external operation must pass through the effect dispatcher.

## 2. Execution modes

Core runtimes may provide:

- interpreter mode for reference behavior and debugging;
- bytecode mode for portable execution;
- native or WASM mode for production targets.

All modes MUST implement the same observable Core semantics for the same inputs
and the same effect-handler trace. Host policy and unavailable capabilities may
still produce different structured boundary failures.

## 3. Values and memory

Core values are immutable and may be copied, shared or represented indirectly
by the runtime. The memory strategy is an implementation detail as long as it
preserves value semantics and does not expose invalid references to Axiom code.

Mutable runtime resources are represented by opaque handles and are accessed
only through declared effects and capabilities.

## 4. Calls and failure

The runtime maintains a call frame for each active function. A runtime failure
contains the active function, causing node and structured failure code when
available. Unhandled failure terminates the current execution boundary; it does
not become an undeclared exception inside the language.

## 5. Effect dispatch

An effect request is sent to a host handler. The handler may return a typed
value, a typed failure or a cancellation result. An absent or unauthorized
handler produces a structured runtime failure.

## 6. Determinism and replay

Pure execution must be deterministic. Effectful execution is reproducible only
when effect requests and responses are recorded or supplied by a deterministic
host. The runtime SHOULD support replay logs for debugging and tests.

## 7. Resource limits

The runtime SHOULD enforce configurable limits for execution time, memory,
call depth, output size and effect requests. A limit violation is reported as a
structured failure, never as silent truncation.

## 8. Targets

The initial targets are interpreter and WebAssembly. Native code generation is
an optimization layer and must not define different language behavior.
