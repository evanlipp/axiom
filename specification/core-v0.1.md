# Axiom Core v0.1

Status: draft normative specification.

The keywords `MUST`, `MUST NOT`, `SHOULD` and `MAY` are normative. This
specification defines the smallest executable Axiom language. Features not
defined here are invalid in Core v0.1.

## 1. Scope

Core v0.1 defines:

- modules and declarations;
- primitive, record and sum types;
- `Option` and `Result` values;
- functions with explicit effect sets;
- local immutable bindings;
- conditional expressions;
- typed returns and errors;
- preconditions and postconditions;
- a closed set of named effects;
- deterministic evaluation;
- structured diagnostics.

It does not define mutation, loops, concurrency, exceptions, generics,
reflection, macros, foreign functions or a memory model.

## 2. Source model

A source file contains exactly one module. The canonical program consists of
modules identified by stable `ModuleId`s. Every declaration and expression in
the canonical representation has a stable `NodeId`.

Whitespace is insignificant except as a separator. Comments begin with `//`
and continue to the end of the line. Identifiers are case-sensitive and match
`[A-Za-z_][A-Za-z0-9_]*`.

## 3. Grammar

The following grammar is intentionally compact. `*` means zero or more and
`?` means optional.

```ebnf
program       = module ;
module        = "module" identifier declaration* ;

declaration   = type_decl | function_decl ;
type_decl     = "type" identifier "=" type_definition ;
type_definition = record_decl | sum_decl | type_expr ;
record_decl   = "{" field* "}" ;
sum_decl      = "{" variant ("|" variant)* "}" ;
variant       = identifier ("(" type_expr_list ")")? ;
field         = identifier ":" type_expr ;

function_decl = "function" identifier "(" parameter_list? ")"
                "->" type_expr contract* effect_decl? block ;
parameter     = identifier ":" type_expr ;
parameter_list = parameter ("," parameter)* ;
contract      = "requires" expression
              | "ensures" expression ;
effect_decl   = "effects" "[" identifier_list? "]" ;
identifier_list = identifier ("," identifier)* ;

block         = "{" statement* expression "}" ;
statement     = "let" identifier "=" expression ;
expression    = binary_expr ;
binary_expr   = primary_expr (binary_operator primary_expr)* ;
primary_expr  = literal
              | identifier
              | constructor
              | call
              | if_expr
              | return_expr
              | "(" expression ")" ;
if_expr       = "if" expression block "else" block ;
return_expr   = "return" expression ;
call          = identifier "(" argument_list? ")" ;
constructor   = identifier "(" argument_list? ")" ;
argument_list = expression ("," expression)* ;
type_expr_list = type_expr ("," type_expr)* ;
binary_operator = "+" | "-" | "*" | "/" | "%"
                | "<" | "<=" | "==" | "!=" | ">" | ">="
                | "and" | "or" ;

type_expr     = refined_type | primitive | identifier | "Option" "<" type_expr ">"
              | "Result" "<" type_expr "," type_expr ">" ;
refined_type  = (primitive | identifier) "where" expression ;
primitive     = "Integer" | "Boolean" | "String" | "Unit" ;
literal       = integer | boolean | string ;
```

In a refined-type predicate, `value` is an implicit binding with the base type.

The grammar is not sufficient to determine validity. Name resolution, typing,
effect checking and contract checking are semantic phases.

## 4. Types

Core types are:

```text
Integer  signed arbitrary-precision integer
Boolean  true or false
String   Unicode string
Unit     one value: Unit
```

User-defined record types contain named fields. User-defined sum types contain
named variants. `Option<T>` is either `Some(T)` or `None`. `Result<T, E>` is
either `Ok(T)` or `Error(E)`.

There are no implicit conversions. Equality and binary operators are defined
only for operands of compatible types. A value cannot be `null`.

## 5. Names and modules

Names MUST be declared before use within the resolved module graph. A module
has a private namespace in Core v0.1; imports and exports are not yet defined.
The module containing a reference MUST contain the referenced declaration.

Duplicate declarations in one module are errors.

## 6. Functions and evaluation

For inputs satisfying the function's preconditions and with all required host
capabilities available, a function must produce a value or a typed `Result`
value. A function MUST NOT terminate by an undeclared exception.

Function arguments are evaluated from left to right. `let` bindings are
immutable and are evaluated once, at the point of declaration. In an `if`
expression, only the selected branch is evaluated.

The last expression of a function block is its result. `return expression` is
an explicit early return. A function body MUST have a type compatible with its
declared return type.

Core evaluation is deterministic: the same valid program and the same inputs
produce the same result and diagnostics.

## 7. Operators

Core v0.1 defines:

```text
Integer: + - * / % < <= == != > >=
Boolean: and or == !=
String:  == !=
```

Division by zero is a type-checking error when statically provable and a
runtime diagnostic otherwise. Operator evaluation order is left to right.
Operators are left-associative. Precedence, from highest to lowest, is:

```text
1. parentheses and primary expressions
2. * / %
3. + -
4. < <= > >=
5. == !=
6. and
7. or
```

## 8. Effects

Core v0.1 defines these effect names:

```text
state.read
state.write
clock.read
random.use
filesystem.read
filesystem.write
network.send
```

Functions without an `effects` declaration are pure and have an empty effect
set. A function MAY call another function only when the caller's declared
effect set contains every effect required by the callee. Effects are
transitive through calls.

The Core interpreter does not provide external resources yet. Calling an
effectful primitive without a bound capability produces a structured runtime
diagnostic.

## 9. Contracts

`requires` is evaluated immediately before a function body starts. If it is
false, evaluation fails with `PreconditionViolation`.

`ensures` is evaluated after normal function completion. The special name
`result` refers to the returned value. If it is false, evaluation fails with
`PostconditionViolation`.

Contracts are executable checks in Core v0.1. Their result status is
`Checked`, not `Proven`. A future verifier MAY additionally assign `Proven`
when it has a sound proof.

## 10. Errors and diagnostics

A compiler or interpreter MUST emit structured diagnostics with at least:

```text
Diagnostic {
    code: String
    phase: Parse | Resolve | Type | Effect | Contract | Runtime
    message: String
    node: NodeId?
    severity: Error | Warning | Info
}
```

Diagnostic messages are for humans; `code`, `phase` and `node` are stable
machine-readable fields.

## 11. Canonical IR minimum

Every accepted program MUST be representable using these concepts:

```text
Program(modules: List<Module>)
Module(id, name, declarations)
TypeDecl(id, name, definition)
Function(id, name, parameters, return_type, contracts, effects, body)
Expr(id, type, operation, children)
```

The IR MUST preserve source locations when available, but source locations do
not participate in program semantics. IR serialization is outside the syntax
of the language; JSON is the initial interchange format recommendation.

## 12. Conformance examples

Valid:

```axiom
module Math

type DivisionError = { DivisionByZero }

function divide(a: Integer, b: Integer)
    -> Result<Integer, DivisionError>
    requires b != 0
{
    return Ok(a / b)
}
```

Invalid: the declared return type is incompatible with `Integer`:

```axiom
module Invalid

function answer() -> Boolean {
    return 42
}
```

Invalid: a pure function calls an effectful function:

```axiom
module Effects

function read_time() -> Integer
    effects [clock.read]
{
    return 0
}

function pure_wrapper() -> Integer {
    return read_time()
}
```

## 13. Open decisions

The following are intentionally unresolved and MUST NOT be inferred by an
implementation:

- exact serialization schema for canonical IR;
- import and export syntax;
- generic types;
- ownership and memory management;
- concurrency and cancellation;
- proof language and theorem prover integration;
- capability syntax for external resources.
