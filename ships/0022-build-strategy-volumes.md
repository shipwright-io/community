<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-strategy-volumes
authors:
  - "@adambkaplan"
reviewers:
  - "@otaviof"
  - "@HeavyWombat"
  - "@imjasonh"
approvers:
  - "@sbose78"
  - "@SaschaSchwarze0"
creation-date: 2021-08-18
last-updated: 2021-10-19
status: implementable
see-also: []
replaces: []
superseded-by: []
---

# Build Strategy Volumes

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

This proposal will allow build strategy authors to declare volumes which can be shared across build steps and build runs.
The build strategy will declare the mount point for the volume which will be fixed across all build steps.
Volumes can either have a fixed volume source, or an overridable volume source that can be set in a `Build` or `BuildRun`.

## Motivation

Shipwright builds have two limitations that the build APIs do not address:

1. Builds cannot use persistent storage
2. Build strategy authors cannot share information across build steps, outside of the cloned source code.

Persistent storage is a core primitive that is needed to add caching mechanisms to builds.
Caches for container image layers and dependency artifacts are commonly used to improve the performance of application builds.

### Goals

- Build strategies can declare volumes whose data can be reused across steps.
- Developers can use persistent volumes in a `BuildRun`.

### Non-Goals

- Securing volume usage/access with admission control.
- Allow developers to inject arbitrary volumes into `BuildRun`s.

## Proposal

### User Stories

#### Share information in build steps

As a build strategy author
I want to declare volumes in my build steps
So that I can share information across build steps

#### Use persistent volumes in build runs

As a developer
I want to declare persistent volume claims in my Builds
So that I can cache resources like image layers and artifacts for subsequent BuildRuns

#### Use data stored in a secret

As a developer
I want to use credentials stored in a Kubernetes Secret
So that I can access private artifact repositories.

### Implementation Notes

The `BuildStrategy` API will be enhanced so that volumes can be declared alongside the build strategy steps.
The volumes will declare a default source for the underlying filesystem.
Each build step must opt into using the volume as a normal container volume mount.
Volumes can be marked `optional` indicating that the volume is not needed for a `BuildRun` to execute.
Volumes can also indicate if the source can be overrode by a `Build` or `BuildRun`.

Both the `Build` and `BuildRun` APIs will be enhanced to allow volumes to be specified.
The volume name referenced in the `Build` or `BuildRun` must be declared in the build strategy.
If a `BuildRun` references a volume that does not exist (either directly or in its parent `Build` object), the build should fail.

#### Deprecate Implicit emptyDir Volumes

Shipwright currently creates an implicit `emtpyDir` volume if one or more build steps declare a volume mount.
This behavior should be deprecated as a prerequsite to releasing this feature.
Implicit emptyDir volumes can then be removed when this feature is released.

#### Strategy Volumes API

`BuildStrategy`:

```yaml
metadata:
  name: cached-docker-build
spec:
  buildSteps:
  - name: build
    image: quay.io/my-org/my-builder:latest
    volumeMounts:
    - name: build-metadata
      mountPath: /home/build/metadata
    - name: image-cache
      mountPath: /var/lib/containers/cache
    - name: artifact-creds
      mountPath: /path/for/artifact/credentials.xml
      readOnly: true
  volumes:
  - name: build-metadata
    description: "Build metadata"
    optional: true
    volumeSource:
      type: EmptyDir # Type discriminator, this wil let us support new volume sources over time.
      emptyDir: {}
  - name: image-cache
    description: "Container image cache"
    volumeSource:
      overridable: true # indicates the volume source can be different in a Build or BuildRun
      type: EmptyDir
      emptyDir: {}
  - name: artifact-creds
    description: "Private artifact repository credentials"
    volumeSource:
      overridable: true
      type: EmptyDir
      emptyDir: {}
```

`Build` and `BuildRun`:

```yaml
spec:
  ...
  strategy:
    name: cached-docker-build
    kind: BuildStrategy
  volumes:
  - name: image-cache
    volumeSource:
      type: PersistentVolumeClaim # When overriding, the type can be changed
      persistentVolumeClaim:
        name: pvc-image-cache
  - name: artifact-creds
    volumeSource:
      type: Secret
      secret:
        secretName: artifact-creds # Inherited from Kubernetes VolumeSource API
```

#### TaskRun Generation

When Shipwright generates the `TaskRun` for the `BuildRun`, volumes are created as follows:

1. For each volume in the build strategy, a corresponding `volume` is created in the TaskRun's [pod template](https://tekton.dev/docs/pipelines/taskruns/#specifying-a-pod-template).
2. For each volume mount in a build step, a corresponding `volumeMount` is created in the associated TaskRun step.
3. If the `Build` or `BuildRun` specify a different volume source than what is declared in the build strategy, the volume source in the pod template is updated accordingly, with the `BuildRun` taking precedence over the `Build`.
4. If the `BuildRun` attempts to override a volume whose source does not have `overridable: true`, the `BuildRun` should fail.

Misconfigured volume mounts can cause BuildRuns/TaskRuns to remain in the `Pending` state.
Therefore the build controller should fail a `BuildRun` immediately in the following circumstances:

