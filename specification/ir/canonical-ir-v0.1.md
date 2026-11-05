# Axiom Canonical IR v0.1

Status: draft normative specification.

Canonical IR (CIR) is the semantic representation of an Axiom program. The
compiler, verifier and Agent API MUST operate on CIR or on a representation
that is semantically equivalent to it. Text source is an input and output view
of CIR, not a second source of truth.

## 1. Design goals

CIR MUST be:

- typed;
- explicitly linked;
- addressable by stable node identity;
- independent of formatting and source file layout;
- serializable without losing semantic information;
- suitable for structural diffs and transactional changes.

CIR MUST NOT encode meaning through field order, whitespace, generated names or
source positions.

## 2. Identity

Every semantic node has a `NodeId`:

```text
NodeId = { namespace: String, value: String }
```

`value` is opaque to consumers. An implementation MAY use hashes or UUIDs, but
it MUST keep an existing `NodeId` stable when a node is moved or reformatted.
A semantically replaced node receives a new identity.

References use `NodeRef<T>` and MUST point to a node of the expected category.
Dangling or category-incompatible references are invalid CIR.

The interchange form of a reference is:

```json
{
  "category": "FunctionDecl",
  "namespace": "app",
  "value": "fn.divide"
}
```

`category` is mandatory in serialized CIR. A reference with a missing or
incorrect category is invalid even when its namespace and value identify an
existing node.

## 3. Source mapping

Source mapping is optional metadata and has no semantic meaning:

```text
SourceSpan {
    document: String
    start: { line: Integer, column: Integer, offset: Integer }
    end:   { line: Integer, column: Integer, offset: Integer }
}
```

A node MAY have multiple source spans because a single semantic node can be
rendered across multiple locations. A renderer MUST NOT require source spans to
serialize valid CIR.

## 4. Top-level schema

The conceptual schema is:

```text
Program {
    version: String
    modules: List<NodeRef<Module>>
    nodes: NodeStore
    metadata: Metadata
}

NodeStore {
    modules: Map<NodeId, Module>
    types: Map<NodeId, TypeDecl>
    functions: Map<NodeId, FunctionDecl>
    bindings: Map<NodeId, Binding>
    expressions: Map<NodeId, Expr>
    contracts: Map<NodeId, Contract>
}
```

The maps are conceptual. A binary implementation MAY use another storage
format, provided node lookup and serialization preserve the same meaning.

## 5. Modules

```text
Module {
    id: NodeId
    name: String
    declarations: List<NodeRef<Declaration>>
    source: List<SourceSpan>
}
```

Core v0.1 modules have no imports or exports. A module declaration list MUST
contain each declaration at most once. Declaration order does not affect
meaning, although it MAY be preserved for rendering.

## 6. Type declarations

```text
TypeDecl {
    id: NodeId
    name: String
    definition: TypeDefinition
    source: List<SourceSpan>
}

TypeDefinition =
      PrimitiveType { name: Integer | Boolean | String | Unit }
    | AliasType { target: TypeRef }
    | RecordType { fields: List<Field> }
    | SumType { variants: List<Variant> }
    | RefinedType { base: TypeRef, predicate: NodeRef<Expr> }

Field { name: String, type: TypeRef }

Variant {
    name: String
    fields: List<TypeRef>
}
```

Field and variant names are unique within their containing declaration. Record
field order is semantically irrelevant. Sum variant order is semantically
irrelevant but SHOULD be preserved for stable rendering.

## 7. Type references

```text
TypeRef =
      BuiltinType { name }
    | NamedType { declaration: NodeRef<TypeDecl> }
    | OptionType { value: TypeRef }
    | ResultType { value: TypeRef, error: TypeRef }
```

All type references MUST be resolved before a program is accepted. An unresolved
name is a compiler error, not an `Unknown` type.

## 8. Functions

```text
FunctionDecl {
    id: NodeId
    name: String
    parameters: List<Parameter>
    return_type: TypeRef
    effects: Set<EffectRef>
    contracts: List<NodeRef<Contract>>
    body: NodeRef<Expr>
    source: List<SourceSpan>
}

Parameter {
    name: String
    type: TypeRef
}

EffectRef { name: String }
```

Parameter names are unique within a function. A function with no effects MUST
have an empty effect set; absence and an explicit empty set have the same
meaning in CIR.

## 9. Expressions

Every expression has an inferred or declared type after type checking:

