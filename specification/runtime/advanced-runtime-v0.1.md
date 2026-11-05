# Axiom Advanced Runtime v0.1

Status: design specification; platform layer beyond Core v0.1.

This document defines memory, structured concurrency and cancellation without
adding hidden mutable state to the Core language.

## 1. Memory model

Core values are immutable and managed by the runtime. Core v0.1 exposes no raw
pointers, manual deallocation or user-visible object addresses.

The runtime may use tracing GC, reference counting, arenas, stack allocation or
an ownership optimizer. The chosen strategy must preserve:

- value immutability;
- memory safety;
- deterministic pure semantics;
- lifetime validity of opaque handles;
- bounded behavior under configured resource limits.

## 2. Resources and handles

External resources are represented by opaque handles owned by a runtime scope.
A handle cannot be copied into ordinary serialized data unless its capability
policy explicitly allows a portable token.

When a scope ends, all handles owned only by that scope are closed or cancelled
according to their resource contract. Use-after-close produces a structured
failure.

## 3. Shared state

The runtime may contain mutable state, but Core code cannot access it through an
implicit global. Access requires an effect and a capability. Concurrent access
must use an explicit synchronization or message boundary.

## 4. Structured concurrency

Tasks form a tree owned by a parent task:

```text
Task
├── child task
└── child task
```

A parent cannot finish successfully while owned children remain unobserved. A
child failure propagates to the parent according to the task group's policy.
Detached background tasks are not part of the default model.

The runtime operations are:

```text
spawn    create a child task in the current scope
await    observe a child outcome
cancel   request cancellation of a task subtree
join     await all children and collect outcomes
```

Task creation, awaiting and cancellation are platform effects and must be
visible in the task API even though they are not Core v0.1 effects.

## 5. Message passing

Tasks do not share mutable Axiom values directly. Communication uses typed
immutable messages or runtime channels. A channel has a declared element type
and a boundedness policy.

Channel closure and send failure are typed outcomes. A blocked send or receive
must be cancellable.

## 6. Cancellation

Cancellation is cooperative and hierarchical:

1. a parent requests cancellation;
2. the request propagates to descendants;
3. cancellation points observe the request;
4. the task releases owned resources;
5. the task completes with `Cancelled`.

Cancellation points include `await`, blocking I/O, channel operations,
explicit `check_cancelled` and host-defined safe points. Pure CPU computation
must offer an explicit check when it may run for an unbounded duration.

Cancellation is not an exception and must not be silently converted to a normal
success value.

## 7. Cancellation safety

Resource cleanup is guaranteed by runtime scopes. A cancellation handler may
perform only effects allowed by the task's cleanup policy. Cleanup failures are
reported as structured secondary failures attached to the primary cancellation
event.

## 8. Timeouts and deadlines

A timeout is a parent cancellation request with a deadline. The runtime records
the deadline and clock source. Deadline expiration produces `Timeout`, while
explicit cancellation produces `Cancelled`.

## 9. Scheduling

Scheduling order is not part of pure program semantics. Reproducible tests must
use a deterministic scheduler or record scheduling decisions. Data races are
invalid runtime behavior and must be prevented by the task and resource model,
not diagnosed after arbitrary corruption.

## 10. Failure model

Task outcomes are:

```text
Completed(value)
Failed(error)
Cancelled
TimedOut
```

`await` returns a typed task outcome. A parent policy may fail fast, collect all
errors or cancel siblings, but the policy must be explicit.

## 11. Resource limits

The runtime may limit task count, stack or heap memory, channel capacity,
execution time and effect operations. Limit violations must identify the task,
resource and configured policy.

## 12. Core boundary

Memory strategy, concurrency and cancellation remain runtime concerns. They may
be exposed through standard-library APIs and platform effects, but they must
not change the meaning of pure Core expressions.
