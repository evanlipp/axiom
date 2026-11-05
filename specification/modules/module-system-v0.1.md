# Axiom Module System v0.1

Status: design specification; not yet included in the executable Core v0.1
grammar.

Modules are the primary boundaries of ownership, context, verification and
change impact. A module hides its implementation and exposes an explicit
interface to other modules and to agents.

## 1. Module model

A module consists of four distinct parts:

```text
Module {
    identity
    interface
    implementation
    dependencies
}
```

The interface is the contract visible to consumers. The implementation may be
changed without exposing its internal nodes, provided the interface remains
compatible.

## 2. Proposed source form

The following syntax is provisional:

```axiom
module UserService

import Database from "storage/database" version "1.2"

export type User
export function register_user

private type RegistrationState
private function validate_input
```

The exact syntax may change. Its semantic meaning is defined by the module
model below, not by the spelling of these keywords.

## 3. Module identity

A module identity is:

```text
ModuleIdentity {
    package: String
    path: String
    version: Version
}
```

Two modules with different identities are different modules even if their text
is identical. A module version identifies an interface-compatible release, not
an arbitrary source snapshot.

## 4. Visibility

Every declaration has one visibility:

```text
Public       visible through the module interface
Private      visible only inside the module
ReExported   public because it is explicitly exposed from a dependency
```

Declarations are private by default. A consumer cannot refer to a private node
by name or by CIR reference.

An implementation MAY use private declarations to implement public functions,
but private node identities MUST NOT appear in the public interface type,
contract or effect signature.

## 5. Module interface

The canonical interface is:

```text
ModuleInterface {
    identity: ModuleIdentity
    types: List<PublicType>
    functions: List<PublicFunction>
    effects: Set<EffectAtom>
    contracts: List<PublicClaim>
}
```

The interface MUST contain all information needed by a consumer to type-check
its direct use of the module without reading the implementation.

A public function includes:

```text
PublicFunction {
    identity: PublicIdentity
    name: String
    parameters: List<TypeSignature>
    result: TypeSignature
    effects: EffectSet
    preconditions: List<ContractSignature>
    postconditions: List<ContractSignature>
}
```

## 6. Public type closure

Every type reachable from a public function parameter, result, public field,
variant or contract MUST itself be public or be representable entirely in the
interface.

This is the public type closure rule. It prevents a module from exposing an
unnameable private implementation type:

```text
private type DatabaseRow

export function get_user() -> DatabaseRow // rejected
```

An implementation may return a public abstraction whose internal representation
is private.

## 7. Imports and dependency graph

An import binds a local name to a public interface from another module:

```text
Import {
    alias: String
    target: ModuleIdentity
    required_interface: InterfaceConstraint
}
```

The full program dependency graph has modules as nodes and imports as directed
edges. Each edge records the exact public declarations used by the importer.

Unused imports SHOULD be diagnosed. An import that resolves to a private or
missing declaration is an error.

## 8. Cycles

Core module graphs MUST be acyclic. A cycle is rejected with a diagnostic that
lists the complete cycle path.

This restriction keeps interface validation deterministic and lets an agent
load a module's required context without recursively loading an unbounded
graph. A future version MAY add explicitly declared recursive module groups.

## 9. Effects at module boundaries

An imported function's declared effects are part of its public interface. A
caller must declare enough effects for its own body and for all imported calls.

A module MAY declare an allowed effect budget:

```text
module PaymentService
    allows [database.read, database.write]
```

Every public and private function in the module must remain within that budget.
The module budget does not grant concrete capabilities; it is a static upper
bound only.

## 10. Contracts at module boundaries

Public contracts are part of the interface and MUST be available to consumers.
Private contracts may strengthen implementation checks but cannot weaken a
public guarantee.

The following compatibility rules apply:

- a new precondition cannot reject inputs previously accepted by the interface;
- a new postcondition cannot be removed without a breaking change;
- a public invariant cannot be weakened silently;
- changing an `Assumed` claim to `Unknown` is a loss of evidence and is breaking
  where consumers depend on that claim.

## 11. Interface compatibility

An interface change is compatible only if all existing valid consumers remain
valid under the new interface.

Core compatibility rules:

| Change | Compatible? |
|---|---|
| Add a private declaration | Yes |
| Change private implementation | Yes, if interface behavior remains valid |
| Add a new public declaration | Yes |
| Remove a public declaration | No |
| Rename a public declaration | No |
| Change a parameter type | No |
| Change a return type | No |
| Add a required effect | No |
| Remove an effect requirement | Potentially, subject to contract compatibility |
| Strengthen a precondition | No |
| Strengthen a postcondition | Yes, subject to project policy |
| Weaken a postcondition | No |

The postcondition rule is intentionally conservative: exact producer/consumer
variance will be formalized with contracts in a later version.

## 12. Version selection

Dependency resolution MUST select one concrete version for each module identity
within a program build. Two incompatible versions of the same package cannot
silently coexist under one identity.

Version constraints are declarations, not suggestions. If no version satisfies
all constraints, resolution fails before compilation.

Reproducible builds require the resolved identity and integrity information for
every imported module to be recorded in the build artifact.

## 13. Agent context boundary

For a task targeting module `M`, the default agent context is:

```text
Context(M) =
    M's public and private target nodes
  + imported interface signatures
  + directly referenced public contracts
  + effect budget and capability requirements
  + tests attached to affected nodes
```

The agent does not receive unrelated module implementations by default. If a
change crosses an interface, the system must expand the impact set and report
which consumers require revalidation.

## 14. Module validation phases

A module is validated in this order:

1. resolve its identity and imports;
2. validate the dependency graph and reject cycles;
3. type-check declarations and implementation;
4. check public type closure;
5. check effect budgets;
6. check public contracts;
7. generate the public interface;
8. compare the interface with the requested compatibility policy.

Failure in any phase prevents publication of the module artifact.

## 15. Module artifact

A published module artifact contains:

```text
ModuleArtifact {
    identity: ModuleIdentity
    interface: ModuleInterface
    implementation_hash: Hash
    dependency_lock: List<ResolvedDependency>
    verification_summary: VerificationSummary
}
```

Consumers need the interface and dependency lock. They do not need the source
implementation to type-check direct calls.

## 16. Diagnostics

The module system must distinguish at least:

```text
ModuleNotFound
VersionConflict
ImportNotPublic
ImportNotFound
DependencyCycle
PublicTypeClosureViolation
PublicContractViolation
ModuleEffectBudgetExceeded
BreakingInterfaceChange
```

Diagnostics SHOULD include the dependency path and the smallest interface
element that caused the failure.

## 17. Open decisions

The following remain unresolved:

- exact package and version syntax;
- whether interfaces are committed artifacts or generated views;
- whether public types are nominal, structural or both;
- contract variance for interface evolution;
- recursive module groups;
- package registry and integrity format;
- re-export semantics;
- interface-level capability declarations.
