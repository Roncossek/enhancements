# GEP-0054: Worker Capabilities for Machine Image Selection

## Table of Contents

- [GEP-0054: Worker Capabilities for Machine Image Selection](#gep-0054-worker-capabilities-for-machine-image-selection)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Reserved Capability Names](#reserved-capability-names)
    - [Deriving Capability Requirements](#deriving-capability-requirements)
    - [Image Selection Algorithm](#image-selection-algorithm)
    - [Validation](#validation)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)

## Summary

Some Gardener features, like [in-place node updates](../0031-inplace-node-updates/README.md), and (planned) secure boot, only work when the **machine image, the machine type, and Gardener** all support them. The image- and machine-type side fits [GEP-0033](../0033-machine-image-capabilities/README.md)'s capability mechanism, but the *names* of these capabilities form a contract owned by Gardener.

This GEP reserves the prefix `gardener-` inside GEP-0033 capabilities for this family of features and offers an option for existing or future typed worker pool fields (e.g. `updateStrategy`) into the GEP-0033 selection algorithm. **Users keep configuring features via typed worker fields**; Gardener internally derives the matching capability requirements.

## Motivation

Today, when a user enables in-place updates on a worker pool, nothing checks that the selected machine image and machine type actually support them. The misconfiguration surfaces only at runtime. Avoiding it requires manually finding a machine-type / machine-image pair in the CloudProfile that is compatible with the worker pool configuration, and keeping that selection valid across maintenance updates. This means for now automatic OS update operations are off the table for in-place update pools.

This GEP delivers two benefits, both built on GEP-0033:

1. **Automatic image selection.** Gardener picks (and maintains) a machine image that is compatible with both the chosen machine type and the worker pool configuration.
2. **Admission-time validation.** Incompatible combinations of machine image, machine type, and worker pool are rejected up front instead of failing on the node.

The trade is explicit: **less complexity for the user, more for the implementation**.

### Goals

- Reserve the `gardener-` prefix inside `spec.machineCapabilities` for capability names whose keys and values are owned by Gardener.
- Extend GEP-0033's selection and validation algorithm to also satisfy capability requirements derived from typed worker pool fields.
- Reject incompatible combinations of machine image, machine type, and worker pool at admission time.

### Non-Goals

- Defining the full list of reserved capabilities up front — it grows with new features.
- Introducing a generic `capabilities` map on the worker pool API. Worker pools keep typed fields; requirements are derived internally.
- Changing the GEP-0033 mechanism. Reserved capabilities behave like any other — only their **names and values** are owned by Gardener.
- Implementing the features themselves (e.g. wiring secure boot end-to-end is out of scope).

## Proposal

GEP-0033 is unchanged. This GEP adds two ingredients on top:

1. **A reserved namespace.** Capability names starting with `gardener-` are owned by Gardener. Operators must register them in `spec.machineCapabilities` using the keys and values Gardener defines; CloudProfile admission validates this. Anything *outside* `gardener-` is free for operators, exactly as today.
2. **A derivation step.** When a user configures a worker pool, Gardener internally translates relevant typed fields into capability requirements (e.g. `updateStrategy: AutoInPlaceUpdate` → `gardener-update-type: in-place`). These requirements feed the GEP-0033 matching algorithm alongside machine type and machine image capabilities.

The mechanism is **additive**: CloudProfiles and worker pools that don't use any reserved capability behave exactly as today.

**Key terms:**

- **Reserved capability** — capability whose name starts with `gardener-`; key and values defined by Gardener.
- **Capability requirement** — `(name, value)` pair derived internally from a worker pool's typed fields. Never written by users.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| An operator accidentally registers a `gardener-*` capability with values that don't match Gardener's authoritative definition | CloudProfile admission rejects the registration |
| Reserved-capability definitions drift between a Gardener release and a CloudProfile maintained by an operator | CloudProfile admission validates registered values against the Gardener version in use; mismatches surface immediately rather than at runtime |
| Existing CloudProfiles or worker pools break | The mechanism is additive — CloudProfiles and worker pools that don't use any reserved capability behave exactly as today |

## Design Details

### Reserved Capability Names

Reserved names and values are defined as constants in Gardener core. The two driving examples:

```go
const (
    // gardener-update-type — drives in-place node updates (GEP-0031).
    GardenerCapabilityUpdateType        = "gardener-update-type"
    GardenerCapabilityUpdateTypeRolling = "rolling"
    GardenerCapabilityUpdateTypeInPlace = "in-place"

    // gardener-boot-type — drives secure boot (planned).
    GardenerCapabilityBootType         = "gardener-boot-type"
    GardenerCapabilityBootTypeStandard = "standard"
    GardenerCapabilityBootTypeSecure   = "secure"
)
```

CloudProfile admission validates that any `gardener-*` capability registered in `spec.machineCapabilities` matches Gardener's authoritative definition.

### Deriving Capability Requirements

The mapping is part of the image selection process in shoot admission, the maintenance controller and provider extension:

```go
func deriveWorkerCapabilityRequirements(worker core.Worker) map[string][]string {
    requirements := map[string][]string{}

    // updateStrategy is defaulted earlier (default: rolling).
    switch ptr.Deref(worker.UpdateStrategy, core.UpdateStrategyRollingUpdate) {
    case core.UpdateStrategyAutoInPlace, core.UpdateStrategyManualInPlace:
        requirements[GardenerCapabilityUpdateType] = []string{GardenerCapabilityUpdateTypeInPlace}
    default:
        requirements[GardenerCapabilityUpdateType] = []string{GardenerCapabilityUpdateTypeRolling}
    }

    // Planned: worker.Machine.SecureBoot.
    if worker.Machine.SecureBoot != nil && *worker.Machine.SecureBoot {
        requirements[GardenerCapabilityBootType] = []string{GardenerCapabilityBootTypeSecure}
    } else {
        requirements[GardenerCapabilityBootType] = []string{GardenerCapabilityBootTypeStandard}
    }

    return requirements
}
```

Only typed worker pool fields drive requirements — users of the shoot resource never write capability names.

### Image Selection Algorithm

GEP-0033 introduced a selection algorithm based on capability compatibility:

```go
AreCapabilitiesCompatible(imageFlavor, machineType, capabilityDefinitions)
```

This GEP adds a third input:

```go
AreCapabilitiesCompatible(imageFlavor, machineType, workerRequirements, capabilityDefinitions)
```

The algorithm succeeds if, for every capability defined in `spec.machineCapabilities`, the value sets from image flavor, machine type, and worker requirements have a non-empty intersection (with image- and machine-type-side defaulting per GEP-0033).

The same algorithm is invoked from three existing call sites:

1. [Shoot validator admission](https://github.com/gardener/gardener/blob/e6263d6a575e4181f0289345803ccb59117605f6/plugin/pkg/shoot/validator/admission.go#L1000) — rejects user-selected machine image / machine type pairs that are incompatible with the worker pool's capability requirements.
2. [Shoot mutator admission](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/plugin/pkg/shoot/mutator/admission.go#L559) — picks a default machine image compatible with the chosen machine type and worker pool when the user does not specify one.
3. [Maintenance controller](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/pkg/controllermanager/controller/shoot/maintenance/helper/helper.go#L27) — chooses a machine image compatible with the worker pool's machine type and capability requirements during automatic updates.
4. Worker controller in provider extensions.

### Validation

| Concern | Validated by |
|---|---|
| `gardener-*` capability registrations match Gardener's authoritative list | gardener-apiserver CloudProfile admission |
| Worker pool's capability requirements satisfied by the selected machine image and machine type | gardener-apiserver shoot admission |
| Maintenance image selection respects all requirements | maintenance controller |

## Drawbacks

- **Internal complexity grows.** Selection gains a third input and a derivation step. The user-side simplification is the explicit trade.
- **Operator setup per CloudProfile.** Operators who want a reserved feature must register the capability in `spec.machineCapabilities` and declare per-image-flavor support. CloudProfiles that expose no reserved features are unaffected.
- **Coupling between Gardener releases and CloudProfile content.** Reserved definitions are owned by Gardener; operators must keep CloudProfiles in sync with the version they run. CloudProfile admission catches mismatches.

## Alternatives

- **Generic `capabilities` map on the worker pool API.** Most consistent with GEP-0033, but rejected for three reasons: (1) it forces users to learn the CloudProfile's capability vocabulary to configure standard features; (2) most capabilities are infrastructure-level concerns that are irrelevant to shoot users (e.g. hypervisor type, or which bare-metal machine type works with which image) — exposing them on the worker pool API would surface implementation detail with no user benefit; (3) it leaks an internal contract into operator-facing API.
- **No reserved namespace.** Hard-code names only in Gardener code. Lets operators accidentally redefine them with incompatible values. Rejected — the `gardener-` prefix plus admission validation prevents this.
- **Implicit reserved capabilities (no `spec.machineCapabilities` entry).** Inconsistent with GEP-0033, which declares every capability there. Rejected — operators register reserved capabilities like any other.
- **Provider-extension-owned capabilities in this GEP.** Earlier drafts let provider extensions own a `gardener-<provider>-` sub-namespace and derive capability requirements from typed `WorkerConfig` fields (e.g. OpenStack trusted launch). This was deferred because `WorkerConfig` is opaque to gardener-apiserver, the maintenance controller, and the Dashboard — only the extension can decode it — which makes the mapping mechanism a substantial design problem on its own. Solving it is independent of the core mechanism this GEP introduces.
