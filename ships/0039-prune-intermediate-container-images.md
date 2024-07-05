<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: prune-intermediate-container-images
authors:
  - "@HeavyWombat"
reviewers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
  - "@adambkaplan"
approvers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
  - "@adambkaplan"
creation-date: 2024-07-04
last-updated: 2024-07-04
status: implementable
see-also:
  - "/ships/0026-shipwright-managed-push.md"
replaces:
superseded-by:
---

# Prune intermediate container images

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

n/a

## Summary

Shipwright builds allow for (post) processing of container images as part of the build, previously called mutation of the image. This includes changes to the labels/annotations of an image or the image creation timestamp.

Depending on the actual build software (Buildpacks, BuildKit, etc.) the build itself can result in two images being pushed to the container registry, which is sometimes also referred to as a double push. This is due to the fact that for example Paketo Buildpacks builds don't have an option to store the container image on disk during the build for which there are very good reasons. However, this results in any post processing to require to obtain the image content in the build pod via a pull. Subsequently, changing image metadata and pushing it to the target location leads to a second push and therefore also two images in the container registry.

The following excerpt from a build shows the additional loading of the image (image pull) followed by an image push.

```text
Loading the image from the registry "registry.com/namespace/image-name:latest"
Loaded single image
Mutating the image timestamp
Pushing the image to registry "registry.com/namespace/image-name:latest"
Image registry.com/namespace/image-name:latest@sha256:2302fa7c3af6599a62cb33a79260cbeb0e091657de40a3c63a2f3633dd455874 pushed
```

Since the final push will use the same tag, the first image that was pushed will be untagged.

## Motivation

Conceptionally, there are two paths that we can follow to improve on the situation:

- Avoid the first push to the target registry all together by introducing a temporary intermediate container registry in which the image is being pushed. Shipwright would then only push to the configured target registry once everything is done. This however will increase the resource requirements of a build and will also result in a performance degredation, since the image needs to be pushed to the intermediate registry, which will be fast, but still require time.
- Mitigate the storage effects of the double push by cleaning up the first container image that was pushed in case post processing results in a second container image.

This enhancement proposal is about the later one.

### Goals

Users will not have two images in their container registry when a double push scenario occurrs.

### Non-Goals

Avoid having two container images in the target registry.

## Proposal

Introduce additional configuration settings in the BuildSpec that enables the possibility to remove the untagged image.

### User Stories

#### Story 1

User can define a setting so that the untagged image from the first push will be pruned.

#### Story 2

User can use Shipwright as-is with the same behavior as before, but end up with an untagged image.

### Implementation Notes

Add `PruneUntaggedImage` to define whether intermediate output images should be pruned. It defaults to `false`, which is the current behavior of doing nothing with the additional image.

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: buildpack-nodejs-build
spec:
  source:
    type: Git
    git:
      url: https://github.com/shipwright-io/sample-nodejs
    contextDir: source-build
  strategy:
    name: buildpacks-v3
    kind: ClusterBuildStrategy
  output:
    image: image-registry.openshift-image-registry.svc:5000/build-examples/taxi-app
    timestamp: SourceTimestamp
    pruneUntaggedImage: true
```

### Test Plan

This feature can only be tested in an E2E test case, since it requires a container registry.

Run a build with default settings and verify that there are two images being created.

Run a build with pruning of untagged images and verify that there is only one image in the registry.

### Release Criteria

n/a

### Risks and Mitigations

## Drawbacks

This is only a mitigtion, there will be still an addtional image in the container registry for a brief moment.

## Alternatives

As touched briefly in the beginning, the alternative would be to rewrite the build in such a way that an additional intermediate container registry can be used. This additional registry doesn't necessarily have to be a temporary registry that is deployed in the build pod or along with the build, but could be a standalone container registry that needs to be configured and acts as a temporary location. Conceptually, such an intermediate container registry would also allow to fully use vulnerability scanning for strategy-managed push with `failOnFindings` enabled. However, this approach requires a significant more planning work and additional enhancement proposal to clarify the requirements.

## Infrastructure Needed

n/a

## Implementation History

_Major milestones in the life cycle of a proposal should be tracked in `Implementation History`._
