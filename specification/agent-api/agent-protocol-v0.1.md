# Axiom Agent API v0.1

Status: design specification.

The Agent API is a model-independent protocol for inspecting and changing an
Axiom program. It is not an AI prompt format and does not assume a particular
model, vendor or user interface.

The API operates on Canonical IR snapshots and never treats an agent's textual
claim of success as evidence that a change succeeded.

## 1. Protocol properties

Every operation MUST be:

- structured;
- addressable by stable node identity;
- explicit about its target snapshot;
- deterministic where the underlying compiler operation is deterministic;
- auditable;
- safe to reject without changing the committed program.

Mutating operations are staged in a transaction. Only a successful `commit`
changes the active program snapshot.

## 2. Program snapshots

An agent always works against a named immutable snapshot:

```text
Snapshot {
    id: SnapshotId
    parent: SnapshotId?
    program_hash: Hash
    cir_version: String
    created_at: Timestamp
}
```

The timestamp is metadata and does not affect program semantics. A request that
targets an obsolete snapshot MUST be rejected instead of silently applying a
change to a newer program.

## 3. Request envelope

Every request uses the following envelope:

```text
Request {
    request_id: String
    operation: Operation
    target_snapshot: SnapshotId
    context: ContextSelector?
    options: RequestOptions?
}
```

Every response contains:

```text
Response {
    request_id: String
    status: Accepted | Rejected | NeedsContext | Conflict
    diagnostics: List<Diagnostic>
    result: OperationResult?
}
```

The initial transport is JSON over a request/response channel. The wire
envelope is:

```json
{
  "protocol": "axiom-agent-0.1",
  "request_id": "req-001",
  "operation": "inspect",
  "target_snapshot": "snap-abc",
  "payload": {
    "node": {"category": "FunctionDecl", "namespace": "app", "value": "fn.main"},
    "projection": "Interface"
  }
}
```

Responses use the same `request_id` and contain `status`, `diagnostics` and an
operation-specific `result`. Unknown fields must be preserved by gateways and
ignored by older clients. A transport adapter may use stdio, HTTP or another
channel, but it must preserve request identity, ordering within a transaction
and atomic commit semantics.

`request_id` is used for tracing and idempotency. Repeating an operation with
the same request identity MUST NOT create duplicate commits.

## 4. Context selection

The agent may request a bounded context:

```text
ContextSelector {
    roots: List<NodeId>
    include: Interface | DirectDependencies | Contracts | Tests | ImpactSet
    depth: Integer?
}
```

The server MAY return `NeedsContext` when a request cannot be safely resolved
from the supplied context. It must identify the missing node or interface
rather than returning a generic failure.

## 5. Inspect

`inspect` reads program structure and never mutates it:

```text
InspectRequest {
    node: NodeId
    projection: Full | Interface | Types | Dependencies | Contracts | Effects
}
```

The result is:

```text
Inspection {
    snapshot: SnapshotId
    node: NodeId
    projection: Projection
    content: StructuredValue
    referenced_nodes: List<NodeId>
}
```

An inspection MUST preserve node identities and verification statuses.

## 6. Query

Agents need targeted search without reading a full project:

```text
Query {
    snapshot: SnapshotId
    kind: Module | Type | Function | Contract | Effect
    name: String?
    references: NodeId?
    effect: EffectAtom?
}
```

Query results contain stable identities and interface summaries. Querying does
not grant permission to modify a node.

## 7. Create

`create` proposes a new node under a specified parent:

```text
CreateRequest {
    transaction: TransactionId
    parent: NodeId
    draft: NodeDraft
}
```

The draft MUST include enough information to determine its category and local
name. Missing type, effect or contract information is not silently guessed;
the server returns a diagnostic or requests context.

The result contains a provisional `NodeId` and the candidate snapshot. The node
is not visible in the committed program until commit.

## 8. Update

Updates use structural patches, not arbitrary text replacement:

```text
UpdateRequest {
    transaction: TransactionId
    target: NodeId
    expected_revision: Revision
    patch: StructuralPatch
}
```

A patch MAY change:

- a type reference;
- a function parameter;
- a return type;
- an effect declaration;
- a contract;
- an expression subtree;
- a visibility declaration;
- a module dependency.

The patch MUST identify the exact target field or child node. A patch that
depends on an ambiguous name match is rejected.

## 9. Delete

Deletion is structural and conservative:

```text
DeleteRequest {
    transaction: TransactionId
    target: NodeId
    expected_revision: Revision
}
```

The operation is rejected when the target has live references unless the
request includes an explicit migration plan. Core v0.1 does not define
automatic reference rewriting.

## 10. Transactions

All mutating operations occur inside a transaction:

