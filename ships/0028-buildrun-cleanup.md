<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: buildrun-cleanup
authors:
  - @SaschaSchwarze0
reviewers:
  - @GabeMontero
  - @qu1queee
  - @raghavbhatnagar96
approvers:
  - @GabeMontero
  - @qu1queee
creation-date: 2022-02-14
last-updated: 2022-03-02
status: implementable
---

# BuildStrategy Parameter Enhancements

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

None

## Summary

This ship proposes means for Build users to define that their completed BuildRuns should be automatically cleaned up after some time or when a certain number of BuildRuns is reached.

## Motivation

In Shipwright, we separated the definition of a build and the run of a build into two artifacts, the Build and BuildRun. The BuildRun by its nature is a run-to-completion workload. Once it is completed, its only purpose is to view its status. Retaining the BuildRun for some time makes sense. In case of a succeeded BuildRun, one finds there the digest of the image. For a failed BuildRun, one finds the error details and a pointer to the pod logs. The lifecycle of the BuildRun is coupled with the TaskRun and Pod using [owner references](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/).

Over times, the number of BuildRuns in the system grows. This consumes resources in Kubernetes' database (usually etcd) and on the nodes (the pod logs).

Eventually, users must manually clean up old BuildRuns which is not nice. Providing them means to define that BuildRuns get automatically cleaned up simplifies their lives.

### Goals

- Allow Build user to define how many completed BuildRuns should be kept.
- Allow Build user to define how long a completed BuildRun should be kept.
- Allow Build user to separate succeeded and failed BuildRuns in their cleanup handling.

### Non-Goals

- Allowing BuildRuns to be archived to an external database.

## Proposal

### API Changes

We will extend the Build resource with an additional optional `retention` section`:

```yaml
spec:
  retention:
    ttlAfterFailed: duration
    ttlAfterSucceeded: duration
    succeededLimit: uint
    failedLimit: uint
