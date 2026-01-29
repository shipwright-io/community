<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: override-strategy-resources-in-build-and-buildruns
authors:
  - "@anchi205"
reviewers:
  - "@adambkaplan"
  - "@sayan-biswas"
approvers:
  - TBD
creation-date: 2026-01-23
last-updated: 2026-01-29
status: provisional
see-also:
replaces: []
superseded-by: []

---

# SHIP-0046 Override Strategy Step Resources in Build and BuildRuns

## Release Signoff Checklist

- [✔] Enhancement is `implementable`
- [✔] Design details are appropriately documented from clear requirements
- [✔] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

1. How should we make use of Tekton's computeResources API or Kubernetes pod resources? Should these be an implementation detail, or a separate feature?

2. Should we add features to the shp CLI, or leave this as a pure "YAML-only" developer experience?

3. Do we need to add anything to the operator? For example, does our implementation logic need to change depending on the version of Tekton or Kubernetes is deployed?

## Summary

This feature introduces a `stepResources` field in the Build and BuildRun APIs, enabling
developers to override CPU, memory, or ephemeral storage requirements for specific steps
defined in a BuildStrategy or ClusterBuildStrategy, without needing to duplicate and modify
the strategy itself.

Currently, resource requirements are hardcoded at the strategy level, for which either we
need to duplicate strategies for different resource needs or request platform admins to
modify shared strategies.

With this change, users can specify per-step resource overrides directly in their Build or
BuildRun specs, and the controller will merge these overrides at runtime when generating
the Tekton TaskRun with BuildRun overrides taking precedence over Build overrides, which in
turn override strategy defaults.

This follows the existing Shipwright override patterns for parameters, volumes, and env,
maintaining API consistency while enabling fine-grained, per-build resource customization.

## Motivation

Currently, CPU and memory resources for build steps can only be defined at the BuildStrategy or ClusterBuildStrategy level, meaning all builds using a strategy inherit the same fixed resource allocations. So when a developer needs more memory for a larger container image build, or they require additional CPU, they must either create a duplicate copy of the entire strategy just to change resource values, or accept that their build may fail due to insufficient resources. 

Developers should be able to customize resource requirements for their specific builds without touching the strategies. This feature bridges that gap by allowing per-build resource overrides, giving developers the flexibility they need while preserving the simplicity and maintainability of the shared build strategies.

### Goals

- Users can specify step-level resource overrides in Build spec and BuildRun spec, without creating duplicate BuildStrategies or requesting changes to shared ClusterBuildStrategies.
- When resources are specified at multiple levels, BuildRun overrides take precedence over Build overrides, which take precedence over strategy defaults.

### Non-Goals

- Modifying BuildStrategy or ClusterBuildStrategy resources - This feature allows overriding resources at Build/BuildRun level; it does not change how strategies define default resources.
- Using Tekton's task-level `computeResources` - We will not use Tekton's resource allocation where requests are divided among steps. [Follow-up feature]
- Using Tekton's `stepSpecs` override mechanism - We implement our own Shipwright-native override mechanism rather than passing through to Tekton's `stepSpecs`.
- Injecting organization-wide default resources at strategy deployment time is a separate concern for an operator layer, not Shipwright core.
- Validating whether requested resources fit within namespace `ResourceQuota` or `LimitRange` constraints is out of scope; Kubernetes handles this at pod creation time.
- Suggesting or automatically calculating optimal resources based on build history or image size is not part of this proposal.
- Using Kubernetes native pod-level resource specification.`Kubernetes v1.34 [beta]`(enabled by default). Resources are shared across all containers in pod. This requires PodLevelResources feature gate enabled; build steps run sequentially so no concurrent sharing benefit. No per-step control. [Follow-up feature]
- Overriding resources for Shipwright-managed supporting steps (Git clone, image-processing).
These steps are already configurable cluster-wide by administrators via container templates (`GIT_CONTAINER_TEMPLATE`, `IMAGE_PROCESSING_CONTAINER_TEMPLATE`).

## Proposal

### Implementation Notes

#### API changes

1. Add `StepResourceOverride` struct and `StepResources` field to `Strategy` struct in `pkg/apis/build/v1beta1/buildstrategy.go`:

```go
type StepResourceOverride struct {
    Name      string                      `json:"name"`      // Step name from strategy
    Resources corev1.ResourceRequirements `json:"resources"` // CPU/memory override
}

type Strategy struct {
    Name          string                 `json:"name"`
    Kind          *BuildStrategyKind     `json:"kind,omitempty"`
    StepResources []StepResourceOverride `json:"stepResources,omitempty"` // NEW
}
```

2. Update CRD schemas in `deploy/crds/shipwright.io_builds.yaml` and `deploy/crds/shipwright.io_buildruns.yaml`.

#### API Usage Examples

**Build with step resource override:**
```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
spec:
  strategy:
    name: buildah
    kind: ClusterBuildStrategy
    stepResources:
      - name: build
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
  # ...
```

**BuildRun override (takes precedence):**
```yaml
apiVersion: shipwright.io/v1beta1
kind: BuildRun
spec:
  build:
    name: my-build
    spec:
      strategy:
        stepResources:
          - name: build
            resources:
              requests:
                memory: "16Gi"   # Overrides Build value
```

How It Fits in the Broader Context?
- The `stepResources` field follows the established Shipwright pattern for overriding strategy-defined values (like paramValues, volumes, env) 

#### Implementation

Broadly, the steps will be:

1. Merge logic(Reconciler): When generating a TaskRun, the controller merges resources with the following precedence: BuildRun.stepResources > Build.stepResources > Strategy.steps[].resources.

2. Validation: Before creating the TaskRun, validate that step names exist in the referenced strategy.

3. TaskRun generation: Apply merged resources to `TaskRun.spec.taskSpec.steps[].computeResources`.

- Feature will work with both BuildStrategy and ClusterBuildStrategy.
- Invalid step names in `stepResources` will reject the Build with clear errors.
- Existing builds without stepResources continue to work unchanged (The feature is backward compatible). Builds that do not specify `stepResources` continue to use strategy-defined defaults with no behavioral change.
- The Tekton TaskRun created by the controller will contain the merged resource values in `spec.taskSpec.steps[].computeResources`.

### Test Plan

#### Unit Tests 

- Test merge logic to verify correct precedence (BuildRun > Build > Strategy defaults)
- Test validation to ensure invalid step names are rejected with appropriate error messages
- Test TaskRun generation correctly applies merged resources to steps
- Test edge cases: empty `stepResources`, partial overrides (non-overridden steps retain strategy defaults), nil values, multiple resource types (CPU, memory, ephemeral-storage)

#### Integration Tests 
- Create a Build with `stepResources`, trigger a BuildRun, verify the generated TaskRun contains correct resource values
- Test with both `BuildStrategy` and `ClusterBuildStrategy` to ensure feature works for both

#### E2E Tests
- Create Build with `stepResources` overrides, run BuildRun, verify build completes successfully
- Create Build with overrides, then BuildRun with different overrides, verify build completes with BuildRun values applied

### Release Criteria

TBD

### Risks and Mitigations

No new permissions granted. Feature only adds override capability so that users can override default resources as per their requirement. Users requesting excessive resources → Can be mitigated by Kubernetes `ResourceQuota` and `LimitRange`.

## Drawbacks

- `stepResources` will be placed under `spec.strategy` while similar override fields (`paramValues`, `volumes`, `env`) are at the `spec` level. This inconsistency may confuse users.
- Allowing arbitrary overrides may lead to builds failing in unexpected ways (e.g., setting memory too low causes OOMkilled, or too high wastes cluster resources).
- Adds a new field to the Build and BuildRun APIs, increasing complexity users need to understand, like where resources come from (Strategy vs Build vs BuildRun) and how precedence works in this case.

## Alternatives

- Pass step resource overrides directly to Tekton's `TaskRun.spec.taskSpec.stepSpecs[].computeResources` instead of implementing Shipwright-native abstraction. Trade-off: Shipwright's abstraction is broken and creates inconsistency.
- Set resources at task level and let Tekton distribute among steps. Trade-off: Resource requests are divided, so no per-step control.
- Users create their own namespace-scoped `BuildStrategy` with custom resources. Trade-off: It is cumbersome as user needs to duplicate the BuildStrategy to add their custom resource requirements.
- Inject default resources into ClusterBuildStrategies at deployment time via operator. Trade-off: Only sets global defaults, no per-build control.
- Use `LimitRange` to set default resources across namespace. Trade-off: Applies to entire namespace, not build-specific.

## Implementation History

TBD