<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-scheduler-options
authors:
  - "@adambkaplan"
reviewers:
  - "@apoorvajagtap"
  - "@HeavyWombat"
approvers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
creation-date: 2024-05-15
last-updated: 2024-06-20
status: Implementable
see-also: []
replaces: []
superseded-by: []
---

# Build Scheduler Options

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

- Should this be enabled always? Should we consider an alpha -> beta lifecycle for this feature? (ex: off by default -> on by default)

## Summary

Add API options that influece where `BuildRun` pods are scheduled on Kubernetes. This can be
acomplished through the following mechanisms:

- [Node Selectors](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Custom Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

## Motivation

Today, `BuildRun` pods will run on arbitrary nodes - developers, platform engineers, and admins do
not have the ability to control where a specific build pod will be scheduled. Teams may have
several motivations for controlling where a build pod is scheduled:

- Builds can be CPU/memory/storage intensive. Scheduling on larger worker nodes with additional
  memory or compute can help ensure builds succeed.
- Clusters may have mutiple worker node architectures and even OS (Windows nodes). Container images
  are by their nature specific to the OS and CPU architecture, and default to the host operating
  system and architecture. Builds may need to specify OS and architecture through node selectors.
- The default Kubernetes scheduler may not efficiently schedule build workloads - especially
  considering how Tekton implements step containers and sidecars. A custom scheduler optimized for
  Tekton or other batch workloads may lead to better cluster utulization.

### Goals

- Allow build pods to run on specific nodes using node selectors.
- Allow build pods to tolerate node taints.
- Allow build pods to use a custom scheduler.

### Non-Goals

- Primary feature support for multi-arch builds.
- Allow node selection, pod affinity, and taint toleration to be set at the cluster level.
  While this may be desirable, it requires a more sophisticated means of configuring the build
  controller. Setting default values for scheduling options can be considered as a follow-up
  feature.
- Prevent use of build pod scheduling fields. This is best left to an admission controller like
  [OPA Gatekeeper](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/) or
  [Kyverno](https://kyverno.io/).
- Allow build pods to set node affinity/anti-affinity rules. Affinity/anti-affinity is an
  incredibly rich and complex API (see [docs](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)
  for more information). We should strive to provide a simpler interface that is tailored
  specifically to builds. This feature is being dropped to narrow the scope of this SHIP. Build
  affinity rules can/should be addressed in a follow up feature.

## Proposal

### User Stories

#### Node Selection - platform engineer

As a platform engineer, I want builds to use node selectors to ensure they are scheduled on nodes
optimized for builds so that builds are more likely to succeed

#### Node Selection - arch-specific images

As a developer, I want to select the OS and architecture of my build's node so that I can run
builds on worker nodes with multiple architectures.

#### Taint toleration - cluster admin

As a cluster admin, I want builds to be able to tolerate provided node taints so that they can
be scheduled on nodes that are not suitable/designated for application workloads.

#### Custom Scheduler

As a platform engineer/cluster admin, I want builds to use a custom scheduler so that I can provide
my own scheduler that is optimized for my build workloads.

### Implementation Notes

#### API Updates

The `BuildSpec` API for Build and BuildRun will be updated to add the following fields:

```yaml
spec:
  ...
  nodeSelector: # map[string]string
    <node-label>: "label-value"
  tolerations: # []Toleration
    - key: "taint-key"
      operator: Exists|Equal
      value: "taint-value"
  schedulerName: "custom-scheduler-name" # string
```

The `nodeSelector` and `schedulerName` fields will use golang primitives that match their k8s
equivalents.

#### Tolerations

The Tolerations API for Shipwright will support a limited subset of the upstream Kubernetes
Tolerations API. For simplicity, any Shipwright Build or BuildRun with a toleration set will use
the `NoSchedule` [taint effect](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

```yaml
spec:
  tolerations: # Optional array
    - key: "taint-key" # Aligns with upstream k8s taint labels. Required
      operator: Exists|Equal # Aligns with upstream k8s - key exists or node label key = value. Required
      value: "taint-value" # Alights with upstream k8s taint value. Optional.
```

As with upstream k8s, the Shipwright Tolerations API array should support
[strategic merge JSON patching](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#notes-on-the-strategic-merge-patch).

#### Precedence Ordering and Value Merging

Values in `BuildRun` will override those in the referenced `Build` object (if present). Values for
`nodeSelector` and `tolerations` should use strategic merge logic when possible:

- `nodeSelector` merges using map keys. If the map key is present in the `Build` and `BuildRun`
  object, the `BuildRun` overrides the value.
- `tolerations` merges using the taint key. If the taint key is present in the `Build` and
  `BuildRun` object, the `BuildRun` overrides the value.

This allows the `BuildRun` object to "inherit" values from a parent `Build` object.

#### Impact on Tekton TaskRun

Tekton supports tuning the pod of the `TaskRun` using the
[podTemplate](https://tekton.dev/docs/pipelines/taskruns/#specifying-a-pod-template) field. When
Shipwright creates the `TaskRun` for a build, the respective node selector, tolerations, and
scheduler name can be passed through.

#### Command Line Enhancements

The `shp` CLI _may_ be enhanced to add flags that set the node selector, tolerations, and custom
scheduler for a `BuildRun`. For example, `shp build run` can have the following new options:

- `--node=<key>=<value>`: Use the node label key/value pair in the selector. Can be set more than
  once for multiple key/value pairs..
- `--tolerate=<key>` or `--tolerate=<key>=<value>`: Tolerate the taint key, in one of two ways:
  - First form: taint key `Exists`.
  - Second form: taint key `Equals` provided value.
  - In both cases, this flag can be set more than once.
- `--scheduler=<name>`: use custom scheduler with given name. Can only be set once.


#### Hardening Guidelines

Exposing `nodeSelector` and `tolerations` to end developers adds risk with respect to overall
system availability. Some platform teams may not want these Kubernetes internals exposed or
modifiable by end developers at all. To address these concerns, a hardening guideline for
Shipwright Builds should also be published alongside documentation for this feature. This guideline
should recommend the use of third party admission controllers (ex: OPA, Kyverno) to prevent builds
from using values that impact system availability and performance. For example:

- Block toleration of `node.kubernetes.io/*` taints. These are reserved for nodes that are not
  ready to receive workloads for scheduling.
- Block node selectors with the `node-role.kubernetes.io/control-plane` label key. This is reserved
  for control plane components (`kube-apiserver`, `kube-controller-manager`, etc.)
- Block toleration of the `node-role.kubernetes.io/control-plane` taint key. Same as above.

See the [well known labels](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-role-kubernetes-io-control-plane)
documentation for more information.

### Test Plan

- Unit testing can verify that the generated `TaskRun` object for a build contains the desired pod
  template fields.
- End to end tests using `KinD` is possible for the `nodeSelector` and `tolerations` fields:
  - KinD has support for configuring multiple [nodes](https://kind.sigs.k8s.io/docs/user/configuration/#nodes)
  - Once set up, KinD nodes can simulate real nodes when it comes to pod scheduling, node labeling,
    and node taints.
- End to end testing for the `schedulerName` field requires the deployment of a custom scheduler,
  plus code to verify that the given scheduler was used. This is non-trivial (see
  [upstream example](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/#specify-schedulers-for-pods))
  and adds a potential failure point to the test suite. Relying on unit testing alone is our best
  option.


### Release Criteria

TBD

**Note:** *Section not required until targeted at a release.*

#### Removing a deprecated feature [if necessary]

Not applicable.

#### Upgrade Strategy [if necessary]

The top-level API fields will be optional and default to Golang empty values.
On upgrade, these values will remain empty on existing `Build`/`BuildRun` objects.


### Risks and Mitigations

**Risk:** Node selector field allows disruptive workloads (builds) to be scheduled on control plane
nodes.

*Mitigation*: Hardening guideline added as a requirement for this feature. There may be some
cluster topologies (ex: single node clusters) where scheduling builds on the "control plane" is not
only desirable, but necessary. Hardening guidelines referencing third party admission controllers
preserves flexibility while giving cluster administrators/platform teams the knowledge needed to
harden their environments as they see fit.


## Drawbacks

Exposing these fields leaks - to a certain extent - our abstraction over Kubernetes. This proposal
places k8s pod scheduling fields up front in the API for `Build` and `BuildRun`, a deviation from
Tekton which exposes the fields through a `PodTemplate` sub-field. Cluster administrators may not
want end developers to have control over where these pods are scheduled - they may instead wish to
control pod scheduling through Tekton's
[default pod template](https://github.com/tektoncd/pipeline/blob/main/docs/podtemplates.md#supported-fields)
mechanism at the controller level.

Exposing `nodeSelector` may also conflict with future enhancements to support
[multi-architecture image builds](https://github.com/shipwright-io/build/issues/1119). A
hypothetical build that fans out individual image builds to nodes with desired OS/architecture
pairs may need to explicitly set the `k8s.io/os` and `k8s.io/architecture` node selector fields on
generated `TaskRuns`. With that said, there is currently no mechanism for Shipwright to control
where builds execute on clusters with multiple worker node architectures and operating systems.


## Alternatives

An earlier draft of this proposal included `affinity` for setting pod affinity/anti-affinity rules.
This was rejected due to the complexities of Kubernetes pod affinity and anti-affinity. We need
more concrete user stories from the community to understand what - if anything - we should do with
respect to distributing build workloads through affinity rules. This may also conflict with
Tekton's [affinity assistant](https://tekton.dev/docs/pipelines/affinityassistants/) feature - an optional configuration that is enabled by default in upstream Tekton.

An earlier draft also included the ability to set default values for these fields at the cluster
level. This would be similar to Tekton's capability with the Pipeline controller configuration.
Since this option is available at the Tekton pipeline level, adding nearly identical features to
Shipwright is being deferred. Tuning pod template values with the Tekton pipeline controller may
also be an acceptable alternative to this feature in some circumstances.


## Infrastructure Needed [optional]

No additional infrastructure antipated.
Test KinD clusters may need to deploy with additional nodes where these features can be verified.

## Implementation History

- 2024-05-15: Created as `provisional`
- 2024-06-20: Draft updated to `implementable`
