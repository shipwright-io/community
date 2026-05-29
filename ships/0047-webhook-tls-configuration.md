<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: webhook-tls-configuration
authors:
  - "@anchi205"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-04-28
last-updated: 2026-04-28
status: provisional
see-also: []
replaces: []
superseded-by: []

---

# SHIP-0047 Webhook TLS Configuration Flags

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

The shipwright-build-webhook currently uses a hardcoded TLS configuration (TLS 1.2 minimum, a static list of ECDHE cipher suites, and 3 curve preferences). This configuration cannot be changed without modifying source code and rebuilding the binary.

This proposal defines a **platform-agnostic** approach: Shipwright will not embed any platform-specific API calls or “TLS profile” parsing logic. Instead, Shipwright will:

- stop hardcoding TLS curve preferences and TLS 1.2 cipher suite lists in the webhook's `tls.Config`, while keeping a safe baseline of `MinVersion: tls.VersionTLS12`.
- introduce **CLI flags** to configure the webhook's TLS minimum version and (optionally) TLS 1.2 cipher suites at deployment time.

Platform operators can translate a cluster TLS profile into these flags and inject them into the webhook Deployment.

## Motivation

Hardcoding TLS configuration creates a security gap:

Today, the shipwright-build-webhook hardcodes TLS 1.2 with a fixed cipher list. This means:

- The webhook can diverge from a platform's centrally managed TLS policy/profile.
- Administrators cannot adjust TLS 1.2 cipher suites or minimum TLS version without patching Shipwright.
- By pinning curves/ciphers, Shipwright can accidentally block improvements in newer Go releases (e.g., evolving defaults, hybrid PQ/TLS key exchange where available).
- Downstreams must either carry patches or accept mismatched TLS behavior.

This proposal addresses the problem in two complementary ways:

- **Remove hardcoded curves/ciphers**: avoids pinning crypto policy in code, lets Go defaults evolve (including TLS 1.3 behavior and curve preferences), and reduces the risk of Shipwright blocking improvements.
- **Add flags**: provides an escape for strict environments where compliance/policy requires pinning a minimum TLS version or a TLS 1.2 cipher allowlist; deployers can enforce that without patching Shipwright.

### Goals

- The webhook supports **platform-agnostic TLS configuration** via CLI flags suitable for operator injection.
- By default, the webhook relies on **Go's secure defaults** (except for a baseline minimum TLS version) rather than hardcoding cipher suite or curve lists.
- Effective TLS configuration is **logged at startup** for observability (minimum version, whether a TLS 1.2 cipher allowlist was set, and the count of configured ciphers).
- Test helpers avoid duplicating TLS configuration logic, so test servers match production defaults.

### Non-Goals

- **Securing the build controller's metrics endpoint.** The metrics endpoint in `cmd/shipwright-build-controller/main.go` serves plain HTTP on `:8383`. Adding TLS there requires changes to a different binary and is a separate enhancement. This proposal only covers the webhook server.
- **Managing TLS certificates or certificate rotation.** The existing Secret flow (`hack/setup-webhook-cert.sh`) handles certificate provisioning. This proposal only changes the TLS *protocol settings* (versions, ciphers, curves), not certificate management.
- **Enforcing TLS profiles on registry connections.** The `--insecure` flag in `pkg/image/options.go` controls whether the `image-processing` binary (which runs inside build pods, not the webhook) verifies registry TLS certificates. This is a per-build, user-specified setting for scenarios like internal registries with self-signed certificates — a different concern from cluster-wide TLS policy for the webhook server.
- **Controlling Tekton component TLS settings.** Tekton's own webhook and controller have their own TLS configurations managed by the Tekton project. Shipwright does not own or configure those components.
- **Modifying build strategy TLS verification flags.** Build strategies use tool-specific flags like `--tls-verify` (Buildah) and `--skip-tls-verify` (Kaniko) to control whether the build tool verifies the registry's TLS certificate when pushing images. Enforcing the cluster TLS profile here would require mapping profiles to each build tool's specific flag semantics and modifying every strategy sample — out of scope for this proposal.
- **Embedding platform-specific APIs or TLS profile parsing in Shipwright.** Shipwright will not read platform-specific resources at runtime.
- **In-process watching/hot-reload of platform TLS profile changes.** Operators should rollout configuration changes through normal Deployment updates.
- **Configuring TLS 1.3 cipher suites.** Go does not allow configuring TLS 1.3 cipher suites via `tls.Config.CipherSuites`; TLS 1.3 ciphers are negotiated automatically.

