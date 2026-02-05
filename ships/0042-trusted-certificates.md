---
title: trusted-certificates
authors:
  - "@sayan-biswas"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-11-11
last-updated: 2025-12-11
status: provisional
see-also:
replaces:
superseded-by: []
---

# SHIP-0042: Trusted Certificates

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]


## Summary

This proposal aims to enhance Shipwright by allowing users to add user certificates in the build container's
system trust store by extending the Build APIs to let user define a list of Kubernetes Secrets/ConfigMaps, which will
hold the certificate data and will be added to the trust store.
This capability is essential for scenarios requiring access to external resources like self-signed
certificates for Git/image registries or other necessary build artifacts, independent of BuildStrategy
definitions, overcoming a current limitation where all volumes must be pre-defined or overridable via the strategy.

## Motivation

Currently, Shipwright Builds face challenges when needing to utilize resources like self-signed
certificates for interacting with private Git repositories or image registries.

### Goals

- Extend Build resources API to declare a list of Secrets/ConfigMaps that holds certificate data.
- Ensure these certificates are loaded and accessible to all primary containers involved
in executing the build steps throughout the lifecycle of the build's execution Pod.

### Non-Goals

What is out of scope for this proposal? 
Listing non-goals helps to focus on discussion and make progress.

## Proposal

### User Stories

#### Using self-signed certificates

As a user, I want to use Git repositories or image registries that are secured with self-signed
certificates. These certificates need to be accessible by the build process, potentially across
multiple steps (e.g., source cloning, image push/pull).

### Implementation Notes

The current BuildRun APIs will be extended to support a list of certificates accepted as kubernetes Secret
resources that should be loaded in the container's trust store.

```yaml
apiVersion: shipwright.io/v1beta1 # Ensure this is the correct API group
kind: Build
metadata: 
  namespace: test
  # ...
spec:
  # ...
  certificates:
    - name: custom-cert-1
      secretName: ca-cert-1
      mountPath: /etc/ssl/cert # This is optional and will default to configuration from system parameter (see below)
    - name: custom-cert-2
      ConfigMapName: ca-cert-2
```

The default path where the certificates will be mounted is defined in the system parameter API of the build strategy.

https://shipwright.io/docs/build/buildstrategies/#system-parameters

params.shp-certificate-directory

Consideration for mount point:
- BuildStrategy should always have a default mount path set for certificates. This shouldn't be hardcoded or assumed.
- Use subPath: Unless you want to delete all existing system certificates, use subPath.
- Environment Variables: For Node.js, additionally set `NODE_EXTRA_CA_CERTS=/path/to/your/bundle.pem`. For Python (Requests), set `REQUESTS_CA_BUNDLE`.
- Java: Use the jks or pkcs12 target in your trust-manager Bundle spec and mount it to the Java cacerts path `$JAVA_HOME/lib/security/cacerts`.
- Mount points should not overwrite a symlink.

Note:
<br> Debian-based (Ubuntu, Debian, Alpine): Use `/etc/ssl/certs/ca-certificates.crt`.
<br> Red Hat-based (UBI, RHEL, Fedora): Use `/etc/pki/tls/certs/ca-bundle.crt`.


If a certificate with the same name is defined at multiple levels, the following precedence will apply
for its definition (e.g., which Secret or ConfigMap to use):

- BuildRun.spec.certificates (highest precedence)
- Build.spec.certificates
- BuildStrategy.spec.certificates (lowest precedence, acts as a default or template)

- For new, directly mounted volumes, a mountPath must be specified.
- The name of the certificate must be unique within the list of certificates defined in any of Build, BuildRun, or BuildStatey.
- Standard Kubernetes validation for the source secret will apply.

#### Distributing Certificates
Although users can create their own ConfigMap and Secret with the certificates data, we have few options which automate this distribution of trust anchors:
- Using [TrustManager](https://cert-manager.io/docs/trust/trust-manager/) generate trust bundle and mount it in workload during reconciliation.
- Use [Cluster trust bundles](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#cluster-trust-bundles) and [clusterTrustBundle projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/#clustertrustbundle) to mount it in workload.


### Test Plan

### Release Criteria


#### Upgrade Strategy [if necessary]


### Risks and Mitigations

What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
both security and how this will impact the larger Shipwright ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

## Drawbacks

TBD

## Alternatives

TBD

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new subproject, repos
requested, GitHub details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources started right away.

## Implementation History

- 2025-11-11: Initial Draft
