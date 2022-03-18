<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: one-off-builds
authors:
  - "@dalbar"
  - "@raghavbhatnagar96"
reviewers:
  - "@imjasonh"
approvers:
  - "@SaschaSchwarze0"
  - "@qu1queee"
  - "@imjasonh"
creation-date: 2022-02-23
last-updated: 2022-01-31
status: provisional
---

# One-Off Builds

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, DA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This ship proposes to embed build specifications in buildruns to allow a user to build an image without
creating and maintaining a dedicated build resource. The newly introduced field `buildSpec` is capable
to hold a full `Build` specification.

## Motivation

Building an image with `shipwright/bulid` currently requires the user to create a build and a corresponding buildrun
resource. This approach has the advantage that a build resource can be reused for multiple buildruns without requiring
to redefine properties such as the target repository. However, not every workload requires a reusable build. Tools built on top of `shipwright/build` such as [build-load](https://github.com/homeport/build-load) create builds
for the sake of referencing them in a follow-up buildrun without the intention of creating multiple runs.

A user might try to build a repository remotely in a `docker build` style and a single run is enough. This can be
combined with the `local` source builds to give the user a single ergonomic command to push their source code and build the image in one step.

The one-off build removes the burden of maintenance and cleanup of multiple resources for users and tools that do not
profit from the re-usability of builds.

### Goals

- Execute builds without explicitly creating a `Build` resource
- Extend `BuildRun` resource to allow embedding of the build specification
- Extend `BuildRun` reconciler to pickup build specifications and translate them into `TaskRuns` directly
- Extend shp cli to create a buildrun with an embedded build specification
- Offer the same level of validation for embedded builds as regular builds

### Non-Goals

this ship focuses on `BuildRun` that embeds builds not vice versa

- Extend `Build` resource to start a "one-off" buildrun
- Use embedded build specification in a `BuildRun` in combination with `Build` references to create a "merged" version
  of an embedded build

## Proposal

We propose to extend the `BuildRun` API with a new field `buildSpec`. It is equivalent to the `spec` field in a `Build`
resource and allows the same granularity of configuration. Thus, the enhancement is also referred to as an embedded build.
The goal of this proposal are to avoid creating an actual build as a side-effect and to translating the build specification
directly into a `TaskRun` without losing any of the validation steps for registering a build. This implies specifically
that an embedded build spec does not correspond to any build resource in the cluster, nor are we proposing to create one
in the process (more in alternatives). We also propose to not count embedded build specifications as builds in
`shipwright/build` metrics.

The following describes how such a `BuildRun` object can be specified:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: buildpack-nodejs-buildrun-
spec:
  buildSpec:
    source:
      url: https://github.com/shipwright-io/sample-nodejs
      contextDir: source-build
    strategy:
      name: buildpacks-v3
      kind: ClusterBuildStrategy
    output:
      image: docker.io/${REGISTRY_ORG}/sample-nodejs:latest
      credentials:
        name: push-secret
```

When an embedded specification is invalid, we propose not to start a `TaskRun`. Instead, the buildrun should be set
to a failure state and the reasons for the invalid specification should be reflected using the `Succeded` condition in `status.conditions`.

Additionally, we require that `buildSpec` is never used in conjunction with any of `buildRef`, `output`, `paramValues`,
`env`, `timeout` or `sources`.
When there is `buildRef` and `buildSpec`, one might decide to prefer `buildSpec` due to locality
properties. However, it would leave a dead property in the resource description, hurting its declarative nature and
adding the additional overhead of implementation details for the user to remember. So to
prevent confusion in the first place, a validation functions should fail the creation of such buildrun and inform the
user about the exclusive usage of those properties.
Properties like `output` and `sources`, that overwrite the referenced build's properties, have no usage when an embedded
build specification is set. Instead, these properties can be directly specified inside `buildSpec`.
To keep the user from guessing implementation details, a validation function
should fail the creation of such buildrun again and advise the user to directly set the fields within the embedded
build specification.

So the following examples are invalid resource definitions:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: buildpack-nodejs-buildrun-
spec:
  buildRef:
    name: buildpack-nodejs-build
  buildSpec:
    ...
```

```yaml
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: buildpack-nodejs-buildrun-
spec:
  output:
    image: my-user/nodejs-build
  buildSpec:
    ...
```

### User Stories

#### Story 1

As a shipwright user, I want to be able to build a container image without creating and managing multiple resources.

#### Story 2

As an application developer, I want my application to utilize `shipwright/build` without the need to create builds only for
the sake of creating buildruns since the re-usability of builds does not provide any advantages for my application.

#### Story 3

As a developer that uses shipwright, I want to build an image from my local source-code ergonomically in a single step without having
to maintain multiple resources similar to `docker build`.

### Implementation Notes

#### API Changes

The proposal adds the new optional field `buildSpec` to the `BuildRun` resource specification. It also changes the
`buildRef` field to optional, since either a `buildSpec` or a `buildRef` would be required to run a build:

```go
type BuildRunSpec struct {
    // BuildSpec refers to an embedded build specification
    //
    // +optional
    BuildSpec *BuildSpec `json:"buildSpec,omitempty"`

    // BuildRef refers to the Build
    //
    // +optional
    BuildRef *BuildRef `json:"buildRef,omitempty"`

    ...
}
```

#### Backend

The changes can be summarized in three major working items (code snippets are go-like pseudo code and simplified) :

##### Refactor Build reconciler logic
We have to extract verification logic for builds into its own functions. The buildrun reconciler should be able to import
and reuse them. This is also a good opportunity to add some unit-test with various verification cases.

##### New buildrun and embedded build verification
As proposed above, a new set of constraints has to be enforced for the validity of buildrun specifications.
Ideally some sort of empty intersection-set can be asserted between `spec.buildSpec` and build-overwriting properties
for a given buildrun specification:

```go
// alternatively create struct with sets as members
func hasIntersection(...disjunctSets) ([]br *v1alpha1.BuildRun) bool {
    // true if any elements in disjunctSets is present within br's specs
    // otherwise, false
}
```

Additionally, the buildrun verification has to use the extracted set of build verification on an embedded build
specification if present.

Finally the new overall verification could be a predicate similar to:

```go
func isValidBuildrun(br *v1alpha1.BuildRun) bool {
    return !hasIntersection(someSets)(br) && isValidBuildSpec(br.buildSpec) && ...
}
```

##### Inject buildSpec if present

So far, the buildrun reconciler uses a build reference to fetch a build and use its specification throughout the
reconciling process. We agree with the approach, but propose to fetch a build (like we
[currently](https://github.com/shipwright-io/build/blob/main/pkg/reconciler/buildrun/buildrun.go#L126) do)
only if `buildRef` is present. Otherwise, there has to be an embedded build specification in `buildSpec` which we
validate and use to create a temporary `v1alpha1.Build` instance. The instance should only exist in memory and not be an
actual cluster resource. After that point, both branches merge in the current workflow that eventually ends up in a
`TaskRun`.

Note, we do not have to perform any cleanup operations using this approach and a minimal amount of code changes in the reconciler.

#### CLI

Allow the same set of flags for build creation that correspond to a build's specification to be used in buildrun
creation. Similar to builds, we have to verify that `--buildref-name` is not used in conjunction with any build spec
related flag such as `--output-image` (not final concept for UX, will be elaborated in a later phase of the ship lifecycle).

### Test Plan

The implementation has to be tested on a `unit`, `integration` and `e2e` level to ensure correctness.
We have to make sure to include "negative" tests, that prevent `buildSpec` and build property overwrites such as
`sources` to be present in the same buildrun specification.

### Release Criteria

tbd


### Risks and Mitigations

*risk*: The proposal increases the complexity of buildruns by adding the new `spec.buildSpec` for build specifications.

mitigation: We need to provide proper documentation for the new feature. The build specs themselves are already
well documented. Besides, this proposal only adds a new workflow for certain cases. The `spec.buildSpec` is optional and a
user can always fall back to the approach of using a dedicated build resource.

*risk*: Build validation and error signaling can get lost with embedded specifications

mitigation: Provide the same level of validation for embedded fields as regular builds and reflect issues in the BuildRun status.

## Drawbacks

As an administrator, it would no longer be possible to assign users a role that is only allowed to create buildruns. It would require further additions to the ecosystem.

## Alternatives

### deprecating and merging spec.buildRef into spec.build

This alternative proposes to have an additional API change that merges
`spec.buildRef` into the new `spec.build` field, deprecating `spec.buildRef` in the process. As a result, it keeps semantically related
fields under the same umbrella of `spec.build`. The following example shows a buildrun that references a build with name `my-build`.

```yaml
apiVersion: shipwright.io/v1alpha1
kind: BuildRun
metadata:
  generateName: buildpack-nodejs-buildrun-
spec:
  build:
    name: my-build
```

A further extension could be to have a spec and a reference at the same time. The semantics in that case would be a
merge of the referenced build and the embedded build specification, where the embedded specification overrules the
existing bulid properties. Therefore, the fields `buildrun.spec.output` and `buildrun.spec.sources` would become
unnecessary.

### prefer build reference over embedded build

As an alternative, we prioritize build references over embedded builds when both are present.

### counting embedded builds still as builds

As the title suggests, technically it would be possible to increase the build count for every embedded build spec that
succeeds.

### create an implicit build

This approach is vastly different to the one we propose. The controller should pick up the build specification, generate
meta-data and create a build for the user. At the end of the buildrun, that build is cleaned up automatically to prevent
any pollution. Thus, the build is an intermediate layer to achieve the one-off build.

The main advantage of this strategy is that it integrates well with the current controllers for buildruns and build. Especially for
builds, it guarantees the same level of validation to register the build.

One of the disadvantages of this strategy is that there is a higher cost of creating and deleting build resources in a
cluster like additional network operations.

## Implementation History

tbd
