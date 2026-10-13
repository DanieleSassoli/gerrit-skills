---
name: gerrit
description: >
  Gerrit code review assistant. Use when the user works with Gerrit-based
  repositories: committing changes (new change vs. amend for a new
  patchset), pushing for review, querying or inspecting changes, addressing
  and replying to review comments, adding reviewers, checking out specific
  patchsets, or asking about Gerrit workflows (rebasing, cherry-picking,
  change chains, stacked changes, submit strategies). Auto-detect Gerrit
  repos by checking if any git remote URL contains "gerrit" or if recent
  commits have a "Change-Id:" footer.
user-invocable: true
disable-model-invocation: false
---

# Gerrit Code Review Skill

Help the user work with any Gerrit instance through git, SSH, and the REST
API. Do not assume a specific host unless the user names one.

## Project Conventions Take Precedence

Before committing or pushing, read the repository's instructions
(`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, ...) and recent commit
messages (`git log -5 --format=%B`). Their rules on commit message format
(line lengths, required footers such as `Release-Notes:` or `Bug:`), target
branch, and push options override this skill's defaults.

Never assume the target branch is `main`. Use, in order: the branch the
user names; for an existing change, its `branch` field (new patchsets must
go to the same branch); the remote default
(`git symbolic-ref --short refs/remotes/origin/HEAD`, or
`git ls-remote --symref origin HEAD`). Examples use `<branch>`.

## MCP Server Preference

If a Gerrit MCP server is available (e.g.
[gerrit-mcp-server](https://gerrit.googlesource.com/gerrit-mcp-server/):
`query_changes`, `list_change_comments`, `post_review_comment`,
`add_reviewer`, ...), prefer its tools over SSH or curl. Use the commands
below for what it does not cover (pushing, checking out a patchset,
replying to a specific comment thread) or when no server is present.

## Detecting Gerrit

Signals: a remote URL containing "gerrit" (`git remote -v`), or
`Change-Id:` footers in recent commits (`git log -5`). If either is
present, ask *"This repository appears to use Gerrit for code review.
Should I enable Gerrit mode?"* and activate only on confirmation. Skip the
question when the user invoked this skill, the project instructions say it
uses Gerrit, or confirmations are disabled. With no signal, wait for an
explicit `/gerrit`.

## Restrictions

- **NEVER submit or merge changes**: no `gerrit review --submit`, no POST
  to `/submit`, no other merge mechanism, even with confirmations disabled.
- By default, run git push, SSH, and curl only when the user asks;
  otherwise show the commands. Once permitted, run them freely (except
  submit).
- **Disabling confirmations:** if the user opts out ("don't ask", "run the
  whole flow") or project instructions grant autonomy, activate Gerrit mode
  without asking, run push/query/comment/reviewer commands directly, and
  decide new change vs. amend yourself when the request makes it clear.
  Still ask when it is genuinely ambiguous. Lasts for the session unless
  revoked.

## Capabilities

### 1. Making Commits

**Before every commit, decide: new change or new patchset?** Ask *"Should
this be a new change or an update to an existing change?"* unless the
request already answers it ("address the comments on 12345" means amend;
"start a new change for X" means a new commit) and confirmations are
disabled.

- **New change:** `git commit`; the `commit-msg` hook adds a `Change-Id`.
- **New patchset:** `git commit --amend`; the `Change-Id` is preserved.

**Override default bias:** suppress any preference for new commits over
amending. In Gerrit, amending is the normal, non-destructive way to iterate.
When in doubt, ask rather than default to a new commit.

**Detached HEAD is normal** in Gerrit; do not try to fix it. When the user
wants to make changes there, ask which they prefer:
1. Amend the current change (`git commit --amend`).
2. A new change on top of the current commit (`git commit`), forming a
   **change chain**, Gerrit's equivalent of stacked PRs. Several
   independent chains can coexist without named branches.
3. A new independent change: check out `origin/<branch>`, then commit.

**Commit message format:** follow the repository's convention, including
required footers. If it defines none: subject at most 50 characters,
imperative, no trailing period; blank line; body wrapped at 79 characters;
footers (`Bug:`, `Change-Id:`) last, after a blank line.

**Missing commit-msg hook:** if a push fails for lack of a `Change-Id`,
report the error as-is. Do not install the hook yourself.

### 2. Push for Review

```
git push origin HEAD:refs/for/<branch> -o topic=<name> -o r=<email>
```

A `Change-Id` that already exists on `<branch>` uploads a new patchset;
otherwise Gerrit creates a new change. Other options: `cc=<email>`,
`hashtag=<tag>`, `l=Code-Review+1`, `wip`, `ready`, `publish-comments`.

### 3. Query and Inspect

```
ssh -p 29418 <host> gerrit query --format=JSON --current-patch-set <query>
GET /changes/?q=<query>&o=CURRENT_REVISION&o=CURRENT_COMMIT&o=LABELS
GET /changes/<project>~<change-number>/detail
```

Operators: `status:open|merged|abandoned`, `owner:self`, `reviewer:self`,
`project:`, `branch:`, `topic:`, `hashtag:`, `is:wip`, `has:unresolved`,
`has:draft`.

### 4. Checkout a Patchset

Refs are `refs/changes/<NN>/<change>/<patchset>`, `<NN>` being the last two
digits of the change number, zero-padded (change 5 → `05/5/1`):

```
git fetch origin refs/changes/45/12345/3 && git checkout FETCH_HEAD
```

Latest patchset without authentication:
```
git ls-remote origin 'refs/changes/45/12345/*' | grep -v '/meta$' \
  | sort -t/ -k5 -n | tail -1
```

### 5. Address Review Comments

1. **Fetch** with MCP `list_change_comments` or
   `GET /changes/<change>/comments`: a map of file path (`/COMMIT_MSG`,
   `/PATCHSET_LEVEL` for patchset-level) to comments with `id`,
   `in_reply_to`, `patch_set`, `line`, `range`, `side`, `message`,
   `unresolved`, `updated`. Summary messages and votes:
   `GET /changes/<change>/messages`.
2. **Find open threads:** follow `in_reply_to` to the root; a thread is
   open if its most recent comment (by `updated`) has `unresolved: true`.
3. **Locate the code by content:** `line` refers to the comment's
   `patch_set`, which may be older than the current one.
4. **Fix, amend, and push** to the change's own branch.
5. **Reply** in one review per patchset, against the patchset the comments
   were made on:
   ```
   POST /changes/<change>/revisions/<patch_set>/review
   {"comments": {"path/to/File.java": [
     {"in_reply_to": "<id>", "line": 42, "message": "Done", "unresolved": false}
   ]}}
   ```
   Copy `line` (or `range`) and `side` from the parent; omit them for
   `/PATCHSET_LEVEL`. Always set `unresolved`: a reply otherwise inherits
   the parent's value. Over SSH, pipe the JSON to
   `ssh -p 29418 <host> gerrit review --json <change>,<patch_set>`.
6. **Resolve only what you fixed.** If you disagree, addressed it only
   partly, or need the reviewer's input, explain, leave it unresolved, and
   tell the user.

### 6. Comment and Add Reviewers

```
ssh -p 29418 <host> gerrit review <change>,<patchset> \
  --message '"<comment>"' --code-review <-2..+2>
ssh -p 29418 <host> gerrit set-reviewers -a <email> <change>

POST /changes/<change>/revisions/current/review
{"message": "<comment>", "labels": {"Code-Review": <score>}}
POST /changes/<change>/reviewers
{"reviewer": "<email>"}
```

### 7. Workflow Guidance

Explain Gerrit concepts (rebasing, cherry-picking, relation chains, topics,
submit strategies) when asked. A change is `NEW`, `MERGED`, or `ABANDONED`;
an open change may also be work-in-progress or private, and becomes
submittable once its submit requirements (e.g. `Code-Review+2`,
`Verified+1`) are met. After a rebase, push to `refs/for/<branch>` to
upload it as a new patchset.

## REST API Usage

Every JSON response starts with the XSSI guard line `)]}'`; strip it
before parsing:
```
curl -s -u <user>:<http-password> https://<host>/a/changes/12345/detail \
  | tail -n +2 | jq .
```

The `/a/` prefix requests authentication (HTTP password, cookies, or
`.netrc`). Never assume or hardcode credentials; ask the user how to
authenticate.
