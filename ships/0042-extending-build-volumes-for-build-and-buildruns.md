<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: extend-build-volumes-to-mount-directly-to-builds-and-buildruns
authors:
  - "@avinal"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-04-21
last-updated: yyyy-mm-dd
status: provisional
see-also:
  - "0022-build-strategy-volumes.md"  
replaces:
  - "/docs/proposals/that-less-than-great-idea.md"
superseded-by:
  - "/docs/proposals/our-past-effort.md"
---

# Extending Build Volumes to mount directly to Builds and Buildruns
## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]


## Summary

This proposal aims to enhance Shipwright by allowing users to define additional volumes directly
within Build and BuildRun resources. These volumes can then be mounted into the build containers.
This capability is essential for scenarios requiring access to external resources like self-signed
certificates for Git/image registries or other necessary build artifacts, independent of BuildStrategy
definitions, overcoming a current limitation where all volumes must be pre-defined or overridable via the strategy.

## Motivation

Currently, Shipwright Builds face challenges when needing to utilize resources like self-signed
certificates for interacting with private Git repositories or image registries. While BuildStrategy
resources can define volumes that Build and BuildRun resources can override, there's no mechanism
to introduce entirely new volumes directly at the Build or BuildRun level without them being
pre-declared in the BuildStrategy. These strategy-defined volumes, even when overridden, are typically
scoped to the build steps defined by the strategy.

The original proposal for strategy volumes (SHIP-0022) explicitly listed adding arbitrary volumes to
builds as a non-goal to simplify its initial scope. This proposal addresses that gap. 

### Goals

- Allow Build and BuildRun resources to declare their own volumes, in addition to those provided or
overridden from the BuildStrategy.
- Ensure these directly defined volumes are mountable and accessible to all primary containers involved
in executing the build steps throughout the lifecycle of the build's execution Pod.

### Non-Goals

What is out of scope for this proposal? Listing non-goals helps to focus discussion and make
progress.

## Proposal

### User Stories

#### Using self-signed certificates

As a user, I want to use Git repositories or image registries that are secured with self-signed
certificates. These certificates need to be accessible by the build process, potentially across
multiple steps (e.g., source cloning, image push/pull).

#### Volumes indepedndent of Build Strategies

As a user, I want to provide necessary artifacts or configurations (e.g., custom configuration files, pre-downloaded dependencies)
to my builds using standard Kubernetes volumes (ConfigMap, Secret, EmptyDir). I need the flexibility
to do this directly in my Build or BuildRun without requiring the BuildStrategy to pre-define a
corresponding volume for me to override.

### Implementation Notes

The current Build and BuildRun APIs will be extended to support a mountPath field for each volume
entry and to allow volume definitions that are not simply overrides of BuildStrategy volumes.

The spec.volumes array already exists in Build and BuildRun.

```yaml
apiVersion: shipwright.io/v1beta1 # Ensure this is the correct API group
kind: Build
metadata:
  # ...
spec:
  # ...
  volumes:
    - name: custom-certs
      secret:
        secretName: my-ca-certs
      mountPath: /etc/ssl/certs/custom-ca.crt # Proposed field for direct mount
      # For ConfigMap
    - name: config
      configMap:
        name: my-build-configmap
      mountPath: /etc/build/config
      # For EmptyDir
    - name: shared-workspace
      emptyDir: {}
      mountPath: /workspace/shared
```

The current implicit requirement that every volume in Build.spec.volumes or BuildRun.spec.volumes
must correspond to a volume name in the BuildStrategy.spec.volumes will be relaxed.

- If a volume name in Build or BuildRun matches a volume name defined in the BuildStrategy, the
existing override logic and validation (e.g., ensuring the type is compatible if specified by the strategy)
will apply. The mountPath for these overridden strategy volumes would still be determined by the
BuildStrategy's step definitions.
- If a volume name in Build or BuildRun is new (i.e., does not match any volume in the BuildStrategy),
it will be treated as a direct volume mount. For these volumes, the mountPath field in the Build/BuildRun
volume definition becomes mandatory.


Directly defined volumes (those with a mountPath) will be added to the Pod template generated for the build.
These volumes will be mounted into all primary containers responsible for executing the build steps
defined by the BuildStrategy, as well as any essential helper containers injected by Shipwright
(e.g., for source checkout, if applicable and technically feasible to receive these mounts).
This ensures wide availability for common use cases like certificates.

If a volume with the same name is defined at multiple levels, the following precedence will apply
for its definition (e.g., which Secret or ConfigMap to use):

- BuildRun.spec.volumes (highest precedence)
- Build.spec.volumes
- BuildStrategy.spec.volumes (lowest precedence, acts as a default or template)


- For new, directly mounted volumes, a mountPath must be specified.
- The name of the volume must be unique within the list of directly defined volumes for a given Build or BuildRun.
- Standard Kubernetes validation for the volume source (e.g., configMap, secret) will apply.
- If a mountPath defined in a Build/BuildRun for a direct volume conflicts with a mountPath used by
a BuildStrategy volume, or another direct volume, this should result in a validation error during
Build/BuildRun admission to prevent ambiguity.


### Test Plan


### Release Criteria


#### Upgrade Strategy [if necessary]


### Risks and Mitigations

What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
both security and how this will impact the larger Shipwright ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

## Drawbacks

- All-Step Mounting: Mounting directly defined volumes to all primary build step containers by
default offers simplicity but lacks granularity. If a volume is only needed in one specific step,
it will still be mounted in others. This could slightly clutter the Pod spec and provide
wider-than-necessary access within the Pod's lifecycle. However, for common use-cases like certificates
or shared workspace, this broad availability is often desired. The alternative, per-step mount
configuration, would significantly increase API complexity.
- Potential for mountPath Overlap: While validation should prevent this, users need to be mindful
of mountPaths used by the underlying build tools within the strategy containers to avoid unintentional behavior.

## Alternatives

- Dedicated Certificate Management Feature:

  - Description: Introduce a specific API section within Build/BuildRun solely for providing CA
  certificates or other trust bundles, rather than using general-purpose volumes.
  - Reason for Not Choosing: While this would address the self-signed certificate use case directly,
  it is less flexible. General-purpose volumes can solve a wider range of problems, such as providing
  configuration files, tools, or shared workspace artifacts, beyond just certificates. The proposed
  solution leverages existing Kubernetes volume concepts which are familiar to users.

- Relying Solely on BuildStrategy Volume Overrides:

  - Description: Not implement this feature and require all volumes to be predefined in BuildStrategy
  and then overridden in Build/BuildRun.
  - Reason for Not Choosing: This lacks flexibility for users who don't control the BuildStrategy
  or when a BuildStrategy is designed to be generic. It forces strategy authors to anticipate all
  possible volume needs, which is impractical.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new subproject, repos
requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources started right away.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation History`.


