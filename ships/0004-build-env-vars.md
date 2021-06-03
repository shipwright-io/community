<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: build-env-vars
authors:
  - "@adambkaplan"
reviewers:
  - "@gabemontero"
  - "@ImJasonH"
approvers:
  - "qu1queee"
  - "@sbose78"
creation-date: 2021-06-03
last-updated: 2021-06-29
status: implementable
see-also: []
replaces:
- https://github.com/shipwright-io/build/pull/726
superseded-by: []
---

# Build Environment Variables

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

1. How can we validate against the environment variables present in the base image of the build?
   We're considering this out of scope - this would be very difficult to implement in practice.

## Summary

This proposal will give developers the abilty to add environment variables to build strategy steps.
Conversely, this proposal will also give build strategy authors the ability to fix the values of environment variables in build steps or provide non-empty default values.

## Motivation

Developers using Shipwright may need to alter the behavior of a build by overriding or augmenting the environment variables.
These may not be known ahead of time by the build strategy author, and hence not exposed via build parameters.
For example, source-to-image and Cloud Native Buildpacks rely on environment variables to tune the behavior of their supported language/framework runtimes.
The set of environment variables to support is effectively unbounded, as each underlying runtime has its own tooling (`npm`, `mvn`, etc.) that can read dozens of environment variables at build time.

At the same time, build stategy authors may want to control which environment variables can be altered by the end developer.
Many tools use environment variables to override basic security settings, like TLS verification, that are otherwise enabled by default.
A build strategy author may not want developers to change these settings, or would perfer to provide default values that adhere to best practices.

### Goals

* Allow developers to set environment variables used in a build step.
* Provide a means for build strategy authors to keep fixed values for environment variables.

### Non-Goals

* Allow developers to alter or override other aspects of the build step, such as commands.
* Set environment variables in the output image of the build.
* Validate environment variables against those in the base image of a build step.
* Provide direct support for Dockerfile [build arguments](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg).

## Proposal

### User Stories

#### Story: Alter build behavior via environment variables

As a nodejs developer using buildpacks
I want to set the `NPM_INCLUDE` environment variable in my Build and BuildRuns
So that I can include development and debug resources in my container image

#### Story: Prevent insecure behavior in a build strategy

As a build strategy author
I want to keep fixed values for some environment variables in my build strategy
So that I don't allow end users to enable insecure or potentially harmful features.

#### Story: Provide credentials via environment variables

As a Java developer using maven to build my application (via buildpacks or source-to-image)
I want to use environment variables to pass usernames and passwords to values defined in my `pom.xml` file
So that I don't store sensitive information in source control.

### Implementation Notes

#### Build Strategy Behavior

Build strategies which declare an environment variable in a build strategy step will by default be fixed.
A developer will not be able to change this value directly.

If a build strategy author wants to allow an environment variable to be tuned, they have two options:

1. Omit the environment variable - this works if an empty/zero default value is acceptable in the build step.
2. Declare the environment variable, but use a build strategy parameter to set the value.
   This is useful if the build strategy should use a nonempty/nonzero default value in the build step.

For example, a BuildKit build strategy author could use the following to ensure that TLS is always verified when pushing the resulting image:

```yaml
spec:
  steps:
  - name: build
    ...
    env:
    - name: DOCKER_TLS_VERIFY
      value: "true"
```

Or alternatively, the build strategy author can make this configurable with a build parameter:

```yaml
spec:
  parameters:
  - name: TLS_VERIFY
    description: "Verify TLS when pushing the resulting container image."
    default: "true"
  steps:
  - name: build
    ...
    env:
    - name: DOCKER_TLS_VERIFY
      value: $(params.TLS_VERIFY)
```

#### Build/BuildRun Environment Variables

The `Build` and `BuildRun` APIs will be extended with a `spec.env` field.
This is where a developer declares that they are appending environment variables to a build step.
`env` will have an array of Kubernetes envrionment variables that will be appended to every step of the build strategy.

Referring to the nodejs buildpacks story above, a developer who wants their build to install `dev` dependencies would do the following in their `Build` or `BuildRun`:

```yaml
spec:
  sources:
  - name: git
    type: Git
    git:
      uri: https://github.com/sclorg/nodejs-ex.git
  strategy:
    kind: BuildStrategy
    name: buildpacks-v3
  env:
  - name: NPM_INCLUDE
    value: dev
```

