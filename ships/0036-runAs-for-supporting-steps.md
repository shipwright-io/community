<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: runAs-for-supporting-steps
authors:
  - "@SaschaSchwarze0"
reviewers:
  - "@adambkaplan"
  - "@qu1queee"
approvers:
  - "@adambkaplan"
  - "@qu1queee"
creation-date: 2023-03-24
last-updated: 2023-05-03
status: implementable
---

# RunAs user and group for supporting steps

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, DA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This ship proposes a new configuration option that allows to run the supporting steps that Shipwright is adding to a BuildRun (the source steps, the image mutation step) as the same user that runs the build strategy steps.

## Motivation

In build strategies, authors define the steps that are needed to turn source code into a container image. For a complete BuildRun, shipwright prepends, and appends additional steps. Prepended steps take care of the loading of the source code that is defined in the Build. Appended steps process the image that was built.

In its [configuration](https://github.com/shipwright-io/build/blob/v0.11.0/docs/configuration.md), Shipwright allows to define the container templates for these supporting steps. The container template can include a security context with a runAsUser and runAsGroup. This template applies to all BuildRuns. The runAs user and group of the source step has two consequences:

1. The source code on the file system is owned by this user and group.
2. As long as we have creds-init enabled, the `.docker` directory in `/tekton/home` is owned by this user and group.

Build strategy authors can also define a securityContext for each step where they can define a runAsUser and runAsGroup.

Different sample build strategies need to run as different users and groups. For example:

- [Kaniko is running as user 0](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/kaniko/buildstrategy_kaniko_cr.yaml#L15) (root) due to the way is building the container image.
- [BuildKit runs as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml#L58-L59)
- [Paketo Buildpacks today run as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml#L40-L41). That's only the case as long as we are using the Bionic builder image there. For Jammy, the Paketo community actually updated the stack to user 1001 and group 1000 at build time. We will need to adapt this in our sample strategy when we switch to Jammy there.
- [ko runs as user 1000 and group 1000](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/ko/buildstrategy_ko_cr.yaml#L55-L56)

All sample build strategies that run as non-root currently contain a [prepare step](https://github.com/shipwright-io/build/blob/v0.11.0/samples/buildstrategy/ko/buildstrategy_ko_cr.yaml#L29-L49). This step ensures that `/tekton/home` is owned by the build strategy runAsUser/runAsGroup so that creds-init also works there IF the source step runs as a different user. With today's default configuration, this is a no-op because all source steps by default run as user=1000 and group=1000 which is the same non-root user of all our sample build strategies. It is only there in case somebody uses a different user, or group for the source step. Though, I think we really never tested this and are currently having an issue because we do not `chown` the source directory as well. Basically if the source step is reconfigured as something else, then the runAsUser/runAsGroup of our sample build strategies may not have full access insight the source directory which can lead to failures.

To improve this, I am suggesting to change our logic to run the supporting steps as the user that is used in the build strategy IF all build strategy steps are defined to run as the same user. This means that all steps will run as the same user. The same logic applies to the group.

### Goals

- Run all Shipwright-managed steps as the same user that is used by the build strategy.
- Remove all prepare steps from sample build strategies. This implies that we will have many build strategies that then fully run as non-root user.
- Change the Paketo sample build strategy to Jammy to prove that non-root strategies that run as a non-root user that is not 1000 are functional.

### Non-Goals

- Read the runAsUser and runAsGroup from the step image when they are not defined directly on the step. This can be a future extension, but for now we expect the build strategy author to specify runAsUser and runAsGroup even if they match the image configuration.

### User Stories

#### Story 1

As a Shipwright User, I want to have sample build strategies that really run everything as a non-root user.

#### Story 2

As a Shipwright Build Strategy author, I want to read about recommendations to specify runAsUser and runAsGroup so that I correctly define the steps.

## Proposal

### API changes

The Build Strategy API is extended with a top-level `securityContext` inside the `spec`. It can be used to define the `runAsUser` and `runAsGroup`. If present, those settings override the `runAsUser` and `runAsGroup` form the container templates in the Shipwright configuration. They are also the default for the strategy steps. Strategy steps can still overwrite those two fields in their step-level `securityContext`.

In the following example, all Shipwright-managed steps will run as user 1001 and group 1000 while the single build strategy step will run as user 1001 and group 0.

```yaml
apiVersion: shipwright.io/v1alpha1
kind: ClusterBuildStrategy
metadata:
  name: buildpacks-v3
spec:
  securityContext:
    runAsUser: 1001
    runAsGroup: 1000
  buildSteps:
  - name: build
    securityContext:
      runAsGroup: 0
    ...
```

### Base image changes

Container images define which users and groups exist in the container by specifying the content of `/etc/passwd` and `/etc/group`. Though, it is generally possible to run a container with as a user or group that is not included in the container image.

One change that we must make, is to introduce a common world-writable home directory that we call `/shared-home`. We will reference this in the Shipwright configuration's container template.

Unfortunately, binaries may rely on an entry in the `/etc/passwd` and `/etc/group` files to be present to properly work. For the binaries that we run in supporting steps, this is the case for `ssh` which we need in the Git step when cloning using the SSH protocol. [SSH requires an entry and fails if it is not present.](https://github.com/openssh/openssh-portable/blob/V_9_3_P1/ssh.c#L671-L676)

We therefore must ensure that those files contain proper information.

We will change the base image to contain `/etc/passwd` and `/etc/group` files with the content that applies to the default Shipwright configuration. That will be for `/etc/passwd`:

```txt
shp:x:1000:1000:shp:/shared-home:/sbin/nologin
```

And for `/etc/group`:

```txt
shp:x:1000
```

### BuildRun reconciler changes

The BuildRun reconciler is changed to check the strategy-level `securityContext`. If it is defined, then their `runAsUser` and `runAsGroup` override those settings from the container templates of Shipwright-managed steps in the configuration. They are also defaults for the strategy steps' securityContext.

If the strategy-level `securityContext` is defined, then the reconciler will setup the `/etc/passwd` and `/etc/group` files to match this configuration. The following annotations are added to the TaskRun (assuming a build strategy that runs as user 1010 and group 1020):

```yaml
metadata:
  annotations:
    buildrun.shipwright.io/security-context-group: shp:x:1020
    buildrun.shipwright.io/security-context-passwd: shp:x:1010:1020:shp:/shared-home:/sbin/nologin
```

Tekton will automatically propagate these annotations to the Pod. There, they are used in a [downward API volume](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi):

```yaml
spec:
  volumes:
    - name: shp-security-context
      downwardAPI:
        defaultMode: 0444
        items:
          - path: group
            fieldRef:
              fieldPath: ".metadata.annotations['buildrun.shipwright.io/security-context-group']"
          - path: passwd
            fieldRef:
              fieldPath: ".metadata.annotations['buildrun.shipwright.io/security-context-passwd']"
```

Shipwright-managed steps will mount the `/etc/group` and `/etc/passwd` files through these volume mounts:

```yaml
spec:
  steps:
    - name: source-default
      volumeMounts:
        - name: shp-security-context
          mountPath: /etc/group
          subPath: group
        - name: shp-security-context
          mountPath: /etc/passwd
          subPath: passwd
```

Through this mechanism, the user in the container cannot change the `/etc/group` and `/etc/passwd` files, and has no chance to switch to a user.

Downward APIs in their nature mean a dynamic file content as a change to the referenced field, in our case the annotations, would update the mounted files. For our specific setup, this will not happen as [Kubernetes documents the following](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi):

> **Note**: A container using the downward API as a subPath volume mount does not receive updates when field values change.

This can be considered a current limitation in Kubernetes and could change in the future. Even when this is lifted, we will still be safe because a Kubernetes user who is allowed to update the pod, can only change the content of `/etc/passwd` but not the `runAsUser` and `runAsGroup` of the container as we will also reduce the default configuration of our supporting steps to not allow privilege escalation and to drop all capabilities (`SETGID` and `SETUID` would be required to switch the user). We actually should have configured them like this all the time already to give them no unnecessary privileges.

### Sample updates

The prepare step of all build strategies is removed. The Paketo build strategy is updated to use the Jammy builder and run as user 1001. The strategy-level `securityContext` is introduced in those strategies that do not run as user/group 1000, such as Paketo and Kaniko.

### Documentation updates

The build strategy documentation is updated to explain the consequences of defining, or not defining runAsUser and runAsGroup on the steps.

### Test Plan

Unit and integration tests should cover the part that ensures that everything behaves correctly. Our existing e2e tests should be able to cover the changes to the build strategies.

The e2e tests that depend on a Git secret have to succeed locally as they do not run as part of a build.

## Release Criteria

## Risks and Mitigations

None

## Drawbacks

## Alternatives

- Instead of setting the `securityContext` of all the supporting steps to the same value as the build strategy steps, we could also set the Pod's `securityContext` (through the [TaskRun's pod template](https://tekton.dev/docs/pipelines/taskruns/#specifying-a-pod-template)), and clear the `runAsUser` and `runAsGroup` on the steps. The resulting behavior would mostly be the same beside that the Pod's `securityContext` would also apply to Tekton's init containers. As we so far do not set any pod template and do not require to run the init containers as the same user, we will go with the step settings for now.
- The whole feature could be implemented as a opt-in feature through a configuration setting. This is not done because all sample build strategies will be changed. Even the existing build strategies would continue to behave as today because they do NOT have a common `runAsUser` or `runAsGroup` configuration on their steps.
- Instead of running all build strategy steps as the same user, we can also make use of supplemental groups: we collect the runAsGroups of all build strategy, and Shipwright-managed steps, and configure those as supplemental groups on the Pod's security context. In addition, we change all our source steps to make the files not only user- but also group-writable. As a result, all strategy steps will have write access to all source files due to group ownership. This works for most cases, but not for build strategies that try to run operations that only the owning user is permitted to, such as `chmod` operations as [performed by the Paketo Maven buildpack](https://github.com/paketo-buildpacks/maven/blob/v6.14.0/maven/maven_manager.go#L136-L138).
- Instead of injecting the `/etc/group` and `/etc/passwd` files through volume mounts, we could have made those two files writable in the container images. Shipwright-managed steps that require their runAs user to be in those files, would then update those files. This would have opened the door that somebody could exec into a running container and write any data to those files. This would have had no harm as long as we do not grant `SETGID` and `SETUID` capabilities to those containers.
- Instead of providing any means to have the correct data in the `/etc/group` and `/etc/passwd` files, we can change the logic to fully rely on code and external binaries that are able to run without the current runAs user being defined in those files. As of now, this is not possible. [go-git](https://github.com/go-git/go-git) is the alternative that could be used instead of the `git` binary to clone repositories. But, this comes [without lfs support](https://github.com/go-git/go-git/issues/381). In the Golang space, an alternative does not exist, root problem is that [git-lfs, while being implemented in Go, does not provide a stable Go API](https://github.com/git-lfs/git-lfs#limitations). Hacking around this is not reasonable because git-lfs internally calls out to the `git` binary. Libraries in other languages such as [libgit2](https://libgit2.org/) have not been evaluated.
- Instead of introducing the strategy-level `securityContext`, we could have implicitly assumed that Shipwright-managed steps should run as the user and group that is defined consistently on all strategy steps. This would have meant a behavioral change that potentially could have broken existing build strategies.

## Implementation History

Draft pull requests are here: [SHIP-0036: Part 1: Update base images to allow an arbitrary user to run it #1268](https://github.com/shipwright-io/build/pull/1268) and [SHIP-0036: Part 2: Update runtime logic to use a common runAsUser and runAsGroup in the steps when possible #1266](https://github.com/shipwright-io/build/pull/1266)
