---
name: update-owners
description: Update OWNERS file across all Shipwright repositories
---

# Update OWNERS file across all Shipwright repositories

## When to offer this skill

User asks to update the OWNERS file of repositories.

## Context gathering

The user must provide which user to add, remove or move in the OWNERS file. The user must also provide the repositories where the changes should be made. If no repositories are provided, then all Shipwright-related repositories will be updated.

## Instruction

In general, use both the `gh` and `git` commands outside of sandboxes so that you can use the user's authentication.

In general, if you create some temporary scripts that you then run to perform some of the work, then put them into the `.github/skills/update-owners/work/scripts` directory so that they are not accidentally committed.

Use the `gh` CLI to verify that users that should be added to an OWNERS file are actually member of the shipwright-io GitHub organization. If they are not, then inform the user that they need to be invited to the organization first.

Use the `gh` CLI to list all repositories in the shipwright-io organization.

Clone the relevant repositories using `git clone --single-branch --depth 1` with SSH protocol into the .github/skills/update-owners/work/repos directory. Some of the repositories may already be locally present. If so, reuse them. Make sure the default branch of the repository (usually main) is checked out and that the branch is up-to-date. If there are dirty files in any locally available repository, then discard those.

In all repositories, there is a file called OWNERS in the repository root which is in YAML format and lists approvers, reviewers and emeritus_approvers and emeritus_reviewers. You can check the existing files to get an overview.

Make the user-requested changes to the OWNERS file. When making somebody approver, make sure that user is also a reviewer.

In each repository where you made changes, do the following:

- Checkout a feature branch and commit your change using `git commit -s -m "<summary>"` where the summary should summarize the changes (for example "add user-abc to reviewers") you made. Use one message per change that you made.
- Push the branch to the remote.
- Open a pull request using the `gh` CLI with the following constraints:
  - If only one change was made, use that as PR description. Otherwise use `Update OWNERS`.
  - Use `.github/.github/pull_request_template` as template for the body. .github is one of the repositories of shipwright-io. You should have cloned it already earlier, if not do so now so that you can access the template.
    - In the **Changes** section, list the changes that you also used in your commit message.
    - Use `/kind cleanup`
    - Use `NONE` as release notes
    - Remove comment blocks that are instructing humans that are opening pull requests.
    - Have a new line at the end of the body.