```text
Expr {
    id: NodeId
    type: TypeRef?
    operation: Operation
    source: List<SourceSpan>
}

Operation =
      IntegerLiteral { value: String }
    | BooleanLiteral { value: Boolean }
    | StringLiteral { value: String }
    | VariableRef { declaration: NodeRef<Parameter | Binding> }
    | Let { binding: Binding, value: NodeRef<Expr>, body: NodeRef<Expr> }
    | If { condition: NodeRef<Expr>, then_branch: NodeRef<Expr>, else_branch: NodeRef<Expr> }
    | Call { function: NodeRef<FunctionDecl>, arguments: List<NodeRef<Expr>> }
    | Construct { type: TypeRef, variant: String, arguments: List<NodeRef<Expr>> }
    | RecordConstruct { type: TypeRef, fields: Map<String, NodeRef<Expr>> }
    | Refine { target: RefinedTypeRef, value: NodeRef<Expr> }
    | FieldRead { value: NodeRef<Expr>, field: String }
    | Binary { operator: BinaryOperator, left: NodeRef<Expr>, right: NodeRef<Expr> }
    | Return { value: NodeRef<Expr> }

Binding {
    id: NodeId
    name: String
    type: TypeRef?
}

BinaryOperator = Add | Subtract | Multiply | Divide | Remainder
                | Less | LessOrEqual | Equal | NotEqual
                | Greater | GreaterOrEqual | And | Or
```

`type` is null only before type checking. Accepted CIR MUST contain resolved
types for every expression. A `VariableRef` MUST refer to a parameter or a
binding visible at that expression.

## 10. Contracts and evidence

```text
Contract {
    id: NodeId
    kind: Requires | Ensures | Invariant
    predicate: NodeRef<Expr>
    status: EvidenceStatus
    evidence: EvidenceRef?
}

EvidenceStatus = Proven | Checked | Assumed | Unknown
EvidenceRef {
    kind: String
    identity: String
    details: String?
}
```

The predicate of a `Requires` contract is evaluated in the function input
environment. The predicate of an `Ensures` contract may additionally refer to
`result`. An `Invariant` is reserved in the schema but is not attachable to a
mutable resource in Core v0.1.

## 11. Validity invariants

A CIR instance is accepted only if:

1. every node identity is unique;
2. every reference resolves to an existing node of the expected category;
3. every type reference is resolved;
4. every expression has a valid type;
5. every call's argument types match its target parameters;
6. the caller's effect set contains the callee's effect set;
7. every contract predicate has type `Boolean`;
8. every function body is compatible with its return type;
9. no variable reference escapes its lexical scope;
10. no unsupported operation is present in Core v0.1.

Validation failure MUST produce structured diagnostics and MUST NOT produce an
executable program artifact.

## 12. Canonical serialization

JSON is the initial interchange format. The following is illustrative; exact
field ordering and whitespace are not significant:

```json
{
  "version": "axiom-cir-0.1",
  "modules": [{"namespace": "app", "value": "module.math"}],
  "nodes": {
    "modules": {
      "app:module.math": {
        "id": {"namespace": "app", "value": "module.math"},
        "name": "Math",
        "declarations": [
          {"namespace": "app", "value": "fn.divide"}
        ]
      }
    }
  }
}
```

Implementations MUST preserve unknown metadata fields when performing a
read-modify-write cycle, unless the metadata namespace is explicitly owned by
the implementation.

## 13. Structural operations

The initial Agent API operates on CIR using these abstract operations:

```text
Inspect(node: NodeId, projection: Projection) -> Snapshot
Create(parent: NodeId, node: NodeDraft) -> NodeId
Update(node: NodeId, patch: StructuralPatch) -> Candidate
Delete(node: NodeId) -> Candidate
Validate(candidate: Candidate) -> ValidationReport
Commit(candidate: Candidate) -> ChangeSet
Rollback(change: ChangeSet) -> ChangeSet
```

`Update` and `Delete` never mutate the committed program directly. A candidate
program is validated before `Commit`. Commit MUST be atomic.

## 14. Change reports

```text
ChangeSet {
    identity: String
    parent_program: String
    changed_nodes: List<NodeId>
    affected_nodes: List<NodeId>
    diagnostics: List<Diagnostic>
    evidence: List<EvidenceRef>
}
```

`affected_nodes` is the transitive impact set calculated by dependency edges.
The implementation MUST distinguish directly changed nodes from nodes whose
analysis result may have changed.

## 15. Compatibility

Two CIR interfaces are compatible in Core v0.1 only if:

- existing parameter names and types remain unchanged;
- the return type remains unchanged;
- no required effect is removed from the caller's known environment;
- no precondition is strengthened;
- no postcondition is weakened.

Strengthening a postcondition is compatible because every result satisfying the
new interface also satisfies the old guarantee. A consumer may still reject a
strengthened postcondition when its project policy treats proof or runtime
checking cost as part of compatibility.

Full module versioning and import compatibility are deferred until the module
system is specified.

## 16. Open IR decisions

The following remain open:

- exact identity generation and persistence across branches;
- whether node maps are exposed in public JSON;
- a standard patch format;
- representation of generic and recursive types;
- how effect parameters and concrete capabilities are linked;
- proof artifact storage;
- schema evolution between CIR versions.
