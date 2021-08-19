<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: shipwright-image-api
authors:
  - "@imjasonh"
reviewers:
  - "@shbose78"
  - "@adambkaplan"
  - "@SaschaSchwarze0"
  - "@ricardomaraschini"
approvers:
  - "@adambkaplan"
  - "@SaschaSchwarze0"
creation-date: 2021-08-16
last-updated: 2021-08-16
status: provisional
see-also:
  - https://github.com/shipwright-io/community/issues/8
---

# Shipwright Image API

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

1. Is this a worthwhile and well-aligned addition to the existing Shipwright project scope?

1. Should Shipwright Image subproject release at the same cadence as the Shipwright Build subproject? (don't need to answer soon)

## Summary

Add a subproject to the Shipwright project that focuses on describing a set of general-purpose `Image` APIs.
This may include a registry-protocol proxy for in-cluster or out-of-cluster image registries, that updates `Image` resources when images change, and can use Kubernetes RBAC rules on `Image` resources to guard pushes/pulls/etc of proxied image resources.

This was previously briefly discussed at https://github.com/shipwright-io/community/issues/8

## Motivation

Shipwright's existing scope has been mainly about describing and executing `Build`s, which produce container image artifacts.
This has produced a set of APIs similar to OpenShift's Build API, among others, and has been generally well received.

While the Build API helps developers and operators reason about and manage executions of builds, it lacks a concept for the container image artifact itself.
This means that, though operators can use existing APIs to learn about images that were built by Shipwright, they can't do much to learn about other images that might exist in their platform.
With existing APIs they can also only reason about images in the context of their _builds_, and not as first-class resources.

OpenShift has had a similar concept for years, in [ImageStreams](https://docs.openshift.com/container-platform/4.8/openshift_images/images-understand.html#images-imagestream-use_images-understand) -- this proposal suggests a spiritual successor to this concept, separated from OpenShift, as a Shipwright subproject.

An OpenShift developer, @ricardomaraschini, has prototyped https://github.com/ricardomaraschini/tagger, which implements most of what's being proposed, and more.
He's interested in identifying a subset of this project that meets the Shiwpright community's needs and contributing that to a prospective Shipwright Image API subproject.

### Goals

Create a Shipwright subproject focused on designing and implementing Image-focused CRD APIs that work well with the Build APIs.

It should be possible to use Build APIs without Image APIs, and to use Image APIs without Build APIs, but to have the two work well together if both are being used.


### Non-Goals

This proposal doesn't suggest fully specifying the Image API from the outset, only to create organization repos to serve as a testbed for further exploration of these APIs.

The proposal should not include a full registry API implementation with storage; there are many in-cluster and out-of-cluster implementations in existence, and it's not a good use of our time to produce another one.

## Proposal

Spawn a subproject under the Shipwright umbrella to develop these new APIs, and ensure they work well with existing Shipwright Build APIs, CLI, and Operator efforts.

### User Stories

#### Story 1

As an operator, I want to be able to configure a Kubernetes `Deployment` to be updated automatically when an `Image` resource in my cluster changes, whether that's as the result of a `Build`, or some external event such as an external registry push.

#### Story 2

As an operator, I want to have a Shipwright API that can tell me the history of an image tag, and let me use this information to perform a rollback to a previous version of my software.


### Implementation Notes

None so far.

### Test Plan

As with the Build APIs today, the Image subproject should include thorough and high-quality unit tests, integration tests, and end-to-end tests.

### Release Criteria

Not currently targetting a release.

### Risks and Mitigations

Widening the scope of the Shipwright project beyond what its contributors or users find useful is a potential risk.
This can be mitigated by identifying new contributors who have an interest in maintaining this new featureset, and attracing users with a broader set of useful APIs that work well together.

## Drawbacks

Expanding the scope of any software project is inherently risky.
Existing contributors will have wider responsibilities, and current and prospective Shipwright users will have more APIs and features to learn and use (or to take time to learn why they don't care to use).

## Alternatives

We could simply not embark on this effort, and remain focused on the existing Build APIs, CLI, operator, etc.

## Infrastructure Needed

GitHub repository for Image APIs, teams to organize its collaborators, release processes to release any artifacts the subproject produces.

## Implementation History

None so far.
