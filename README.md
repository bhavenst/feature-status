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
  - `jiraBoards` — kanban boards to track
  - `jiraIssues` — Jira issues to fetch details from

## State

Mutable state (run history, draft outputs) is stored in `~/.claude/feature-status/`:

```
~/.claude/feature-status/
  state/{area-name}/last-run.json    # last execution metadata
  output/{area-name}-{date}-draft.md # generated status drafts
```

## Installation

The plugin is installed as a local plugin via symlink:

```bash
mkdir -p ~/.claude/plugins/local/feature-status/.claude-plugin
cp .claude-plugin/plugin.json ~/.claude/plugins/local/feature-status/.claude-plugin/
ln -s /path/to/feature-status/skills ~/.claude/plugins/local/feature-status/skills
```

## Requirements

- `gh` CLI (authenticated with access to monitored repos)
- Jira MCP server (for board/issue data and posting comments)
