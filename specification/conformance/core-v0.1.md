# Axiom Core v0.1 Conformance Fixtures

Status: normative fixture set.

Each implementation of the Core parser, checker and interpreter should run
these fixtures. Expected diagnostics use stable codes from the specifications.

## Fixture 1: integer arithmetic

```axiom
module Arithmetic

function calculate(a: Integer, b: Integer) -> Integer {
    a + b * 2
}
```

Expected:

```text
parse: pass
type: pass
effects: pass
calculate(3, 4): IntegerValue(11)
```

This fixture checks precedence of multiplication over addition.

## Fixture 2: short-circuiting

```axiom
module BooleanLogic

function safe(value: Boolean) -> Boolean {
    false and (1 == 2)
}
```

Expected:

```text
parse: pass
type: pass
run: BooleanValue(false)
```

The division must not be evaluated.

## Fixture 3: records

```axiom
module Records

type User = { name: String age: Integer }

function user_age(user: User) -> Integer {
    user.age
}
```

Expected:

```text
parse: pass
type: pass
```

## Fixture 4: result values

```axiom
module Results

type DivisionError = { DivisionByZero }

function divide(a: Integer, b: Integer)
    -> Result<Integer, DivisionError>
    requires b != 0
{
    Ok(a / b)
}
```

Expected:

```text
parse: pass
type: pass
effects: pass
```

## Fixture 5: precondition failure

Use `divide` from Fixture 4 with input `(10, 0)`.

Expected:

```text
run: PreconditionViolation
```

The division operation itself must not run.

## Fixture 6: undeclared effect

```axiom
module Effects

function read_clock() -> Integer
    effects [clock.read]
{
    2026
}

function pure_wrapper() -> Integer {
    read_clock()
}
```

Expected:

```text
type: fail
diagnostic: UndeclaredEffect
```

## Fixture 7: type mismatch

```axiom
module Invalid

function wrong() -> Boolean {
    42
}
```

Expected:

```text
type: fail
diagnostic: InvalidReturnType
```

## Fixture 8: unknown identifier

```axiom
module InvalidName

function wrong() -> Integer {
    missing_value
}
```

Expected:

```text
resolve: fail
diagnostic: UnknownName
```

## Fixture 9: contract failure

```axiom
module Contracts

function positive(value: Integer) -> Integer
    requires value > 0
{
    value
}
```

Use input `0`.

Expected:

```text
run: PreconditionViolation
```

## Fixture 10: refinement syntax

```axiom
module Refined

type AdultAge = Integer where value >= 18 and value <= 150
```

Expected:

```text
parse: pass
type: pass
```

Construction from an unchecked external integer remains outside this fixture
until the refinement-construction operation is finalized.
