<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: tekton-custom-task
authors:
  - "@imjasonh"
reviewers:
  - "@adambkaplan"
  - "@sbose78"
  - "@qu1queee"
approvers:
  - "@adambkaplan"
  - "@sbose78"
  - "@qu1queee"
creation-date: 2021-06-03
last-updated: 2021-06-03
status: provisional
---

# Tekton Custom Task

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

Implement a Tekton Custom Task controller that watches for Tekton `Run` objects that reference a Shipwright `Build` resource by name.
When such a `Run` object is created, create a `BuildRun` referencing the referenced `Build`, and as the `BuildRun` progresses, update the `Run` status accordingly.

## Motivation

Implementing a Custom Task controller has two main benefits:

1. This will allow us to integrate Shipwright with Tekton Triggers in a future change.
   The Custom Task is necessary here because Tekton Triggers is only designed to create Tekton resources, and not arbitrary non-Tekton resources such as a `BuildRun`.
1. This will allow users to integrate Shipwright Builds in their Tekton Pipelines.
   This would enable them to, for example, run unit tests before calling Shipwright to build a container image, or to test, scan or deploy an image produced by Shipwright.

Because Shipwright already necessarily requires Tekton to be installed, this change should not incur any additional dependencies on users or operators.

### Goals

A Shipwright installation should be able to watch and react to Tekton `Run` objects by creating `BuildRun`s, and update the `Run` status with the `BuildRun`'s status.

This behavior should be documented and demoable, and tested to guard against regressions.

### Non-Goals

Shipwright should not require or recommend any particular integration with a Tekton pipeline, only take advantage of features provided by Tekton to integrate with a pipeline.

## Proposal

### Implementation Notes

Add a reconciler to the Shipwright Build controller that watches for Tekton `Run` objects that reference a Shipwright `Build`.

The reconciler should handle timeouts and cancellations.
In the future, when BuildRuns support parameterization, the reconciler should also support `Run` parameters by passing them through.

The core implementation of this controller has been prototyped in https://github.com/imjasonh/build-task, using a reconciler based on knative/pkg.
If we want to integrate this into Shipwright's existing controller, we might want to port that to controller-runtime to remain consistent with Shipwright's existing implementation.

Tekton added support for `Run`s in [version 0.19.0](https://github.com/tektoncd/pipeline/releases/tag/v0.19.0), released December 2020.
As of the latest Tekton release, running Custom Tasks in a Pipeline requires opting in to the alpha feature gate, but Shipwright can enable the controller regardless.

### Test Plan

In addition to unit tests, add end-to-end tests that cover this behavior:

1. create a `Run` directly and check that a `BuildRun` is created; check that the `Run` status is updated when the `BuildRun` executes.
1. create a `PipelineRun` that references such a `Run`, and check that the `PipelineRun` progresses and creates a `BuildRun`.

### Release Criteria

**Note:** *Section not required until targeted at a release.*

#### Upgrade Strategy [if necessary]

Currently, Tekton Custom Task `Run` resources have the `v1alpha1` API version.
In the future, we should expect these to graduate to `v1beta1` and eventually `v1`.

The shape of the `Run` resource is fairly small and not expected to grow in breaking ways.

### Risks and Mitigations

Integrating with Custom Tasks requires the Shipwright controller to have permission to view and update the status of _all `Run` resources_, even those that don't reference Shipwright `Build`s.
This is due to limitations in Kubernetes RBAC -- there's no way to request access to only `Run` objects that reference `Build`s.
As a result, a bug or exploit of the Shipwright controller could lead to unexpected or malicious behavior in the user's cluster.

## Drawbacks

This integration presents a somewhat convoluted semi-circular dependency between Shipwright and Tekton.
A Tekton `PipelineRun` executes a `Run` that tells Shipwright to create a `BuildRun`, which executes by creating a Tekton `TaskRun` under the hood.

Because Custom Tasks are alpha, most Tekton pipeline visualization UIs don't have good support for showing the status of `Run`s, and this will extend to Shipwright builds in a Tekton pipeline execution.
By having more stable Custom Task implementations in the wild, this should encourage better support in these UIs.

## Alternatives

We could try to encourage Tekton Triggers to support creating arbitrary external types; they've been (rightfully, I think) resistant to this in the past.

## Infrastructure Needed [optional]

None.

## Implementation History

TBD
