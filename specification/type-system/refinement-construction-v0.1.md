# Axiom Refined Value Construction v0.1

Status: design specification.

Refined types prevent an unchecked external value from being treated as a value
that satisfies a predicate. Construction is therefore an explicit operation.

## 1. Construction operation

The canonical operation is:

```text
Refine {
    target: RefinedTypeRef
    value: Expr
}
```

If the target is `T where P`, the operation evaluates the source value, binds
it to `value`, and evaluates predicate `P`.

## 2. Result type

For a refined target `R` with base type `T`:

```text
refine(value: T, target: R) : Result<R, RefinementError>
```

The operation returns `Ok(refined_value)` only when the predicate is true. It
returns `Error(RefinementFailed)` when the predicate is false.

```text
RefinementError =
    RefinementFailed
  | RefinementEvaluationFailed
```

The error type may carry the target type identity and a redacted diagnostic,
but must not leak secrets from the input value.

## 3. Typing rule

```text
Γ ⊢ value : T
target = Refined(T, P)
────────────────────────────────────
Γ ⊢ Refine(target, value) : Result<target, RefinementError>
```

The predicate must have type `Boolean` when `value` is bound to the base type.
Refinement predicates must be pure and deterministic.

## 4. Evidence

Successful runtime construction produces `Checked` evidence. A static verifier
may replace it with `Proven` evidence only when it has a sound proof that `P`
holds for the source value.

`Unknown` evidence cannot construct a refined value. `Assumed` evidence is
available only through the explicit trust-boundary operation below.

## 5. Trusted assumption

Host code may provide:

```text
assume_refined(value: T, target: R, authority: AssumptionAuthority)
    : Result<R, RefinementError>
```

This operation is unavailable to ordinary Core code. It requires a declared
trust-boundary capability and produces `Assumed` evidence. The authority,
source boundary and policy identity must be recorded in the evidence object.

## 6. No implicit refinement

The following are invalid without explicit evidence:

- assigning `Integer` to `AdultAge`;
- returning an arbitrary `Integer` where `AdultAge` is required;
- passing an unvalidated external field as `AdultAge`;
- erasing a refinement and reconstructing it without a check.

Refinement construction is not an implicit cast.

## 7. Boundary behavior

Deserialization, FFI, database reads, network responses and user input produce
base or untrusted types. Adapters should immediately call `Refine` when a
refined type is required.

Example:

```axiom
type AdultAge = Integer where value >= 18 and value <= 150

function parse_age(raw: String)
    -> Result<AdultAge, AgeError>
{
    let number = parse_integer(raw)
    refine(number, AdultAge)
}
```

The exact source spelling of `refine` is provisional; the CIR operation and
typing rules are normative.

## 8. Runtime semantics

The source value is evaluated exactly once. The predicate is evaluated only
after the source value succeeds. A false predicate returns `Error` and does not
produce a value of the refined type.

If predicate evaluation fails, the operation returns
`RefinementEvaluationFailed`; it must not manufacture a successful refined
value.
