# Contributing to ToolHive RFCs <!-- omit from toc -->

First off, thank you for taking the time to contribute to ToolHive! :+1: :tada:
ToolHive is released under the Apache 2.0 license. If you would like to
contribute an RFC or want to participate in the design process, this document
should help you get started.

## Table of contents <!-- omit from toc -->

- [Code of conduct](#code-of-conduct)
- [Reporting security vulnerabilities](#reporting-security-vulnerabilities)
- [How to contribute](#how-to-contribute)
  - [Using Discord Threads](#using-github-issues-and-discussions)
  - [Not sure how to start contributing?](#not-sure-how-to-start-contributing)
  - [RFC submission process](#rfc-submission-process)
  - [RFC file naming format](#rfc-file-naming-format)
  - [RFC content guidelines](#rfc-content-guidelines)
  - [Commit message guidelines](#commit-message-guidelines)

## Code of conduct

This project adheres to the
[Contributor Covenant](https://github.com/stacklok/toolhive/blob/main/CODE_OF_CONDUCT.md)
code of conduct. By participating, you are expected to uphold this code. Please
report unacceptable behavior to
[code-of-conduct@stacklok.dev](mailto:code-of-conduct@stacklok.dev).

## Reporting security vulnerabilities

If you think you have found a security vulnerability in any ToolHive project,
please DO NOT disclose it publicly until we've had a chance to fix it. Please
don't report security vulnerabilities using GitHub issues; instead, please
follow this
[process](https://github.com/stacklok/toolhive/blob/main/SECURITY.MD).

## How to contribute

### Using Discord Threads

You can reach out to the team using [Discord](https://discord.gg/stacklok),
which we use for all our interactions with users and contributors, and also
for RFC ideas and general design conversations. To avoid confusion, we
recommend using Discord threads, this will make easier to follow conversations.

If you have an idea for a significant change but aren't sure if it warrants an
RFC, start a thread first.

GitHub Issues in this repository are used to track RFC status and any
meta-discussions about the RFC process itself.

For general usage questions about ToolHive, please ask in
[ToolHive's discussion forum](https://discord.gg/stacklok).

### Not sure how to start contributing?

If you're new to the RFC process:

1. Read through existing RFCs to understand the format and level of detail expected
2. Start with a thread on [Discord](https://discord.gg/stacklok) to validate your idea
3. Review the [RFC template](rfcs/0000-template.md) to understand what sections are required

### RFC submission process

1. **Start a thread (optional but recommended)**: Open a thread on Discord
   to gather initial feedback on your idea before investing time in a full RFC.

2. **Fork and create your RFC**:
   - Fork this repository to your own GitHub account
   - Copy `rfcs/0000-template.md` to `rfcs/THV-XXXX-descriptive-name.md`
   - Use the next available RFC number (check existing RFCs)
   - Fill in all required sections of the template

3. **Submit a Pull Request**:
   - All commits must include a Signed-off-by trailer at the end of each commit
     message to indicate that the contributor agrees to the Developer Certificate
     of Origin
   - Ensure the PR title clearly describes the RFC (e.g., "RFC: Add token exchange middleware")
   - Link to any related GitHub Discussions or issues in other repositories

4. **Address feedback**:
   - Respond to reviewer comments
   - Update the RFC based on feedback
   - Be prepared for multiple rounds of review

5. **RFC decision**:
   - Maintainers will make a decision to accept, reject, or postpone the RFC
   - The RFC status will be updated accordingly
   - Accepted RFCs can proceed to implementation

### RFC file naming format

All RFC files must follow this naming pattern:

```
THV-{NUMBER}-{descriptive-name}.md
```

Where:
- `{NUMBER}` is a four-digit sequential number that must be equal to the PR number (e.g., 0001, 0002)
- `{descriptive-name}` is a short description in kebab-case

#### Examples of valid RFC names:
- `THV-0001-token-exchange-middleware.md`
- `THV-0002-kubernetes-crd-improvements.md`
- `THV-0003-registry-api-v2.md`

A CI job will make sure you're following the right numbering.

### RFC content guidelines

A good RFC should:

- **Clearly state the problem**: Explain what problem you're trying to solve and
  why it matters
- **Be specific**: Include concrete examples, API definitions, and implementation
  details
- **Address security**: The Security Considerations section is required for all
  RFCs. Consider threats, authentication, authorization, data security, and
  mitigations
- **Consider alternatives**: Show that you've thought about other approaches and
  explain why your proposal is the best choice
- **Plan for compatibility**: Address backward and forward compatibility concerns
- **Include diagrams**: Use Mermaid diagrams to illustrate complex flows or
  architectures
- **Keep it focused**: Each RFC should address a single cohesive change. Split
  large proposals into multiple RFCs if needed

### Commit message guidelines

We follow the commit formatting recommendations found on
[Chris Beams' How to Write a Git Commit Message article](https://chris.beams.io/posts/git-commit/):

1. Separate subject from body with a blank line
1. Limit the subject line to 50 characters
1. Capitalize the subject line
1. Do not end the subject line with a period
1. Use the imperative mood in the subject line
1. Use the body to explain what and why vs. how
