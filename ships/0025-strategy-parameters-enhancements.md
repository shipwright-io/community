<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: buildstrategy-parameter-enhancements
authors:
  - @SaschaSchwarze0
reviewers:
  - @HeavyWombat
  - @GabeMontero
  - @ImJasonH
approvers:
  - @adambkaplan
creation-date: 2021-10-06
last-updated: 2021-10-06
status: implementable
---

# Surface Step Results in BuildRun Status

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

None

## Summary

Recently, we introduced build strategy parameters with default values support through [ship 0014](0014-parameterize-strategies.md). This only supported single strings as values.

This ship proposes an extension to support arrays, Secret and ConfigMap values for strategy parameters.

## Motivation

Build strategies need their own way to define parameters to enable certain features. In certain scenarios, single strings are not enough for all purposes:

- There can be many [ARGs in a Dockerfile](https://docs.docker.com/engine/reference/builder/#using-arg-variables) for which the corresponding value needs to be provided when running the build.
- Values of a build-arg are sometimes secrets, for example to pass a value for the [NPM_TOKEN environment variable](https://docs.npmjs.com/using-private-packages-in-a-ci-cd-workflow) into a Dockerfile build. Users do not want to provide those through a text field in a Build or BuildRun but rather using a Secret key reference.
- Values of a build-arg are sometimes common across builds, for example the version of a tool that is used across builds, for example the version of a base image. Users do not want to update several builds individually, potentially forgetting one and continuing to build with an outdated version. Instead they want re-usable values stored in ConfigMaps and reference its keys in multiple Builds or BuildRuns.

That's where the need for the additional features comes from.

### Goals

- Allow the build strategy author to define if a parameter is a single value or an array.
- Allow the build strategy author to define a default value also for an array parameter.
- Provide updated build strategy samples that use arrays so that the build strategy author can understand how to do this.
- Allow build users to define values for an array parameter using simple strings.
- Allow build users to define values for a string parameter, or an element of an array parameter, using a Secret key reference.
- Allow build users to define values for a string parameter, or an element of an array parameter, using a ConfigMap key reference.
- Allow build users to define a prefix for the Secret value.
- Allow build users to define a prefix for a ConfigMap value.

### Non-Goals

## Proposal

### API Changes

We will extend the Build Strategy Parameter type to support the following:

```yaml
spec:
  parameters:
    - name: build-args
      description: The values for the ARGs in the Dockerfile.
      type: array
      defaults: []
```

The `type` property is new and allows either `string` or `array` with `string` being the default.

The `defaults` parameter is new and is a string array. No possibility here to reference Secret or ConfigMap values.

The Build and BuildRun parameterValue will be extended to support the following:

```yaml
spec:
  paramValues:
    - name: single-secret-value
      secretValue:
        name: secret-name
        key: secret-key
    - name: single-configmap-value
      configMapValue:
        name: configmap-name
        key: configmap-key
    - name: simple-list
      values:
        - value: first-item
        - value: second-item
    - name: build-args
      values:
        - value: KEY1=myarg1
        - secretValue:
            name: build-arg-secret
            key: value
            format: "KEY2=${SECRET_VALUE}" 
        - configMapValue:
            name: build-configuration
            key: base-image-digest
            format: "BASE_IMAGE_DIGEST=${CONFIGMAP_VALUE}"
```

The `secretValue` property is new and can be used instead of value to reference a key in a Secret. Optionally, the `format` property (shown above at the end inside an array) can be used to add text around the Secret value. The scenario is clear there: the value for the build arg comes from a Secret, but the key of the build arg also needs to go there.

The `configMapValue` property is also new and works in the same way as the `secretValue`. A `format` is also supported here.

The `values` property is new and allows to specify multiple values. Each value can either be a simple string `value`, or a `secretValue`, or a `configMapValue`.

Validation will be added ...

- to the Build and BuildRun to verify that the value for an array parameter is not a single value, and vice versa.
- to the Build and BuildRun to verify the existence of the Secret and the referenced key.
- to the Build and BuildRun to verify the existence of the ConfigMap and the referenced key.

### Backend

We will continue to use Tekton results. This is straight-forward for the array support.

For Secrets, we will do the following:

- For each `secretValue`, we will add an environment variable to those steps that reference the param.
  - The environment variable name will be `SHP_SECRET_PARAM_{randomSuffix}`
  - The env variable on the step will have a [`secretKeyRef`](https://github.com/kubernetes/api/blob/release-1.20/core/v1/types.go#L1946).
- The value of the parameter is set to the raw value of `$(SHP_SECRET_PARAM_{randomSuffix})` if no `format` field is specified, or we replace all occurrences of `${SECRET_VALUE}` in the `format` field from our yaml example above with the value of `$(SHP_SECRET_PARAM_{randomSuffix})`.
- **NOTE**: The `$(ENV_VAR_NAME)` notion is supported by Kubernetes. Secret values will therefore not be usable in all places that Tekton supports params for, but only where Kubernetes can replace environment variable references.

For Config Maps, we will do the same as for Secrets, but environment variable name will be `SHP_CONFIGMAP_PARAM_{randomSuffix}` and they will use a [`configMapKeyRef`](https://github.com/kubernetes/api/blob/release-1.20/core/v1/types.go#L1943).

Example:

Build strategy:

```yaml
spec:
  parameters:
    - name: some-var
  buildSteps:
    - name: test
      image: alpine:latest
      command:
        - /bin/bash
      args:
        - -c
        - |
          echo "Parameter: $(params.some-var)"
```

BuildRun:

```yaml
spec:
  paramValues:
    - name: some-var
      secretValue:
        name: a-secret
        key: a-secret-key
        format: KEY=${SECRET_VALUE}
```

Will result in this TaskRun

```yaml
spec:
  taskSpec:
    params:
      - name: some-var
        type: string
    steps:
      - name: test
        image: alpine:latest
        command:
          - /bin/bash
        args:
          - -c
          - |
            echo "Parameter: $(params.some-var)"
        env:
          - name: SHP_SECRET_PARAM_4M8SL
            valueFrom:
              secretKeyRef:
                name: a-secret
                key: a-secret-key
  params:
    - name: some-var
      value: KEY=$(SHP_SECRET_PARAM_4M8SL)
```

And in this pod:

```yaml
spec:
  containers:
    - name: test
      image: alpine:latest
      command:
        - /tekton/tools/entrypoint
      args:
        - -wait_file
        - ...
        - -entrypoint
        - /bin/bash
        - --
        - -c
        - |
          echo "Parameter: KEY=$(SHP_SECRET_PARAM_4M8SL)"
      env:
        - name: SHP_SECRET_PARAM_4M8SL
          valueFrom:
            secretKeyRef:
              name: a-secret
              key: a-secret-key
```

### Sample updates

The sample build strategies for Buildah and BuildKit will get updates to support a build-args parameter. In Kaniko on the other side it is not possible to support build-args easily because some simple scripting is necessary to translate the parameter value into arguments for the Kaniko binary (in the same way as it will be done for Buildah and BuildKit), but Kaniko is based on scratch based on its design. Therefore, no shell is available there.

### Documentation updates

We will need to update the documentation about build strategies to describe that array parameters are supported and how they can be used (the Tekton syntax in the args, `$(params.param-name[*])`). Our sample build strategy updates will help to understand this.

We will need to update the documentation about Builds and BuildRuns to describe how to use array parameters and how to use Secrets or ConfigMaps as values.

### User Stories

#### As a Strategy Author, I want to define and use array parameters in my steps

Or more specific:

#### As an author of a Dockerfile strategy, I want to provide support for the user to provide values for build-args so that we can support a broader set of Dockerfile builds

#### As a Build user, I want to provide values for array parameters in my Build and BuildRun

Or more specific:

#### As a Build user, I want to provide values in the Build or BuildRun for the ARGs in my Dockerfile so that I can run these builds with Shipwright

#### As a Build user, I want to provide values for certain parameters through a Secret so that I do not need to specify them in clear text in the Build or BuildRun

#### As a Build user, I want to provide values for certain parameters through a ConfigMap so that I can re-use the value across multiple Builds or BuildRuns

### Test Plan

It should include:

- Integration tests
- Unit tests
- e2e tests

### Release Criteria

Should be available before release v1.0.0

### Risks and Mitigations

- We rely here in Tekton params as we already do with the existing support. If Tekton ever changes behaviors we might have to adopt.
- The existence validation for Secrets and ConfigMaps can become expensive. The fact that they contain potentially a large amount of data makes it even more problematic and rules out options like informer-based caches due to the too large memory consumption. But, for our validations, we only need to know the secret or config map keys, not their data. This would allow for an efficient caching.

## Drawbacks

None.

## Alternatives

To address features like build-args specifically, we could introduce first class support in the Build or BuildRun. But, as build-args are only for Dockerfile based builds, they imo make more sense as parameters.

ConfigMaps do not contain values that necessarily need protection. Instead of using environment variables as we must do it for Secrets, we could also read the ConfigMap and directly place the value in the param value in the TaskRun.

## Implementation History

There is a spike branch atm that is doing some shortcuts but is fully functional and contains an update to the BuildKit build strategy on how to use it: [sascha-spike-parameters-v2](https://github.com/SaschaSchwarze0/build/tree/sascha-spike-parameters-v2). It does not contain the support for ConfigMap references.