```text
Transaction {
    id: TransactionId
    base_snapshot: SnapshotId
    operations: List<Operation>
    state: Open | Validated | Committed | Aborted
}
```

Transaction operations are applied in order to a candidate graph. The base
snapshot remains unchanged while the transaction is open.

Transactions MUST support:

```text
begin(base_snapshot)
stage(operation)
validate(transaction)
diff(transaction)
commit(transaction)
abort(transaction)
```

## 11. Validate

Validation checks the candidate program without committing it:

```text
ValidationReport {
    transaction: TransactionId
    status: Valid | Invalid | Incomplete
    diagnostics: List<Diagnostic>
    changed_nodes: List<NodeId>
    affected_nodes: List<NodeId>
    evidence: List<EvidenceRef>
    checks: List<CheckResult>
}
```

Checks include, at minimum:

1. CIR structural validity;
2. name and reference resolution;
3. type checking;
4. effect checking;
5. contract evaluation where configured;
6. module interface compatibility;
7. attached tests and invariants within the impact set, where a test host is
   configured.

`Incomplete` means that validation did not obtain enough information or did
not finish. It is not equivalent to `Valid`.

## 12. Diff

`diff` returns a semantic change set:

```text
StructuralDiff {
    transaction: TransactionId
    additions: List<NodeId>
    removals: List<NodeId>
    modifications: List<FieldChange>
    interface_changes: List<InterfaceChange>
    effect_changes: List<EffectChange>
    contract_changes: List<ContractChange>
    impact: ImpactReport
}
```

The diff MUST distinguish a moved or reformatted node from a deleted and
recreated node. Stable identities make this possible.

## 13. Impact analysis

```text
ImpactReport {
    directly_changed: List<NodeId>
    affected_functions: List<NodeId>
    affected_modules: List<NodeId>
    invalidated_checks: List<CheckId>
    required_revalidation: List<ValidationTarget>
}
```

Impact is calculated from type references, call edges, effect edges, contract
references and module imports. The implementation MUST report uncertainty when
the graph is incomplete.

## 14. Commit

Commit is permitted only when:

- the transaction is based on the current active snapshot;
- validation status is `Valid`;
- no unresolved conflict exists;
- all required policy checks pass.

The result is:

```text
CommitResult {
    snapshot: Snapshot
    parent: SnapshotId
    change_set: ChangeSet
    validation: ValidationReport
}
```

Commit is atomic: either the complete candidate becomes the new active
snapshot, or the active snapshot remains unchanged.

## 15. Conflicts

A conflict occurs when another commit changes the transaction's base snapshot
or when an expected node revision no longer matches.

```text
Conflict {
    target: NodeId
    expected_revision: Revision
    actual_revision: Revision
    conflicting_changes: List<FieldChange>
}
```

The API MUST NOT silently merge conflicting semantic changes. A future merge
operation may propose a new transaction, but the agent must receive explicit
conflict information first.

## 16. Rollback

Rollback creates a new transaction that reverses a committed change set:

```text
RollbackRequest {
    target_snapshot: SnapshotId
    change_set: ChangeSetId
}
```

Rollback is itself validated and committed as a new snapshot. It does not erase
history or mutate an old snapshot. Rollback may be rejected if later changes
depend on the reverted interface; the impact report must identify them.

## 17. Authorization

The Agent API has two separate authorization layers:

```text
Program authority: may the agent modify this module or node?
Runtime capability: may the resulting program access this resource?
```

Permission to edit a function does not grant permission to add a network
capability. A request that expands effects or capabilities MUST identify that
expansion and pass the applicable policy check.

## 18. Idempotency and retries

Read operations MAY be retried freely. A mutating request is idempotent by
`request_id` and transaction identity. Retrying after an unknown network result
must return the original operation status rather than staging a duplicate node.

## 19. Agent-facing diagnostics

Diagnostics SHOULD include machine-actionable remediation:

```text
Diagnostic {
    code
    phase
    severity
    message
    node
    related_nodes
    expected
    actual
    suggested_operations
}
```

Suggested operations are proposals, not automatic changes. The agent must
submit a new explicit request to apply them.

## 20. Example workflow

```text
1. inspect(UserService, Interface + DirectDependencies)
2. begin(snapshot=current)
3. update(User.age, add_refinement(Integer, value >= 18))
4. validate(transaction)
5. diff(transaction)
6. commit(transaction)
```

If validation fails, no program state changes. If another agent commits first,
the commit returns a conflict and the original snapshot remains available for
inspection.

## 21. Open decisions

The following remain unresolved:

- transport format and session protocol;
- exact JSON schema for every request and response;
- long-running validation jobs;
- authentication of agents;
- human approval gates;
- merge proposals for non-overlapping changes;
- test execution sandbox policy;
- persistent transaction storage.
