<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: surfacing-errors-to-buildrun
authors:
  - "@dalbar"
reviewers:
  - "@qu1queee"
  - "@HeavyWombat"
approvers:
  - "@adambkaplan"
  - "@sbose78"
creation-date: 2021-09-10
last-updated: 2021-09-10
status: draft

---

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This ship provides a mechanism to surface errors in the build process
into BuildRun objects granting more visibility into
without being Tekton/Implementation specific. To
achieve this goal, we introduce two new generic results
`shp-error-reason` and `shp-error-message`, that a TaskRun can emit
and a BuildRun can pick up. 


## Motivation
A strategy's container logs differ for their steps and utilized
tooling. Therefore, finding an error message and extracting the error's reason is challenging for machine processing and consumption from
third parties due to the logs' heterogeneous nature and the need to parse
`stdout`. 
For a better user experience, we propose that a strategy author
can define the failure reason in a compact way that can
be represented in the BuildRun status, for example, through TaskRun results.
Build strategy authors are already aware of this tooling as they can emit the size
digest of the output image similarly, see [the BuildKit strategy](https://github.com/shipwright-io/build/blob/main/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml#L65).

Additionally, the proposal enables a user to keep the troubleshooting on the BuildRun level and only descend further into Tekton
specifics when necessary.

### Goals

- Allow build strategy users to define error details that will show up on the BuildRun status.
- Allow our steps like the git step to also use this mechanism to provide additional error details in the BuildRun status

### Non-Goals

- The specifics of errors regarding tooling are not part of this proposal. Neither is their means for reporting them. Errors highly
  depend on each strategy

## Proposal

We propose to introduce two generic results for errors:
`shp-error-message` and `shp-error-reason`.  The result
`shp-error-reason` has a machine-recognizable value that is ideally prefixed
with the problematic technology, for example, `git` in case of missing
a secret. The result `shp-error-message` should be descriptive text to
give the user enough information to troubleshoot the BuildRun.  A task
run emits the new results as Tekton results.  
Finally, we must extend the BuildRun controller
logic to propagate the new results into the BuildRun CRD's
`status.failureDetails` property. 
Moreover, there exists a pointer to the failed container and pod under `status.failedAt` to give
a user hints for debugging his problem:

```go
 type FailedAt struct {                                                                                                                          
           Pod       string `json:"pod,omitempty"`
           Container string `json:"container,omitempty"`
   }
```

However, we are worried about the sprawl of our API, and having two failure-related pointers only makes that sprawl worse.
Our proposal for error results is yet another pointer to the underlying reason why a build has failed.
Therefore, we propose to merge the `status.failedAt` property into `status.errorDetails` to achieve a semantic grouping.


BuildRun with new error results and merged `status.failedAt`:

```yaml
status:
    failure:
      reason: git-auth-ssh
      message: Authentication failed for ...
      location:
        pod: aPod
        container: aContainer
```

We do not suggest mutating the `status.conditions.message` field
since it contains valuable hints to troubleshoot on a lower level if
the propagated error message and reason is not sufficient.

### User Stories

#### Story 1 - Developer and direct user of shipwright perspective
As a user of Shipwright and developer, I want to identify why my
BuildRun is in a failed state without going through the lengthy
process of identifying the correct container and manually processing
its logs.
#### Story 2 - Github Actions developer (third party application developer)
As a third-party application developer that uses shipwright under the hood, I want to supply my users with meaningful hints to fix their builds, thus requiring a straightforward API to consume error details.

### Implementation Notes

`tbd in the next phase of the proposal`

### Test Plan

`tbd in the next phase of the proposal`

### Release Criteria

`tbd in the next phase of the proposal`

### Risks and Mitigations
Tekton results are limited in size (4096 bytes) which applies to the
proposed error results (see [code ref](https://github.com/kubernetes/kubernetes/blob/96e13de777a9eb57f87889072b68ac40467209ac/pkg/kubelet/container/runtime.go#L632)).
We should document this limitation.

There is no strict typing that would prevent strategy authors from
using proper error reasons (value has to be a `string`). So two authors can use different names for
the same error reason or use the same name for different errors. To
prevent these issues, we need clear documentation and consistent error reporting in our strategies.

Ideally, this ship requires that TaskRuns emit results in error
cases. However, Tekton currently does not emit any results for failed
TaskRuns. Therefore, I have reached out to the Tekton community to
include that functionality
(https://github.com/tektoncd/pipeline/issues/3749).  For the case that
the request is denied, for example, because of Tekton's API design
philosophy, I have researched a workaround. In every TaskRun step's
status, there is a message property (JSON string), including the step
results. We can extract the failed step and use it to access the correct message property.

Our proposal merges `status.failedAt` and therefore has API breaking changes that third party tools rely on. To mitigate the risks for breaking those applications,
we propose to keep `status.failedAt`. With the release of the surfacing error results feature, we intend to deprecate it and educate our users with a proper migration
strategy that shall be added to our docs.

# Drawbacks
Since we only expose a field for an error's message and reason, we do
not allow strategy authors to introduce additional custom fields for
clarifying the build problem.

We assume that a BuildRun fails after the build throws an error, and
build steps are executed in sequence, thus making it sufficient to have a
single failure slot for a BuildRun.

If the container exits with a non-zero exit code, we ignore error results including the strategy author's contributions to errors.
To surface errors, the container strictly requires a completed TaskRun with failed reason.

## Alternatives
In the first of the alternatives, we used the `status.results` field to publish errors as introduced in ship `0023`.
The drawback is that results are commonly associated with success. Additionally, it makes looking up errors more inconvenient
since `results` is a list of key/value pairs.

Variant 1

```yaml
...
  status:
    results:
      - name: shp-error-reason
        value: git-ssh-auth
      - name: shp-error-message
        value: Authentication failed for ...
```

Having specific structs for individual errors can provide the user
with even more details. However, I did not find any compelling
examples with practical fields while looking into git-based
errors. The user most likely is interested in the error message and
some recognizable error reason (or exit code) only. I am open to
counterexamples.

Variant 2
```yaml
...
   status:
     shpErrors:
        - name: shp-source-git-error
          git:
            message: Authentication failed for ...
            reason: git-ssh-auth
            customFields: ...
```

Variant 3

The third variant is the same proposal for error details without merging `status.failedAt`. One drawback of this variant is that there would be two distinct pointers
to the same failure (sloppy API design-wise). The advantage on the other hand is that we do not introduce any breaking API changes. However, since we are in alpha, 
we can afford to introduce breaking API changes for the sake of a cleaner API design.

Variant 4

The fourth variant is adding error results to the conditions field of the `BuildRun` e.g:

```yaml
conditions:
  - lastTransitionTime: "2021-10-12T16:57:07Z"
    message: |
      "step-build-and-push" exited with code 1 (image: "icr.io/obs/codeengine/buildkit/builder@sha256:a11e2348f9ee40822fc28dcb501c57cd02ebd31fb441841bfe5c144cc9d77fc6"); for logs run: kubectl -n baran-test logs apparmor-build-buildrun-gkxld-zrrtv-pod-8qpkt -c step-build-and-push
    reason: Failed
    status: "False"
    type: Succeeded
  - lastTransitionTime: "2021-10-12T16:57:07Z"
    message: |
      The key is invalid for the specified target. Please make sure that the remote source exists, you have sufficient rights and the key is in the right format.
    reason: GitAuthSSH
    status: "False"
    type: ImageBuildSucceded
```

It seems to be the most idiomatic approach in K8s world. Other controllers would be able to consume them without worrying about the specific CRD. Tools like `kubectl` could also use its existing features for conditions to react to failures e.g. wait. Based on the conventions for conditions there are some possible downsides.

Conventions From [community repo](https://github.com/kubernetes/community/blob/4f95b9210536ee62165e0a186ef5fab19a8be63f/contributors/devel/sig-architecture/api-conventions.md):

> Controllers should apply their conditions to a resource the first time they visit the resource, even if the status is Unknown. This allows other components in the system to know that the condition exists and the controller is making progress on reconciling that resource.

Conditions are always present in the status. I don't think it is too noisy, but there was no consensus on this. In fact, reviewers felt providing the failing condition at the top level of the status made it easier for the human user to parse and consume. Also, the distinction of failures being a terminal or ending condition, vs. other conditions being intermediate or not permanent in their outcome, contributed to the desire to isolate the failing condition from the other condition.  For example, during a build run, a given condition would be `Unknown` and for a successful build would change to `False` if it was the reason a build run failed.

> Condition type names should describe the current observed state of the resource, rather than describing the current state transitions. This typically means that the name should be an adjective ("Ready", "OutOfDisk") or a past-tense verb ("Succeeded", "Failed") rather than a present-tense verb ("Deploying"). Intermediate states may be indicated by setting the status of the condition to Unknown.

Good naming would be key to avoid confusion specifically for the `type` property of conditions. The example above includes the current condition of type `Succeded`. As discussed above, we are not mutating it anyway. It then requires us to come up with a new condition type that indicates a failed build. However, that condition has the same meaning as the overall `Succeeded` field. Additionally, it does not solve having two failure pointers in separate places (see `failedAt`).

## Implementation History

tbd
