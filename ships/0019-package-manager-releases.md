<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: Release Shipwright CLI in well known package managers

authors:
  - "@heavywombat

reviewers:
  - "@otaviof"
  - "@alicerum"
  - "@adambkaplan"
  - "@qu1queee

approvers:
  - "@qu1queee"
  - "@sbose78

creation-date: 2021-07-07

last-updated: 2021-07-09

status: implementable

see-also: n/a

replaces: n/a

superseded-by: n/a
---

# Release Shipwright CLI in well known package managers

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

_n/a_

## Summary

Users of CLI tools are accustomed to the usage of package managers to install a tool and to keep it up-to-date. The `shp` CLI tool should have an option to be installed with such a package manager.

## Motivation

Make `shp` installable on the common platforms as easy, reliable and trustworthy as possible.

### Goals

- Setup a Homebrew Tap for Shipwright tools and publish `shp` in it.
- Create a `snap` to be available in [snapcraft.io](https://snapcraft.io/) store.

### Non-Goals

Make us available in package managers that require a lengthy approval process, e.g. `apt`, or `yum`.

## Proposal

Create a new repository https://github.com/shipwright-io/tap to serve as a Homebrew Tap. Register `shp` as a new `snap` in the [snapcraft.io](https://snapcraft.io/) store. Introduce GoReleaser configuration that builds and publishes to the repository and store on each release.

### Implementation Notes

The setup for Homebrew and `snap` are pretty straightforward. For Homebrew, we only need a repository in GitHub with a well-known name, ie. `tap` or `homebrew-tap`. The [snapcraft.io](https://snapcraft.io/) is also relatively easy, it requires some metadata and a logo to be registered. The rest is handled by GoReleaser as it comes with publishing capabilities for these targets. As a side effect, this will also release a GitHub release with a change log and binaries. The [`dyff` GoReleaser configuration](https://github.com/homeport/dyff/blob/main/.goreleaser.yml) contains the respective configuration snippets for reference.

### Test Plan

- On systems that support `brew`, such as macOS and Linux, run a test installation with `brew install shipwright-io/tap/shp` and verify the CLI version.
- The `snap` system is also supported on a series of Linux systems, the test installation should be `snap install shp`.

### Release Criteria

**Note:** *Section not required until targeted at a release.*

### Risks and Mitigations

With the release process fully automated, there should be no risks of additional maintenance. For this to work, access credentials need to be configured to publish the binaries as a GitHub release and to commit to the Homebrew Tap, however, this can be done completely within GitHub and only accessed through GitHub Actions. The risk of credential exposure is therefore at the same level as the other credentials that are in place and relies heavily on the integrity of GitHub itself.

## Drawbacks

None.

## Alternatives

Present "install from source" and "curl to pipe" style install options.

## Infrastructure Needed

- GitHub
- GitHub Actions

## Implementation History

_n/a_
