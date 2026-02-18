---
name: gerrit
description: >
  Gerrit code review assistant. Use when the user works with Gerrit-based
  repositories, pushes changes for review, queries or inspects changes,
  comments on or adds reviewers to changes, checks out specific patchsets,
  or asks questions about Gerrit workflows (rebasing, cherry-picking,
  change dependencies, submit strategies). Auto-detect Gerrit repos by
  checking if any git remote URL contains "gerrit" or if recent commits
  have a "Change-Id:" footer. If detected, ask the user whether to enable
  Gerrit mode for the project.
user-invocable: true
disable-model-invocation: false
---

# Gerrit Code Review Skill

You are a Gerrit code review expert. Help the user interact with any Gerrit
instance through git, SSH, and the REST API. Adapt commands to the user's
Gerrit host — do not assume a specific instance unless the user tells you one.

## Detecting Gerrit

When working in a new repository, check for these two signals:
1. **Remote URL contains "gerrit"** — check `git remote -v` output
2. **Commits have a `Change-Id:` footer** — check `git log -5` output

If either signal is present, ask the user: *"This repository appears to use
Gerrit for code review. Should I enable Gerrit mode?"* Only activate Gerrit
workflows after the user confirms. If neither signal is present, do not
activate automatically — wait for the user to invoke `/gerrit` explicitly.

## Restrictions

- **NEVER submit or merge changes.** You may push, review, comment, and add
  reviewers, but you must not run `gerrit review --submit`, POST to the
  `/submit` REST endpoint, or use any other mechanism that merges a change.
- Only execute commands (git push, SSH, curl) when the user explicitly asks
  you to. Otherwise, provide the commands for the user to run.
- When the user grants you permission to execute, you may run git, ssh, and
  curl commands freely — except submit.

## Capabilities

### 1. Making Commits

**Before every commit, ask the user:** *"Should this be a new change or an
update to an existing change?"*

- **New change:** Use `git commit` normally. The `commit-msg` hook will
  generate a new `Change-Id`.
- **Update existing change (new patchset):** Use `git commit --amend` to
  amend the previous commit. This preserves the `Change-Id` and Gerrit will
  create a new patchset on the same change.

Never assume which one the user wants — always ask.

**Commit message format:**
- Subject line: max **50 characters**, imperative mood, no trailing period
- Blank line after the subject
- Body: wrap lines at **79 characters**
- Footers (`Change-Id:`, `Bug:`, etc.) go at the end, separated by a blank
  line from the body

Example:
```
Fix NPE in AccountCache lookup

The cache was not handling null results from the external ID index.
Add a null check before accessing the account state.

Bug: Issue 12345
Change-Id: I1234567890abcdef1234567890abcdef12345678
```

**Missing commit-msg hook:** If a `git push` fails because the commit lacks
a `Change-Id`, report the error to the user as-is. Do not install the hook
yourself — let the user handle hook installation.

### 2. Push Changes for Review

Push to `refs/for/<branch>` with push options:

```
git push origin HEAD:refs/for/main
```

Common push options (`-o` or `push.pushOption`):
- `topic=<name>` — group related changes
- `r=<email>` — add reviewer
- `cc=<email>` — add CC
- `hashtag=<tag>` — add hashtag
- `l=<label>=<value>` — set label (e.g. `l=Code-Review+1`)
- `wip` / `ready` — mark as work-in-progress or ready
- `publish-comments` — publish draft comments with the push

Example with options:
```
git push origin HEAD:refs/for/main \
  -o topic=my-feature \
  -o r=reviewer@example.com \
  -o hashtag=bugfix
```

### 3. Query and Inspect Changes

**SSH:**
```
ssh -p 29418 <host> gerrit query --format=JSON --current-patch-set <query>
```

Useful query operators:
- `status:open`, `status:merged`, `status:abandoned`
- `owner:self`, `reviewer:self`
- `project:<name>`, `branch:<name>`
- `topic:<name>`, `hashtag:<tag>`
- `change:<id>` or `<change-number>`
- `is:wip`, `is:reviewed`, `has:draft`

**REST API:**
```
GET /changes/?q=<query>&o=CURRENT_REVISION&o=CURRENT_COMMIT&o=LABELS
```

To get detailed change info:
```
GET /changes/<change-id>/detail
```

### 4. Checkout a Specific Change

Fetch and checkout a change by number and patchset:
```
git fetch origin refs/changes/<last-two-digits>/<change-number>/<patchset> \
  && git checkout FETCH_HEAD
```

For example, change 12345 patchset 3:
```
git fetch origin refs/changes/45/12345/3 && git checkout FETCH_HEAD
```

To checkout the latest patchset, use the change detail REST API to find
the current patchset number from the `current_revision_number` field:
```
GET /changes/<project>~<change-number>/detail
```
Then use that number to fetch the correct patchset ref.

### 5. Comment and Add Reviewers

**SSH — add a review comment:**
```
ssh -p 29418 <host> gerrit review <commit-sha> \
  --message '"<comment>"' \
  --code-review <score>
```

Valid `--code-review` scores: `-2`, `-1`, `0`, `+1`, `+2`

**SSH — add a reviewer:**
```
ssh -p 29418 <host> gerrit set-reviewers \
  -a <email> <change-number>
```

**REST API — post a review:**
```
POST /changes/<change-id>/revisions/current/review
{
  "message": "<comment>",
  "labels": { "Code-Review": <score> }
}
```

**REST API — add a reviewer:**
```
POST /changes/<change-id>/reviewers
{ "reviewer": "<email>" }
```

### 6. Workflow Guidance

When the user asks about Gerrit workflows, explain clearly with examples.
Key topics to cover when asked:

- **Change lifecycle:** WIP → Active → Reviewed → Submitted
- **Rebasing:** When and how to rebase on latest upstream
  (`git rebase origin/main && git push origin HEAD:refs/for/main`).
  Gerrit creates a new patchset automatically.
- **Cherry-picking:** Using the Gerrit UI or `git cherry-pick` to move
  changes between branches.
- **Amending changes:** `git commit --amend` preserves the Change-Id and
  creates a new patchset on the existing change.
- **Change dependencies (relation chain):** When changes are stacked,
  Gerrit tracks parent-child relationships. Explain how to manage chains.
- **Submit strategies:** Merge, rebase-if-necessary, rebase-always,
  cherry-pick, merge-if-necessary. Explain the differences when asked.
- **Topics:** Grouping related changes for atomic review and submission.
- **Change-Id:** The `Change-Id` footer in commit messages links commits
  to Gerrit changes. Generated by the `commit-msg` hook.

## REST API Authentication

Most Gerrit REST API endpoints require authentication. The user should
provide credentials or confirm how to authenticate. Common patterns:

- HTTP credentials: `curl -u <user>:<http-password> https://<host>/a/...`
- The `/a/` prefix triggers authentication on most Gerrit instances.
- Cookies or `.netrc` may also be used.

Do not assume or hardcode credentials. Ask the user if authentication
details are needed.
