# Axiom Verification Model v0.1

Status: design specification.

Verification in Axiom is the collection of evidence that a program satisfies
its declared types, effects, contracts and tests. It does not mean that every
possible property of a program can be proven automatically.

## 1. Evidence statuses

Every claim has one status:

```text
Proven   a sound verifier produced a proof accepted by the trust policy
Checked  an executable check passed for the configured inputs/environment
Assumed  an explicitly trusted external premise was supplied
Unknown  no accepted evidence exists
```

Statuses are ordered by strength only for reporting, not as an automatic
conversion rule:

```text
Unknown → Checked → Proven
Unknown → Assumed
```

`Checked` cannot silently become `Proven`. A passing test is not a mathematical
proof unless the verifier explicitly defines it as one.

## 2. Claims

A claim is a typed assertion attached to a program node:

```text
Claim {
    id: ClaimId
    owner: NodeId
    predicate: NodeRef<Expr>
    status: EvidenceStatus
    evidence: List<EvidenceRef>
    scope: VerificationScope
}
```

Claims include preconditions, postconditions, invariants, security properties
and test assertions.

## 3. Tests as program artifacts

Tests are first-class artifacts in the platform, even though the test syntax is
not part of Core v0.1 yet:

```text
Test {
    id: NodeId
    name: String
    target: List<NodeId>
    setup: NodeRef<Expr>?
    body: NodeRef<Expr>
    expected: ExpectedOutcome
    effects: EffectSet
    tags: Set<String>
}
```

A test MUST declare or inherit the effects required by its setup and body. A
test that writes to external state must run inside a configured isolated test
environment.

## 4. Expected outcomes

```text
ExpectedOutcome =
    Returns { value: NodeRef<Expr> }
  | Fails { failure_code: String }
  | Satisfies { claim: NodeRef<Claim> }
```

Expected values are compared using typed equality. Comparing rendered strings
instead of typed values is not a conformance test for program semantics.

## 5. Unit tests

A unit test targets one function or one type operation. It must provide all
inputs explicitly and should run without unrelated module implementations.

Unit test result:

```text
Passed | Failed | Blocked | NotRun
```

`Blocked` means the environment or required capability was unavailable. It is
not equivalent to `Passed`.

## 6. Property tests

Property tests execute a claim over generated inputs:

```text
PropertyTest {
    generator: Generator
    claim: NodeRef<Claim>
    cases: Integer
    seed: Integer
}
```

Core v0.1 does not standardize generator syntax. A property test may establish
`Checked` evidence only for its configured finite sample. The seed and generator
configuration must be recorded for reproducibility.

## 7. Contract checks

Runtime contract checking follows the semantic boundaries:

- `requires` before function execution;
- `ensures` after normal completion;
- invariants at the boundaries where they are attached.

A passing contract check produces `Checked` evidence for that execution. A
failed contract is a verification failure and must prevent a successful commit
when the contract belongs to the impact set.

## 8. Proofs

A proof-producing verifier may attach a proof artifact:

```text
ProofEvidence {
    verifier: String
    verifier_version: String
    theorem: ClaimId
    artifact_hash: Hash
    assumptions: List<ClaimId>
}
```

The verifier MUST identify all assumptions. A proof that depends on an
`Unknown` assumption cannot receive `Proven` status unless the trust policy
explicitly allows that assumption.

Core v0.1 defines the status and evidence interface but does not require a
theorem prover.

## 9. Verification scope

Checks have an explicit scope:

```text
VerificationScope =
    Node
  | Function
  | Module
  | DependencyClosure
  | Program
```

Evidence scoped to one node must not be presented as evidence for the entire
program. A module-level result requires all required checks in that module and
its declared interface dependencies.

## 10. Change impact and test selection

After a candidate change, the platform calculates:

```text
RequiredChecks {
    directly_changed_claims
    invalidated_claims
    affected_tests
    affected_contracts
    affected_interfaces
}
```

A check is invalidated when its target, dependency, input type, effect set,
contract or referenced implementation has changed in a semantically relevant
way.

Unrelated checks MAY be skipped, but the validation report must say that they
were outside the impact set rather than implying that they passed.

## 11. Commit policy

The default commit policy is:

```text
commit allowed when:
    CIR validation = Valid
    type checking = Passed
    effect checking = Passed
    all required checks = Passed
    no required claim = Unknown
```

Projects MAY define stricter policies, such as requiring all public contracts
to be `Proven`. A policy must be versioned and included in the commit report.

## 12. Verification report

```text
VerificationReport {
    snapshot: SnapshotId
    policy: PolicyId
    status: Passed | Failed | Incomplete
    checks: List<CheckResult>
    claims: List<ClaimStatus>
    invalidated: List<ClaimId>
    assumptions: List<ClaimId>
    diagnostics: List<Diagnostic>
}
```

`Incomplete` means at least one required check did not run or lacked evidence.
It cannot be used as a synonym for success.

## 13. Reproducibility

A check result must record enough information to repeat it:

```text
CheckResult {
    id: CheckId
    status: Passed | Failed | Blocked | NotRun
    target: List<NodeId>
    runner: String
    runner_version: String
    inputs_hash: Hash
    environment_hash: Hash
    duration: Duration
    output_hash: Hash
}
```

For nondeterministic effects, the environment must provide a replay record or
the result must be marked non-reproducible.

## 14. Test isolation

Tests must not modify the developer's real environment by default. A test host
provides isolated capabilities for filesystem, network, clock, randomness and
state.

The host must report capability violations as `Blocked` or `Failed`, never hide
them as ordinary test success.

## 15. Test failure and diagnostics

A failed check should identify:

- the test or claim;
- the target node;
- the input or seed;
- expected outcome;
- actual outcome;
- relevant effect requests;
- the smallest causal trace available.

The human-readable message is secondary to stable machine-readable fields.

## 16. Verification and rollback

If a committed change later fails a scheduled or environment-dependent check,
the platform must create a diagnostic event and may propose rollback. Automatic
rollback is a project policy, not a language semantic rule.

Rollback itself is a new change and must pass the same validation process.

## 17. Example

```text
Claim: balance >= 0
Owner: Account.transfer
Status: Checked
Evidence: property-test#8741, seed=42, cases=10000
Scope: Function
```

This means the property passed for the recorded generated cases. It does not
mean the property is proven for every possible execution.

## 18. Open decisions

The following remain unresolved:

- standard test syntax;
- standard input generators;
- proof language and trusted kernel;
- mutation testing;
- coverage semantics for contracts and effects;
- differential testing against foreign implementations;
- scheduled verification jobs;
- policy inheritance across modules and packages.
