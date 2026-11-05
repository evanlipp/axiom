# Axiom Core Concepts

This document defines the conceptual model behind Axiom. It is more abstract
than the grammar and is intended to guide the canonical IR, compiler and Agent
API.

## 1. Program as a typed graph

Axiom does not fundamentally treat a program as a sequence of text files. A
program is a typed directed graph:

```text
Program
├── Modules
│   ├── Declarations
│   │   ├── Types
│   │   └── Functions
│   └── Contracts
└── Dependency edges
```

Nodes have stable identities. Edges describe relationships such as:

- a function uses a type;
- a function calls another function;
- a contract refers to an input or result;
- a function requires an effect;
- a module exposes a declaration.

The graph is the canonical object that tools inspect, validate and modify.
Textual source is a serialized human-oriented view of the graph.

## 2. Five layers of a program

Every Axiom program is understood through five layers:

### Data

The values and types that can exist: records, variants, primitive values and
restricted domains.

### Logic

Pure transformations of values. Logic should be deterministic and free of
external effects whenever possible.

### Effects

Interactions with state or the outside world. Effects are declared capabilities
of a function, not comments about its implementation.

### Contracts

Conditions that constrain valid inputs, outputs and persistent invariants.
Contracts can be checked at runtime and may later be proven statically.

### Composition

The way modules and functions are assembled into a larger program. Composition
must preserve type, effect and contract information.

## 3. Values and computations

A value is a completed piece of data. A computation is a description of work
that may produce a value and may require effects.

The distinction is important:

```text
Integer                  value
Integer -> Integer       pure computation
Integer -[clock.read]-> Integer  effectful computation
```

An effect is not itself an error. An effect is a declared permission and
dependency. Failure remains an explicit result value or a structured runtime
diagnostic.

## 4. Function identity

A function is identified by its stable node identity, not only by its name.
Its semantic interface contains:

```text
FunctionInterface {
    identity
    name
    parameters
    return_type
    required_effects
    preconditions
    postconditions
}
```

The function body is separate from the interface. An implementation may change
while the interface remains compatible.

## 5. Type meaning

A type is both a set of possible values and a set of operations allowed on
those values. A type declaration may therefore narrow a domain:

```text
type AdultAge = Integer where value >= 18 and value <= 150
```

Constructing a value of a restricted type requires evidence that its predicate
holds. Evidence may be:

- `Proven` by static reasoning;
- `Checked` by an executed validation;
- `Assumed` from a trusted boundary.

The type system must preserve the evidence status. An agent cannot silently
turn an `Unknown` predicate into a valid value.

## 6. Contracts and invariants

Contracts constrain behavior at boundaries:

- preconditions constrain callers and inputs;
- postconditions constrain successful results;
- invariants constrain values or module state across operations.

Contracts do not automatically make behavior correct. They are claims attached
to graph nodes and must carry their verification status.

```text
Claim {
    expression
    status: Proven | Checked | Assumed | Unknown
    evidence: EvidenceRef?
}
```

## 7. Effects and capabilities

An effect describes what kind of external interaction a computation may need.
A capability describes which concrete resource it may access.

For example:

```text
effect: filesystem.read
capability: filesystem.read("/input")
```

An effect declaration alone does not grant access to every resource. Runtime
capabilities bind abstract effects to permitted concrete resources.

This separation allows the same function interface to run in different
environments without making resource access implicit.

## 8. Boundaries

A boundary is a place where Axiom stops trusting an assumption. Examples are:

- user input;
- data read from a file or database;
- a foreign function;
- a network response;
- an agent modification.

Every boundary must either validate incoming data or mark it as an explicit
assumption. Unvalidated external data must not automatically inhabit a refined
or trusted type.

## 9. Change as a transaction

An agent change is not a text patch. It is a transaction over the program graph:

```text
ChangeRequest
    → resolve targets
    → propose graph operations
    → calculate impact set
    → validate candidate graph
    → commit or reject atomically
```

A rejected change leaves the previous program unchanged. A committed change
records the changed nodes, affected interfaces, diagnostics and evidence
produced by validation.

## 10. Context for an agent

An agent should receive the smallest sufficient context for a requested change.
The default context consists of:

- target nodes;
- their public interfaces;
- directly referenced types and functions;
- required effects and capabilities;
- related contracts and tests;
- the calculated impact set.

The agent may request more context, but the system should make expansion
explicit and auditable.

## 11. Core design consequences

These concepts imply the following rules:

1. Names and text locations are not stable program identity.
2. Interfaces must be separable from implementations.
3. Effects must be visible in the graph and transitively checkable.
4. External data must cross a typed boundary.
5. Verification status is part of the program model.
6. Agent modifications must be atomic and produce an impact report.
7. The text syntax must be derived from these semantics, not define them by accident.

## 12. Deliberately unresolved questions

The following decisions require a later design phase:

- whether refined types are first-class types or validated wrappers;
- how recursive types and recursive functions are represented;
- whether state is modeled as an effect parameter, an owned resource or both;
- how invariants are attached to long-lived resources;
- how proofs and runtime evidence are serialized;
- how compatible interface evolution is determined.
