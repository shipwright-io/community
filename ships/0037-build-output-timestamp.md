<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-output-timestamp
authors:
  - "@HeavyWombat"
reviewers:
  - "@adambkaplan"
  - "@qu1queee"
  - "@SaschaSchwarze0"
approvers:
  - "@adambkaplan"
  - "@qu1queee"
creation-date: 2023-04-24
last-updated: 2023-12-11
status: implemented
---

# Build Output Image Timestamp

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, DA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This _ship_ proposes an extension to the `spec.output` of a Build (or BuildRun through the build specification) to explicitly set the result image timestamp.

## Motivation

With Shipwright, we can help users to achieve [reproducible builds](https://reproducible-builds.org/docs/timestamps/) depending on the tools that are used in the strategy. One factor of reproducible builds is avoiding the usage of build system specific timestamps in the final image. Ideally, when using a `Dockerfile` based build, running the build twice with the exact same source input should yield the same image.

### Goals

- Allow a Build user to specify that a neutral timestamp is used for the creation time of the container image, i.e. unix timestamp zero.
- Allow a Build user to specify that -- if possible -- Shipwright is suppose to come up with a reasonable deterministic timestamp of the build, e.g. the Git commit timestamp for Git source based builds similar to how GoReleaser does it.
- Allow a Build user to specify that Shipwright is not suppose to do anything with regards to the container image creation timestamp.

### Non-Goals

- Modification of timestamps of the source files.
- No post-processing of container images that used strategy managed push, i.e. Paketo Buildpacks.
- No magic build strategy manipulation of steps other than the `image-processing` step.

### User Stories

#### Story 1

As a Shipwright user, I want to use a `Dockerfile` based build where the resulting container image creation timestamp is set to unix timestamp zero.

#### Story 2

As a Shipwright user, I want to use build a container image that has the image creation timestamp that matches the one of the Git commit that was used for the build.

#### Story 3

As a Shipwright user, I want write a build that results in a build run that does not touch the image timestamp at all.

## Proposal

### API Changes

We will extend the Build resource by an additional (Unix epoch) timestamp entry in `spec.output`:

```yaml
spec:
  output:
    timestamp: Zero | SourceTimestamp | BuildTimestamp | <timestamp> | (null)
```

The `timestamp` entry is an optional string value field with the default being `null`. It can have multiple values that lead to different behavior:

- `Zero` refers to the Unix epoch start date of January, 1st 1970.
- `SourceTimestamp` would lead to taking the provided source code meta data to define the timestamp. In case the source is a Git repository, the Git commit that is used for the build defines the source timestamp. In any other case, the newest creation time stamp of the source files is used. Note, the timestamp needs to be taken from the user provided source files since the build process might introduce new files as part of the build itself. The source step needs to be extended to introduce a reliable point and single spot in which the source timestamp is being obtained.
- `BuildTimestamp` defines that the image timestamp is to be set to the current actual execution timestamp of the Build Run.
- `<timestamp>` must be a parsable Unix epoch timestamp that lies in the past and will be then used as-is for the creation timestamp of the image, i.e. custom neutral timestamp. An empty string is considered not parsable and therefore results in an error.
- Null (unset) results in no timestamp usage at all, which means that the respective build strategy timestamp is used.

The same fields will be made available in a BuildRun's `spec.output` and `spec.buildSpec.output`.

The BuildRun status will be extended to include the timestamp in the output field.

### Backend

The timestamp processing will be performed as part of the [image-processing step that is introduced with Shipwright-managed push](0026-shipwright-managed-push.md#backend).

It can only work with Shipwright managed push strategies and it will rely on the source step providing the necessary timestamp piece of information. If not provided with a timestamp, it will fail with condition:
```yaml
conditions:
- type: Succeeded
  reason: Failed
  status: "False"
  lastTransitionTime: "2023-10-10T10:10:10Z"
  message: 'Cannot change image timestamp in the context of a build strategy that pushes the image itself as part of its build process.'
```

The respective source step needs to be extended with a flag or feature so that it produces a result file for the commit timestamp, similar to the result files for author or commit SHA. This would only be done and subsequently used in case the `SourceTimestamp` is configured in the build.

- `Git` build source will use the timestamp of the last commit to produce the timestamp file.
- `Local` build source will produce a timestamp file with the timestamp of the newest file in the uploaded set of files.
- `OCI` build source will produce a timestamp file with the timestamp of the newest file in the uploaded bundle container.

The `image-processing` step and code needs to be updated to take the configured timestamp and mutates the image to set the respective timestamp prior to pushing the image.

### Configuration

n/a since no deployment configuration is required

### CLI

The new output settings will be added as flags to the appropriate CLI commands.

### Sample updates

Extend build samples to show all possible options at least once.

### Documentation updates

The Build and BuildRun documentation are updated to explain the new fields and their behavior.

### Test Plan

The implementation has to be tested on a `unit`, and `e2e` level to ensure correctness.

## Release Criteria

n/a since this can be added at any point since the (new) default behavior is the current behavior

## Risks and Mitigations

Having a `SourceTimestamp` convenience setting is great for the main use case of using a Git repository, it could be confusing for a user when they accidentally try to use it with a non-Git source build and their Build or BuildRun fails to be registered. A clear and easy to understand error message is required in this case.

## Drawbacks

n/a since this does not has to be used by a Build user

## Alternatives

### Generic post-processing step

Instead of using the `image-processing` step, we could inject a common post-processing step for all strategies which could modify the timestamp of the image after it got pushed. This would work for all strategies that push images, independently whether it is strategy managed push or Shipwright managed push.

## Implementation History

Nothing so far.
