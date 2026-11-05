# Axiom Core Effect System v0.1

Status: draft normative specification.

The effect system records interaction with state, time, randomness and external
resources. It answers a question separate from ordinary typing:

> What may this computation do outside of transforming its input values?

## 1. Effect sets

An effect set is a finite set of named effect atoms:

```text
EffectSet = Set<EffectAtom>
```

Core effect atoms are:

```text
state.read
state.write
clock.read
random.use
filesystem.read
filesystem.write
network.send
```

An empty set means that a computation is pure with respect to the Core effect
model. Effects are not values and cannot be stored in ordinary data types.

## 2. Effect judgment

The effect judgment is:

```text
Γ ⊢ e : T ! E
```

Read as: expression `e` has value type `T` and may require effects `E`.

The type system and effect system are checked together. A program is accepted
only when both judgments succeed.

## 3. Effect subset rule

An implementation may analyze an expression to discover a minimal effect set,
but a declared function effect set is an upper bound:

```text
inferred_effects ⊆ declared_effects
```

If an expression requires an effect not present in the function declaration,
the function is rejected. Declaring extra effects is allowed in Core v0.1, but
tools SHOULD warn about effects that are declared and never used.

## 4. Pure expressions

Literals, variable references and pure arithmetic have no effects:

```text
Γ ⊢ literal : T ! {}
Γ ⊢ variable : T ! {}
Γ ⊢ left : T₁ ! E₁   Γ ⊢ right : T₂ ! E₂
──────────────────────────────────────────
Γ ⊢ left op right : T ! (E₁ ∪ E₂)
```

The result type `T` is determined by the type-system rules. Evaluation order is
defined by the operational semantics and is not changed by effect analysis.

## 5. Let and conditional effects

For `let x = value; body`:

```text
Γ ⊢ value : T ! E₁
Γ[x : T] ⊢ body : U ! E₂
────────────────────────────
Γ ⊢ let x = value; body : U ! (E₁ ∪ E₂)
```

For a conditional:

```text
Γ ⊢ condition : Boolean ! E₀
Γ ⊢ then_branch : T ! E₁
Γ ⊢ else_branch : T ! E₂
────────────────────────────────────────────
Γ ⊢ if condition then then_branch else else_branch
       : T ! (E₀ ∪ E₁ ∪ E₂)
```

The static effect set includes both branches even though runtime execution
evaluates only one. This gives callers a safe upper bound independent of input.

## 6. Function declarations

A function declaration contains an explicit effect set:

```text
FunctionType {
    parameters: List<Type>
    result: Type
    effects: EffectSet
}
```

The body must satisfy:

```text
Γ_parameters ⊢ body : ReturnType ! E_body
E_body ⊆ E_declared
```

A function with no `effects` clause has `E_declared = {}`. Such a function is
pure and cannot call an effectful function.

## 7. Function calls

For a function `f` with declared effects `E_f`:

```text
Γ ⊢ arg₁ : T₁ ! E₁ ... Γ ⊢ argₙ : Tₙ ! Eₙ
────────────────────────────────────────────────────
Γ ⊢ f(arg₁, ..., argₙ) : R ! (E₁ ∪ ... ∪ Eₙ ∪ E_f)
```

The caller's declared effect set must contain the complete resulting set. This
rule makes effect requirements transitively visible:

```text
read_clock   effects [clock.read]
wrapper      effects [clock.read]
pure_wrapper effects []                 // rejected
```

Indirect calls are resolved through CIR references before effect checking.
Core v0.1 does not support unknown function pointers or dynamic dispatch.

## 8. Effect declarations are not permissions

An effect declaration states what a function may require; it does not grant
access to a concrete resource. Resource access is bound separately by runtime
capabilities.

```text
Effect:      filesystem.read
Capability:  filesystem.read("/input")
```

A function may declare `filesystem.read` and still fail at runtime if no host
capability is bound. A host MUST NOT bind a capability whose effect atom is not
declared by the active call chain.

## 9. Capability model

A capability is an opaque runtime authority:

```text
Capability {
    effect: EffectAtom
    resource: ResourceSelector
    authority: AuthorityId
}
```

Core v0.1 does not standardize the syntax of `ResourceSelector` or the policy
language for authorities. It does require these rules:

1. capabilities are supplied by the host, not manufactured by ordinary code;
2. a capability cannot be widened by a function;
3. a capability cannot be serialized as ordinary program data;
4. an effect request without a matching capability fails;
5. the host records the capability boundary in diagnostics when possible.

## 10. Effects and contracts

Contract predicates MUST be pure:

```text
Γ ⊢ requires_predicate : Boolean ! {}
Γ ⊢ ensures_predicate  : Boolean ! {}
```

A contract cannot read the clock, generate randomness, access a file or query a
database. This ensures that checking a contract does not change the behavior
being checked.

## 11. Effects and `Result`

`Result<T, E>` does not represent an effect. Returning `Error(e)` is a normal
value-level outcome and has the same effect set as returning `Ok(v)` from the
same function.

For example:

```text
function parse_age(text: String)
    -> Result<Integer, ParseError>
    effects []
```

The function is pure even though it may return an error value.

## 12. Determinism classification

Core v0.1 classifies effects as follows:

```text
Deterministic by value alone: {}
Environment-dependent: state.read, filesystem.read, clock.read
Externally mutating: state.write, filesystem.write, network.send
Nondeterministic by default: random.use
```

This classification is metadata for tooling and replay. It does not replace
the effect-set check. A host MAY provide a deterministic replay handler for an
otherwise environment-dependent effect.

## 13. Effect diagnostics

At minimum, effect checking must distinguish:

```text
UndeclaredEffect
EffectInContract
UnknownEffect
MissingCapability
CapabilityEffectMismatch
DynamicCallNotSupported
```

An `UndeclaredEffect` diagnostic SHOULD include:

- the function node;
- the call chain that introduced the effect;
- the missing effect atom;
- the nearest declaration where it could be added.

## 14. Worked example

```axiom
function current_year() -> Integer
    effects [clock.read]
{
    // Host binding omitted in Core v0.1.
    return 2026
}

function report() -> Integer
    effects [clock.read]
{
    let year = current_year()
    year
}
```

The example returns `Integer`; a text formatting operation is not part of Core
v0.1. Such formatting belongs in the standard library.

The call to `current_year` contributes `clock.read` to `report`. A function
declared with `effects []` could not call it.

## 15. Effect safety invariant

For every accepted function call at runtime:

```text
required_effects(call_chain) ⊆ declared_effects(active_functions)
```

If a malformed IR artifact violates this invariant, execution MUST stop with a
structured failure. The runtime must not infer permission from the attempted
operation.

## 16. Deferred effect features

The following are intentionally excluded from Core v0.1:

- effect polymorphism;
- effect variables and row types;
- user-defined effect atoms;
- effect handlers in the language;
- scoped capability delegation;
- transactional effects;
- effect cancellation and compensation;
- distributed effect propagation;
- automatic effect inference across foreign code.
