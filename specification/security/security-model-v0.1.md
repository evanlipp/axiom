# Axiom Security Model v0.1

Status: design specification.

Axiom security is capability-based and defense-in-depth. Language checks are
combined with runtime isolation; neither layer is considered sufficient alone.

## 1. Capabilities

A capability is an unforgeable authority bound by the host. Code can use a
resource only when it holds a matching capability and declares the associated
effect.

```text
Capability {
    effect
    resource_selector
    authority
    limits
}
```

Capabilities are not ordinary serializable values.

## 2. Least authority

Modules receive only the capabilities required by their declared interface and
runtime policy. A child task or imported module receives no additional
authority automatically.

## 3. Isolation

The runtime SHOULD isolate modules using process, container, WASM or equivalent
sandbox boundaries. A sandbox must restrict filesystem, network, process,
secret and resource access according to policy.

## 4. Secrets

Secrets MUST NOT be embedded into source, Canonical IR, logs or ordinary values.
Secret access is an explicit capability and diagnostic output must redact secret
contents.

## 5. Trust boundaries

Foreign code, host callbacks, deserialized data and agent modifications are
untrusted boundaries by default. Data crossing a boundary must be validated or
marked as an explicit assumption.

## 6. Agent permissions

The ability to inspect a module, edit it, add effects, commit changes and grant
capabilities are separate permissions. An agent edit must not implicitly grant
runtime authority.

## 7. Security events

Denied access, policy violations, sandbox escapes and capability mismatches
must produce structured security events. The runtime must fail closed when a
policy decision is unavailable.
