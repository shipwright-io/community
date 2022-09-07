<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: Shipwright BETA API Release
authors:
  - "@qu1queee"
reviewers:
  - "@SaschaSchwarze0"
  - "@coreydaley"
  - "@otaviof"
  - "@dalbar"
  - "@adambkaplan"
approvers:
  - "@SaschaSchwarze0"
  - "@adambkaplan"
creation-date: 2022-07-09
last-updated: 2022-07-09
status: implementable
---

# Shipwright BETA API Release

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

## Summary

Propose a path to release Shipwright Beta API. Achieving a Beta API will provide users a more stable version they can depend on, with the certainty of not dropping features. In this document we provide the modifications, deprecations or additions to the Beta API.

## Motivation

1. The majority of the Shipwright Alpha API did not change over time. Part of the current Alpha API is considered mature enough for graduating to Beta.

2. The Shipwright API got many novel ideas over time, supporting different use cases. These features were implemented as part of the Alpha API, but the desired state was not reach. Planning for Beta API puts into triage these features and their potential deprecation.

3. The Shipwright API was created with some assumptions in mind, over time new features force us to rethink on better states of the API.

### Goals

1. Layout the changes for Beta API.

2. Document the deprecation policy for Beta API.

3. Define supported use cases for Beta API.

### Non-Goals

1. Document migration strategies of API conventions or object definitions from Alpha to Beta.

## Proposal

We follow the Kubernetes API [versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning). As documented by Kubernetes, we stick to the same levels (_Alpha, Beta or Stable_), depending on the levels of stability and support.

### Deprecation Policy

For the policy of Beta, we rely also on the Kubernetes Deprecation [policies](https://kubernetes.io/docs/reference/using-api/deprecation-policy/), which translates to the following four rules:

1. _API elements may only be removed by incrementing the version of the API group._
2. _API objects must be able to round-trip between API versions in a given release without information loss, with the exception of whole REST resources that do not exist in some versions._
3. _An API version in a given track may not be deprecated in favor of a less stable API version._
4. _Minimum API lifetime is determined by the API stability level_

Related to 4), in this document we propose that for Beta API versions, we provide coverage after deprecation for **9** months. We do not encourage to be based on releases quantity, per the faster cadence we have compared to Kubernetes.

### API Changes

1. In all resources, introduce a convention for objects with an array, where each element contains the `name` key. An existing example is `.spec.volumes`.
2. In the `Build` resource, replace `.spec.source.credentials` and `.spec.output.credentials` with a single field(_key_). This will become `.spec.source.cloneSecret` and `.spec.output.pushSecret` and will reference a secret name in the current namespace.
3. In the `BuildStrategy` resource(_namespace and cluster scope_), replace `.spec.buildSteps` in favor of `steps`. The same applies to the `BuildStep` Go type.
4. In the `BuildStrategy` resource(_namespace and cluster scope_), replace the `.spec.buildSteps[]`items with customize fields, instead of the whole `corev1.Container` Go type.
5. In the `Build` resource, replace the `build.build.dev/build-run-deletion` annotation in favor of `.spec.retention.atBuildDeletion`.
6. In the `BuildRun` resource, rename `.status.latestTaskRunRef` to `.status.taskRunName`.
7. In the `BuildRun` resource, consolidate `.spec.buildRef.name` and `.spec.buildSpec` into `.spec.build.name` and `.spec.build.spec`.
8. Use `type discriminators` if an API object has some form of implied polymorphism. For example, source types or volume sources.

### Deprecation

1. In the `Build` resource, deprecate `.spec.sources`. We only allow `.spec.source`.
2. In the `Build` resource, deprecate `.spec.dockerfile`.
3. In the `Build` resource, deprecate `.spec.builder`.
4. In the `Build` resource, deprecate `.spec.volumes[].description`.
5. In the `Build`, `BuildStrategy` and `ClusterBuildStrategy` resource, remove the `status` subresource.
6. In the `BuildRun` resource, remove the support for auto-generating a service account.

### Supported Use Cases

### Implementation Details/Notes/Constraints [optional]

### Risks and Mitigations

## Design Details

### Test Plan

### Graduation Criteria



### Upgrade / Downgrade Strategy


### Version Skew Strategy

## Implementation History

## Drawbacks


