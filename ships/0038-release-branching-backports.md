<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: release-branching-backports
authors:
  - "@adambkaplan"
reviewers:
  - "@apoorvajagtap"
  - "@HeavyWombat"
approvers:
  - "@qu1queee"
  - "@SaschaSchwarze0"
creation-date: 2024-02-20
last-updated: 2024-03-05
status: provisional
see-also:
  - https://github.com/shipwright-io/community/issues/85
  - https://github.com/shipwright-io/build/issues/1496
replaces: []
superseded-by: []
---

# Release Branching and Backports

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [x] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

None.

## Summary

Proposed release process changes for Shipwright projects so we can issue candidate
releases and provide backports for high-impact bug fixes/security advisories.

## Motivation

As the community provides stronger promises of stability in its projects, end users
and downstream distributions may not be able to wait for new upstream releases to
receive important bug fixes or security patches. Backporting is a common practice in
mature open source projects to provide these fixes without introducing new features
or other potential breaking changes.

### Goals

- Define processes and procedures Shipwright projects should use when creating a
  release with respect to code branching.
- Define processes for applying bug fixes and security advisories to the most recent
  released version.
- Apply the backport process to current released versions of the Shipwright build,
  cli, and operator projects.

### Non-Goals

- Identify which bugs and security advisories are eligible for backport moving
  forward.
- Apply the backport process to "alpha" maturity projects in Shipwright (ex:
  triggers).
- Docs/website content strategy for released versions.
- Apply backport process to older released versions.
- Prescribe a cadence for feature and bugfix releases.
- Establish long term support (LTS) versioning and backport processes.
- Alter our current process for generating release notes.
- Standardize on a single release toolchain.

## Proposal

Shipwright projects will continue to use the `main` branch for feature
development. When the project maintainers decide to create a release, they **MUST**
create a branch that adheres to the `release-vX.Y` naming convention. `X.Y`
represents the semantic _major_ and _minor_ version of the given release,
respectively.

Release tags and artifacts for a given `X.Y` version **MUST** be created from the
respective `release-vX.Y` release branch. Releases **MUST** use a valid semantic
version when tagging or versioning artifacts, of the format `vX.Y.Z`. Projects
**MAY** append a hyphen to indicate a release candidate or other form of "pre-
release" version, in accordance with the SemVer 2.0 specification. For example,
`v0.13.0-rc2` can be used to represent version 0.13.0, _release candidate 2_.
CI checks to merge code in the the `main` branch **MUST** also be run against code
in release branches. Other measures applied to `main` branches (ex: branch
protection rules) **MUST** also be applied to release branches.

All code changes **SHOULD** be merged into the `main` branch before being applied
to a previous release branch, or a release branch that is issuing release
candidates. Merging directly into a release branch without an equivalent code
change in the `main` branch is allowed at sole the discretion of project
maintainers. Once code is merged in `main`, it is eligible to be backported to an
appropriate release branch using `git cherry-pick` or equivalent commands.

Backport code changes **SHOULD** be merged into release branches in descending
semantic version order if the code change should be applied to more than one release
version. This ensures upgrades from one patch version to the next minor version do
not introduce regressions with respect to bugs or security fixes. For example, if
the `release-v0.13` branch is issuing release candidates, and a bug fix for the
release needs to be backported to `release-v0.12`, code should merge in the
following order:

1. Merge fix to `main`
2. Cherry pick to `release-v0.13`
3. Cherry pick to `release-v0.12` _after_ code merges in `release-v0.13`.

In general, there **SHOULD** be at most two branches accepting backports (the most
recent release, and a current release candidate). Maintainers **MAY** accept
backports to prior releases at their discretion, for instance if a security advisory
with "Critical" severity impacts multiple prior releases. Merging out of semantic
version order **MAY** likewise be done at the discretion of project maintainers in
such situations.

### User Stories [optional]

- As a maintainer of a Shipwright distribution, I want Important/Critical severity
  security advisories to be backported to the most recent release so that I can
  patch security vulnerabilities without introducing new features or breaking
  changes.
- As a cluster admin / architect deploying Shipwright to my engineering teams, I
  want to receive bug fixes for the most recent release so that I don't need to wait
  for a new feature release upstream to receive bug fixes and security advisories.

### Implementation Notes

#### Release Branching Workflow

Shipwright has set up the `.github` repository which - amongst other features -
allows GitHub actions to be shared consistently across the organization. We already
have a standing "add to project" workflow that adds all new issues and pull requests
to the main "Shipwright Overview" GitHub project. [1]