Environment variables will always be appended to the environment variables declared in the build strategy's steps.
If a `BuildRun` is started and either the referenced `Build` or the `BuildRun` has an environment variable name that collides with any step in the build strategy, the `BuildRun` should fail.
A `Build` could reference environment variable names that collide with the build strategy, as it would be hard to guarantee that a `Build` does not have bad enviroment variable names over time.

The environment variable API will take advantage of Kubernete's core `EnvVar` interface, which allows environment variable data to be sourced from a Secret or ConfigMap.

A `BuildRun` can override the environment variables specified in a `Build` object.

#### CLI Enhancements

The `shp build run` command should be extended to allow environment variables to be passed in using the `-e` and `--env` flag.
This flag can be specified more than once, and the value should use the `<key>=<value>` syntax.
For example, to set the `NPM_INCLUDE` environment variable when invoking a run of the `buildpacks-nodejs` build, use the following:

```shell
$ shp build run buildpacks-nodejs -e NPM_INCLUDE=dev
```

### Test Plan

**Note:** *Section not required until targeted at a release.*

TBD

### Release Criteria

**Note:** *Section not required until targeted at a release.*

TBD

#### Upgrade Strategy [if necessary]

TBD

### Risks and Mitigations

There is a risk that developers will want an environment variable to be set in one build step, but unset or restored to a default value in a different build step.
This could be addressed in a future enhancemenent via separate API field (ex: `excludeFromSteps`).
Users who are comfortable modifying build strategies can work around this issue by writing their own strategies with the desired environment variables fixed or exposed as a parameter.

## Drawbacks

The primary drawback is that build strategy authors need to provide fixed or default values for environment variables they want to control.
A strategy author cannot simply declare that an environment variable is "off limits" - they must declare when a controlled env var is used and what value it should have.
This is a fair compromise to establish a clear contract between a build strategy author and the developer augmenting the build strategy's behavior.

This proposal won't allow developers to set environment variables in some build strategy steps, but not others.
This capability implies that the developer knows some information about the build strategy, which the community consideres an advanced skill.
The Shipwright community does not expect developers to know the names of the steps declared in build strategies.
Appending environment variables to all build strategy steps is simpler behavior that is easier to comprehend.
This capabilty could be added in the future, but we do not consider this a requirement at present.

## Alternatives

### Allow/deny lists for environment variables

A previous [build enhancement proposal](https://github.com/shipwright-io/build/pull/726) took a different approach to supporting environment variable overrides.
In that proposal, it was incumbent on the build strategy authors to declare which environment variables could be overrode, and which environment variables could not be set.
This was accomplished through allow/block lists, as well as a separate list of environment variables that could be declared with default values (and which were not declared in build steps).
The downside of this proposal was that it severed the relationship of an environment variable from the place where the environment variable is used.
Its complexity also led to competing paths to implement similar capabilites.

This proposal avoids some of these problems by tying environment variables directly to build steps.
Environment variables declared in build steps are assumed to be fixed unless they are exposed via a build parameter.

### Apply env vars to individual build steps

A draft of this proposal applied environment variables to individual build steps, rather than all build steps.
This required developers to discover the step names in the referenced build strategies to fully utilize this feature.
These names are not immediately obvious to a developer using Shipwright.

Mitigating this concern proved difficult, requiring one or more of the following:

- Establishing a convention where `build` is the main step in any build strategy.
  Most build strategies only have one step - this convention will allow developers to use `env.build` to provide overrides in a natural fashion.
- Enhancing the CLI to print the steps of a build strategy.

We could not easily find a "kubernetes-native" means of printing the step names for each build strategy.
Kubebuilder allows CRDs to have [additional printer columns](https://book.kubebuilder.io/reference/generating-crd.html#additional-printer-columns), however it does not support aribitrary-length arrays well.
One could not instruct Kubernetes to print the `name` field of all items in a JSONPath list as comma-separated values.
Instead, Kubernetes printed the specified value of the first item in the array.

## Implementation History

- 2021-06-03: Initial draft proposal.
- 2021-06-07: Move away from "overrides".
- 2021-06-28: Implementable
