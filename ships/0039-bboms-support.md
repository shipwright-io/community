<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: BBOMs Support
authors:
  - "@qu1queee"
  - "@MaheshRKumawat"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-06-24
last-updated: 2024-06-24
status: provisional
---

# Neat Enhancement Idea

This is the title of the enhancement. Keep it simple and descriptive. A good title can help
communicate what the enhancement is and should be considered as part of any review.

The YAML `title` should be lowercased and spaces/punctuation should be replaced with `-`.

To get started with this template:

1. **Make a copy of this template.** Copy this template into the main
   `proposals` directory, with a filename like `NNNN-neat-enhancement-idea.md`
   where `NNNN` is an incrementing number associated with this SHIP.
2. **Fill out the "overview" sections.** This includes the Summary and Motivation sections. These
   should be easy and explain why the community should desire this enhancement.
3. **Create a PR.** Assign it to folks with expertise in that domain to help
   sponsor the process. The PR title should be like "SHIP-NNNN: Neat
   Enhancement Idea", where "NNNN" is the number associated with this SHIP.
4. **Merge at each milestone.** Merge when the design is able to transition to a new status
   (provisional, implementable, implemented, etc.). View anything marked as `provisional` as an idea
   worth exploring in the future, but not accepted as ready to execute. Anything marked as
   `implementable` should clearly communicate how an enhancement is coded up and delivered. Aim for
   single topic PRs to keep discussions focused. If you disagree with what is already in a document,
   open a new PR with suggested changes.

The `Metadata` section above is intended to support the creation of tooling around the enhancement
process.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding to implement the
design. For instance:

> 1. This locks a build strategy to run privileged pods. Should we do this?

## Summary

The `Summary` section is incredibly important for producing high quality user-focused documentation
such as release notes or a development roadmap. It should be possible to collect this information
before implementation begins in order to avoid requiring implementors to split their attention
between writing release notes and implementing the feature itself.

A good summary is probably at least a paragraph in length.

## Motivation

This section is for explicitly listing the motivation, goals and non-goals of this proposal.
Describe why the change is important and the benefits to users.

### Goals

List the specific goals of the proposal. How will we know that this has succeeded?

### Non-Goals

What is out of scope for this proposal? Listing non-goals helps to focus discussion and make
progress.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories [optional]

Detail the things that people will be able to do if this is implemented. Include as much detail as
possible so that people can understand the "how" of the system. The goal here is to make this feel
real for users without getting bogged down.

#### Story 1

#### Story 2

### Implementation Notes

**Note:** *Section not required until feature is ready to be marked 'implementable'.*

Describe in detail what you propose to change. Be specific as to how you intend to implement this
feature. If you plan to introduce a new API field, provide examples of how the new API will fit in
the broader context and how end users could invoke the new behavior.

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:

- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as
tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).

### Release Criteria

**Note:** *Section not required until targeted at a release.*

#### Removing a deprecated feature [if necessary]

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

#### Upgrade Strategy [if necessary]

If applicable, how will the component be upgraded? Make sure this is in the test
plan.

Consider the following in developing an upgrade strategy for this enhancement:

- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to
  make on upgrade in order to make use of the enhancement?

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
both security and how this will impact the larger Shipwright ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
possible approaches to delivering the value proposed by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new subproject, repos
requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources started right away.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation History`.


