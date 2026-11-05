# Axiom Foreign Function Interface v0.1

Status: design specification.

FFI allows Axiom to use existing ecosystems without allowing foreign code to
silently violate Axiom guarantees.

## 1. Boundary model

Every foreign function is represented by an Axiom declaration containing:

```text
ForeignFunction {
    name
    ABI
    parameters
    result
    effects
    failure_model
    safety_status
}
```

The declaration is the trusted boundary contract; the foreign implementation
is not assumed to satisfy it automatically.

## 2. Supported directions

The initial interop targets are:

- C ABI for native libraries;
- WebAssembly imports and exports;
- JavaScript bridge for browser APIs;
- host-process adapters for Python and other runtimes.

## 3. Data crossing

Only explicitly supported representations may cross an FFI boundary. Strings,
bytes, numbers and declared records require defined ownership and lifetime
rules. Opaque handles must not be dereferenced by Axiom code.

## 4. Safety

Every FFI declaration is `unsafe` by default at the trust-model level. A safe
wrapper may validate inputs, outputs, error values and effect behavior, then
expose a normal typed Axiom interface.

## 5. Failures

Foreign failures must be mapped to a declared `Result` or structured boundary
failure. Undefined foreign exceptions, invalid memory and ABI mismatches are
runtime failures, not ordinary Axiom values.

## 6. Verification

The compiler must preserve the declared effects and capability requirements of
foreign calls. It must never infer purity merely because a foreign signature
omits effects.
