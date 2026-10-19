---
title: Extract BMC package into a standalone Go module
iep-number: 0026
creation-date: 2026-10-19
status: implementable
authors:
  - "@asergeant01"
reviewers:
  - "TBD"
---

# IEP-0026: Extract BMC package into a standalone Go module

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
- [Security Considerations](#security-considerations)
- [Testing Strategy](#testing-strategy)
- [Alternatives](#alternatives)

## Summary

The `bmc` package in `metal-operator` is a self-contained Redfish/BMC client
library that both `metal-operator` (MO) and `metal-maintenance-operator` (MMO)
depend on. Today it lives inside the MO module, forcing MMO to declare a
full dependency on MO just to consume the BMC library. This IEP proposes
extracting it into a new standalone Go module, `github.com/ironcore-dev/bmc`,
so both operators — and any other future consumer — can depend on it directly
and independently.

## Motivation

MMO's `go.mod` currently declares:

```
github.com/ironcore-dev/metal-operator v0.8.1-0.20261016140341-c0b87c0bb354
```

This pseudo-version pin is required solely because the `bmc` package lives
inside MO. As a result:

- A breaking change anywhere in MO's API types, controllers, or transitive
  dependencies can affect MMO's build, even when entirely unrelated to BMC
  functionality.
- MMO's Kubernetes dependencies are already one minor version ahead of MO's
  (v0.37 vs v0.36). Every dependency upgrade in either repo risks transitive
  conflicts rooted in this coupling.
- Issue #991 ("Move BMCUtils package to public") was a workaround for this
  coupling, not a fix — MMO needed bmc-adjacent utilities and the only path
  was a structural change inside MO.
- Issue #967 improved the `bmc` API for third-party vendor implementations,
  but external consumers must still take a dependency on the full MO module,
  including its Kubernetes operator stack.

The `bmc` package's own dependency surface is minimal: it requires only
`github.com/stmcginnis/gofish` (the Redfish transport) and
`k8s.io/apimachinery` (for `resource.Quantity` in hardware info structs).
It has no dependency on controller-runtime, client-go, or any MO CRD types.
It is a library and should be versioned and released as one.

### Goals

- Extract the `bmc` package and its Redfish mock server into a new
  `github.com/ironcore-dev/bmc` module.
- Both MO and MMO become equal consumers of the new module; neither depends
  on the other for BMC functionality.
- Third-party vendors can implement and register custom BMC clients by
  importing only `github.com/ironcore-dev/bmc`, with no Kubernetes operator
  dependency.
- No changes to the `BMC` interface or existing behaviour.

### Non-Goals

- Changing the `BMC` interface or any method signatures.
- Moving `pkg/bmcutils` out of MO. It bridges `bmc` with MO's CRD types
  (`api/v1alpha1`) and belongs in MO.
- Moving MO's `api/v1alpha1` CRD types. MMO's dependency on MO for CRD types
  is correct and remains unchanged.
- Removing the `k8s.io/apimachinery` dependency from the new module.
  That is a separate, optional follow-up (see Alternatives).

## Proposal

### New repository

Create `github.com/ironcore-dev/bmc` with module path `github.com/ironcore-dev/bmc`.

### What moves

| Current location | New location |
|---|---|
| `github.com/ironcore-dev/metal-operator/bmc` | `github.com/ironcore-dev/bmc` |
| `github.com/ironcore-dev/metal-operator/bmc/mock/server` | `github.com/ironcore-dev/bmc/mock/server` |

The mock server is test infrastructure shared by both operators and belongs
with the library it tests.

### Dependencies of the new module

```
github.com/stmcginnis/gofish  — Redfish transport (already a direct dep of both MO and MMO at the same version)
k8s.io/apimachinery           — resource.Quantity in hardware info structs only; no controller-runtime or client-go
```

### Changes to metal-operator

- Remove the `bmc/` directory.
- Add `github.com/ironcore-dev/bmc` to `go.mod`.
- Update import paths in:
  - `cmd/main.go`
  - `internal/controller/bmc_controller.go`
  - `internal/controller/server_controller.go`
  - `internal/controller/endpoint_controller.go`
  - `internal/controller/suite_test.go`
  - `pkg/bmcutils/` (import path update only; no logic change)

### Changes to metal-maintenance-operator

- Replace all `github.com/ironcore-dev/metal-operator/bmc` imports with
  `github.com/ironcore-dev/bmc` across ~20 files (16 production files,
  4 test suite files, 2 hack tools).
- Replace `github.com/ironcore-dev/metal-operator/bmc/mock/server` test
  imports with `github.com/ironcore-dev/bmc/mock/server`.
- MMO's `go.mod` dependency on MO narrows to what it actually needs:
  `api/v1alpha1` CRD types and `pkg/bmcutils`. The pseudo-version pin
  can be replaced with a proper tagged release once MO cuts one.

### Migration sequence

1. Create the `ironcore-dev/bmc` repository and copy the package contents.
2. Tag an initial release (e.g. `v0.1.0`).
3. Update MO: remove `bmc/`, add dep, update imports, cut a new MO release.
4. Update MMO: replace `bmc` imports, update MO dep to the new tagged release.

Steps 3 and 4 should land in a single coordinated PR pair to avoid a window
where MO no longer exports the package but MMO still imports it from MO.

## Security Considerations

The `bmc` package handles BMC credentials (passwords, account management,
certificate-less TLS connections). Extracting it into a standalone module
makes no changes to credential handling, session management, or TLS
behaviour. The security model is identical to today.

The new repository will be under `ironcore-dev` and subject to the same
review and release policies as existing repos in the org.

## Testing Strategy

- All existing unit tests in `bmc/` move with the package and run in the
  new repo's CI.
- The Redfish mock server (`bmc/mock/server`) moves with the library;
  integration tests in both MO and MMO that use it will continue to work
  via the updated import path.
- Both MO and MMO CI pipelines must pass end-to-end after the migration
  before the old `bmc/` directory is removed from MO.
- No new tests are required — this is a structural change with no behaviour
  delta.

## Alternatives

### Keep the status quo

Continue with MMO depending on MO via a pseudo-version. This requires ongoing
manual coordination for every MO release that MMO needs, and the k8s version
skew will widen over time.

### Vendor the bmc package in MMO

Copy `bmc/` into MMO and maintain two diverging copies. Rejected: duplicated
vendor-specific logic is a maintenance burden and will drift.

### Use a Go workspace (`go.work`) for development

A workspace file eases local development across repos but does not resolve the
released module coupling. It would need to be paired with the extraction
anyway.

### Remove `resource.Quantity` to make the module fully Kubernetes-free

Replacing `resource.Quantity` with plain `int64` in hardware info structs
would remove the `k8s.io/apimachinery` dependency entirely and make `bmc` a
pure Redfish library. This is a minor breaking change for existing callers and
is better addressed as a follow-up once the extraction is complete.
