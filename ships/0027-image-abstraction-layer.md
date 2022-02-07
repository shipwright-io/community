---
title: shipwright-image-abstraction-layer
authors:
  - "@ricardomaraschini"
reviewers:
  - "@ImJasonH"
  - "@adambkaplan"
  - "@GabeMontero"
  - "@SaschaSchwarze0"
  - "@qu1queee"
approvers:
  - "@ImJasonH"
  - "@adambkaplan"
  - "@GabeMontero"
  - "@SaschaSchwarze0"
  - "@qu1queee"
creation-date: 2022-02-07
last-updated: 2022-02-07
status: implementable
see-also:
  - "/ships/0020-shipwright-image-api.md"
---
# Shipwright Image Abstraction layer

This enhancement proposal suggests an image management abstraction layer for a Kubernetes cluster
in the form of an Operator; this abstraction layer is directly inspired by OpenShift’s ImageStream.
The idea behind this EP is to design the base API types needed to support such an abstraction as
well as lay down a basic set of features aiming to have a minimal viable product to further
validate the idea. As chosen set of features intended for the minimal viable product are:

* The ability to create tag-to-hash references for Containers Images hosted in a remote registry.
* The ability of mirroring remote images in an  “on-premise” image registry.

## Motivation

Keeping track of all Container Images in use in a Kubernetes cluster is a complicated task.
Container Images may come from numerous different Image Registries. To add to this, Container
Runtimes rely on remote registries (from the cluster's point of view) when obtaining Container
Images, potentially making the process of pulling their blobs (manifests, config, and layers)
slower.

The notion of indexing Container Image versions by tags (“latest”, “development”, etc) is helpful.
Still it does not provide users with the full confidence to always use the intended Container
Image – today's "latest" tag might not be tomorrow's "latest" tag. In addition to that, these
Image Registries allow access to Container Images by their Manifest content's hash (i.e., usually
sha256), which gives users full confidence at a cost in semantics. Other factors may pop up when
running, for instance, in an air-gapped environment, where the cluster nodes may not be allowed to
reach any external Image Registry.

This enhancement proposal suggests an image management abstraction layer. For instance, by
providing a direct mapping between a Container Image tag (e.g., "latest") and its corresponding
Manifest content's hash, users can then refer to the Container Image by its name – and be sure to
use that specific version.

In summary, the proposed operator mirrors and indexes remote Container Images into a Kubernetes
cluster, allowing for other Operators to easily navigate through these multiple indexed Images.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

Design the Kubernetes API types (CRDs) and create a minimal viable product for such an abstraction
layer.

### Non-Goals

At this stage to provide the integration with Shipwright’ builds is not a goal of this proposal,
this will need to be tackled once we have the basic types and functionality in place. To provide
Image storage for Container Images in the form of an integrated Image Registry inside the cluster
is also not part of the scope of this proposal.

## Proposal

The operator would leverage a _custom resource definition_ called Image. An Image represents an
image tag in a remote registry. For instance, an Image called `myapp-devel` may be created to keep
track of the image `quay.io/company/myapp:latest`. A Image _custom resource_ layout would look
like this:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: Image
metadata:
  name: myapp-devel
  namespace: my-namespace
spec:
  source: quay.io/company/myapp:latest
  mirror: false
  insecure: false
```
The meaning of each property in an `Image.Spec` is as follow:

| Property   | Description                                                                       |
| ---------- | --------------------------------------------------------------------------------- |
| source     | Indicates the source of the image (from where to import it)                       |
| mirror     | Informs if the Image should be mirrored to another registry                       |
| insecure   | Indicates that we should skip tls verification during the image import/mirror     |

Once such a custom_ resource_ is created (with `kubectl create`, for example) the operator won’t
act on it immediately but it will instead wait until an ImageImport, referring to the created
Image, is instantiated by the user. Upon ImageImport creation the operator will attempt to process
the ImageImport and populate the created Image with the resulting tag-to-hash reference. Once an
ImageImport has been processed and the tag-to-hash reference is populated inside the Image object
the ImageImport is considered "finished" and no further processing will be executed.

An ImageImport object looks like:

```
apiVersion: shipwright.io/v1alpha1
kind: ImageImport
metadata:
  name: myapp-import-00
  namespace: my-namespace
spec:
  image: myapp-devel
  source: quay.io/company/myapp:latest
  mirror: false
  insecure: false
