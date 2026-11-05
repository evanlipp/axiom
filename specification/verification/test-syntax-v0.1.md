# Axiom Test Syntax v0.1

Status: platform extension; not part of Core language semantics.

Tests are represented in the verification model and may be written in a
separate test view. The initial textual form is:

```axiom-test
test "addition" for add {
    expect add(2, 3) == 5
}

test "division rejects zero" for divide {
    expect failure(DivisionByZero)
        from divide(10, 0)
}
```

## 1. Test declarations

```ebnf
test_decl      = "test" string "for" identifier test_block ;
test_block     = "{" test_statement+ "}" ;
test_statement = "let" identifier "=" expression
               | "expect" expression
               | "expect" "failure" "(" identifier ")" "from" expression ;
```

The test target and all referenced functions must resolve before execution.
Tests have no access to private declarations outside their declared test scope.

## 2. Generators

Property tests use a host-provided generator declaration:

```axiom-test
property "addition is commutative" for add
    cases 1000
    seed 42
    inputs a: Integer, b: Integer
{
    expect add(a, b) == add(b, a)
}
```

Generator semantics and shrinking are platform concerns. The runner must record
the seed, cases, generator version and failing input.

## 3. Status

Test results use the verification statuses and never imply `Proven` merely
because a finite test set passed.
