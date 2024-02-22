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
last-updated: 2024-02-22
status: provisional
see-also:
  - https://github.com/shipwright-io/community/issues/85
  - https://github.com/shipwright-io/build/issues/1496
replaces: []
superseded-by: []
---

# Release Branching and Backports

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD as we iterate through the enhancement proposal lifecycle.

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

**Note:** *Section not required until feature is ready to be marked 'implementable'.*

TBD - requires investigation into the following:

- Current release script assumptions with respect to branch names.
- Current CI assumptions with respect to branch names.
- Applying the process "retroactively" to build, cli, and operator repos.

### Test Plan

TBD

**Note:** *Section not required until targeted at a release.*

```
Consider the following in developing a test plan for this enhancement:

- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as
tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).
```

### Release Criteria

TBD - ideally this is introduced to release `v0.13.0` and applied retroactively to
`v0.12.z`.

#### Removing a deprecated feature [if necessary]

Not Applicable

#### Upgrade Strategy [if necessary]

TBD

### Risks and Mitigations

TBD once the proposal is flushed out.

```
What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
both security and how this will impact the larger Shipwright ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?
```

## Drawbacks

TBD once the implementation details are documented.

```
The idea is to find the best form of an argument why this enhancement should _not_ be implemented.
```

## Alternatives

TBD as the enhancement proposal progresses through its lifecycle.

```
Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
possible approaches to delivering the value proposed by an enhancement.
```

## Infrastructure Needed [optional]

TBD - this will likely require new GitHub actions, changes to existing GitHub
Actions, or enabling features in Prow (currently provided by OpenShift CI).

## Implementation History

- 2024-02-20: Initial proposal (`provisional`)
