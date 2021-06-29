<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: neat-enhancement-idea
authors:
  - @qu1queee
reviewers:
  - @ImJasonH
  - @SaschaSchwarze0
approvers:
  - @ImJasonH
  - @SaschaSchwarze0
creation-date: 20201-June-29
last-updated: 20201-August-26
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

- The status of a custom resource ( _.status_ ) is a "privileged" part, in the sense that only a controller can modify it. How much freedom do we want to provide to strategy authors when surfacing results in the BuildRun _.status_ .

- Do we need to limit the amount of results we will surface under the BuildRun _.status_ somehow ?

## Summary

Provide support in the Shipwright BuildRun controller to surface Tekton TaskRun Results ( _.status.taskResults_ ) into the BuildRun _.status_, at a specific or multiple subpaths.

## Motivation

Today Shipwright Strategies already emit Tekton [results](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#emitting-results) on their Step definitions, see [example](https://github.com/shipwright-io/build/blob/main/samples/buildstrategy/buildkit/buildstrategy_buildkit_cr.yaml#L65). These results contain valuable metadata for users, like the _image digest_  or the _commit sha_ of the source code used for building. Therefore, we need to ensure that the BuildRun controller can surface these results into it´s own _.status_ subresource.

### Goals

- Provide a mechanism to surface higher-level information in BuildRuns reported by the strategies, by enabling users to get insights into valuable metadata without the need of understanding strategy specific details.
- Decide on the most optimal and minimal API changes for the BuildRun _.status_ subresource, in order to host multiple results.

### Non-Goals

- Limit the amount of results that we can define in the _.status_ subresource of a BuildRun.
- Override the .status.conditions with Tekton results.

## Proposal

A high level proposal will require three things:

- [1] Define the API changes for the BuildRun CRD, this should only be for the _.status_ subresource. See [API Changes](#api-changes).

- [2] Modify the BuildRun CRD to extend the _.status_ subresource with the proposed changes in [1].

- [3] Modify the BuildRun controller business logic, to be able to propagate the Tekton _.status.taskResults_ into [2].

### API Changes

The API changes require a categorization of two types of results. To provide a clear distinction, we will classify them as follows:

- **Strategy Results:** Results that are intended for surfacing higher-level information of a strategy. Is the Strategy Author responsibility to define and emit this results, if desired.

- **Build Results:** Results that are generated at runtime by the BuildRun controller. These results are expected to be present all the time, as long as the step that defines them, exist.

### Strategy Results API Proposal

This will require to introduce `results` under the _.status_ subresource of a BuildRun. This is a list of **name/value**, where Strategy Authors can surface their desire result. This new API change for the BuildRun will be looking as follows:

```yaml
 status:
   results:
   - name: a-strategy-result
     value: a-value
   - name: another-strategy-result
     value: another-value
```

Additionally, we will need to allow Strategy Authors to define a list of results in the strategy definition. This new API change in the Strategies will be looking as follows:

```yaml
spec:
  results:
    - name: a-strategy-result
      description: a-meaningful-description
    - name: another-strategy-result
      description: a-meaningful-description
  buildSteps:
    ...

```

### Build Results

These are results defined at runtime by the BuildRun controller. In this proposal we categorize them in two:

- sources
- output

Following what was mentioned in [EP removal of tekton resources](https://github.com/shipwright-io/build/blob/main/docs/proposals/removal-tekton-resources.md#3-tekton-results), we need to consider the following:

#### Current State of Build Results for Sources

Assuming the following Build definition:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  name: a-build
spec:
  source:
    url: https://github.com/shipwright-io/sample-go

```

We will have the following:

| Result Name                       | Naming Convention                      | Emitted Today | Definition  | Notes                                |
| --------------------------------: | ------------------------------------------: | -------: | ----------: | -----------------------------------: |
| shp-source-default-commit-sha     |  `shp-source-${sourceType}-${resultName}`   | yes     |  at runtime | `${sourceType}` is `default` as is the only supported git source today |

#### Future State of Build Results for Sources

Assuming the following Build definition:

```yaml
apiVersion: shipwright.io/v1alpha1
kind: Build
metadata:
  name: a-build
spec:
  sources:
  - name: a-git-source
    type: git
    url: https://github.com/shipwright-io/sample-go
  - name: an-http-source
    type: http
    url: https://licenses.company.com/customer-id/license.tar
```

We will have the following:

| Result Name                       | Naming Convention                           | Emitted Today | Definition  |
| --------------------------------: | ------------------------------------------: |-------: | ----------: |
| shp-source-a-git-source-commit-sha     |  `shp-source-${spec.sources[0].name}-${resultName}` | no     |  at runtime |
| shp-source-a-git-source-commit-author     |  `shp-source-${spec.sources[0].name}-${resultName}` | no     |  at runtime |
| shp-source-an-http-source-size     |  `shp-source-${spec.sources[1].name}-${resultName}` | no     |  at runtime |

#### Build Results Sources API Proposal

Because of the future adoption of sources, the proposed API for `Build Results` should consider the following:

- The _.status_ subresource should provide a relationship between a source and the results emitted in it´s step definition.
- The _.status_ subresource should be able to surface results emitted from different sources.

Therefore the **proposal** is to introduce `_.status.sources_` subresource which holds a list of `SourcesResult`:

```go
type BuildRunStatus struct { // This struct already exists on the BuildRun API, we just add a new field
  Sources []SourceResult `json:"sources"`
}

type SourceResult struct {
  Name   string             `json:"name"`
  Git    *GitSourceResult   `json:"git,omitempty"`
  Http   *HttpSourceResult  `json:"http,omitempty"` // A new field can be always introduced to support new types, e.g. bundle
}

type GitSourceResult struct{
  CommitSha     string `json:"commit-sha,omitempty"`
  CommitAuthor  string `json:"commit-author,omitempty"`
}

type HttpSourceResult struct{
  Size int `json:"size"`
}
```

A BuildRun will surface these results as follows:

```yaml
status:
  sources:
  - name: default
    git:
      commit-sha: cf156cfb0da9be144a40c7d5a110487e5c524
```

#### Build Results Output API Proposal

Today, we only have a single output step in our strategies. There is no plan to modify this, therefore we propose here to introduce the _.status.output_ subresource in a BuildRun. This will require the following structures:

```go
type BuildRunStatus struct { // This struct already exists on the BuildRun API, we just add a new field
  Output *Output `json:"output"`
}

type Output struct {
  DigestOutputResult string `json:"digest,omitempty"`
}
```

which will looks as follows:

```yaml
 status:
   output:
     digest: sha:30213102
```

#### Proposal References

The following go playground [example](https://play.golang.org/p/T3EsAnVmOhQ) highlights the usage of both Sources and Output Build Resource Objects.

### User Stories [optional]

#### As a Strategy Author, I want to surface strategy higher-level information

Via the `Strategy Results` we provide flexibility for Strategy Authors to decide on particular results they would
like to surface. All they need to do is:

- Define the result in their strategy, under `spec.results`, this consist of a name and a description.
- Populate or Emit the result to `'$(results.<result-name>.path)'`

#### As a Shipwright Build Developer, I want to surface Build meaningful results

Shipwright Developers should conclude on which results defined at runtime will belong under `.status.sources` and `.status.output`. Today, we emit the following:

- shp-image-size
- shp-image-digest
- shp-source-default-commit-sha

therefore we should populate `.status` subresource with the following:

```yaml
 status:
   sources:
   - name: default
     git:
       commit-sha: cf156cfb0da9be144a40c7d5a110487e5c
   output:
     image-digest: sha:30213102
```

### As a Shipwright CLI developer, I want to consume surfaced results

Workflows on top of Shipwright should be able to consume higher-level data and present that in different ways. As long as they consume from `.status.sources`, `.status.output` or `.status.results`.

### Test Plan

It should include:

- Integration-tests
- Unit-tests

### Release Criteria

Should be available before release v1.0.0

### Risks and Mitigations

- `Strategy Results` could pile up, ending-up in heavy BuildRun objects. As we populate results via Tekton, the size limit of a BuildRun results is limited to the one from a Task results, which is 4096 bytes, as mentioned in the Tekton Results [documentation](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md#emitting-results).
- `Build Results` should consider the future adoption of new types, like the `bundle` [one](https://github.com/shipwright-io/build/blob/main/docs/proposals/enable-local-source-code-support.md) or the `local-type` [one](https://github.com/shipwright-io/community/pull/11/).
- Categorizing `Build Results` per type name might be complex, as we will consume results directly from Tekton, where there is no clear indication of their related step, as they come in the form of key/values. Therefore we need to stick to the `shp-source-${sourceType}-${resultName}` naming convention when defining results at runtime.

## Drawbacks

None.

## Alternatives

### For Results generated at Runtime Option 1

- Keep `Strategy Results` under the _.status_ subresource of a BuildRun.

- Introduce `commit-sha` under the _.status_ subresource of a BuildRun. This serves as the identifier for the version of the source code from which this image was built. If the version control system is **git**, then this is the SHA. This value is already available as of today, see related [code](https://github.com/shipwright-io/build/blob/main/pkg/reconciler/buildrun/resources/sources/git.go#L24-L27). This will be looking as follows:

   ```yaml
    status:
      commit-sha: <sha>
   ```

- Introduce `image-digest` under the _.status_ subresource of a BuildRun. This serves as the _sha256_ of the generated container image. This value is already available as of today, see an [strategy example](https://github.com/shipwright-io/build/blob/main/samples/buildstrategy/buildpacks-v3/buildstrategy_buildpacks-v3_cr.yaml#L52). This will be looking as follows:

   ```yaml
    status:
      image-digest: <sha>
   ```

## Implementation History

There is already a WIP [branch](https://github.com/shipwright-io/build/pull/787) that aims to implement this feature, but is currently on hold, waiting
for this SHIP to be approved.

[Local Source Upload]: https://github.com/shipwright-io/community/pull/11
