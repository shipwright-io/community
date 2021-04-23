<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

# Contributing Guidelines

Welcome to Shipwright, we are glad you want to contribute to the project!
This document contains general guidelines for submitting contributions.
Each component of Shipwright will have its own specific guidelines.

## Getting Started

All contributors must abide by our [Code of Conduct](/code-of-conduct.md).

The core code for Shipwright is located in the following repositories:

* [build](https://github.com/shipwright-io/build) - the Build APIs and associated controller to run builds.
* [cli](https://github.com/shipwright-io/cli) - the `shp` command line for Shipwright builds
* [operator](https://github.com/shipwright-io/operator) - an operator to install Shipwright components on Kubernetes via OLM.

Technical documentation is spread across the code repositories, and is consolidated in the [website](https://github.com/shipwright-io/website) repository.
Content in `website` is published to [shipwright.io](https://shipwright.io)

## Creating new Issues

We recommend to open an issue for the following scenarios:

- Asking for help or questions. (_Use the **discussion** or **help_wanted** label_)
- Reporting a bug. (_Use the **kind/bug** label_)
- Requesting a new feature. (_Use the **kind/feature** label_)

Use the following checklist to determine where you should create an issue:

- If the issue is related to how a Build or BuildRun behaves, or related to Build strategies, create an issue in [build](https://github.com/shipwright-io/build).
- If the issue is related to the command line, create an issue in [cli](https://github.com/shipwright-io/cli).
- If the issue is related to how the operator installs Shipwright on a cluster, create an issue in [operator](https://github.com/shipwright-io/operator).
- If the issue is related to the shipwright.io website, create an issue in [website](https://github.com/shipwright-io/website).

If you are not sure, create an issue in this repository, and the Shipwright maintainers will route it to the correct location.

## Writing Pull Requests

Contributions can be submitted by creating a pull request on Github. 
We recommend you do the following to ensure the maintainers can collaborate on your contribution:

- Fork the project into your personal Github account
- Create a new feature branch for your contribution
- Make your changes
- If you make code changes, ensure tests are passing
- Open a PR with a clear description, completing the pull request template if one is provided
  Please reference the appropriate GitHub issue if your pull request provides a fix.

## Code review process

Once your pull request is submitted, a Shipwright maintainer should be assigned to review your changes.

The code review should cover:

- Ensure all related tests (unit, integration and e2e) are passing.
- Ensure the code style is compliant with the [coding conventions](https://github.com/kubernetes/community/blob/master/contributors/guide/coding-conventions.md)
- Ensure the code is properly documented, e.g. enough comments where needed.
- Ensure the code is adding the necessary test cases (unit, integration or e2e) if needed.

Contributors are expected to respond to feedback from reviewers in a constructive manner.
Reviewers are expected to respond to new submissions in a timely fashion, with clear language if changes are requested.

Once the pull request is approved and marked "lgtm", it will get merged.

## Community Meetings Participation

We run the community meetings every Monday at 13:00 UTC time.
For each upcoming meeting we generate a new issue where we layout the topics to discuss.
See our [previous meetings](https://github.com/shipwright-io/build/issues?q=is%3Aissue+label%3Acommunity+is%3Aclosed) outcomes.
Please request an invite in our Slack [channel](https://kubernetes.slack.com/archives/C019ZRGUEJC) or join the [shipwright-dev mailing list](https://lists.shipwright.io/admin/lists/shipwright-users.lists.shipwright.io/).

All meetings are also published on our [public calendar](https://calendar.google.com/calendar/embed?src=shipwright-admin%40lists.shipwright.io&ctz=America%2FNew_York).

## Contact Information

- [Slack channel](https://kubernetes.slack.com/archives/C019ZRGUEJC)
- End-user email list: [shipwright-users@lists.shipwright.io](https://lists.shipwright.io/admin/lists/shipwright-users.lists.shipwright.io/)
- Developer email list: [shipwright-dev@lists.shipwright.io](https://lists.shipwright.io/admin/lists/shipwright-dev.lists.shipwright.io/)
