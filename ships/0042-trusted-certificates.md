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

- Extend `Build` and `BuildRun` resources API to declare a Secrets/ConfigMaps that holds certificate data.
- Ensure these certificates are loaded and accessible to all primary containers involved
in executing the build steps throughout the lifecycle of the build's execution Pod.

### Non-Goals

What is out of scope for this proposal?
Although users can create their own ConfigMap and Secret with the certificates data, we have few options which automate this distribution of trust anchors:
- Using [TrustManager](https://cert-manager.io/docs/trust/trust-manager/) generate trust bundle and mount it in workload during reconciliation.
- Use [Cluster trust bundles](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#cluster-trust-bundles) and [clusterTrustBundle projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/#clustertrustbundle) to mount it in workload.

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
  caBundle: # can define either a secret or a configmap
    secret: 
      name: gitea-tls # name of the secret in the same namespace that holds the CA data
      key: tls.crt # key in the secret data to look for. We don't want to mount every key
    configMap: 
      name: gitea-tls # name of the secret in the same namespace that holds the CA data
      key: tls.crt # key in the secret data to look for. We don't want to mount every key
```

The CA bundle will be mounted on common locations is linux distributions. 
Reference: https://go.dev/src/crypto/x509/root_linux.go
```text
	"/etc/ssl/certs/ca-certificates.crt",                // Debian/Ubuntu/Gentoo etc
	"/etc/pki/tls/certs/ca-bundle.crt",                  // Fedora/RHEL
```

Consideration for mount point:
- Certificate defines a CA Bundle. 
- User should provide the CA bundle in a secret or configmap in the same namespace as Build and BuildRun resource.
- Priority of certificate spec: BuildRun >> Build
- Use subPath: Sub paths will be used mount for single file injection. This method doesn't support auto-update of when secret is modified. In CI build context, this not relevant.
- Environment Variables: Mount certain environment variables..
  - Node.js `NODE_EXTRA_CA_CERTS`
  - Python `REQUESTS_CA_BUNDLE`
  - Openssl `SSL_CERT_FILE`
  - Curl CA bundle	`CURL_CA_BUNDLE`
- Java: Use the jks or pkcs12 target in your trust-manager Bundle spec and mount it to the Java cacerts path `$JAVA_HOME/lib/security/cacerts`.
- Mount points should not overwrite a symlink.
- The mounted path will be available as system parameter at `shp-ca-bundle`, so that it can be references in the `BuildRun` resource.

If a certificate with the same name is defined at multiple levels, the following precedence will apply
for its definition (e.g., which Secret or ConfigMap to use):

- `BuildRun.spec.caBundle` (highest precedence) >> `Build.spec.caBundle`
- For new, directly mounted volumes, a mountPath and subPath must be specified.
- Volume names to have postfix (hash or randon) to avoid conflict
- Standard Kubernetes validation for the source secret will apply.
- A `CABundle` definition can have either Secret and ConfigMap reference.
- Validation will be done to check if the referenced resource exists in the cluster and namespace. 
- Syntax validation of CA data in the secret/configmap to be done.

### Test Plan

TBD

### Release Criteria

TBD

#### Upgrade Strategy [if necessary]

This API change doesn't break any existing functionalities. Upgrade strategy is not relevant.

### Risks and Mitigations

- File injection into the operating system's trust store will replace the existing CA bundle which might have CAs for several public domains.
    - This can be solved by mounting the certificates in common places (different linux distros) which provide to mount user certs and then allow OS to index and place it in the right place. (Requires Pod Lifecycle hook, not available in tekton yet)
    - Instead of replacing the CA bundle in the system, append the content of the new CA to the existing one. (Requires Pod Lifecycle hook, not available in tekton yet)

## Drawbacks

- If we don't want to replace the existing certificate, then we have to mount the system's certificate directory (eg: /etc/ssl/cert) as a volume, since we have ReadOnlyRootFileSystem enabled.

## Alternatives

- Define volumes directly in the Strategy and allow Build to override it. Mount volumes to git and image-push containers and user manages everything from defining secret and mounting it to the correct location. Already proposed in [https://github.com/shipwright-io/community/pull/265]

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new subproject, repos
requested, GitHub details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources started right away.

## Implementation History

- 2025-11-11: Initial Draft
- 2026-02-23: Updated Draft
