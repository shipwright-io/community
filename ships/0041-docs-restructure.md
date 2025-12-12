<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
title: SHIP-0041 Documentation Restructure
authors:
  - "@adambkaplan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-04-21
last-updated: 2025-04-21
status: provisional
see-also:
  - https://github.com/shipwright-io/build/blob/main/docs/proposals/shipwright-website.md
replaces: []
superseded-by: []
---

# Documentation Restructure

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Open Questions [optional]

TBD

## Summary

This proposes a restructure of the documentation on [shipwright.io](https://shipwright.io) so that
it incorporates the CNCF best practices for documentation as outlined in the CNCF 
[tech docs primer](https://github.com/cncf/techdocs/blob/main/docs/sandbox-doc-primer.md).

## Motivation

Our current docs practice has engineers write docs alongside code in respective git repositories.
The documentation (in Markdown) is then synced to the website repository. We intended on writing
automation for this process but that never came to fruition.

Keeping docs review alongside code review has led to several negative outcomes:

- Bugs related to broken links, as developers have no means to preview how their content can/should
  be linked.
- Inability to define or test the Docsy metadata that impacts the docs presentation on the website.
- Tendency to use "flat" trees to organize documentation in git.
- Mixing of different documentation
  [types](https://github.com/cncf/techdocs/blob/main/docs/sandbox-doc-primer.md#an-information-model-for-user-documentation) 
  within a single article.

This proposal aims to fix these issues through improved content structure and docs review process.

### Goals

- Reduce documentation bugs related to broken links.
- Present existing content in ways that are relevant to end users and administrators.
- Encourage end user evaluation and adoption.
- Provide a single source of truth for documentation.

### Non-Goals

- Migrate off of Docsy and Markdown to another platform/docs stack.
- Change structure of blogs and other content.
- Provide multi-version support in the documentation.
- Improve/overhaul contributor guidelines.
- Create net new content for missing features.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories [optional]

Detail the things that people will be able to do if this is implemented. Include as much detail as
possible so that people can understand the "how" of the system. The goal here is to make this feel
real for users without getting bogged down.

#### Quick Start

As a developer evaluating Shipwright, I want a quick start guide to demonstrate the project's value.

#### Concept Articles

As a developer using Shipwright, I want to understand what the different API objects represent.

#### Task Articles

As a developer using Shipwright, I want to know how to perform specific types of builds.

#### Reference Articles

As a developer using Shipwright, I want to review the full API specification so that I can
configure Builds for my specific situation.

### Implementation Notes

#### Key Personas

The documentation will be updated with the following user personas in mind:

- **Evaluators**: These are leaders who are considering Shipwright's merits for a team/organization.
- **Developers**: These are individuals who use Shipwright to build their applications on a daily
  basis. These may also be platform engineers/architects who which to incorporate Shipwright into
  CI/CD processes.
- **Administrators**: These are individuals who deploy and manage Shipwright for others.

#### Overall Structure

Shipwright's documentation will be updated to contain four main sections:

- "Quick Start"
- "Concepts"
- "How To"
- "Reference"

**_Quick Start_** will contain a small number of articles aimed to help evaluators demonstrate
Shipwright for others. The goal is to get an individual from "zero" to a successful "hello world"
build as quickly as possible.

**_Concepts_** will contain articles that explain "what" something is in Shipwright, and "why" it
exists. It will also help developers and evaluators understand how the different parts of
Shipwright relate to one another.

**_How To_** will contain articles that explain how to perform specific tasks. These may be divided
into sub-sections or groups, such as "Installation", "Running Builds", and so forth.

**_Reference_** will contain reference documentation for developers and administrators. Ideally
some of this content is generated directly from source code.

Accessing the main [docs page](https://shipwright.io/docs) will direct visitors to a landing page,
with clear links to these other sections.

#### Docs Migration

With the new outline in place, the existing content will be moved to new articles in appropriate
sections. Content should be moved in focused stages to ensure continuity. Changes to structure and
format should be expected. Net new content should be minimized to narrow the scope of changes.

#### Redirects

For content that is moved, Hugo [aliases](https://gohugo.io/methods/page/aliases/) should be used
to automatically configure redirects.

#### Removal of "in-tree" Documentation

Once content has been restructured, existing "in-tree" end user documentation in the `build`,
`cli`, and `operator` repositories should be migrated to the website repository. Not all docs that
are currently "in-tree" need to be migrated - only those that are relevant to the evaluator,
developer, or administrator personas described earlier. Once completed, this content should be
removed and replaced with a link to the appropriate Shipwright website article.

Moving forward, documentation can be submitted in one of two ways:

- Filing an appropriate PR against the `website` repository.
- Adding appropriate code comments or content that leads to auto-generated reference content.

Shipwright maintainers should insist that all features provide documentation at minimum via
generated reference content. Ideally contributors or the community work together to produce other
forms of content.


### Test Plan

Existing CI/CD infrastructure will be utilized to validate the new docs structure is deployable.

### Release Criteria

#### Removing a deprecated feature [if necessary]

N/A

#### Upgrade Strategy [if necessary]

N/A

### Risks and Mitigations

TBD

> What are the risks of this proposal and how do we mitigate? Think broadly. For example, consider
> both security and how this will impact the larger Shipwright ecosystem.

> How will security be reviewed and by whom? How will UX be reviewed and by whom?

## Drawbacks

TBD

> The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

TBD

> Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other
> possible approaches to delivering the value proposed by an enhancement.

## Infrastructure Needed [optional]

No new infrastructure.

## Implementation History

- 2025-04-21: Created as `provisional`
