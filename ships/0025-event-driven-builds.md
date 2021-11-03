<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: event-driven-builds
authors:
  - "@sbose787"
  
reviewers:
  - "@gmontero"
  - "@adamkaplan"
  - "@ImJasonH"
  - "@SaschaSchwarze0"
  - "@HeavyWombat"
  
approvers:
  - "@adamkaplan"
  - "@SaschaSchwarze0"


 
 
creation-date: 2021-11-03
status: implementable

---

# Git Event-driven triggering of Shipwright Builds


## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

1. Almost every user would need an exposed `Service`, how do we create a vendor-agnostic `Ingress` object ?


## Summary

Trigger the execution of an image build based on a commit/push event from a relevant source code repository. 



## Motivation

This enhancement proposal aims to provide an API to enable users to express the intent of having their builds triggered by events from a Git Repository.


### Goals

* Build a technology-agostic user experience for Git-triggered build executions.
* Add support for 'reacting' to events from Github and Gitlab repositories.



### Non-Goals

* Support events from generic git servers. The implementation should be extensible to support those in a non-breaking manner.
* Creation of the vendor-specific Ingress resources to expose the webhook URL.


## Proposal


### User Stories [optional]

Detail the things that people will be able to do if this is implemented. Include as much detail as
possible so that people can understand the "how" of the system. The goal here is to make this feel
real for users without getting bogged down.

#### Story 1
As a user, I would like to define a `Build` and trigger the execution of the same upon pushes to my Git Repo

#### Story 2
As a user, I would like to define a `Build` and the execution of the same for pushes to any branch in that Git Repo.

#### Story 3
As a user, I would like to configure a secure webhook URL for triggering `Builds`.


### Implementation Notes

Upon specification of the **new** API field `.spec.webhook`, the Shipwright Build Controller will do the needful to generate a webhook URL and 
provide the information on the same in the `status` of the `Build` resource.


```
spec:
  ...
  ...
  webhook:
    type: github
    imageTagPolicy: short_sha    # optional, allowed values: 'short_sha' , 'branch'. Defaults to 'branch'.
    secretRef:                   # optional, will be genereated if not specified.
      name: my-webhook-secret.  
```

The `.status` sub-resource would contain the information

```
status:
  webhook:
    status: live
    reason: ""          # to be populated in case of error.
    secretRef:          # mandatory field, in-secure not an option.
      name: _user_specified_or_generated
    serviceRef:         # kubernetes service which needs to be exposed.
      name: _name_of_the_recieveing_webhook_traffic  
```

Here's what a full Build resource would look like :
```
kind: Build
metadata:
  name: buildpack-nodejs-build
spec:
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
  webhook:
    type: github
    imageTagPolicy: short_sha    # optional, allowed values: 'short_sha' , 'branch'. Defaults to 'branch'.
    secretRef:                   # optional, will be genereated if not specified.
    name: my-webhook-secret. 
 status:
  ...
  ...
  webhook:
    status: live        
    reason: ""          # to be populated in case of error.
    secretRef:          # mandatory field, in-secure not an option.
      name: _user_specified_or_generated
    serviceRef:         # kubernetes service which needs to be exposed.
      name: _name_of_the_recieveing_webhook_traffic
 ```
 
 #### Pre-requisities
 
The following items ( part of "Bill of materials" ) would need to be shipped with the Shipwright installation so that they could be consumed in the webhook-generation process:

1. A `ClusterTriggerBinding` which exposes `$(body.head_commit.id)` and `$(body.ref)` from the webhook payload.

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: ClusterTriggerBinding
metadata:
 name: github-shipwright-webhook
spec:
  params:
    - name: commit
      value: $(body.head_commit.id)
    - name: branch
      value: $(body.ref)
```

Similar `ClusterTriggerBinding`s need to be shipped for `Gitlab` and `BitBucket`.

2. A "Custom Tekton Task" controller with the following behaviour:
    * _watches_ `Tekton` `Run` resources referencing Shipwright's `Build` resources.
    * Expects the commit ID and branch name in the `params`.
    * Based on the above information, the controller would generate the following:
          * A `Build` with the revision and output image tag overwritten with the above information based on the optional `.spec.webhook.imageTagPolicy`.
          * A `BuildRun` referencing the above `Build`.



#### Webhook endpoint creation process

 
Upon creation of a `Build` resource by the user, the following would occur:

1. The Shipwright Build Controller would create a `TriggerTemplate` that would map the information coming in from the event into the "parameters" 
   expected by the `Run` resource ie, the branch name & the commit ID.
   
 ```
 
### Generated by the `Build` reconciler. 
 
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
 name: my-build-name
spec:
 params:
   - name: branch
   - name: commit
 resourceTemplates:
   - apiVersion: tekton.dev/v1alpha1
     kind: Run
     metadata:
       generateName: build-execution-
     spec:
       ref:
         apiVersion: shipwright.io/v1alpha1
         kind: Build 
         name: my-build-bame
       timeout: 3000s
       params:
         - name: branch
           value: $(tt.params.branch)
         - name: commit
           value: $(tt.params.commit)
 ```

2. The Shipwright Build Controller would create a `EventListener` under-the-hood per build. 
    
```
### Generated by the `Build` reconciler. 

apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
 name: name-of-the-build
spec:
 serviceAccountName: pipeline
 triggers:
   - bindings:
       - ref: github-shipwright-webhook
     template:
       name: name-of-the-build
   
```

And that's it, you may now go ahead and 

### Test Plan

To be filled.

### Release Criteria

To be filled.

#### Removing a deprecated feature [if necessary]

N/A

#### Upgrade Strategy [if necessary]

N/A

### Risks and Mitigations

1. This feature opens up the possiblility of triggering Build executions for branches which weren't explicitly specified. 

This isn't a risk per se since repo committers would have the permissions needed to push a branch to the repo. Therefore,
as long as the image is tagged appropriately to indiciate that it is not built off 'main' ( or the branch specified iniially by the user in the `Build`), 
this should not be a problem.

While the enhancement could have been scoped to only deal with the branch/revision explicitly specified in the `Build` resource, 
doing the same would have have excluded some key use cases where users push to a different branch before merging to 'main'. 

2. Exposing a webhook URL enables creation of pods ( ie, processes on the node ) by actors who may not necessarily have access to the cluster.

This does open up an attack vector given the execution of the build is done using the configured service account. 

* Webhook-driven builds are being designed to be secure by default - the usage of a webhook secret is mandatory.
* The resulting `BuildRun` would be annotated with relevant metadata from the webhook event so that it's easy to trace the actor responsible for the trigger. 


## Drawbacks

1. Is this even a Shipwright concern or should this be the concern of a general-purpose CI ?
( to be filled ) 




## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
possible approaches to delivering the value proposed by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new subproject, repos
requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources started right away.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation History`.

