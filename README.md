# feature-status

A Claude Code plugin that aggregates status from GitHub PRs, Jira kanban boards, and Jira feature issues into concise status updates suitable for posting as Jira comments.

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