We can create a reusable "release branching" workflow [2] as follows:

- Use `workflow_call` as the trigger, to be run on the main branch. [2]
- Receive a semantic version as input (in `vX.Y` format) to create the release branch
  name.
- Create the release branch via standard git commands, ensuring the workflow is
  granted write permission to the appropriate repository.

Example (`shipwright-io/.github/.github/workflows/release-branch.yml`):

```yaml
name: Release Brancher
on:
  workflow_call:
    inputs:
      release-version:
        required: true
        type: string
      git-ref:
        required: false
        type: string
jobs:
  release-brancher:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ inputs.git-ref }}
    - name: Create release branch
      env:
        RELEASE_VERSION: release-${{ inputs.release-version }}
      run: |
        git switch -c ${RELEASE_VERSION}
        git push --set-upstream origin ${RELEASE_VERSION}
```

Each repository can then be onboarded using a starter workflow that is hosted in the
.github repository (`.github/workflow-templates/release-branch.yml`) [3]:

```yaml
name: Create Release Branch
on:
  workflow_dispatch:
    inputs:
      release-version:
        required: true
        type: string
        description: "Semantic version for the release branch (vX.Y format)"
jobs:
  create-release-branch:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    uses: shipwright.io/.github/.github/workflows/release-branch.yml@main
    with:
      release-version: ${{ inputs.release-version }}
```

[1] https://github.com/shipwright-io/.github/blob/main/.github/workflows/issues.yml

[2] https://docs.github.com/en/actions/using-workflows/reusing-workflows

[3] https://docs.github.com/en/actions/using-workflows/creating-starter-workflows-for-your-organization

#### Release Workflows

Component repositories will retain the core of their existing release workflows.
However, each will need to be modified so the release workflow runs against the
appropriate `release-vX.Y` branch:

- `build`: The `release.yml` workflow will receive a new required parameter,
  `release-branch`. This will be passed to the standard git checkout action. The
  current release workflow creates a follow-up PR to update the README with the
  latest release version/tag. This will be opened against the `release-branch` (not)
  `main`
- `operator`: The `release.yml` workflow will receive a new required parameter,
  `release-branch`. This will be passed to the standard git checkout action.
- `cli`: The `cli` repository will add a new workflow, `release-tag.yml`, which will
  create the tag for the upcoming release. This will accept `release-branch` as
  required parameter, to be passed to the standard git checkout action:

  ```yaml
  name: Create Release Tag
  on:
    workflow_dispatch:
      inputs:
        release-branch:
          required: true
          type: string
          description: "Branch to use for creating the release tag"
        version-tag:
          required: true
          type: string
          description: "Full semantic version of the release (in vX.Y.Z format)"
  jobs:
    create-release-tag:
      ... # permissions, check out ${{ inputs.release-branch }}
      - name: Tag release
        env:
          - VERSION_TAG: ${{ inputs.version-tag }}
        run: |
          git tag ${VERSION_TAG}
          git push --tags
  ```

  Once the tag is created, the `cli` release workflow should work as-is.

#### GitHub Branch Protection

