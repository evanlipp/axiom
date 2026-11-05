# Axiom Manifesto

## Purpose

Axiom is a programming language whose primary developer may be an AI agent,
while remaining readable and inspectable by people.

The language should reduce the cost of creating correct software by making
intent, dependencies, effects and guarantees explicit.

## Principles

### 1. Intent before implementation

Types, contracts and interfaces describe what a program is allowed to do. An
implementation is accepted only when it satisfies those declarations.

### 2. Explicit dependencies

External resources and side effects are part of a function's interface. A
function must not silently access global state, the network or the filesystem.

### 3. One meaning

The language avoids implicit conversions, hidden control flow and unspecified
evaluation order. A valid program has one defined meaning.

### 4. Structured programs

The canonical program representation is typed and addressable. Text is one
view of the program; agents may work through structural operations instead of
rewriting arbitrary text.

### 5. Evidence over assertion

The system distinguishes properties that are proven, checked, assumed or
unknown. Neither a compiler nor an agent may present an unverified property as
proven.

### 6. Local change

Modules expose explicit boundaries. A change should have an analyzable impact
set, and multi-module changes should be applied transactionally.

### 7. Safe failure

Expected failures are typed values. Diagnostics are structured data containing
location, cause, affected nodes and possible remediation information.

### 8. Small trusted core

The language kernel must stay small enough to specify, implement and test
independently. Advanced features should be added as explicit layers.

## What Axiom does not promise

Axiom cannot make arbitrary programs automatically correct. Static checking,
contracts and effect analysis have limits. The language instead makes the
remaining uncertainty visible and machine-readable.
