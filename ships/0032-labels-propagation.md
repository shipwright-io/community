<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: propagating-labels-to-the-pod
authors:
  - "@Pinolo"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-05-12
last-updated: 2022-05-15
status: -
see-also:
  - "/ships/0010-buildstrategy-annotation-propagation.md"
---

# Propagating labels to the pod

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

1. Do we want to allow all of `BuildStrategy`, `ClusterBuildStrategy`, `Build`, `BuildRun` to be able to define labels to be propagated to the Tekton `TaskRun` `Pod`?
1. In case of multiple allowed label sources, what is the hierarchy for overrides? (e.g. `ClusterBuildStartegy` or `BuildStrategy` > `Build` > `BuildRun`)
1. Which label prefixes do we want to deny-list?

## Summary

The labels set for a Shipwright build resource will be passed to the generated Tekton `TaskRun`, and from there to the corresponding `Pod`.

## Motivation

Labels are useful to administrators and developers, in order to be able to filter resources. An example use case is the need for getting logs of a specific group of `BuildRun`s, based on e.g. a label identifying a certain service.

### Goals

- Enable users to define labels on `ClusterBuildStrategy`,`BuildStrategy`, `Build`, `BuildRun`, that are copied to the `BuildRun`'s `Pod`

### Non-Goals

Propagation for annotations has aleady been implemented.

## Proposal

Shipwright resources administrators can define labels in the metadata, as with all Kubernetes objects, see the [Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) topic in the Kubernetes documentation.

As we did with annotations, we reserve certain label prefixes that are either used by Kubernetes or Tekton or Shipwright's controller:

* `?kubernetes.io`
* `?k8s.io`
* `tekton.dev`
* `*.shipwright.io`

When generating a Tekton `TaskRun`, the idea is to look at the labels of the `BuildStrategy` or `ClusterBuildStrategy` or `Build` or `BuildRun` and copy all labels over to the `TaskRun`, except those that use one of the reserved prefixes.

Valid labels found along the `strategy > build > run` hierarchy will be merged.

Tekton automatically copies all `TaskRun` labels to the `Pod`, see [pod.go](https://github.com/tektoncd/pipeline/blob/v0.35.1/pkg/pod/pod.go#L377).

For example, this metadata of a build:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  labels:
    somelabelkey: somelabelvalue
    custom.prefix.io/somelabelkey: somelabelvalue
    build.shipwright.io/somelabelkey: Value
```

will lead to the following metadata on the `TaskRun` (and `Pod`):

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  labels:
    somelabelkey: somelabelvalue
    custom.prefix.io/somelabelkey: somelabelvalue
```

### Implementation Notes

The implementation requires the [BuilderStrategy interface](../../pkg/apis/build/v1alpha1/buildstrategy.go) to be extended with a `GetLabels` functions that is implemented in the [BuildStrategy](../../pkg/apis/build/v1alpha1/buildstrategy_types.go) and [ClusterBuildStrategy](../../pkg/apis/build/v1alpha1/clusterbuildstrategy_types.go) types by returning the object's labels.

It also requires `GetLabels` functions to be implemented in the [Build](../../pkg/apis/build/v1alpha1/build_types.go) and [BuildRun](../../pkg/apis/build/v1alpha1/buildrun_types.go) types.

The assignment of the `TaskRun` labels needs to be done in the [generate_taskrun.go](../../pkg/reconciler/buildrun/resources/taskrun.go) file in the `GenerateTaskRun` function. The labels from the build strategy need to be copied to the `TaskRun` except those with reserved prefixes mentioned under [Proposal](#proposal).

Making sure the assignment takes place in the desired order (first labels from `BuilderStrategy`, then from `Build`, and finally from `BuildRun`) will allow cascading overrides.

### Test Plan

TBD

### Release Criteria

TBD

#### Upgrade Strategy [if necessary]

N/A

### Risks and Mitigations

N/A

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A

## Implementation History

N/A