## Proposal

### Scope

Modify the webhook server TLS configuration (and its test webhook servers) to:

- `cmd/shipwright-build-webhook/main.go`: remove `CurvePreferences` and `CipherSuites`, keep `MinVersion: tls.VersionTLS12`.
- `test/utils/webhook.go`: same change for the integration test webhook server.
- `test/utils/v1beta1/webhook.go`: same change for the v1beta1 test webhook server.

- Add webhook flags (`--tls-min-version`, `--tls-cipher-suites`) and a shared TLS config builder (`pkg/webhook/tlsconfig/tlsconfig.go`) used by both the webhook and test servers.

Rationale: newer Go releases provide safer and more modern defaults (including updated curve preferences and TLS 1.3 negotiation behavior). We explicitly retain `MinVersion: tls.VersionTLS12` as a baseline security requirement.

### Implementation Notes

#### CLI flags 

Add platform-agnostic flags to the webhook binary and use them to build `tls.Config`:

- Why these flags are needed:
  - `--tls-min-version` is required so deployers can enforce stricter platform policy (e.g., TLS 1.3-only) without patching Shipwright; a permanently hardcoded `MinVersion: tls.VersionTLS12` would prevent compliance with tighter requirements.
  - `--tls-cipher-suites` is required because for TLS 1.2 some platforms/compliance profiles mandate an explicit allowlist; relying only on Go defaults does not reliably satisfy a required cipher set.

- `--tls-min-version`
  - Sets `tls.Config.MinVersion`.
  - Accepted values: `VersionTLS10`, `VersionTLS11`, `VersionTLS12`, `VersionTLS13`.
  - Default when unset: `VersionTLS12`.

- `--tls-cipher-suites`
  - Comma-separated list of cipher suite names (Go cipher suite names), e.g. `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`.
  - Sets `tls.Config.CipherSuites` (TLS 1.2 and below only).
  - Default when unset: do not set `CipherSuites` explicitly (use Go defaults).
  - If `--tls-min-version=VersionTLS13`, ignore `--tls-cipher-suites` and log a warning (since TLS 1.3 ciphers are not configurable in Go).

Input validation:
- Unknown `--tls-min-version` values: fail startup with a clear error.
- For `--tls-cipher-suites`, reject an empty list (e.g., `--tls-cipher-suites=""` or commas with no names).
- Unknown cipher suite names: fail startup with a clear error listing invalid entries.

#### Shared helper for production + tests

To avoid drift, implement TLS configuration in `pkg/webhook/tlsconfig/tlsconfig.go` with a shared builder used by:

- `cmd/shipwright-build-webhook/main.go`
- `test/utils/webhook.go`
- `test/utils/v1beta1/webhook.go`

The helper should provide a function similar to:

`BuildServerTLSConfig(minVersionFlag, cipherSuitesFlag string) (*tls.Config, warning string, err error)`

Builder behavior:

- **Defaults**: set `MinVersion = tls.VersionTLS12`; leave `CurvePreferences` and `CipherSuites` unset so Go defaults apply.
- **`--tls-min-version` parsing**: accept `VersionTLS10|VersionTLS11|VersionTLS12|VersionTLS13`; default to `VersionTLS12`; reject unknown values with a clear error.
- **`--tls-cipher-suites` parsing**: comma-separated list of Go cipher suite names; map names to IDs using `tls.CipherSuites()` plus `tls.InsecureCipherSuites()` for name resolution; reject unknown names and empty lists.
- **TLS 1.3 rule**: if min version is `VersionTLS13`, ignore `--tls-cipher-suites` and return a warning (since Go controls TLS 1.3 ciphers).

