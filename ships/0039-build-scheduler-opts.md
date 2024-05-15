<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-scheduler-options
authors:
  - "@adambkaplan"
reviewers:
  - "@apoorvajagtap"
  - "@HeavyWombat"
approvers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
creation-date: 2024-05-15
last-updated: 2024-05-15
status: provisional
see-also: []
replaces: []
superseded-by: []
---

# Build Scheduler Options

<!-->

This is the title of the enhancement. Keep it simple and descriptive. A good title can help
communicate what the enhancement is and should be considered as part of any review.

The YAML `title` should be lowercased and spaces/punctuation should be replaced with `-`.

To get started with this template:

1. **Make a copy of this template.** Copy this template into the main
   `proposals` directory, with a filename like `NNNN-neat-enhancement-idea.md`
   where `NNNN` is an incrementing number associated with this SHIP.
2. **Fill out the "overview" sections.** This includes the Summary and Motivation sections. These
   should be easy and explain why the community should desire this enhancement.
3. **Create a PR.** Assign it to folks with expertise in that domain to help
   sponsor the process. The PR title should be like "SHIP-NNNN: Neat
   Enhancement Idea", where "NNNN" is the number associated with this SHIP.
4. **Merge at each milestone.** Merge when the design is able to transition to a new status
   (provisional, implementable, implemented, etc.). View anything marked as `provisional` as an idea
   worth exploring in the future, but not accepted as ready to execute. Anything marked as
   `implementable` should clearly communicate how an enhancement is coded up and delivered. Aim for
   single topic PRs to keep discussions focused. If you disagree with what is already in a document,
   open a new PR with suggested changes.

The `Metadata` section above is intended to support the creation of tooling around the enhancement
process.

<-->

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

Add API options that influece where `BuildRun` pods are scheduled on Kubernetes. This can be
acomplished through the following mechanisms:

- [Node Selectors](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)
- [Affinity/anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

## Motivation

Today, `BuildRun` pods will run on arbitrary nodes - developers, platform engineers, and admins do
not have the ability to control where a specific build pod will be scheduled. Teams may have
several motivations for controlling where a build pod is scheduled:

- Builds can be CPU/memory/storage intensive. Scheduling on larger worker nodes with additional
  memory or compute can help ensure builds succeed.
- Clusters may have mutiple worker node architectures and even OS (Windows nodes). Container images
  are by their nature specific to the OS and CPU architecture, and default to the host operating
  system and architecture. Builds may need to specify OS and architecture through node selectors.
- Left unchecked, builds may congregate on a set of nodes, impacting overall cluster utilization
  and stability.

### Goals

- Allow build pods to run on specific nodes using node selectors.
- Allow build pods to set node affinity/anti-affinity rules.
- Allow build pods to tolerate node taints.
- Allow node selection, pod affinity, and taint toleration to be set at the cluster level.

### Non-Goals

- Primary feature support for multi-arch builds.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories [optional]

#### Node Selection - platform engineer

As a platform engineer, I want builds to use node selectors to ensure they are scheduled on nodes
optimized for builds so that builds are more likely to succeed

#### Node Selection - arch-specific images

As a developer, I want to select the OS and architecture of my build's node so that I can run
builds on worker nodes with multiple architectures.

#### Pod affinity - platform engineer/admin

As a platform engineer/cluster admin, I want to set anti-affinity rules for build pods so that
running builds are not scheduled/clustered on the same node.

#### Taint toleration - cluster admin

As a cluster admin, I want builds to be able to tolerate provided node taints so that they can
be scheduled on nodes that are not suitable/designated for application workloads.

### Implementation Notes

TBD

<!-->
**Note:** *Section not required until feature is ready to be marked 'implementable'.*

Describe in detail what you propose to change. Be specific as to how you intend to implement this
feature. If you plan to introduce a new API field, provide examples of how the new API will fit in
the broader context and how end users could invoke the new behavior.
<-->

### Test Plan

TBD

<!-->
**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:

- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as
tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).
<-->

### Release Criteria

TBD

**Note:** *Section not required until targeted at a release.*

#### Removing a deprecated feature [if necessary]

Not applicable.

#### Upgrade Strategy [if necessary]

<!-->

If applicable, how will the component be upgraded? Make sure this is in the test
plan.

Consider the following in developing an upgrade strategy for this enhancement:

- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to make use of the enhancement?
<-->

### Risks and Mitigations

TBD

<!-->
What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
both security and how this will impact the larger Shipwright ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?
<-->

## Drawbacks

TBD - The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

TBD

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
possible approaches to delivering the value proposed by an enhancement.

## Infrastructure Needed [optional]

No additional infrastructure antipated.
Test KinD clusters may need to deploy with additional nodes where these features can be verified.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation History`.


