# Axiom Core Semantics v0.1

Status: draft normative specification.

This document defines the meaning of valid Core IR. It is written in terms of
the canonical IR, but the same rules apply to any source representation that
compiles to equivalent IR.

## 1. Semantic domains

### 1.1 Values

Core values are:

```text
IntegerValue(n)
BooleanValue(true | false)
StringValue(text)
UnitValue
RecordValue(type, fields)
VariantValue(type, variant, values)
OptionValue(None | Some(value))
ResultValue(Ok(value) | Error(value))
```

Values are immutable. A value is always associated with a type after checking.

### 1.2 Environments

An environment maps visible bindings to values:

```text
Environment = Map<BindingId, Value>
```

Bindings are lexical. A nested environment may shadow a name, but a binding's
identity remains distinct. A reference resolves to exactly one binding before
execution starts.

### 1.3 Outcomes

Evaluation produces one of three outcomes:

```text
Outcome<T> =
    Returned(T)
  | Failed(RuntimeFailure)
  | RequestedEffect(EffectRequest)
```

Core v0.1 has no built-in external effect handlers. Therefore an effect request
that reaches an unbound runtime boundary becomes `Failed(EffectUnavailable)`.

## 2. Evaluation judgment

The main judgment is:

```text
Γ ⊢ e ⇓ o
```

Read as: under environment `Γ`, expression `e` evaluates to outcome `o`.

Function calls use an additional function environment `F` containing resolved
function declarations.

All subexpressions are evaluated from left to right unless a rule explicitly
says that a branch is skipped.

## 3. Literal expressions

Literal evaluation returns the corresponding value:

```text
Γ ⊢ IntegerLiteral(n) ⇓ Returned(IntegerValue(n))
Γ ⊢ BooleanLiteral(b) ⇓ Returned(BooleanValue(b))
Γ ⊢ StringLiteral(s)  ⇓ Returned(StringValue(s))
```

Integer literals use arbitrary precision in Core v0.1. The parser MUST reject a
literal that is not a valid integer representation.

## 4. Variable references

If `Γ[x] = v`, then:

```text
Γ ⊢ VariableRef(x) ⇓ Returned(v)
```

An unresolved reference cannot occur in valid CIR. If it occurs due to a
corrupt or manually constructed runtime artifact, execution fails with
`InvalidReference`.

## 5. Let bindings

For `let x = value; body`, evaluate the value first:

```text
Γ ⊢ value ⇓ Returned(v)
Γ[x ↦ v] ⊢ body ⇓ o
────────────────────────
Γ ⊢ Let(x, value, body) ⇓ o
```

If evaluating `value` fails or requests an unavailable effect, `body` is not
evaluated and the same outcome is returned.

The binding is immutable and is evaluated exactly once.

## 6. Conditional expressions

For `if condition then a else b`:

```text
Γ ⊢ condition ⇓ Returned(BooleanValue(true))
Γ ⊢ a ⇓ o
────────────────────────────────
Γ ⊢ If(condition, a, b) ⇓ o
```

```text
Γ ⊢ condition ⇓ Returned(BooleanValue(false))
Γ ⊢ b ⇓ o
────────────────────────────────
Γ ⊢ If(condition, a, b) ⇓ o
```

The non-selected branch is never evaluated. A condition that is not Boolean
cannot occur in valid CIR; a malformed runtime artifact fails with
`InvalidConditionType`.

## 7. Binary operations

For a binary operation `left op right`, the left operand is evaluated first.
If it succeeds, the right operand is evaluated. If either operand fails, the
operation is not applied.

The operation is then computed according to the operator table:

| Operator group | Operand types | Result |
|---|---|---|
| arithmetic | `Integer`, `Integer` | `Integer` |
| ordering | `Integer`, `Integer` | `Boolean` |
| equality | same type | `Boolean` |
| boolean | `Boolean`, `Boolean` | `Boolean` |

`And` and `Or` are short-circuiting:

- `false and right` returns `false` without evaluating `right`;
- `true or right` returns `true` without evaluating `right`.

Operators at the same precedence level evaluate left-associatively. The
precedence table is defined in the Core grammar specification.

Integer division and remainder by zero fail with `DivisionByZero`.

## 8. Construction and field access

Constructing a record or variant evaluates constructor arguments from left to
right. The constructor succeeds only if the values match the declared fields.

For a record value containing field `f`:

```text
FieldRead(RecordValue(..., f = v), f) ⇓ Returned(v)
```

Accessing a missing field is invalid CIR. A malformed runtime value fails with
`InvalidFieldAccess`.