#### Changes to `cmd/shipwright-build-webhook/main.go`

Replace the existing hardcoded `tls.Config` with the shared helper result, roughly:

```go
tlsCfg, warning, err := tlsconfig.BuildServerTLSConfig(tlsMinVersionFlag, tlsCipherSuitesFlag)
if err != nil { ... }
if warning != "" { ... }
log.Info("Effective TLS configuration",
    "minVersion", tlsCfg.MinVersion,
    "cipherSuitesConfigured", tlsCfg.CipherSuites != nil,
    "cipherSuitesCount", len(tlsCfg.CipherSuites))

server := &http.Server{
    Addr:              ":8443",
    Handler:           mux,
    ReadHeaderTimeout: 32 * time.Second,
    TLSConfig:         tlsCfg,
}
```

#### Test Helper Updates

Update any test webhook servers (e.g., under `test/utils/`) to use the same helper defaults (and optionally the same flag-driven overrides if tests need them), rather than duplicating cipher/curve configuration.

### Test Plan

#### Unit Tests (new helper package)

- `TestMinVersionParsing` — accept allowed values, reject unknown values.
- `TestCipherSuiteParsing` — accept known cipher suite names; reject unknown names; preserve order.
- `TestDefaults` — verify baseline is TLS 1.2 minimum and that ciphers/curves are not explicitly set by default.
- `TestTLS13IgnoresCipherSuitesFlag` — if min version is TLS 1.3, cipher suite flag is ignored with a warning indicator in the summary.

#### Existing Tests

- Existing webhook e2e tests continue to pass with defaults.

### Release Criteria

TBD

#### Upgrade Strategy

- **Upgrade**: no configuration changes required for existing installations. The webhook will stop advertising the previously hardcoded TLS 1.2 cipher suite list and curve list and will instead rely on Go defaults (with TLS 1.2 minimum retained). Operators may optionally begin injecting flags to match platform TLS profile requirements.

- **Downgrade**: downgrading restores the previous hardcoded behavior. This should not require any action, but the downgrade may remove newly available key exchange options introduced by newer Go defaults.

### Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Go TLS defaults change across Go versions | This is intended (to pick up security improvements). Keep an explicit minimum TLS version baseline and provide flags for environments that require stricter control. |
| Misconfigured cipher allowlist causes handshake failures | Validate cipher names at startup and fail fast with clear errors. Log the effective TLS settings so operators can confirm configuration. |
| Operators need platform TLS profile behavior | The Shipwright operator (or downstream installer) is responsible for translating platform TLS profiles into the platform-agnostic flags and rolling the Deployment. |

## Drawbacks

- Relying on Go defaults means TLS behavior can vary across Go versions, which can complicate strict compliance environments. This is mitigated by offering explicit flags for policy control.
- Adding flags increases configuration surface area and requires documentation and validation to prevent misconfiguration.

## Alternatives

- **Dynamic platform TLS profile adherence inside Shipwright**: Have the webhook read and watch a platform-specific API and apply TLS settings dynamically. This was rejected because it embeds platform-specific behavior into upstream Shipwright and duplicates policy logic better owned by the operator/downstream.
- **ConfigMap-based TLS configuration**: Read TLS settings from a Shipwright-owned ConfigMap. Trade-off: works on all clusters but introduces an additional configuration object and still requires an operator/admin to keep it aligned with platform policy.
- **Do nothing**: Keep the hardcoded TLS 1.2 + static curves/ciphers. Trade-off: brittle against Go TLS improvements.

## Infrastructure Needed

None.

## Implementation History

TBD