Each repository will need to create a new branch protection rule that applies to
`release-v*` branches. Branch protection settings from the `main` branch will need to
be manually copied over to the release branch rule.
This is due to simplifying limitations in how GitHub does pattern matching for
[branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule#about-branch-protection-rules).

The branch protection policy needs to allow GitHub actions to push branches and tags.

#### Backport Process

Backporting can be done in one of three ways:

1. "Manually": this would involve a contributor running `git cherry-pick` on their
   local machine against a release branch, resolving any merge conflicts, then
   opening a pull request.
2. Prow `cherrypick` bot: our existing configuration for OpenShift CI will be
   updated to enable the Prow cherrypick plugin. The bot can automatically create
   cherrypick pull requests by adding a `/cherrypick <branch-name>` comment to an
   existing pull request, either when it is open or after merge. See the Prow
   docs [1] for more details.
3. Standard pull request - typically reserved for the following:
   1. Dependency updates, where changes to `go.mod` and `go.sum` would likely lead
      to merge conflicts across release branches.
   2. Bug fixes for code that was refactored in a subsequent release.

[1] https://docs.prow.k8s.io/docs/components/external-plugins/cherrypicker/

### Test Plan

CI jobs for each component repository will need to support execution from a branch
that matches the release branch pattern. This is done by updating the `on.pull_request` stanza of
the GitHub action to match `release-v*` branch names (same as branch protection
rules). This should be updated for both push events as well as pull requests that
target the release branch.

### Release Criteria

This process will apply to the `v0.13.0` release moving forward. It will also be
applied retroactively to `v0.12.0` by running the release branch workflow,
referencing an appropriate release tag.

#### Removing a deprecated feature [if necessary]

Not Applicable

#### Upgrade Strategy [if necessary]

Not directly applicable.

Care must be taken when introducing "breaking" changes to the release brancher
workflow. The workflow parameters should be treated like an API.

### Risks and Mitigations

**Risk**: Unreviewed code merges in release branches.

_Mitigation:_ Branch protection rules for `main` will be copied over to release
branches. Checks that ensure unreviewed code is blocked from merge will be applied
to release candidates and bugfix releases. At present, no workflow requires code to
be modified during the release process.


**Risk**: Code will be released untested.

_Mitigation_: CI workflows will be updated to run on the release branch for push
events as well as pull requests.

**Risk**: Unauthorized releases.

_Mitigation_: Manually triggered workflows require the "write" permission to a
repository. These are individuals who typically have been promoted to the "Reviewer"
role in the project and have established some level of trust within the community.
Membership and contributor status should be reviewed regularly (at least annually) to
reduce risk.

## Drawbacks

- The "release brancher" workflow needs to be copied to individual repositories via
  the [GitHub starter worflows](https://docs.github.com/en/actions/learn-github-actions/using-starter-workflows)
  process. This makes it easier to check out code and create a relase branch via git
  commands.
- Branch protection rules will need to be copied over by hand, due to limitations in
  the pattern matching behavior in GitHub. To simplify matching rules, GitHub does
  not have full RegEx support for branch protection matching. They instead support
  syntax that is closer to "glob" matching on a Unix system.
- The release branching workflow itself is "copied" to each repository via a starter
  workflow. Defining a reusable workflow mitigates this drawback - the bulk of the
  logic to set up a release branch is centralized.
- CI jobs will need to be updated by hand to run against the release branch.
- The release workflow for `build` will by default open the README update pull
  requests against the release branch. This means that updates to `main` may have to
  be done manually, or the README in `main` will need to reference a floating
  "latest" release tag.

## Alternatives

### Single Release Brancher Workflow

Instead of using a starter workflow for release branching, we could have set up a
single workflow in `.github` that applied to _all_ Shipwright repositories. Doing
this would require some extra logic to tell the checkout action which repository to
check out and push the release branch to. It also removes flexibility in case a
repository needs to run other workflows as pre/post branching "hooks." For example,
after branching the `operator` repository may want to update the `VERSION` variable
in its Makefile and regenerate the operator bundle manifest.

### Standardize on a release toochain (GoReleaser)

Since most projects in Shipwright are go-based, we could choose a single toolchain
like GoReleaser to build our release artifacts [1]. This would require substantially
more effort, requiring re-writes to how the `build` and `operator` repositories
release code. The current proposal takes advantage of existing project release
workflows, only requiring small modifications in specific spots.

### Status Quo

Keeping the status quo (_no release branching_) means that the community cannot
provide sanctioned bugfix/security patch releases. Downstream distributions remain
free to build this infrastructure on their own. Lack of process for backports/
release branching could impact our evaluations for maturity status (ex: "Graduated"
in CNCF or CDF).

### Release .0 from `main`

Instead of branching first, then releasing, we could release the "dot zero" release
from `main`. This simplifies the initial release process and makes it easier to use
our current release tooling "as is." The short-term savings are negated by long-term
costs/tech debt with this approach:

- Bugfix releases will use a slightly skewed set of release scripts compared to "dot
  zero."
- Release branches for z-streams will need to be created after the fact, at the will
  of the project maintainers. This is a potential "open loop" that is not critical
  to the initial release process, putting it at risk of being skipped or forgotten.
- Release branching makes it easier for "experimental" features to merge earlier.
  Often this work will start in a "feature" branch, then merge into the `main`
  branch early in a release cycle so bugs can be identified and patched before
  end-users consume it.

## Infrastructure Needed [optional]

New workflows in the `.github` repository:

- Reusable workflow for release branching
- Starter workflow for release branching (to be copied over)

Also needs a new workflow in the `cli` repository to create release tags. This was
a pre-existing need.

## Implementation History

- 2024-02-20: Initial proposal (`provisional`)
- 2024-03-05: Implementable version