`Some(v)`, `None`, `Ok(v)` and `Error(v)` are ordinary immutable variant values.
They do not automatically alter control flow.

## 9. Function calls

For a call `f(a1, ..., an)`:

1. resolve `f` to a function declaration;
2. evaluate arguments in order from `a1` to `an`;
3. create a fresh environment binding parameters to argument values;
4. check preconditions;
5. evaluate the function body;
6. check postconditions on normal completion;
7. return the resulting outcome.

The caller's environment is not modified by the callee. Local bindings do not
escape the call.

If an argument fails, later arguments and the function body are not evaluated.

## 10. Return expressions

`return e` evaluates `e` and converts a successful value into a function-level
return. In the CIR expression model this is represented as an internal control
outcome:

```text
InternalOutcome = Value(Value) | Return(Value) | Failed(RuntimeFailure)
```

An explicit return exits the innermost function immediately. A `Return` that
escapes a function body with an incompatible type is a validation error and
cannot occur in valid CIR.

The final expression of a function body is returned normally.

## 11. Result values and error propagation

`Result<T, E>` is data, not an implicit exception mechanism. A function may
return `Error(e)` as an ordinary successful evaluation of type
`Result<T, E>`.

Core v0.1 does not define a propagation operator such as `?`. If propagation is
added later, it must be specified as an explicit IR operation with clear type
and control-flow rules.

## 12. Effects

Effect declarations are checked statically. They do not change the value
semantics of pure expressions.

An effectful primitive or host binding evaluates to an `EffectRequest`:

```text
EffectRequest {
    effect: EffectRef
    operation: String
    arguments: List<Value>
}
```

An execution host MAY handle the request and resume computation with a value.
If no handler is installed, the interpreter returns:

```text
EffectUnavailable(effect, operation)
```

The host MUST NOT resume an effect request with a value that violates the
declared result type.

## 13. Preconditions

Before a function body is evaluated, each `Requires` contract is evaluated in
the parameter environment.

For a contract that evaluates to `true`, execution continues. For `false`,
execution fails with:

```text
PreconditionViolation { function: NodeId, contract: NodeId }
```

A contract evaluation failure is propagated as a runtime failure. Preconditions
are checked in declaration order.

## 14. Postconditions

After a function body returns normally, each `Ensures` contract is evaluated in
an environment containing the input bindings and the special `result` binding.

If any postcondition is false, execution fails with:

```text
PostconditionViolation { function: NodeId, contract: NodeId }
```

Postconditions are checked in declaration order. They are not checked after an
unhandled runtime failure.

## 15. Runtime failure taxonomy

Core runtime failures include:

```text
DivisionByZero
InvalidReference
InvalidConditionType
InvalidFieldAccess
InvalidConstructor
PreconditionViolation
PostconditionViolation
EffectUnavailable
ContractEvaluationFailure
```

Every failure MUST retain the node that caused it when that node is known.
Failures are not strings only; the textual message is a rendering of the
structured failure.

## 16. Determinism

For a valid pure program, fixed function definitions and fixed inputs:

```text
evaluate(program, inputs) = one unique outcome
```

The implementation MUST NOT depend on map iteration order, memory addresses,
thread scheduling or source formatting. Host effect handlers are responsible
for documenting and controlling nondeterminism introduced at effect boundaries.

## 17. Example trace

For:

```axiom
function add_one(x: Integer) -> Integer {
    let y = x + 1
    y
}
```

and input `x = 4`:

```text
Γ0 = { x ↦ IntegerValue(4) }
Γ0 ⊢ x + 1 ⇓ Returned(IntegerValue(5))
Γ1 = Γ0 ∪ { y ↦ IntegerValue(5) }
Γ1 ⊢ y ⇓ Returned(IntegerValue(5))
function result = IntegerValue(5)
```

The trace is explanatory metadata. It is not part of program semantics and
implementations are not required to produce it by default.

## 18. Conformance requirements

An implementation conforms to Core Semantics v0.1 if it:

1. rejects every invalid CIR instance required by the IR specification;
2. evaluates valid pure expressions according to these rules;
3. preserves left-to-right evaluation order;
4. does not evaluate skipped conditional or short-circuited branches;
5. checks contracts at the defined boundaries;
6. reports runtime failures structurally;
7. never silently handles an unavailable effect.

## 19. Deferred semantic features

The following have no Core v0.1 semantics yet:

- mutation and aliasing;
- loops and recursion limits;
- concurrency;
- cancellation;
- memory allocation strategy;
- foreign calls;
- transactions over external state;
- exceptions and process failure;
- proof-producing contract verification.