```

All Image properties also show up in an ImageImport object, so, if the user does not provide data
for these properties in the ImageImport the operator will use whatever is set in the Image instead.
Follow a list of meaning for each property in an ImageImport.Spec:

| Property    | Description  |
| ----------- | ------------ |
| image       | The Image to which this ImageImport refers to. This is a mandatory field. (required) |
| source      | Indicates the source of the image (from where the operator should import it), if not set the operator will use the spec.image.source by default. (optional) |
| mirror      | Informs if the Image should be mirrored in the configured mirror registry. If not set image.Spec.Mirror is used instead. (optional) |
| insecure    | Indicates if the remote registry hosting the image is using “invalid certificates”, i.e. skip TLS verification. If not set image.Spec.Insecure is used instead. (optional) |

If an ImageImport has in its `spec.image` an Image that does not exist yet one should be created
by the operator before starting to process the ImageImport object.

### Mirroring images

If mirroring is set in an Image or ImageImport the operator will attempt to mirror the image
content into another registry provided by the user. To mirror images locally one needs to inform
the operator about the mirror registry location and authentication methods. Such mirror registry
could be configured through a Secret called `mirror-registry-config`, this secret would contain
the following properties:

| Name       | Description                                                                       |
| -----------| --------------------------------------------------------------------------------- |
| address    | The mirror registry URL                                                           |
| username   | Username                                                                          |
| password   | The password to be used                                                           |
| token      | The auth token to be used                                                         |
| insecure   | Allows access to insecure registries if set to "true"                             |
| repository | If set all images are going to be mirrored inside this registry repository        |

### Image status

The status field for a given Image should contain all its previously imported versions of the
same image sorted by import time. Follow below the proposed properties found in an Image `.status`
property and their meaning:

| Name              | Description                                                                |
| ----------------- | -------------------------------------------------------------------------- |
| hashReferences    | A list of all imported versions of the Image. Sorted by import time descending (the first item in this list is the last imported version) |

The property `.status.hashReferences` is a list of imported image versions, The operator would
hold up to twenty five imports per image. Each item in the list is composed of the following
properties:

| Name           | Description  |
| -------------- | ------------ |
| source         | Keeps a reference from where the image got imported |
| importedAt     | Date and time of the import |
| imageReference | Where this reference points to (by hash), may point to the mirror registry in case of a mirrored Image |

### ImageImport status

On a ImageImport status, one would be able to find information about the import attempts executed
by the operator, it looks like:

| Name           | Description  |
| -------------- | ------------ |
| hashReference  | Once the import has succeeded this is populated with the data about the image. Properties for hashReference are equal to Image.Status.HashReferences above. This is what is copied into the Image.Status.HashReferences list. |
| conditions     | An slice of metav1.Condition{}, represents the import state |
| importAttempts | A list of all attempts made at processing the ImageImport |

The properties for the object `.status.hashReference` are exactly the same as for the objects
inside `Image.Status.HashReferences`. As for `status.importAttempts` objects they contain the
following properties:

| Name    | Description                                                                          |
| ------- | ------------------------------------------------------------------------------------ |
| when    | Date and time of the import attempt                                                  |
| succeed | A boolean indicating if the import was successful or not                             |
| reason  | In case of failure (succeed = false), what was the error                             |

The operator must try to import multiple times the same ImageImport in case of failures. These
imports should implement exponential backoff to avoid “too many requests” from remote registries.
After ten failures the operator should stop trying to process the ImageImport.


### User Stories

As an operator I want to be able to create a tag-to-hash reference to an image hosted in a remote
registry so I can easily refer to it by its hash instead of by tag.

As an operator I want to be able to mirror an image in my registry so I can take ownership of it
and possibly gain some performance while pulling.


### Implementation Notes

To import images hosted in registries protected by authentication the operator would need to be
able to read docker credentials secrets that exist in the same namespace where the Image exists.
The idea is to mimic OpenShift’s Image Stream here: read all docker config secrets from the target
namespace and use the appropriate one (if present).

### Test Plan

As with the Build APIs today, this operator should include thorough and high-quality unit tests,
integration tests, and end-to-end tests. End to end tests are to be designed using kuttl utility.

### Release Criteria

Not currently targeting a release.


### Risks and Mitigations

**Confusion with OpenShift’s ImageStream implementation**

As presented above OpenShift’s also has an Image object that may cause confusion. This should be
documented and very clear to the user from day one.


## Implementation History

None so far.