```

All fields inside the `retention` section are optional.

The `ttlAfterFailed` and `ttlAfterSucceeded` fields holds a duration. They define the duration on how long a BuildRun should live after it failed, or succeeded.

A succeeded BuildRun is a BuildRun where the `type=Succeeded` condition is set to `"true"`. A failed BuildRun has the `type=Succeeded` condition is set to `"false"`, and contains therefore for example canceled BuildRuns as well.

The two TTL fields behaves the same as the [`ttlSecondsAfterFinished` field for Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/). I decided against naming consistency with that feature, in favor of consistency inside our resource. Builds and BuildRuns already have a `timeout` field where we also use a duration while Kubernetes Jobs use an integer to store just seconds. Durations are easier to specify for users because they can simply write something like `6h` and don't need to calculate the amount of seconds.

The `succeededLimit` and `failedLimit` fields hold a positive integer number. They define how many succeeded and failed BuildRuns for a Build can exist.

If both TTL and limit are defined, then the BuildRun will get deleted once the first criteria is met.

The BuildRun resource will be extended with a `retention` section with only the `ttlAfterFailed`, and `ttlAfterSucceeded` fields. They override the values of the Build. This is done for consistency (the `timeout` can also be overwritten in the BuildRun) and in support of standalone BuildRuns that we may have in the future.

Updates to TTL or limit fields in the Build after a BuildRun was created/completed will have the following behavior:

- If a TTL field is updated, then no changes are made to BuildRuns. A TTL therefore applies to an individual BuildRun. This is consistent with timeouts that are also only considered at BuildRun creation.
- If a limit field is updated, then this new value applies. The values in the BuildRun's `status.buildSpec.retention.*Limit` are therefore obsolete. A limit is applicable to all BuildRuns of a Build in a namespace. Handling this differently is not possible as there can only be a single limit for them.

This means that TTL and limit behave slightly different which needs to be explained in the documentation.

### Backend

We will introduce new controllers for the BuildRun cleanup inside the same manager. The reasons for this are that our BuildRun controller is already highly complex - both in its predicate functions as well as in its `Reconcile` function - as it must handle TaskRuns and BuildRuns in various possible state combinations. There is one simple scenario where it simply does nothing: when the BuildRun is already completed. Instead of adding additional complexity for completed BuildRuns, the new controllers will focus exclusively on those aspects.

We will introduce a new `buildrun-ttl-cleanup-controller` which will reconcile completed BuildRuns. It will watch BuildRuns with those predicate functions:

- Create: if the BuildRun is completed (`status` of condition `type=Succeeded` set to `"true"` or `"false"`), and if a applicable TTL is set. This is handled for managers becoming the leader.
- Update: if the BuildRun completed (`status` of condition `type=Succeeded` is changed from `"unknown"` to `"true"` or `"false"`), and if an applicable TTL is set.

In the `Reconcile` function, we will check the BuildRun: When the `ttlAfterFailed` field is set for a BuildRun that failed, or when the `ttlAfterSucceeded` field is set for a BuildRun that succeeded, then we will calculate `.status.conditions[type=Succeeded].lastTransitionTime + TTL - now`. If the value is smaller than or equal zero, then we will delete the BuildRun. Otherwise, we will return a reconciliation result where `Requeue` is set to `true` and `RequeueAfter` is set to the calculated duration.

We will introduce a new `build-limit-cleanup-controller` which will reconcile Builds. It will watch Builds with those predicate functions:

- Create: if the Build has limits configured. This is handled for managers becoming the leader.
- Update: if the Build has limits introduced, or changed to lower values.

It will watch BuildRuns with those predicate functions:

- Update: if the BuildRun completed (`status` of condition `type=Succeeded` is changed from `"unknown"` to `"true"` or `"false"`), and if it is related to a Build (standalone BuildRuns when introduced with SHIP 0029 will be ignored).

In the `Reconcile` function, we will check the Build: if it has any limit set, then

- We will run a list for BuildRuns in the namespace with the label filter `build.shipwright.io/name: <BUILD_NAME>`.
- All BuildRuns will be put into three buckets: succeeded (`status` of condition `type=Succeeded` set to `"true"`), failed (`status` of condition `type=Succeeded` set to `"false"`) and the rest.
- If the number of succeeded BuildRuns is larger than `succeededLimit`, then the oldest succeeded BuildRun(s) sorted by `.metadata.creationTimestamp` will be deleted.
- If the number of failed BuildRuns is larger than `failedLimit`, then the oldest failed BuildRun(s) sorted by `.metadata.creationTimestamp` will be deleted.

### Staging

The implementation can be done in stages. In particular, the TTL-based cleanup and the limit-based cleanup are very much independent.

### Documentation updates

We will need to update the documentation about builds to introduce this new feature.

### User Stories

#### As a Build user, I want to define how many finished BuildRuns I want to keep for my Build so that the amount of finished BuildRuns does not grow endlessly

#### As a Build user, I want to define how long a finished BuildRun should stay so that I do not have obsolete and very old BuildRuns in the system

### Test Plan

It should include:

- Integration tests
- Unit tests
- e2e tests

### Release Criteria

Should be available before release v1.0.0

### Risks and Mitigations

We will have more controllers and reconcilers. Consumers may have to increase memory limits in the deployment.

For the support of limits, especially the required `list` calls can become a problem if `succeededLimit`, or `failedLimit` are set to high values and we have to retrieve a large amount of BuildRuns from the API server and have them in memory. A mitigation would be that we define maximum values for those values (maybe 100).

## Drawbacks

None.

## Alternatives

For the use case:

- We can help users who manually cleanup BuildRuns by providing CLI commands, for example to delete all BuildRuns that are older than some provided time.
- We can provide samples to periodically cleanup BuildRuns, for example using [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

For the implementation:

- Instead of adding new controllers, we could have extended the existing one.

Extensions:

- User may want to mark a BuildRun as not applicable for cleanup. For example through an annotation. The scenario is that there is a run one wants to keep for debugging purpose. We can consider this once somebody asks for it.

## Implementation History

None
