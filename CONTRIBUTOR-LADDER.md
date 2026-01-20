# Contributor Ladder

Welcome! This guide explains how people participate in Shipwright and how you can grow from a
`Participant` to a `Maintainer`. It sets clear expectations for each role and what privileges come
with them, so you always know what’s next and how to get there. You don’t need permission to get
started — jump in where you’re comfortable, and we’ll help you level up!

## Participants

Participants are any members of the public who interact with the Shipwright project.

### Expectations

- Follow the [Code of Conduct].
- Ask questions, report issues, and engage constructively in discussions.
- Share feedback and propose ideas in issues, discussions, or meetings.

### Privileges

- Read access to public repositories (default GitHub behavior).
- Open and comment on issues and pull requests.
- Attend public community meetings and join public communication channels.

### Becoming a Participant

No process required. Participation is open to everyone - [join us](./README.md)!

## Contributors

Contributors are recognized members of the community who have demonstrated meaningful
participation and are invited into the `shipwright-io` GitHub organization.

Typical contributions include (non-exhaustive):

- Submitting or reviewing pull requests.
- Authoring or improving documentation, examples, and tests.
- Triaging issues, reproducing bugs, and improving labels.
- Participating actively in discussions and meetings.

### Expectations

- Consistently contribute high-quality changes or reviews.
- Follow project workflows and guidelines (e.g., contribution and review processes).
- Review code contributions from others, encouraging community members to follow the project
  [Contribution Guide] and feedback from Approvers/Maintainers.
- Communicate clearly and respectfully; help newcomers.

### Privileges

- GitHub organization membership in `shipwright-io`.
- Read permission to `shipwright-io` private repositories.
- `Triage` permission for all `shipwright-io` repositories.
- Ability to apply the `lgtm` label to pull requests, making code contributions a candidate for
  merging.
- Eligibilty to be added as a `reviewer` in a repository `OWNERS` file, communicating their
  interest in a specific subject area.
- Eligibility for nomination to Approver on specific subject areas.

### Becoming a Contributor

- Create a GitHub Account and enable [two-factor authentication](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/about-two-factor-authentication).
- Demonstrate a track record of constructive participation. Contributor scores from [LFX Insights]
  may be considered, however there is no minimum score required for promotion.
- Express interest to current community members or be nominated by an existing community member.

Once approved, Shipwright maintainers will extend an invitation to join the `shipwright-io` GitHub
organization. You must accept the invitation in order to be considered a `Contributor`.

## Approvers

Approvers are recognized authorities for one or more subject areas and are allowed to approve
code changes in their scope.

### Expectations

- Perform high-quality, timely reviews in scoped areas; ensure correctness, security, and
  maintainability.
- Approve pull requests when ready by applying the `approve` label. Request actionable changes when
  necessary and in accordance with the [Contribution Guide].
- Guide `Contributors` and mentor new reviewers.
- Uphold project standards and help refine processes and documentation.
- Provide feedback on [SHIP proposals] impacting their area of expertise.

### Privileges

- Approval permissions on repositories for their subject areas, enforced via repository
  permissions and `OWNERS` files.
- `Write` permission for all Shipwright repositories.
- Abilty to manage GitHub action workflows, including release workflows.
- Listed as `Approvers` in the Shipwright [Maintainers List].
- Influence on technical direction within their scope.

### Becoming an Approver

- Demonstrate sustained, high-quality contributions and reviews within a subject area. Contribuor
  scores from [LFX Insights] may be considered, however there is no minimum score required to
  become an Approver.
- Be nominated by a `Maintainer` or `Approver` based on proficiency and responsiveness.
- Gain approval by Maintainers via lazy consensus or vote according to [Governance].
- Be added to relevant `OWNERS` files and granted necessary repository permissions.

## Maintainers

Maintainers have administrative rights for all of Shipwright and are responsible for the overall
health and direction of the project.

### Expectations

- Set technical direction, maintain project quality, and steward the roadmap.
- Actively participate in the [SHIP Proposals] process, providing feedback in their areas of
  expertise.
- Manage repository settings, CI, labels, and release processes as needed.
- Ensure inclusive, respectful collaboration and enforce the [Code of Conduct].
- Triage escalations and make decisions when consensus cannot be reached.
- Sponsor and mentor `Contributors` and `Approvers`.

### Privileges

- `Admin` permission for all Shipwright repositories.
- Ability to nominate and approve new Approvers and Maintainers per [Governance].
- Approval permission for [SHIP Proposals].
- Entry in the top-level [Maintainers List], granting them access to CNCF maintainer resources.
- Access to the project admin and security disclosure mailing lists.

### Becoming a Maintainer

- Demonstrate a successful track record as an `Approver` across one or more subject areas.
- Show leadership, reliability, and judgment in technical and community matters.
- Be nominated and approved by existing Maintainers as defined in [Governance].

## Inactivity and Involuntary Removal

Roles may be adjusted when responsibilities and requirements aren’t met, including extended
inactivity or violations of the [Code of Conduct]. Maintainers will attempt to re-engage inactive
role-holders with a grace period (typically 15–30 days). If unresponsive, role changes (e.g.,
removal from `OWNERS`, revocation of permissions) may be enacted to protect the community and its
deliverables.

Community members in good standing whose roles are adjusted may be granted an "emeritus" status
to recognize their prior contributions to the project. Community members who are removed due to
code of conduct violations are not eligible for emeritus status.

## Stepping Down and Stepping Back In

Community members may step down from a role or move to emeritus status if availability changes.
When circumstances permit, they may be re-nominated to resume prior responsibilities, subject to
the standard approval process.

## Mapping to GitHub Permissions

Shipwright maps community roles to GitHub repository
permissions as follows:

| Shipwright Role | GitHub Permission | Scope |
| --- | --- | --- |
| Participant | Read | Organization |
| Contributor | Triage | Organization |
| Approver | Write | Organization |
| Maintainer | Admin | Organization |

Note: Some Approvers may require `Maintain` or `Admin` permission for specific tasks (e.g. security,
repository settings). Admin access is granted sparingly and at the discretion of the project's
respective Maintainers.

## Related Documents

- [Governance charter][Governance]
- [Maintainers list][Maintainers List]
- `OWNERS` files — per repository/per directory for scoping approvals
- [Contributing Guide][Contribution Guide]


[Code of Conduct]: https://github.com/shipwright-io/.github/blob/main/CODE_OF_CONDUCT.md
[Contribution Guide]: https://github.com/shipwright-io/.github/blob/main/CONTRIBUTING.md
[LFX Insights]: https://insights.linuxfoundation.org/project/shipwright/contributors
[Governance]: https://github.com/shipwright-io/community/blob/main/GOVERNANCE.md
[SHIP Proposals]: https://github.com/shipwright-io/community/blob/main/ships/README.md
[Maintainers List]: https://github.com/shipwright-io/community/blob/main/MAINTAINERS.md
