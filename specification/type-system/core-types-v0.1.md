# Axiom Core Type System v0.1

Status: draft normative specification.

The type system determines which values may exist, which operations are valid
and which function calls are well-formed. It is static: an accepted program
cannot fail because of an ordinary type mismatch.

## 1. Typing judgment

The main typing judgment is:

```text
Γ ⊢ e : T
```

Read as: in typing environment `Γ`, expression `e` has type `T`.

The environment maps visible binding identities to types:

```text
Γ = Map<BindingId, Type>
```

Function and type declarations are resolved before expression typing begins.

## 2. Type equality

Two types are equal when they have the same canonical identity and equal type
arguments. Type aliases are resolved before comparison.

Core v0.1 has no implicit subtyping and no implicit conversion. In particular:

```text
Integer ≠ Boolean
Integer ≠ String
Integer ≠ Option<Integer>
Integer ≠ refined Integer
```

A refined type may be used where its exact base type is expected only through
an explicit `erase` operation, which is not available in the source language
yet. Implementations MUST NOT silently erase refinement evidence.

## 3. Core types

The built-in types are:

```text
Integer
Boolean
String
Unit
```

They are closed types: no user-defined value can inhabit a built-in type other
than through the operations defined by the language.

## 4. User-defined types

### 4.1 Records

A record type defines named fields:

```text
type User = {
    name: String
    age: Integer
}
```

Two record declarations with identical fields are still distinct nominal types.
Record construction requires exactly the declared fields. Extra or missing
fields are type errors.

### 4.2 Sum types

A sum type defines named variants:

```text
type PaymentError = { InsufficientFunds | AccountBlocked }
```

Each variant belongs to exactly one declared sum type. Variant names are
resolved through their containing type in canonical IR.

### 4.3 Option

`Option<T>` contains either no value or one value of type `T`:

```text
None       : Option<T>
Some(x)    : Option<T>  if x : T
```

`None` is polymorphic at the IR typing level and is resolved from the expected
type. If no expected type exists, the compiler must report an ambiguous type.

### 4.4 Result

`Result<T, E>` contains either a successful value or an error value:

```text
Ok(x)      : Result<T, E>  if x : T
Error(e)   : Result<T, E>  if e : E
```

The error type is part of the function's type. Errors cannot be silently
converted to strings or thrown as exceptions.

## 5. Refined types

A refined type narrows a base type with a Boolean predicate:

```text
AdultAge = Integer where value >= 18 and value <= 150
```

In CIR:

```text
RefinedType {
    base: Integer
    predicate: value >= 18 and value <= 150
}
```

To construct a refined value, the compiler or runtime must obtain evidence that
the predicate is true. Evidence has one of four statuses:

```text
Proven | Checked | Assumed | Unknown
```

`Unknown` evidence cannot satisfy a refined type. `Assumed` evidence is allowed
only at an explicitly declared trust boundary. A normal constructor produces
`Checked` evidence after evaluating the predicate.

Core v0.1 does not define a separate source-level refinement constructor. CIR
and host tools may represent one, while the textual syntax remains deferred.

## 6. Literal typing

```text
Γ ⊢ integer_literal : Integer
Γ ⊢ true : Boolean
Γ ⊢ false : Boolean
Γ ⊢ string_literal : String
```

Integer literals do not overflow because Core integers are arbitrary precision.

## 7. Variable typing

If `Γ[x] = T`, then:

```text
Γ ⊢ x : T
```

A variable reference is valid only when its binding is visible at the reference
node. Lexical visibility is determined from CIR binding relationships, not from
the textual spelling of a name.

## 8. Let typing

For `let x = value; body`:

```text
Γ ⊢ value : T
Γ[x : T] ⊢ body : U
────────────────────────
Γ ⊢ let x = value; body : U
```

Bindings are immutable. A binding's inferred type is the type of its value and
cannot later be widened by reassignment.

## 9. Conditional typing

```text
Γ ⊢ condition : Boolean
Γ ⊢ then_branch : T
Γ ⊢ else_branch : T
──────────────────────────────
Γ ⊢ if condition then then_branch else else_branch : T
```

Both branches must have exactly the same type. Core v0.1 does not compute a
common supertype.

