<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: buildrun-executor-field
authors:
  - "@hasan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-12-19
last-updated: 2024-12-19
status: implemented
see-also:
  - "/ships/0023-surface-results-in-buildruns.md"
  - "/ships/0024-surfacing-errors-to-buildrun.md"
replaces: []
superseded-by: []
---

# BuildRun Executor Field

This enhancement proposes adding a new `executor` field to the `BuildRunStatus` to track which resource (TaskRun or PipelineRun) is responsible for executing a specific BuildRun.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

TBD

## Summary

This enhancement adds a new `executor` field to the `BuildRunStatus` that provides visibility into which underlying Tekton resource (TaskRun or PipelineRun) is responsible for executing a BuildRun. This field will contain the name and kind of the executor resource, enabling better debugging, monitoring, and observability of the build execution process.

## Motivation

This enhancement is motivated by the need to support PipelineRun as an alternative execution mode to TaskRun in Shipwright's build logic. As part of refactoring Shipwright to execute builds in Tekton PipelineRuns, cluster admins will be able to switch between the object type Shipwright creates to run builds. This is a necessary prerequisite for fanning out parallel multi-arch builds.

The introduction of PipelineRun support as an "alpha" feature (disabled by default) ensures we don't break existing TaskRun-based behavior while allowing experimentation with the PipelineRun "mode" for operating builds. However, with multiple execution modes available, users and operators need visibility into which specific Tekton resource (TaskRun or PipelineRun) is actually executing their BuildRun to enable proper debugging, monitoring, and troubleshooting.

### Goals

- Provide clear visibility into which Tekton resource (TaskRun or PipelineRun) is executing a BuildRun
- Enable better debugging and troubleshooting capabilities for build execution issues
- Support monitoring and observability tools that need to correlate BuildRun status with underlying Tekton resources
- Maintain backward compatibility with existing BuildRun resources
- Replace the deprecated `TaskRunName` field with a more comprehensive `Executor` field that supports both TaskRun and PipelineRun execution modes

### Non-Goals

- Modifying the core build execution logic
- Changing how BuildRuns are created or managed
- Adding executor information to Build resources (only BuildRun status)

## Proposal

### User Stories

#### Story 1: Introduce Executor field for runner kinds 
As a developer, I want a new Executor field to be added to the BuildRunStatus to define the resource responsible for executing a build.

### Implementation Notes

The new `executor` field will be added to the `BuildRunStatus` struct with the following structure:

```go
type BuildRunStatus struct {
    // ... existing fields ...
    
    // TaskRunName is the name of the TaskRun responsible for executing this BuildRun.
    //
    // Deprecated: Use Executor instead to describe the taskrun.
    // +optional
    TaskRunName *string `json:"taskRunName,omitempty"`

    // Executor is the name and kind of the resource responsible for executing this BuildRun.
    //
    // +optional
    Executor *BuildExecutor `json:"executor,omitempty"`
    
    // ... other existing fields ...
}

// BuildExecutor defines the name and kind of the build runner.
type BuildExecutor struct {
    // Name is the name of the TaskRun or PipelineRun that was created to execute this BuildRun
    Name string `json:"name"`
    
    // Group is the API group of the object that was created to execute the BuildRun (e.g. "tekton.dev")
    Group string `json:"group"`
    
    // Kind is the kind of the object that was created to execute the BuildRun (e.g., "TaskRun", "PipelineRun")
    Kind string `json:"kind"`
}
```

The field will be populated by the Shipwright controller when it creates the underlying Tekton resource. The controller will set this field immediately after creating the TaskRun or PipelineRun, ensuring that the executor information is available throughout the build lifecycle. This will be particularly important when the alpha PipelineRun feature is enabled, as users will need to distinguish between the two execution modes.

**Note**: The existing `TaskRunName` field in `BuildRunStatus` is now deprecated in favor of the new `Executor` field, which provides more comprehensive information about the execution resource.

#### Example

Here's an example of how the new `executor` field will appear in a BuildRun status:

```yaml
apiVersion: shipwright.io/v1beta1
kind: BuildRun
metadata:
  name: buildah-golang-buildrun
  namespace: default
spec:
  build:
    name: buildah-golang-build
status:
  # ... other status fields ...
  executor:
    name: buildah-golang-buildrun-ml6wg
    kind: TaskRun
  # ... other status fields ...
```

When viewed with `kubectl describe`, the executor information will be displayed as:

```
Status:
  # ... other status fields ...
  Executor:
    Name:         buildah-golang-buildrun-ml6wg
    Kind:         TaskRun
  # ... other status fields ...
```

### Test Plan

- Unit tests for the new Executor field
- Integration tests to verify the executor field is populated correctly for both TaskRun and PipelineRun execution paths (including alpha PipelineRun feature)
- End-to-end tests to ensure the field is available in the BuildRun status throughout the build lifecycle
- Tests to verify backward compatibility with existing BuildRuns that don't have the executor field

### Release Criteria

#### Removing a deprecated feature
Not applicable - this is a new field addition.

#### Upgrade Strategy
This is a purely additive change with no breaking modifications:
- Existing BuildRuns will continue to work without the executor field
- New BuildRuns will have the executor field populated automatically
- No migration is required for existing resources

### Risks and Mitigations

**Risk**: The executor field might become stale if the underlying Tekton resource is deleted.
**Mitigation**: The field should be treated as informational and not relied upon for critical operations. Users should always check the actual Tekton resource status.

**Risk**: Additional API surface area increases complexity.
**Mitigation**: The field is optional and backward compatible, minimizing impact on existing integrations.

**Risk**: Performance impact from additional field updates.
**Mitigation**: The field is set once during resource creation and not updated frequently, minimizing performance impact.

## Drawbacks

- Adds complexity to the BuildRunStatus API
- Creates a dependency on the underlying Tekton resource lifecycle
- May encourage tight coupling between BuildRun and Tekton resource management

## Alternatives

1. **Status Conditions Approach**: Instead of a dedicated field, add the executor information as a status condition. However, this would make the information less discoverable and harder to query.

2. **Annotations Approach**: Store executor information in annotations rather than the status field. This would be less structured and harder to validate.

3. **Separate Resource**: Create a separate resource to track executor relationships. This would add unnecessary complexity and resource overhead.

## Infrastructure Needed

None - this enhancement only requires changes to the existing Shipwright controller and API definitions.

## Implementation History

- Work is done in https://github.com/shipwright-io/build/pull/1956
