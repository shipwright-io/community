<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: shipwright-managed-push
authors:
  - @SaschaSchwarze0
reviewers:
  - @qu1queee
  - @GabeMontero
approvers:
  - @adambkaplan
creation-date: 2022-01-19
last-updated: 2022-03-14
status: implementable
---

# Make the push operation to the container registry a shipwright-managed operation

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

None

## Summary

This ship describes why and how we provide means for build strategies to store the produced container image as a tarball in the file system instead of doing the image push. The push will then be performed by a shipwright-managed step.

## Motivation

Shipwrights mission is to turn source code into container images and push them to a container registry. While the sources are handled completely transparent by Shipwright, every build strategy author so far is responsible to implement the push operation. In most cases this is straight-forward because all of the tools that we use in our sample build strategies support this.

It became more complex when we introduced results to pass information about the image to the BuildRun status: the digest and the size. In many cases, additional script-style logic had to be added to build strategies to parse standardized or proprietary files to access this information, and then to write it to response files. This made those strategies more complex. For example, the Buildpacks strategy required an [additional argument](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml#L76), and a [hacky command to extract the digest](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml#L79-L80). [Kaniko even required an additional volume and container.](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/kaniko/buildstrategy_kaniko_cr.yaml#L48-L74).

Another feature introduced another pain: we allow to specify labels and annotations in the Build and BuildRun to be added to the image. With the image being pushed already by the strategy step, this feature was only implementing by re-pushing the augmented image - potentially causing issues in scenarios where the image push triggers other logic such as deployments.

These are already two reasons to move the image push operation to a shipwright-owned step that runs after the build strategy. And there are many more scenarios that we will be able to implement. We will cover one of them in this ship as it is being used already in a sample build strategy:

- Supporting insecure registries. Today, if this is desired, every build strategy author needs to provide the necessary parameters, for example in todays [buildkit strategy](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml#L22-L25). Once this is shipwright managed, common options can be added to the Build's output section and users of all build strategies that use the new approach proposed here, can leverage that option.

Other possible scenarios are just listed here, they'll be looked at in separate ships:

- Supporting custom certificates for the container registry. Similar to the scenario above, we can add this to the Build's output section and honor this in our push logic.
- Vulnerability scanning of the image. Once we own the push operation, we can make this a shipwright managed operation that happens before the push to prevent vulnerable images to appear in the container registry. Today, this can be done by build strategy authors, but again makes the strategy more complex. See [kaniko-trivy](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/kaniko/buildstrategy_kaniko-trivy_cr.yaml).
- Software Bill Of Materials. Having the image locally at hand, we can analyze the content of the image and put together something like an [SPDX document](https://spdx.dev/resources/learn/) and store that somewhere.
- Image signing. This is loosely coupled because images can only be signed after they are pushed, but still this is a feature that fits well in container image operations that run after the build strategy steps.

### Goals

- Allow the build strategy author to store the image as a tarball to a file system location provided though a new Shipwright system parameter.
- Introduce mechanisms in Shipwright to read this tarball and push it to the container registry.
- Continue to support build strategies that perform the push themselves.

### Non-Goals

- Describe any of the follow-on scenarios listed under [Motivation](#motivation) in any detail. These should be covered in separate ships

## Proposal

Today, most BuildRuns result in pods with two contains that run sequentially:

- Shipwright-defined container that loads the sources (can also be multiple if sources of type HTTP are used in combination with a Git or Bundle source)
- Build strategy step(s) that builds and pushed the image

Going forward we will still support this, but also offer the following option:

- Shipwright-defined container that loads the sources (can also be multiple if sources of type HTTP are used in combination with a Git or Bundle source)
- Build strategy step(s) that builds the image and stores it in the file system
- Shipwright-defined container that pushes the image

### API Changes

We will introduce an additional system parameter: `shp-output-directory`. Like `shp-source-root`, this will contain a hardcoded value backed by an emptyDir volume. A good path would be `/workspace/output-image`. This is an extension point for the future for scenarios where we store the image on a persistent volume instead, for example when a Shipwright Build is embedded into a larger pipeline.

A build strategy that uses this parameter = which references `$(params.shp-output-directory)` in any `buildSteps` `commands`, `args` or `env` will be considered a build strategy which does not itself push the image. Any build strategy that does not use this parameter is assumed to perform the image push operation on its own.

Shipwright will push the container image if any build step in the build strategy references the `$(params.shp-output-directory)` parameter in the `commands`, `args`, or `env` spec. Any build strategy that does not use this parameter in this manner is assumed to perform the image push operation on its own.

We will extend the Build's output API to support an optional `insecure` boolean flag with `false` being the default. To allow build strategies that handle the push operation on their own to also access this information, we will introduce another system parameter: `shp-output-insecure`. The `insecure` flag can be overwritten in the BuildRun's output.

### Backend

We will rename today's [`mutate-image` command](https://github.com/shipwright-io/build/tree/v0.7.0/cmd/mutate-image) to `image-processing`. It will be added to the end of the TaskRun if annotations or labels need to be appended to the image (today's mutate-image scenario), or if we determine that the build strategy uses `$(params.shp-output-directory)`.

The command will be extended with the necessary optional parameters and logic to read the image from disk and push it to the container registry. The exact naming of the arguments can be defined in the implementation phase as those are only internal. [go-containerregistry](https://github.com/google/go-containerregistry) provides the necessary library code to read the image from disk and push it to the registry. It is already used for the existing image mutation logic.

The command will write the `shp-image-digest` and `shp-image-size` results like it does today already when it mutates the image.

Even though the image push will be performed by our own step, we will continue to make the image push secret available to the build strategy steps. This is required because in many scenarios, this secret is used today to also perform read operations - such as in the read-from-cache-scenario mentioned under [API changes](#api-changes), or to load `FROM` images in Dockerfile-based build scenarios.

The `image-processing` step will continue to run as `root` user with `DAC_OVERRIDE` capability. This today allows the step to overwrite the result files no matter which user wrote them. It will also allow the step to read the image from file system independent of as which user and with which permissions it was written. Controller configuration can be used to change this configuration like today. This is useful for scenarios where Shipwright users consistently use the same non-root user in all steps (Shipwright-owned and in build strategies).

### CLI

We will make the new `insecure` flag in the output section available as new command line flag when creating or updating a Build, and when creating a BuildRun (aka submitting a Build).

### Sample updates

We will update the sample build strategies to write the tarball where possible:

- [BuildAh](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/buildah/buildstrategy_buildah_cr.yaml) allows to [specify a tarball in buildah push](https://github.com/containers/buildah/blob/main/docs/buildah-push.1.md#example).
- [BuildKit](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml) supports to write [Docker and OCI tarballs](https://github.com/moby/buildkit#docker-tarball).
- [Buildpacks](https://github.com/shipwright-io/build/tree/v0.8.0/samples/buildstrategy/buildpacks-v3) will not change because the [lifecycle methods creator (and exporter) can only write an image as output](https://github.com/buildpacks/spec/blob/main/platform.md#creator).
- [Kaniko](https://github.com/shipwright-io/build/tree/v0.8.0/samples/buildstrategy/kaniko) will use the `--tarPath` argument just like it is [done today already in kaniko-trivy](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/kaniko/buildstrategy_kaniko-trivy_cr.yaml#L42-L43).
- [ko](https://github.com/shipwright-io/build/blob/v0.8.0/samples/buildstrategy/ko/buildstrategy_ko_cr.yaml) needs investigation. It has a [`--local` option](https://github.com/google/ko#local-publishing-options), but that seems to be for the local Docker daemon and not for a tarball.
- [source-to-image](https://github.com/shipwright-io/build/tree/v0.8.0/samples/buildstrategy/source-to-image) is using a Dockerfile strategy as second step and can therefore pick up the approach from BuildAh and Kaniko.

### Documentation updates

We will need to update the documentation about build strategies to describe that there are two options to handle the output image:

The new option is to write a tarball. In such a case the build strategy must use `$(params.shp-output-directory)` and should not write the `shp-image-digest` and `shp-image-size` results. We need to find out which formats go-containerregistry supports and document them.

The existing option is to push. In this case, the build strategy must not use `$(params.shp-output-directory)` and should write the `shp-image-digest` and `shp-image-size` results. It should respect `$(params.shp-image-insecure)`. The known disadvantage, duplicate pushes of the same image, will be outlined as today.

We will extend the documentation for Build users about the new output `insecure` option in the Build and BuildRun's output section.

### User Stories

#### As a Strategy Author, I want to just write the image to a tarball on disk so that I do not need to handle the results manually

#### As a Build user, I want to defined that my output container registry is insecure in a first-class property of the Build API so that I do not need to look how the different build strategies implement this

### Test Plan

It should include:

- Unit tests
- Integration tests
- e2e tests

### Release Criteria

Should be available soon as it is a prereq for other scenarios. It should come before Shipwright goes Beta due to the larger change to how build strategies need to be written.

### Risks and Mitigations

None.

## Drawbacks

The BuildRun Pods will need one more container, not just when labels and annotations are used. At the beginning, this will likely make BuildRuns slightly slower. In the long term, the additional scenarios outlined in the [motivation](#motivation) will bring additional value that will presumable weight higher.

## Alternatives

We could stick to the current approach, but in my opinion we here do not just simplify how build strategies need to be written, but will also open the door to many new scenarios as outlined in the [motivation](#motivation).

## Implementation History

Nothing so far.
