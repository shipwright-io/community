<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-runtime-class
authors:
  - "@adambkaplan"
reviewers:
  - "@dorzel"
  - "@HeavyWombat"
approvers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
creation-date: 2025-03-25
last-updated: 2025-03-25
status: provisional
see-also:
  - "/ships/0039-build-scheduler-opts.md"  
replaces: []
superseded-by: []
---

# Build Runtime Class

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

- Should user namespaces also be in scope for this feature?
  _No, this should be considered separately. Feature is beta as of k8s v1.30_.

## Summary

Extend the `Build` and `BuildRun` APIs to let build pods select their [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
for execution.

## Motivation

The `RuntimeClass` API lets Kubernetes clusters support multiple container runtime environments.
This is most often encountered when using [Kata containers](https://katacontainers.io/), which adds
hardware virtualization to the existing mechanisms for isolating containers.

### Goals

- Allow builds to run with alternative container runtimes, such as Kata containers.

### Non-Goals

- Automatic mechanisms for selecting the container runtime.
- Network isolated builds.
- Isolation with Kubernetes [user namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/).

## Proposal

### User Stories 

#### Kata Containers

As a developer building containers with Shipwright I want to specify that the build pod containers
use Kata containers as their runtime so that I can isolate my builds with hardware virtualization.

### Implementation Notes

#### API Update

The `BuildSpec` API for Build and BuildRun will be updated to add the `runtimeClass` field:

```yaml
spec:
  ...
  runtimeClassName: kata # string
```

This field will correspond to the [runtimeClassName](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#scheduling)
field on Kubernetes pods.

#### Precedence Ordering and Value Merging

Values in `BuildRun` will override those in the referenced `Build` object (if present). This allows
the `BuildRun` object to "inherit" values from a parent `Build` object.

#### Impact on Tekton TaskRun

Tekton supports tuning the pod of the `TaskRun` using the
[podTemplate](https://tekton.dev/docs/pipelines/taskruns/#specifying-a-pod-template) field. When
Shipwright creates the `TaskRun` for a build, the respective `runtimeClassName` can be passed
through.

#### Command Line Enhancements

The `shp` CLI _may_ be enhanced to add flags that set the `runtimeClassName` for a `Build` and/or
`BuildRun`. For example, `shp build run` can have the following new options:

- `--runtime-class=<value>`: this would set the respective `spec.runtimeClassName` value on the
  generated `BuildRun`.

### Test Plan

- Unit testing can verify that the generated `TaskRun` object for a build contains the desired pod
  template fields.
- End to end testing may prove challenging, as KinD is not designed to support multiple container
  runtimes. Implementations like Kata Containers are designed for real clusters on cloud provider
  infrastructure.


### Release Criteria

**Note:** *Section not required until targeted at a release.*

#### Removing a deprecated feature [if necessary]

Not applicable.

#### Upgrade Strategy [if necessary]

The `runtimeClassName` field will be optional and default to Golang empty values.
On upgrade, these values will remain empty on existing `Build`/`BuildRun` objects.

### Risks and Mitigations

#### Pod Scheduling Collision

The `RuntimeClass` object (set and maintained by cluster admins/platform teams) also has options
to set defaults for [pod scheduling](https://kubernetes.io/docs/concepts/containers/runtime-class/#scheduling),
such as node selectors and tolerations. As of Kubernetes 1.32, this is a "beta" feature enabled
on most Kubernetes distributions by default. These scheduler options are merged on pod admission;
there is a risk that pods are rejected and builds fail due to colliding pod scheduler values.

In theory the controller for `BuildRuns` can anticipate this situation and fail the build quickly.
However, other Kubernetes controllers do not appear to do this, and thus it may not be worthwhile
to add this extra logic. Checking the `RuntimeClass` for every build also adds overhead to each
reconcile loop, either by making kubernetes apiserver calls directly or adding an informer-based
cache for `RuntimeClass` that provides little value.

## Drawbacks

This continues our leaking of Kubernetes pod APIs to Shipwright - see
[SHIP-0039](https://github.com/shipwright-io/community/blob/main/ships/0039-build-scheduler-opts.md#drawbacks).
Similar drawbacks also exist with respect to cluster admin control over pod scheduling.

## Alternatives

### Kubernetes User Namespaces

An initial draft of this proposal included Kubernetes
[user namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/) as part of
the scope. This was removed because user namespaces are a beta feature in Kubernetes (as of v1.32),
and did not reach the beta milestone until v1.30. Even with achievement of the beta milestone, the
feature is still disabled by default in Kubernetes v1.32, making it challenging to test.

## Infrastructure Needed [optional]

No additional infrastructure anticipated.


## Implementation History

- 2025-03-25: Created as `provisional`