## 10. Binary operator typing

Arithmetic operators:

```text
Γ ⊢ left : Integer   Γ ⊢ right : Integer
────────────────────────────────────────
Γ ⊢ left (+|-|*|/|%) right : Integer
```

Ordering operators:

```text
Γ ⊢ left : Integer   Γ ⊢ right : Integer
────────────────────────────────────────
Γ ⊢ left (<|<=|>|>=) right : Boolean
```

Equality:

```text
Γ ⊢ left : T   Γ ⊢ right : T
──────────────────────────
Γ ⊢ left (==|!=) right : Boolean
```

Boolean operators:

```text
Γ ⊢ left : Boolean   Γ ⊢ right : Boolean
──────────────────────────────────────
Γ ⊢ left (and|or) right : Boolean
```

No operator is overloaded across unrelated types.

## 11. Record and variant construction

For a record declaration with fields `f₁:T₁ ... fₙ:Tₙ`:

```text
Γ ⊢ e₁ : T₁  ...  Γ ⊢ eₙ : Tₙ
──────────────────────────────────────
Γ ⊢ Record(f₁=e₁, ..., fₙ=eₙ) : RecordType
```

For a variant `V(T₁, ..., Tₙ)`:

```text
Γ ⊢ e₁ : T₁  ...  Γ ⊢ eₙ : Tₙ
──────────────────────────────────────
Γ ⊢ V(e₁, ..., eₙ) : ContainingSumType
```

Field names, variant names and arity are checked statically.

## 12. Field access

If record type `R` contains field `f : T`:

```text
Γ ⊢ value : R
────────────────
Γ ⊢ value.f : T
```

Field access on a sum type is invalid unless a future pattern-matching feature
has established the active variant. Pattern matching is outside Core v0.1.

## 13. Function calls

For a function declaration:

```text
f : (T₁, ..., Tₙ) -> U
```

the call rule is:

```text
Γ ⊢ e₁ : T₁  ...  Γ ⊢ eₙ : Tₙ
────────────────────────────────
Γ ⊢ f(e₁, ..., eₙ) : U
```

Arguments must match parameter types exactly. A function's effect set is
checked in a separate effect judgment and does not alter its value return type.

## 14. Return typing

Inside a function declared to return `T`:

```text
Γ ⊢ expression : T
────────────────────
Γ ⊢ return expression : Return(T)
```

Every normal path through a function body must produce the declared return
type. A return of `Result<T, E>` is not interchangeable with a return of `T`.

## 15. Contract typing

Every contract predicate must be Boolean:

```text
Γ ⊢ predicate : Boolean
────────────────────────
Γ ⊢ contract(predicate) : Contract
```

For an `ensures` contract, `result` is added to the contract environment with
the function's declared return type. Contract expressions may not require
effects in Core v0.1; they must be pure and deterministic.

## 16. Type checking and diagnostics

Type checking is a closed-world validation phase. If no typing derivation can
be constructed for an expression, the program is rejected.

At minimum, type diagnostics must distinguish:

```text
UnknownName
UnknownType
TypeMismatch
WrongArity
InvalidField
InvalidVariant
AmbiguousOptionType
InvalidRefinementEvidence
InvalidContractType
InvalidReturnType
```

Diagnostics should include expected type, actual type and the relevant node
identities when those are available.

## 17. Worked examples

Valid:

```axiom
function add(a: Integer, b: Integer) -> Integer {
    a + b
}
```

Invalid because the branches have different types:

```axiom
function choose(flag: Boolean) -> Integer {
    if flag { 1 } else { "one" }
}
```

Invalid because `Error` has no inferred error type:

```axiom
function bad() -> Result<Integer, String> {
    Error(42)
}
```

Valid:

```axiom
function good() -> Result<Integer, String> {
    Error("not available")
}
```

## 18. Deferred type-system features

The following are intentionally excluded from Core v0.1:

- generics and type parameters;
- typeclasses, traits and interfaces;
- subtyping and variance;
- pattern matching exhaustiveness;
- recursive type definitions;
- implicit type inference across module boundaries;
- linear, affine or ownership types;
- nullable types;
- dynamic or gradual typing;
- automatic refinement propagation.
