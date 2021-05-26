<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: deprecate-runtime
authors:
  - "@imjasonh"
reviewers:
  - "@adambkaplan"
  - "@qu1queee"
approvers:
  - "@adambkaplan"
  - "@qu1queee"
creation-date: 2021-05-26
last-updated: 2021-05-26
status: implementable
replaces:
  - "https://github.com/shipwright-io/build/blob/main/docs/proposals/runtime-image.md"
---

# Deprecate `runtime`

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

What features of `runtime` do we expect future users to want, and how should we deliver them in a manner that's consistent with the future direction of the API?

## Summary

### Background

The `.spec.runtime` field allows users to describe operations that Shipwright should take on images, after they're built by the defined `.spec.strategy`.

These include:

- swapping out the base image layers
- changing the image's working directory, environment variables, labels, user, entrypoint
- injecting file paths
- running arbitrary commands in the container and reflecting any modifications in the output image

These operations are implemented by adding a step after the strategy's steps that executes a generated Dockerfile based on the built image, with `RUN`, `USER`, `ENTRYPOINT`, etc., commands expressed.
This Dockerfile is executed with `kaniko`, which also pushes the image, replacing the originally built image in the registry.

### Proposal

This proposal suggests marking this functionality as deprecated, and removing the feature in a future release.

## Motivation

The original motivation for `runtime` seems to have been to simulate [multi-stage docker builds](https://docs.docker.com/develop/develop-images/multistage-build/) to produce leaner output images.
Using this approach, a build process requiring build tools (e.g., a JDK, `go`) can be modified so that development tools are stripped from the resulting image, leaving only the built, runnable artifact.

This is a noble goal, but since multi-stage docker builds have been available since Docker 17.05, and image build tools like Buildpacks already produce lean images by default, this feature is non-standard.

Furthermore, allowing Build authors to modify the image, and inject files and commands in the container environment, using a generated ephemeral Dockerfile that's not persisted after the build execution, presents an opportunity for confusing behavior at best, and supply chain attacks at worst.

Since the feature is implemented by pushing a new image with the same tag as the image built and pushed by the build strategy steps, clients observing registry changes will be able to observe two tag pushes -- this might lead to downstream automated systems deploying or attempting to deploy the first, unmodified image, leading to confusing or insecure runtime behavior.

### Goals

Deter new users from adopting `runtime`, until we are comfortable removing the feature entirely.

### Non-Goals

Replace every aspect of the feature with new Shipwright-level features.

- Some features should be the responsibility of the build tool (e.g., multi-stage docker builds, Buildpacks), and Shipwright should encourage and document them.
- Some features might end up becoming other more focused Shipwright-level features (e.g., rebasing an image)

In any case, these additions are out of scope for this proposal.

## Proposal

Announce deprecation and set a timeline for removing the feature in a future release.
As an initial proposal, I would suggest two releases from now -- at time of writing, that would be v0.7.0, scheduled to be released in ~3 months.

### Implementation Notes

- Document that the feature is Deprecated in [user-facing documentation](https://github.com/shipwright-io/build/blob/main/docs/build.md#Runtime-Image).
- Mark the `.spec.runtime` field as `// Deprecated` in Go source.
- Notify `shipwright-users@` of the deprecation plan and timeline.

If we [adopt admission controllers](https://github.com/shipwright-io/build/blob/main/docs/proposals/webhook-validation.md), we can [set a warning in the response](https://kubernetes.io/blog/2020/09/03/warnings/#admission-webhooks) that would be shown to `kubectl` users and potentially other clients, when a `Build` request specifies the `.spec.runtime` field.
I'm not sure if admission controllers will land in time to be useful in this case.

### Test Plan

While deprecated, existing tests should continue to guard against regressions.

When the feature is removed, tests covering it can also be removed, and existing tests should ensure no other feature regressions.

### Release Criteria

#### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Risks and Mitigations

At this time we don't believe the feature has significant usage, but due to the nature of installations, there's no way to be sure.
While the feature is marked as deprecated, we might find a user significantly depends on it, and we should be open to considering un-deprecating it, or modifying our removal timeline/process as necessary.

## Drawbacks

We are deprecating and removing a feature without a direct replacement.

## Alternatives

We could modify how the feature is implemented to close potential security holes, and better document and warn against any holes that remain.
Ultimately, we believe the feature isn't widely used, and the functionality is better served by existing build tool features, so this investment (and opportunity cost) would need to be well motivated.

## Infrastructure Needed [optional]

None
