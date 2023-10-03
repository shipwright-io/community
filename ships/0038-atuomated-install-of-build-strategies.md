<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: Automated-install-of-build-strategies
authors:
  - "@jkhelil"
reviewers:
  - "@adambkaplan"
  - "@qu1queee"
approvers:
  - "@adambkaplan"
  - "@qu1queee"
creation-date: 2023-10-02
last-updated: 2023-10-02
status: implementable
---

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]


## Summary

This ship proposes support of installing a catalog of Build Strategies
via the shipwright operator in order to improve the user experience around Shipwright Build

## Motivation

Actually with the Shipwright operator, the user installs the Shipwright Build Controller. 
Installation and upgrade of Build Strategies and Cluster Strategies is currently a manual process
In order to improve user experience around Shipwright, the Shipwright Operator should also install the build strategies.

### Goals

- Install build strategies via the shipwright operator
- Define build strategies catalog source
- Allow a user to install a provided build strategies catalog via the Shipwright Operator
 
### Non-Goals

- Add a new operator to install the Build Strategy catalog
- Curate or rework the Build Strategies in the upstream build repository when copied to the new catalog repository

## Proposal

We propose to add new field `strategyCatalog` to the `spec` of ShipwrightBuild CRD.
The `strategyCatalog` field represents the type of source strategy catalog we use as source to install build strategies.
In the current SHIP we propose one type of source catalog :
- strategy catalog of type: default. the operator installs a manifest of build strategies from the kodata directory
- strategy catalog of type http representing an url of a manifest file containing the strategies.
- strategy catalog of type ociArtifcat

```yaml
apiVersion: operator.shipwright.io/v1alpha1
kind: ShipwrightBuild
metadata:
  name: buildStrategy
spec:
  targetNamespace: shipwright-build
  strategyCatalog:
    type: http
    url:  https://github.com/redhat-developer/openshift-builds-catalog/releases/download/0.1.0/release-strategies.yaml
```
We propose to add a new controller to the Shipwright Operator to install the build strategies
The controllers should:
- Reconcile the ShipwrightBuild instance
- check that cluster build strategies CRD is installed
- check that the shipwright webhook is up and running (this is important as v1beta1 CR are submitted to the conversion webhook)
- fetch the manifests from the catalog provided source or the default local manifest and installs build strategies

### User Stories [optional]

#### Story 1

As a Kubernetes cluster administrator
I want to have a set of installed cluster strategies when i install the shipwright operator
So that developers can start using shipwright builds without any further manual steps

#### Story 2

As a Kubernetes cluster administrator
I want to be able to provide an url of manifest file and install my own catalog of Shipwright Builds Strategies via the Shipwright Operator
So that I can provide a custom subset of Shipwright Build Strategies to my developers

#### Story 3

As a Kubernetes cluster administrator
I want to be able to install a custom ociArtifact of Shipwright Builds Strategies via the Shipwright Operator
So that I can provide a custom subset of Shipwright Build Strategies to my developers


### Implementation Notes

#### API

The proposal adds the new optional field `strategyCatalog` to the `ShipwrightBuild` resource specification. it allow multiple types.
The current SHIP will add support to the `http` type. Other source catalog types can be added in the future.

The following snippet shows how the `ShipwrightBuildSpec` will be represented.

```go
type ShipwrightBuildSpec struct {
	// TargetNamespace is the target namespace where Shipwright's build controller will be deployed.
	TargetNamespace string `json:"targetNamespace,omitempty"`
  // strategyCatalog refers to a source catalog containing strategies manifests to be installed.
	StrategyCatalog StrategyCatalog `json:"strategyCatalog"`
}

type StrategyCatalog struct {
	// Type is the StrategyCatalogType qualifier, the type of the data-source.
	//
	// +optional
	Type StrategyCatalogType `json:"type,omitempty"`

	// HttpSource
	//
	// +optional
	HttpSource *Http `json:"http,omitempty"`
}

type Http struct {
	// URL describes the URL of the tarball repository.
	//
	// +optional
	URL *string `json:"url,omitempty"`
}
```

#### Controller

Actually the Shipwright Operator uses controller-runtime package to bootstrap a manager and setup the Shipwright Build controller within that manager.
In the target implementation we will continue using controller-runtime
We will split the controllers each acheiving a specific function in its own package:
- shipwrightbuild controller to reconcile ShipwrightBuild CRD and installs Shipwright Build manifests in the `targetNamespace` 
- shipwrightbuildstrategy controller to reconcile  ShipwrightBuild CRD and installs Shipwright Build Strategy catalog.
The main package will bootstrap a controller manager and adds all the controllers to the manager.

### Test Plan

The implementation has to be tested on a `unit`, `integration` and `e2e` level to ensure correctness.

### Release Criteria

tbd

### Risks and Mitigations

**Risk**: The http catalog of strategy catalog can be unreachable if the kubernetes cluster needs a proxy to access to the catalog url
**Mitigation**: A future implemenation will be proposed to handle proxy settings

**Risk**: The default strategy catalog or any url provided catalog can be unreachable if the kubernetes cluster is a disconnected cluster and does not have any access to internet
**Mitigation**: 
- the oci artifcats strategy catalog works well with disconnected clusters

## Drawbacks


## Alternatives


## Infrastructure Needed [optional]


## Implementation History

