# feature-status

A Claude Code plugin that aggregates status from GitHub PRs, Jira kanban boards, and Jira feature issues into concise status updates suitable for posting as Jira comments.

## Fork differences

This fork extends the [upstream plugin](https://github.com/matburt/feature-status) with interactive feature-lead workflows designed for daily async status reporting. Changes:

- **Human-authored status narrative** — After collecting data, the skill prompts for a human-written summary that appears at the top of the report, above the auto-generated sections.
- **Health labels** — Prompts for Green / Yellow / Red health status and syncs it to Jira labels on the ANSTRAT feature when posting (`--post`).
- **Self-recycling milestone comment** — Milestones are maintained as a separate Jira comment that gets rotated on each `--post` run, keeping it near the top of recent comments (Jira doesn't support pinning). A new comment is posted and the old one is updated to a strikethrough "superseded" message (the Atlassian MCP has no delete-comment tool). The skill finds the existing milestone comment by ID or by scanning, prompts to update/keep/skip, and rotates it. Milestone comment ID is tracked in state.
- **Target end date management** — Shows the current target end date from the Jira feature and allows updating it inline during the status flow.
- **Feature metadata display** — Before prompting, shows the current state of the Jira feature (health label, target start/end dates) so the user can make informed updates.
- **Jira field updates on post** — `--post` mode now updates the health label and target end date on the Jira feature in addition to posting the status comment.
- **Two-level board metrics** — Board data gathering queries both epics (direct children of the feature) and their sub-tasks. Metrics and staleness are computed from sub-tasks, not epics — an epic with active children is not flagged as stale even if the epic issue itself hasn't been updated.

## Usage

```bash
# Preview status for a specific feature area
/feature-status installer

# Post status as a Jira comment
/feature-status installer --post

# Preview all enabled feature areas
/feature-status --all

# Post all
/feature-status --all --post
```

## Configuration

Feature areas are defined in `skills/status/config/features.json`. Each feature specifies:

- **name** — area identifier (e.g., "installer", "authz")
- **summary** — human-readable description
- **jiraFeature** — top-level Jira issue key for posting status
- **enabled** — whether to include in `--all` runs
- **sources** — data sources to aggregate:
  - `github.repos` — repositories and branch patterns to monitor
  - `github.prSearchTerms` — PR search filters
  - `githubProjects` — GitHub Projects V2 boards to pull item status from
  - `jiraBoards` — kanban boards to track
  - `jiraIssues` — Jira issues to fetch details from

## State

Mutable state (run history, draft outputs) is stored inside the skill directory:

```
{skill-dir}/state/{area-name}/last-run.json    # last execution metadata
{skill-dir}/output/{area-name}-{date}-draft.md # generated status drafts
```

For in-project installation this resolves to `.claude/skills/feature-status/state/` — entirely within the project.

## Installation

### Method A — In-project
Copy or symlink the skill directly into your project's `.claude/skills/` directory:

```bash
# From inside the target project:
mkdir -p .claude/skills

# Symlink (changes to plugin source are reflected immediately):
ln -s /path/to/feature-status/skills/feature-status .claude/skills/feature-status

# Or copy (standalone, no dependency on plugin source):
cp -r /path/to/feature-status/skills/feature-status .claude/skills/feature-status
```

Then copy or symlink the config:
```bash
cp /path/to/feature-status/skills/feature-status/config/features.example.json \
   .claude/skills/feature-status/config/features.json
# Edit features.json with your feature areas
```

State and drafts are written to `.claude/skills/feature-status/state/` and `output/` within the project.

### Method B — Global plugin (available in all projects)

```bash
mkdir -p ~/.claude/plugins/local/feature-status/.claude-plugin
cp .claude-plugin/plugin.json ~/.claude/plugins/local/feature-status/.claude-plugin/
ln -s /path/to/feature-status/skills/feature-status \
      ~/.claude/plugins/local/feature-status/skills/feature-status
```

## Requirements

- `gh` CLI (authenticated with access to monitored repos)
- Jira MCP server (for board/issue data and posting comments)