- A `Secret` volume source is used and any of the associated build step volume mounts do not have `readOnly: true` configured.
- A `ConfigMap` volume source is used and any of the associated build step volume mounts do not have `readOnly: true` configured.

This validation should be extended if additional volume mount configurations lead to "stuck" BuildRuns.

#### Example - Buildah with Image Cache

In this example, the [buildah](https://buildah.io/) build strategy is updated to declare a volume for its container layer storage - `/var/lib/containers`.
By default this is an ephemeral `emptyDir`, which means all images in the build need to be pulled for each `BuildRun`.

```yaml
kind: ClusterBuildStrategy
apiVersion: shipwright.io/v1alpha1
metadata:
  name: buildah
spec:
  buildSteps:
  - name: build
    image: ...
    ...
    volumeMounts:
    - name: var-lib-containers
      mountPath: /var/lib/containers
  ...
  volumes:
  - name: var-lib-containers
    volumeSource:
      overridable: true
      type: EmptyDir
      emtpyDir: {}
```

A `Build`, on the other hand, can run in a namespace that has a Persistent Volume Claim provisioned for the image layers:

```yaml
kind: Build
apiVersion: shipwright.io/v1alpha1
metadata:
  name: app-build
spec:
  source:
    url: https://github.com/shipwright-io/build.git
  strategy:
    kind: ClusterBuildStrategy
    name: buildah
  volumes:
  - name: var-lib-containers
    volumeSource:
      type: PersistentVolumeClaim
      persistentVolumeClaim:
        name: shipwright-build-cache
```

Each `BuildRun` created from this `Build` will then write its image layers to the persistent volume.
The layers can then be used in subsequent build runs, which can reduce the overall runtime of the build.

### Test Plan

Testing will ensure that basic scenarios are covered:

1. Volume mounting Secrets and ConfigMaps
2. Volume mounting Persistent Volumes
3. Verifying BuildRuns succeed or fail of the `overridible` attribute is set to true/false.

### Release Criteria

#### Removing a deprecated feature [if necessary]

The implicit generation of `emptyDir` volumes needs to be deprecated.
Afterwards, this feature can be introduced and the implicit generation of `emptyDir` volumes can be removed.

#### Upgrade Strategy [if necessary]

On upgrade, the volume feature is enabled by default, and the implicit creation of `emptyDir` volumes will be removed.

### Risks and Mitigations

Security is a concern with volumes, especially if arbitrary `HostPath` volume mounts are allowed in the API.
The [Pod Security Admission plugin](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
is a means to mitigate this issue, as it allows risky volume mounts to be blocked per namespace.
Shipwright builds should document how this admission plugin and the volumes feature interact.

## Drawbacks

Volumes support adds complexity to the API that may require developers to fully inspect the referenced build strategies.
We have assumed with other proposals that developers should not be expected to interact fully with `BuildStrategy` objects.
That said, volumes could be grouped with build strategy parameters as features that an "intermediate" user of Shipwright would interact with.

This current proposal requires each build step to opt into using the build volume.
This attempts to overcome the primary drawback of Tekton's workspaces feature, which adds volume mounts to all task steps.
To undo this, Tekton has an [alpha feature](https://tekton.dev/docs/pipelines/workspaces/#isolated-workspaces) which allows workspaces to be isolated to specific task steps.

## Alternatives

### Tekton Workspaces

We could implement similar capabilities by exposing Tekton's [Workspaces](https://tekton.dev/docs/pipelines/workspaces/) feature set directly.
This would potentially leak our abstraction of Tekton, breaking our practice of keeping Tekton an implementation detail.
Tekton workspaces also allow the [default filesystem](https://tekton.dev/docs/pipelines/workspaces/#setting-a-default-taskrun-workspace-binding) for a workspace to be configured cluster-wide.
If a Shipwright user wanted to change the default workspace filesystem type, they would need to change Tekton's configuration.

This proposal purposefully doesn't use Tekton workspaces for the following reasons:

- Workspaces only support limited types of volumes (`secret`, `configMap`, `pvc` and equivalent).
  Other volume types may be desired for build strategies that are granted elevated privileges, such as `HostPath`.
- Workspaces allow the volume source type to be deferred, and the default type to be configured per cluster.
  This would make debugging difficult if Shipwright build strategies assumed the volume defaulted to an `emptyDir`,
  but a cluster configured Tekton to provision PersistentVolumes by default instead.
- Workspaces default to applying the same volumeMount in all build steps.
  The isolated workspaces feature (alpha) is a signal that this behavior is not always desirable.
  Using standard Kubernetes volumes/volumeMounts gives strategy authors maximum flexibility to define which steps get a volume mounted, and where.

### An additional API for overriding volume sources

During review, an alternative API was proposed where the build strategy's volume source could be managed by a separate API.
Such an API would allow volume usage to be discoverable and allow volume sources to be shared across builds.
This was rejected in favor of a simpler API that hews closer to the Kubernetes Volume API.

## Infrastructure Needed [optional]

No new infrastructure.

## Implementation History

2021-08-18: Provisional SHIP proposal
2021-10-19: Updated to implementable
