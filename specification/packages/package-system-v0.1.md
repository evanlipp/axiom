# Axiom Package System v0.1

Status: design specification.

Packages distribute versioned module artifacts and make builds reproducible.

## 1. Manifest

Every package has a manifest containing:

```text
PackageManifest {
    name
    version
    axiom_version
    modules
    dependencies
    targets
    integrity
    capabilities
}
```

Dependencies use explicit version constraints and are resolved before type
checking. The resolved graph is written to a lock file.

## 2. Reproducibility

A build is reproducible when the source, compiler version, standard-library
version, dependency lock and target configuration identify the same artifact.
Package artifacts must be content-addressed or protected by integrity hashes.

## 3. Dependency policy

Dependencies may declare required effects and capabilities. A package cannot
silently expand the capabilities of an application importing it. Application
policy must approve capability expansion explicitly.

## 4. Publication

A package may be published only when its public modules pass interface,
type, effect and required verification checks. Private implementation details
need not be distributed when the selected target supports compiled artifacts.

## 5. Supply-chain metadata

Packages SHOULD record provenance, source revision, signer, integrity hash and
verification summary. Consumers must be able to reject unsigned or untrusted
artifacts through policy.
