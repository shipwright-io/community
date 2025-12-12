<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: multi-arch-image-builds
authors:
  - "@adambkaplan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-05-21
last-updated: 2025-08-19
status: implementable
see-also:
  - "ships/0039-build-scheduler-opts.md"
replaces:
  - https://github.com/shipwright-io/community/pull/275
superseded-by: []
---

# SHIP-0043: Multi-arch Image Builds

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

This extends Shipwright to orchestrate multi architecture container image builds. It aims to solve
the following challenges:

* Scheduling builds on native Kubernetes nodes for a given os + architecture.  
* Providing a parameter to tools that support multi-arch builds through emulation or
  cross-compilation.

## Motivation

### Background

#### OCI Image Indexes

The Open Container Initiative (OCI) provides the industry standards for container image
specifications and formats. It is the successor to Docker’s “v2” specification for container
images, and is designed to be backwards compatible. The specification includes an
[“image index” standard](https://github.com/opencontainers/image-spec/blob/main/image-index.md#image-index-property-descriptions)
for containers that can be run on multiple CPU and operating system architectures. This is
equivalent to the Docker v2 “manifest list,” and the two terms are used interchangeably. For
consistency in this proposal, “image index” will be used moving forward.

#### Multi-Arch Worker Nodes

Many Kubernetes distributions - starting with v1.30 and perhaps earlier - allow clusters to have
worker nodes with different OS and CPU architectures. Clusters expose the node OS and CPU
architecture through [default node labels](https://kubernetes.io/docs/reference/node/node-labels/).
Shipwright began accomodating these scenarios with
[SHIP-0039](https://github.com/shipwright-io/community/blob/main/ships/0039-build-scheduler-opts.md),
whose features were incrementally released in Builds v0.14 and v0.15. However, these features only
let developers create single images for a single architecture. Creating an image index for multiple
architectures requires significant orchestration effort outside of Shipwright.

#### Multi-Arch Capabilities in Build Toolchains

Many popular container build tools, such as `buildkit`, `buildah`,
[cloud native buildpacks](https://buildpacks.io/docs/for-app-developers/how-to/special-cases/build-for-arm/),
and `ko` support multi-arch builds through cross-compilation or qemu-style CPU emulation. These
tools often expose a `--platform` command line option to build the image with a different os +
architecture than the underlying host. The industry appears to have standardized on the `<GOOS>/<GOARCH>`
naming conventions for “platform” (ex: `linux/amd64`). These are identical to the namings used for
Kubernetes node labels.

Support for generating an OCI image index varies by tool. Some - like ko and buildah - do provide
support for creating image indexes. These typically require the build to run in the same process;
“fan out” support to run these builds in parallel is typically not supported or is more challenging
to set up in a containerized environment (ex: `podman farm` command).

### Goals

* Provide a mechanism for developers to request a multi-arch image build, or build for a specific
  OS + CPU architecture.

### Non-Goals

* Generalized matrix builds for Shipwright. This is a capability provided by Tekton.
* Adding retries for failed builds. This is out of scope to simplify the design. Such a feature can
  be considered in a follow-up enhancement.
* Scheduling builds on nodes with specialized hardware (ex: GPUs). This is already supported in
  Shipwright through the scheduler options in v0.15.
* Management of container resources (CPU, memory) at the Build/BuildRun level. At present, these
  can only be defined at the ClusterBuildStrategy/BuildStrategy level. See
  [build#1894](https://github.com/shipwright-io/build/issues/1894).

## Proposal

### User Stories

- As a developer, I want to build containers for x86 and ARM so I can share my app with my team
  using different CPU architectures (Apple Silicon vs. Windows x86)
- As a cluster admin, I want multi-arch builds to be scheduled on native nodes if my Kubernetes
  cluster has multiple CPU architecture worker nodes.
- As a platform engineer, I want to provide a standard way for my teams to run multi-arch container
  builds.

### Implementation Notes

#### “Image Platform” Concept

An image platform is the combination of operating system (“os”), CPU architecture (“arch”), and
other container image “platform” attributes as defined in the OCI [Image Index specification](https://github.com/opencontainers/image-spec/blob/main/image-index.md#image-index-property-descriptions).
Developers can specify the desired platform(s) for a container image build as a JSON/YAML object
with the following attributes:

- `os`: operating system. Required
- `arch`: CPU architecture. Required

The JSON/YAML representation is intended to future-proof the API for additional “features” defined
in the OCI image index spec, or other items that Shipwright can support at a later date (ex: cpu
arch variant, os.version).

A shorter single-string format for “platform” is not allowed within the Kubernetes YAML. However,
it can be supported when invoked from the command line (see below).

#### Multi-arch in `Build` and `BuildRun` objects

The `Build` and `BuildRun` APIs will add a new `multiArch` JSON/YAML object to `spec.output`. This
object will contain the following fields:

- `platforms`: list platforms to build, using the above "image platform" structure. Required.

Below is an example multi-arch Linux image build for x86, ARM, Power, and Z:

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
spec:
  ...
  output:
    image: <url>
    multiArch:
      platforms:
        - arch: amd64
          os: linux
        - arch: s390x
          os: linux
        - arch: arm64
          os: linux
        - arch: ppcle64
          os: linux
```

#### `BuildRun` Controller Reconciliation

##### Validations

The following validations should be run if `spec.output.mutliArch.platforms` is not empty.

For each image platform referened in the `platforms` array, the `BuildRun` controller should
verify that at least one node with the respective `kubernetes.io/os` and `kubernetes.io/arch` label
value is present. If no such node exists, the controller should set the `BuildRun`'s status to
failed with an appropriate message and reason code.

The controller should also check that `spec.nodeSelector` does not have any values set for
`kubernetes.io/os` or `kubernetes.io/arch` in the label matcher. If any value is present, the
controller should likewise set the `BuildRun`'s status to failed with an appropriate message and
error code.

##### Tekton `PipelineRun` Generation

If all checks above pass, the `BuildRun` controller will generate a Tekton `PipelineRun` to
execute the build. This will require significant refactoring of the existing codebase, which
currently generates a single `TaskRun` that is effectively “single-threaded.”

![Multi-arch pipeline](assets/0043-multiarch-pipeline.png)

The containers in the generated `PipelineRun` will be executed in three phases:

**Phase 1: Obtain Source**

The first phase will gather the source code. The mechanism for the generated `TaskRun` will vary
depending on the values in spec.source for the Build/BuildRun.

Code from git will invoke the current Shipwright git clone process as a TaskRun with the following
containers:

- The main “git clone” container that exists today.
- A second “image push” container, leveraging the existing Shipwright container that supports
  “managed push”. This container will package the source code into an OCI artifact, which is then
  pushed to the same registry as the output image. A tag suffix pattern (-src) will be used to
  ensure the source code artifact is persisted on most image registries. This may require
  significant enhancements to the current “image push” container.

Code from “local source” will likewise invoke a TaskRun as above to receive source code from a
remote machine. It will push the source code to an OCI artifact, as above.

Code from an OCI artifact will not invoke any TaskRun during this phase. The push of source code to
the image registry has already been completed.

**Phase 2: Fan Out Builds**

The generated `PipelineRun` will then define a set of Tekton `TaskRuns` that can be run in parallel
- one for each platform in `spec.output.multiArch.platforms`. Each `TaskRun` will set the appropriate
`nodeSelector` to schedule the build on a node with the appropriate `kubernetes.io/os` and
`kubernetes.io/arch` label.

The TaskRun containers will do the following:

- Pull the source code from the referenced OCI artifact. This will utilize existing Shipwright
  logic for pulling source code from OCI artifacts.
- Execute the build per the referenced build strategy.
- Push the output container image to the image registry, with the `-<os>-<arch>` tag suffix.
- Publish the output container image digest as a `TaskRun` result value.

All other fields used to control the build pod definition - such as resources and volumes - will be
inherited from the parent Build/BuildRun object as they are today.

**Phase 3: Assemble Index Image**

The last phase of the generated `PipelineRun` will create a `TaskRun` that assembles the OCI image
index, based on the results of the prior build `TaskRuns`.

If any platform failed to build, the PipelineRun should fail and subsequently mark the BuildRun as
failed.

#### CLI Enhancements

The CLI will add the `--platform` flag to the Build and BuildRun oriented commands. These shall set
the respective value for `spec.output.multiArch`. The `--platform` option can be set multiple
times, and will accept platforms in their single-line `<os>/<arch>` format.

Example experience:

```sh
shp build create sample-go --strategy=source-to-image \
  --output quay.io/adambkaplan/sample-go:v1 \
  --platform=linux/amd64 \
  --platform=linux/arm64

shp build run sample-go --platform linux/amd64 \
  --platform linux/arm64 \
  --platform linux/s390x
```

### Test Plan

Testing will primarily be at the unit level, where the configuration of the Tekton `TaskRun` can be
verified. Current integration tests use KinD clusters, which can't feasibly mimic a Kubernetes
cluster with multiple architecture worker nodes.

End-to-end testing is not feasible unless the project obtains access to a real Kubernetes cluster
with multiple CPU architecture worker nodes.

### Release Criteria

#### Removing a deprecated feature [if necessary]

N/A

#### Upgrade Strategy [if necessary]

The new multiArch field in `Build` and `BuildRun` objects will be optional (fields within it will
be required). Current builds should work as expected.

### Risks and Mitigations

TBD

> What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
> both security and how this will impact the larger Shipwright ecosystem.

> How will security be reviewed and by whom? How will UX be reviewed and by whom?

## Drawbacks

### Verbose API for “Platform”

Developers are used to a single string representation of “platform” - ex “linux/amd64”. This is
provided through the command line, but not in the YAML.

Using a verbose API in the YAML allows us to future-proof builds with more complex manifest list
definitions. The OCI image index spec already has fields that are allowed to be defined on image
index entries which are excluded from this initial API for the sake of simplicity:

- `variant` - some (but not all?) build tools support this. Ex: podman, buildah
- `os.version` - use is not really observed in the field today.
- `features` - this is a catch-all for future extensions to the oci image spec. This might be
  relevant for AI workloads if a container image requires specific hardware to execute.

### Limited Mechanism for Multi-arch Builds

This proposal only supports multi-arch builds on nodes with the respective OS and CPU architecture.
There are other potential mechanisms for building multi-arch container images:

- Use cross-compilation if the programming language/SDK supports it.
- Use emulation if the container build tool supports it.
- Execute builds on virtual machines with appropriate OS + CPU architecture.


## Alternatives

### String Shorthand for “Platform” in YAML

Many developers who do multi-arch builds with Podman or Buildkit are familiar with a single string
representation of “platform”. While convenient, this adds additional complexity to the storage and
serialization of data in Kubernetes. The shorthand also locks the API to a convention that may not
be universally understood by all build tools.

Using the shorthand in the CLI provides this capability in spirit, and closer to where developers
directly interact with builds.

### Use Kata Containers for Multi-Arch Builds

Kata Containers and Confidential Containers support a deployment known as
["peer pods"](https://confidentialcontainers.org/docs/architecture/design-overview/#clouds-and-nesting),
where a remote virtual machine can be provisioned and managed in Kubernetes just like a pod. This
can _hypothetically_ be used to securely run builds on machines with different CPU architectures.
On Kubernetes, Kata peer pods are scheduled by specifying a [runtime class](https://kubernetes.io/docs/concepts/containers/runtime-class/)
for the pod.

The Konflux CI project experimented with this approach for multi-arch container builds, with
[mixed results](https://groups.google.com/g/konflux/c/A0m0JWjYwnc/m/mUOSFkAsAwAJ?utm_medium=email&utm_source=footer).
The proof of concept proved that Tekton can schedule these build pods just fine, however there are
fundamental challenges scaling build workloads using dynamic virtual machines. Creating VMs and
required components for Kata containers is also challenging for a given CPU architecture.

Incorporating runtime class features into any multi-arch build capability would add significant
complexity and may require APIs for cluster administrators (see below).

### Control Multi-Arch Method with New APIs

A previous version of this proposal introduced cluster-level APIs that determined how a multi-arch
build could execute. This would allow an administrator to specify that some OS/CPU architecture
builds could be run on natively on respective Kubernetes nodes, whereas others could use
alternative mechanisms for producing the image for a given OS + CPU arch. These were known as the
`ImagePlatformClass` APIs.

This feature was primarily motivated by the Kata + Confidential containers approach above. Due to
the technical challenges identified above, it does not make sense for Shipwright to provide new
CRDs for this purpose. In addition, the many/many relationship between the `ImagePlatform` and
`ImagePlatformClass` CRDs may have been challenging to implement. This design can be revisited at a
later date, should the Kata + Confidential containers approach prove feasible and scalable.

### Tekton Matrix Builds

Tekton has support for matrixed tasks, and work is underway to improve this feature to support
node selectors and other pod template features. In theory the matrix can be used to run Shipwright
builds per architecture, especially if we release the Triggers project which provides a "Shipwright
build in Tekton pipeline" capability.

Our mission is to provide a simplified, opinionated experience for building container images.
Multi-arch is a clear use case specific to container image builds. Tekton deliberately provides
general-purpose solutions that emphasize flexibility. Using the matrix tasks feature exposes
developers to significant amounts of complexity.

## Infrastructure Needed [optional]

None for unit tests.

End to end testing may require a Kubernetes cluster with multiple CPU architecture worker nodes.

## Implementation History

- 2025-05-21: Initial Draft
- 2025-08-19: Revised draft with narrower scope
