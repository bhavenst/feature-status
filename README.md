# Feature Status Updater

A Claude Code skill that gathers data from GitHub PRs, GitHub Projects boards, Jira kanban boards, and Jira feature issues to produce concise, topical status updates. Supports dry-run mode to preview updates before posting them as Jira comments.

## Problem

Tracking feature progress across GitHub repos, project boards, and Jira issues is tedious. Status updates require manually correlating merged PRs, board movement, and Jira discussions into a coherent summary. This skill automates that correlation and produces a structured update ready to post.

## Quick Start

```bash
# Preview a status update (dry-run, default)
/feature-status my-feature

# Post the update as a comment on the Jira feature
/feature-status my-feature --post

# Preview all tracked features
/feature-status --all

# Post all
/feature-status --all --post
```

## Setup

1. Copy this directory into your project workspace
2. Edit `config/features.json` to define your features and their data sources
3. Run `/feature-status <name>` to generate your first update

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) with skill support
- `gh` CLI authenticated with access to your repos
- Jira MCP server configured (only if using Jira boards or issues — GitHub-only setups don't need this)

## Configuration

Define features in `config/features.json`:

```json
{
  "features": [
    {
      "name": "payments-v2",
      "summary": "Payment Processing Redesign",
      "jiraFeature": "PAY-1234",
      "enabled": true,
      "sources": {
        "github": {
          "repos": [
            { "owner": "myorg", "repo": "payments-api", "branchPatterns": ["feat/pay-*", "*PAY-1234*"] },
            { "owner": "myorg", "repo": "payments-ui" }
          ],
          "prSearchTerms": ["PAY-1234", "payments-v2"]
        },
        "githubProjects": [
          { "owner": "myorg", "number": 12, "statusField": "Status" }
        ],
        "jiraBoards": [
          { "boardId": 500, "project": "PAY", "jql": "status != Done ORDER BY updated DESC" }
        ],
        "jiraIssues": ["PAY-1234"]
      }
    }
  ]
}
```

### Configuration Fields

| Field | Description |
|-------|-------------|
| `name` | Short identifier used in commands |
| `summary` | Human-readable feature name |
| `jiraFeature` | Top-level Jira issue that receives the status comment. Optional — omit for GitHub-only workflows |
| `enabled` | Set `false` to skip without removing |
| `sources.github.repos` | Repos to scan for PRs, with optional branch pattern filters |
| `sources.github.prSearchTerms` | Additional search terms for PR discovery |
| `sources.githubProjects` | GitHub Projects V2 boards to pull item status from |
| `sources.jiraBoards` | Jira kanban/scrum boards to pull issue status from |
| `sources.jiraIssues` | Jira issues to check for comments and status changes |

## How It Works

### Data Gathering

The skill pulls data from four source types. All are optional — configure whichever combination fits your project:

**GitHub PRs** — Scans configured repos for open, recently merged, and recently closed PRs matching branch patterns or search terms. Computes review status and size.

**GitHub Projects** — Queries GitHub Projects V2 boards via GraphQL for item status, assignees, and staleness. Works with both org-level and user-level projects. The `statusField` config (default `"Status"`) maps to whatever single-select field your board uses for columns.

**Jira Boards** — Queries Jira kanban/scrum boards to build a snapshot: items in progress, in review, recently completed, stale, and newly created.

**Jira Issues** — Fetches the feature issue for current status, recent comments, and decisions.

### Incremental Updates

Each run records a timestamp in `state/<name>/last-run.json`. Subsequent runs only report changes since the last run, keeping updates topical rather than cumulative.

### Output Format

```markdown
## Status Update — Payment Processing Redesign (PAY-1234)
**Date:** 2025-03-15
**Period:** Since 2025-03-08

### Progress
- Bullet points of what moved forward

### Active Work
- Open PRs and in-progress board items

### Risks / Blockers
- Stale items, unresolved reviews, unassigned critical work

### Upcoming
- Items ready to start

### Metrics
- PRs: X opened, Y merged, Z closed
- Board: X in progress, Y completed
```

## Directory Structure

```
feature-status/
├── config/
│   └── features.json          # Feature definitions
├── skills/
│   └── status/
│       └── SKILL.md           # Skill implementation
├── state/                     # Per-feature sync state (gitignored)
│   └── <name>/
│       └── last-run.json
├── output/                    # Draft outputs (gitignored)
│   └── <name>-YYYY-MM-DD-draft.md
└── README.md
```

## Usage Patterns

### Daily standup prep

```bash
/feature-status --all
```

Preview status for all features before your standup. Copy what you need.

### Weekly stakeholder update

```bash
/feature-status my-feature --post
```

Post a structured update directly to the Jira feature so stakeholders see progress without asking.

### Tracking a teammate's area during PTO

Add their feature to your config, run dry-run updates to stay informed, post updates if needed.

### Adding manual context

Create `state/<name>/notes.md` with any context the automated sources can't capture. The skill includes this content in the status update.

## Tips

- Run dry-run first. Always preview before posting.
- Keep features focused. One Jira feature = one config entry.
- Branch patterns matter. Broad patterns (`*auth*`) catch more but may include noise.
- Board JQL can be tuned. Filter to specific epics or labels if the board is shared.
- Notes file is optional. Use it for decisions, context, or risks that aren't in PRs or Jira.
