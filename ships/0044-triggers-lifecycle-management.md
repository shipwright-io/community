<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: triggers-lifecycle-management
authors:
- "@sayan-biswas"
  reviewers:
- TBD
  approvers:
- TBD
  creation-date: 2025-12-15
  last-updated: 
  status: provisional
  see-also:
  replaces: []
  superseded-by: []
---

# SHIP-0044: Manage Triggers Lifecycle

This enhancement adds APIs in the `ShipwrightBuild` resource to deploy trigger from Shipwright Operator.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions

TBD

## Summary

This enhancement aims to add APIs and the corresponding logic to manage the lifecycle of the `Triggers` component from `Shipwright Operator`

## Motivation

The Shipwright community already has a functional project [Triggers](https://github.com/shipwright-io/triggers),
which aims to automate and greatly simplify the container image building process. But the component needs to be
deployed separately as of now. Shipwright Operator can be enhanced to manage the life cycle of this component.

### Goals

- There should be APIs to Enable/Disable as well as configure few aspects of `Triggers`
- Maintain backward compatibility with by disabling this as default
- Should be able to install and track the status of `triggers` controller

### Non-Goals

- Modifying the core triggers logic

## Proposal

### Implementation Notes

The new  field will be added to the `ShipwrightBuild` struct with the following structure:

```yaml
---
apiVersion: operator.shipwright.io/v1alpha1
kind: ShipwrightBuild
metadata:
  name: shipwright-operator
spec:
  targetNamespace: shipwright-build
  triggers:
      state: Enabled
      # Any other configurations specific to triggers
status:
  # Standard reconciling status goes here.
```

Also renaming the resource `kind` from `ShipwrightBuild` to `ShipwrightConfig` seems more apt, as `Build` becomes more specific to the Shipwright Build component.

### Test Plan

- Unit tests to check reconciliation of the new fields
- End-to-end tests to ensure successful deployment of `Triggers`
- Tests to verify backward compatibility 

### Release Criteria

TBD

#### Removing a deprecated feature
Not applicable - this is a new field addition.

#### Upgrade Strategy

TBD

### Risks and Mitigations

TBD

## Drawbacks

TBD

## Alternatives

TBD

## Infrastructure Needed

None - this enhancement only requires changes to the existing Shipwright Operator and API definitions.

## Implementation History

TBD
