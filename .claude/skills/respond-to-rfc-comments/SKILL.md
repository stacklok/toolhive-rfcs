---
name: respond-to-rfc-comments
description: Help respond to review comments on an RFC pull request. Use when the user wants to address, reply to, or resolve feedback on an open RFC PR for toolhive, toolhive-studio, toolhive-registry, toolhive-registry-server, toolhive-cloud-ui, or dockyard projects.
allowed-tools: Read, Glob, Grep, Bash(git:*), mcp__github__*, mcp__fetch__fetch, WebFetch, Edit, Write, AskUserQuestion
---

# Respond to RFC Comments Skill

This skill helps you systematically respond to review comments on an RFC pull request by proposing changes, getting user approval, updating the RFC, and posting replies.

## Overview

When an RFC is under review, reviewers leave comments on the PR — questions, suggestions, nitpicks, and blockers. This skill walks through each comment one at a time, proposes a response and RFC change, gets the user's approval (with possible adjustments), applies the change, and posts the reply.

## Workflow

### Step 1: Fetch All Comments

Gather all review feedback from the PR:

1. Use `mcp__github__pull_request_read` with method `get_review_comments` to fetch inline review threads.
2. Use `mcp__github__pull_request_read` with method `get_comments` to fetch PR-level comments.
3. Filter to **unresolved** threads and actionable PR-level comments (skip resolved/outdated threads and the user's own replies).

### Step 2: Read the RFC

Read the full RFC document so you have context for every comment.

### Step 3: Present Comments One at a Time

For each comment, present to the user:

1. **Who** left the comment and **where** (line number or PR-level).
2. **The comment** — quote it or summarize it clearly.
3. **Proposed response** — what you would reply on the PR.
4. **Proposed RFC change** — what specific edit to make (or "none needed").

Then ask: **Approve, adjust, or skip?**

- **Approve**: Apply the RFC change immediately, then move to the next comment.
- **Adjust**: The user provides corrections or different direction. Revise the proposal and re-present for approval.
- **Skip**: Move to the next comment without changes.
- **Go back**: Re-present the previous comment for further adjustment.

Important interaction rules:
- Present exactly **one comment at a time**. Do not batch or summarize multiple comments.
- Wait for explicit user approval before making any edits to the RFC.
- When the user says "approve", apply the RFC change immediately before presenting the next comment.
- When the user provides direction that changes the proposal, revise and re-present — do not apply until approved.
- Track which comments have been approved so you can post all replies together at the end.

### Step 4: Post Replies

Once all comments have been reviewed, ask the user if they want to post the replies. When confirmed:

1. Use `mcp__github__add_reply_to_pull_request_comment` for each inline review comment, using the **numeric comment ID** (not the GraphQL node ID).
2. Use `mcp__github__add_issue_comment` for PR-level comment responses.
3. Post all replies in a single batch (parallel tool calls).

To get numeric comment IDs from GraphQL node IDs, use:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate --jq '.[] | "\(.id) \(.body[:60])"'
```

### Step 5: Commit and Push

After replies are posted, ask the user if they want to commit and push. If confirmed:

1. Stage only the RFC file.
2. Write a commit message summarizing all changes, referencing the reviewers addressed.
3. Push to the PR branch.

## Crafting Good Responses

### Response Tone
- Be direct and constructive.
- Acknowledge the reviewer's point before explaining the change.
- For questions: answer the question, then state what RFC change (if any) you're making.
- For suggestions you agree with: say so concisely, describe the change.
- For suggestions you disagree with: explain the reasoning, propose an alternative if relevant.
- For blockers: treat these as high priority — always propose a concrete resolution.

### RFC Changes
- Make the minimum edit needed to address the comment.
- When renaming fields or terms, use `replace_all` to catch every occurrence.
- When adding new sections, match the style of existing sections in the RFC.
- When adding sections required by the template (e.g., Security Considerations), reference existing RFCs in the repo for structural patterns:
  - Search for the section name across `rfcs/THV-*.md` files.
  - Use 2-3 examples to match the established format (threat model tables, subsection structure, etc.).

### PR-Level Comment Responses
- When responding to a PR-level comment that requests multiple changes, summarize all the changes made in one reply.
- Tag the reviewer with `@username` so they get notified.

## Edge Cases

- **Comment on outdated code**: If the comment references code that has already changed, note this in your proposal and suggest whether the comment is still relevant.
- **Conflicting comments**: If two reviewers disagree, flag this to the user and let them decide the direction.
- **Comments that require design decisions**: These need user input — present the tradeoffs and ask for direction rather than proposing a specific change.
- **Comments that invalidate other parts of the RFC**: Flag cascading impacts (e.g., "if we only rate-limit `tools/call`, then the `prompts:` config section may no longer apply").
