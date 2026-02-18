# gerrit-skills

Claude Code skills for [Gerrit](https://www.gerritcodereview.com/) code review.

## What's Included

- **`/gerrit`** — A skill that helps you interact with any Gerrit instance through git, SSH, and the REST API. It can auto-detect Gerrit repositories, push changes for review, query and inspect changes, checkout patchsets, add reviewers, post comments, and guide you through Gerrit workflows.

## Installation

### Using `claude mcp add-skill` (recommended)

Run the following command to install the skill directly from GitHub:

```bash
claude skill add --url https://github.com/GerritForge/gerrit-skills/tree/main/skills/gerrit-skill
```

### Manual installation

1. Clone this repository:
   ```bash
   git clone https://github.com/GerritForge/gerrit-skills.git
   ```

2. Install the skill by pointing Claude Code to the local path:
   ```bash
   claude skill add --path /path/to/gerrit-skills/skills/gerrit-skill
   ```

## Usage

Once installed, Claude Code will automatically detect Gerrit repositories by checking for:
- Remote URLs containing "gerrit"
- Commits with a `Change-Id:` footer

You can also invoke the skill explicitly with `/gerrit`.

### Examples

- **Push a change for review:** Ask Claude to push your current commit for review on a specific branch.
- **Query changes:** Ask Claude to find open changes owned by you, or search by topic, hashtag, or project.
- **Checkout a patchset:** Ask Claude to fetch and checkout a specific change and patchset number.
- **Add reviewers / comments:** Ask Claude to add reviewers or post review comments on a change.
- **Workflow help:** Ask about rebasing, cherry-picking, change dependencies, submit strategies, and more.

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
