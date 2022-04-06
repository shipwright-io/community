---
title: read-mirror-information-from-target-namespace
authors:
  - "@ricardomaraschini"
reviewers:
  - "@otaviof"
  - "@adambkaplan"
  - "@SaschaSchwarze0"
  - "@ImJasonH"
  - "@GabeMontero"
approvers:
  - "@otaviof"
  - "@adambkaplan"
  - "@SaschaSchwarze0"
  - "@ImJasonH"
  - "@GabeMontero"
creation-date: 2022-04-06
last-updated: 2022-04-06
status: implementable
---

# Allow mirror registry configuration per namespace

Shipwright Image operator reads mirror registry credentials from a Secret in the namespace where
it is running therefore it is not possible for users to provide different mirrors in the same
cluster. This enhancement proposal suggests a way of allowing users to provide authentications for
different mirrors in a per namespace basis (each namespace holds the mirror registry access data).

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

## Summary

This enhancement proposal suggests the introduction a per namespace mirror registry configuration,
without loosing the feature of having a global mirror configuration.

## Motivation

When dealing with a single mirror registry the authentication provided to the operator must be
able to write to different repositories inside the same registry. For example: when mirroring 
images in the `application` namespace images are mirrored into `registry.io/application` repo
while images in the `development` namespace are mirrored into `registry.io/development`. Moreover,
with a single mirror registry multi tenancy becomes more complex to be implemented if possible.

### Goals

1. Allow users to specify a per namespace registry mirror configuration
2. Allow users to specify a global mirror registry configuration as a fallback

### Non-Goals

## Proposal

The Secret will be named `shipwright-mirror-registry-config`, it can exist in any namespace and
it can containg the following properties:

| Name       | Description                                                                        |
| -----------| ---------------------------------------------------------------------------------- |
| address    | The mirror registry URL                                                            |
| username   | Username Shipwright Images should use when accessing the mirror registry           |
| password   | The password to be used by Shipwright Images                                       |
| token      | The auth token to be used by Shipwright Images (optional)                          |
| insecure   | Allows Shipwright Images to access insecure registry if set to "true" (string)     |

If such a Secret exists in a given namespace users can mirror images inside it. If the Secret does
not exist the operator should attempt to read it from the operator namespace allowing the current
setup to keep working. In other words: the operator has a global mirror configuration living in
its own namespace but uses a per namespace secret to overwrite the global setup.

### User Stories [optional]

#### Story 1

As an OpenShift administrator
I would like to be able to specify a different mirror per namespace
So I can easily isolate different users/use cases

### Implementation Notes

This should be relatively easy to implement as the logic to parse the proposed secret already
exists inside the operator.

### Test Plan

We improve the existing `kuttl` end to end tests and deploy multiple mirror registries. Tests
for global (operator namespace) and local secrets (target namespace) should be implemented.

## Implementation History

None so far.
