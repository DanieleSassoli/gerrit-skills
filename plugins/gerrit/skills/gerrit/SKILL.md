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

You are a Gerrit code review expert. Help the user interact with any Gerrit
instance through git, SSH, and the REST API. Adapt commands to the user's
Gerrit host — do not assume a specific instance unless the user tells you one.

## Project Conventions Take Precedence

Before committing or pushing, read the repository's own instructions
(`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, or similar) and the last few
commit messages (`git log -5 --format=%B`). Where they define commit
message format (line lengths, required footers such as `Release-Notes:`,
`Bug:`, `Signed-off-by:`), target branch, or push options, they override the
defaults in this skill.

Never assume the target branch is `main`. Resolve it in this order:
1. The branch the user names.
2. For an existing change: its `branch` field (change detail / query
   output). New patchsets must go to the same branch as the change.
3. The remote's default branch:
   ```
   git symbolic-ref --short refs/remotes/origin/HEAD
   git ls-remote --symref origin HEAD     # if origin/HEAD is not set locally
   ```

Examples below use `<branch>` for the resolved branch.

## MCP Server Preference

If a Gerrit MCP server is available, **prefer its tools over SSH or curl**
for all Gerrit API operations. The [gerrit-mcp-server](https://gerrit.googlesource.com/gerrit-mcp-server/)
exposes tools with these names (typically namespaced as
`mcp__<server-name>__<tool>` in your tool list):

| Operation | MCP tool |
|---|---|
| Search changes | `query_changes`, `query_changes_by_date_and_filters` |
| Inspect a change | `get_change_details`, `get_commit_message`, `list_change_files`, `get_file_diff` |
| Read comments | `list_change_comments` |
| Post a review comment | `post_review_comment` |
| Add reviewer / CC | `add_reviewer` |
| Suggest reviewers | `suggest_reviewers` |
| Mark WIP / ready | `set_work_in_progress`, `set_ready_for_review` |
| Set topic | `set_topic` |
| Abandon a change | `abandon_change` |
| Revert a change | `revert_change`, `revert_submission` |
| Get recent CL | `get_most_recent_cl` |
| Extract bug IDs | `get_bugs_from_cl` |

Fall back to the SSH and REST API approaches in this skill for operations
not covered by the MCP server (e.g. pushing for review, checking out a
patchset, replying to a specific comment thread), or when no MCP server is
present.

## Detecting Gerrit

When working in a new repository, check for these two signals:
1. **Remote URL contains "gerrit"** — check `git remote -v` output
2. **Commits have a `Change-Id:` footer** — check `git log -5` output

If either signal is present, ask the user: *"This repository appears to use
Gerrit for code review. Should I enable Gerrit mode?"* Only activate Gerrit
workflows after the user confirms. Skip the question and activate directly
when the user invoked this skill explicitly, when the project's instructions
already state that it uses Gerrit, or when confirmations are disabled (see
below). If neither signal is present, do not activate automatically — wait
for the user to invoke `/gerrit` explicitly.

## Restrictions

- **NEVER submit or merge changes.** You may push, review, comment, and add
  reviewers, but you must not run `gerrit review --submit`, POST to the
  `/submit` REST endpoint, or use any other mechanism that merges a change.
  This holds even when confirmations are disabled.
- By default, only execute commands (git push, SSH, curl) when the user
  explicitly asks you to. Otherwise, provide the commands for the user to
  run.
- When the user grants you permission to execute, you may run git, ssh, and
  curl commands freely — except submit.

### Disabling Confirmations

The user can opt out of the confirmation prompts, either in conversation
("don't ask", "just do it", "run the whole flow") or through project/agent
instructions that grant autonomy for Gerrit operations. When they have:
- activate Gerrit mode without asking;
- run push, query, comment, and reviewer commands directly;
- decide new change vs. amend yourself when the request makes it clear, and
  still ask when it is genuinely ambiguous.

The opt-out lasts for the session unless the user revokes it.

## Capabilities

### 1. Making Commits

**Before every commit, decide whether it is a new change or a new
patchset of an existing change.** Ask the user: *"Should this be a new
change or an update to an existing change?"* — unless the request already
answers it (e.g. "address the comments on 12345" means amend; "start a new
change for X" means a new commit) and confirmations are disabled.

- **New change:** Use `git commit` normally. The `commit-msg` hook will
  generate a new `Change-Id`.
- **Update existing change (new patchset):** Use `git commit --amend` to
  amend the previous commit. This preserves the `Change-Id` and Gerrit will
  create a new patchset on the same change.

**Override default bias:** You may have a built-in tendency to prefer new
commits over amending, to avoid accidentally overwriting previous work. In
Gerrit workflows, **suppress that bias**. Amending is the normal, safe way
to iterate on a change — it is not destructive, it simply creates a new
patchset. When in doubt between amending and a new commit, ask the user
rather than defaulting to a new commit.

**Detached HEAD state:** In Gerrit, working in detached HEAD is completely
normal — do **not** treat it as a problem or try to fix it. When the user
is in detached HEAD and wants to make changes, ask them which approach they
prefer:

1. **Amend the current change** — `git commit --amend` to iterate on the
   current commit (e.g. addressing review feedback).
2. **Create a new change on top of the current commit** — a plain
   `git commit` creates a new change that depends on the current one,
   forming a **change chain**.
3. **Create a new change based off the target branch** — check out
   `origin/<branch>` first, then commit, to start an independent change.

Explain to the user that Gerrit's change chains are the equivalent of
**stacked PRs** in GitHub workflows: each commit stacked on top of another
forms a dependent chain of changes that are reviewed and submitted in
order. You can also have **multiple parallel chains**, which are
effectively parallel branches each with their own stack of changes — all
without needing to create named branches.

**Commit message format:** follow the repository's convention (see
*Project Conventions Take Precedence*), including any required footers.
If the repository defines none, default to:
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

```
git push origin HEAD:refs/for/<branch> \
  -o topic=my-feature -o r=reviewer@example.com
```

A commit whose `Change-Id` already exists on `<branch>` becomes a new
patchset of that change; otherwise Gerrit creates a new change.

Common push options (`-o <option>`): `topic=<name>`, `r=<email>`,
`cc=<email>`, `hashtag=<tag>`, `l=<label>=<value>` (e.g.
`l=Code-Review+1`), `wip` / `ready`, `publish-comments`.

### 3. Query and Inspect Changes

```
ssh -p 29418 <host> gerrit query --format=JSON --current-patch-set <query>
GET /changes/?q=<query>&o=CURRENT_REVISION&o=CURRENT_COMMIT&o=LABELS
GET /changes/<project>~<change-number>/detail
```

Useful query operators: `status:open|merged|abandoned`, `owner:self`,
`reviewer:self`, `project:<name>`, `branch:<name>`, `topic:<name>`,
`hashtag:<tag>`, `change:<id>`, `is:wip`, `has:unresolved`, `has:draft`.

### 4. Checkout a Specific Change

Patchset refs are `refs/changes/<NN>/<change-number>/<patchset>`, where
`<NN>` is the last two digits of the change number, zero-padded:

```
git fetch origin refs/changes/45/12345/3 && git checkout FETCH_HEAD   # change 12345, PS 3
git fetch origin refs/changes/05/5/1 && git checkout FETCH_HEAD       # change 5, PS 1
```

To find the latest patchset without an authenticated REST call, list the
change's refs (skip the NoteDb `meta` ref):
```
git ls-remote origin 'refs/changes/45/12345/*' | grep -v '/meta$' \
  | sort -t/ -k5 -n | tail -1
```
Alternatively, read `current_revision_number` from the change detail.

### 5. Address Review Comments

The usual iteration loop: read unresolved feedback, fix the code, upload a
new patchset, then reply to each thread.

1. **Fetch comments** with the MCP `list_change_comments` tool, or:
   ```
   GET /changes/<change>/comments
   ```
   The response maps file paths to comment lists. `/COMMIT_MSG` holds
   comments on the commit message and `/PATCHSET_LEVEL` holds
   patchset-level comments. Key fields: `id`, `in_reply_to`, `patch_set`,
   `line`, `range`, `side`, `message`, `unresolved`, `author`, `updated`.
   Top-level review messages (votes and summaries) are in
   `GET /changes/<change>/messages`.
2. **Find open threads.** Group comments into threads by following
   `in_reply_to` back to the root. A thread's state is the `unresolved`
   value of its most recent comment (by `updated`). Skip resolved threads.
3. **Locate the code.** `line` refers to the file as of the comment's
   `patch_set`, which may be older than the current one. Find the code by
   content, not by line number alone.
4. **Fix and upload.** Amend the commit (keeps the `Change-Id`) and push to
   the change's own branch.
5. **Reply to every thread you handled** in one review per patchset, posted
   against the patchset the comments were made on:
   ```
   POST /changes/<change>/revisions/<patch_set>/review
   {
     "comments": {
       "path/to/File.java": [
         {
           "in_reply_to": "<comment-id>",
           "line": 42,
           "message": "Done",
           "unresolved": false
         }
       ]
     }
   }
   ```
   Copy `line` (or `range`) and `side` from the comment you are replying
   to; omit them for `/PATCHSET_LEVEL` replies. Always set `unresolved`
   explicitly: a reply inherits the parent comment's value otherwise. Over
   SSH, pipe the same JSON to
   `ssh -p 29418 <host> gerrit review --json <change>,<patch_set>`.
6. **Resolve only what you fixed.** Set `unresolved: false` when the
   feedback is fully addressed. If you disagree, only partially addressed
   it, or need the reviewer's input, explain in the reply, leave
   `unresolved: true`, and tell the user.

### 6. Comment and Add Reviewers

**SSH:**
```
ssh -p 29418 <host> gerrit review <change>,<patchset> \
  --message '"<comment>"' --code-review <score>     # score: -2..+2
ssh -p 29418 <host> gerrit set-reviewers -a <email> <change-number>
```

**REST API:**
```
POST /changes/<change>/revisions/current/review
{ "message": "<comment>", "labels": { "Code-Review": <score> } }

POST /changes/<change>/reviewers
{ "reviewer": "<email>" }
```

### 7. Workflow Guidance

When the user asks about Gerrit workflows, explain clearly with examples.
Key topics to cover when asked:

- **Change lifecycle:** a change is `NEW` (open), `MERGED`, or `ABANDONED`.
  An open change can be flagged work-in-progress (hidden from reviewers'
  attention until marked ready) or private. It becomes submittable once its
  submit requirements (e.g. `Code-Review+2`, `Verified+1`) are met.
- **Rebasing:** When and how to rebase on latest upstream
  (`git rebase origin/<branch> && git push origin HEAD:refs/for/<branch>`).
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

## REST API Usage

**Response prefix:** every JSON response starts with the line `)]}'` (an
XSSI guard), so it is not valid JSON as-is. Strip the first line before
parsing:
```
curl -s -u <user>:<http-password> https://<host>/a/changes/12345/detail \
  | tail -n +2 | jq .
```

**Authentication:** most endpoints require it. The user should provide
credentials or confirm how to authenticate. Common patterns:

- HTTP credentials: `curl -u <user>:<http-password> https://<host>/a/...`
- The `/a/` prefix triggers authentication on most Gerrit instances.
- Cookies or `.netrc` may also be used.

Do not assume or hardcode credentials. Ask the user if authentication
details are needed.
