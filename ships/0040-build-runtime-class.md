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
- Isolation with Kubernetes [user namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/). _Should this be a goal as well?_

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

The reconcile loop for `BuildRun` could in theory catch this situation ahead of time and fail the
build prior to pod creation. We could also test this scenario and see how Tekton `TaskRuns` behave;
if the `TaskRun` fails with a reasonable error message, then Shipwright `BuildRun`s can simply echo
the information in their status.

## Drawbacks

This continues our leaking of Kubernetes pod APIs to Shipwright - see
[SHIP-0039](https://github.com/shipwright-io/community/blob/main/ships/0039-build-scheduler-opts.md#drawbacks).
Similar drawbacks also exist with respect to cluster admin control over pod scheduling.

## Alternatives

TBD

## Infrastructure Needed [optional]

No additional infrastructure anticipated.


## Implementation History

- 2025-03-25: Created as `provisional`
