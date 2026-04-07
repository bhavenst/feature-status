---
name: feature-status
description: Gather data from GitHub PRs, GitHub Projects boards, Jira kanban boards, and Jira features to produce a concise status update. Supports dry-run mode (default) to preview the update before posting it as a Jira comment. Use when user asks for feature status, status update, progress report, or wants to update a Jira feature with current status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(gh *)
  - Bash(git *)
  - Bash(jq *)
  - Bash(date *)
  - Bash(mkdir *)
  - Bash(cp *)
  - Bash(wc *)
---

# /feature-status - Feature Status Updater

Gathers status from multiple sources (GitHub PRs, GitHub Projects boards, Jira kanban boards, Jira features) and produces a concise, topical status update suitable for posting as a comment on the top-level Jira feature.

**Arguments**: `$ARGUMENTS`

Supported forms:
- `/feature-status <name>` — dry-run (default), outputs status to console
- `/feature-status <name> --post` — posts the status as a Jira comment on the feature
- `/feature-status --all` — dry-run for all enabled features
- `/feature-status --all --post` — posts status for all enabled features

Where `<name>` matches a name in `config/features.json`.

## Implementation

### 1. Setup and Configuration

Locate the feature-status root directory by searching upward from the current working directory for a `feature-status/config/features.json` file, or by checking common locations:

```bash
# Search for config relative to cwd, then common locations
for candidate in \
  "./feature-status" \
  "../feature-status" \
  "$(git rev-parse --show-toplevel 2>/dev/null)/feature-status"; do
  if [[ -f "$candidate/config/features.json" ]]; then
    STATUS_ROOT="$(cd "$candidate" && pwd)"
    break
  fi
done

CONFIG_FILE="$STATUS_ROOT/config/features.json"
DRY_RUN=true

# Parse arguments
AREA_NAME=""
POST_MODE=false

for arg in $ARGUMENTS; do
  case "$arg" in
    --post) POST_MODE=true; DRY_RUN=false ;;
    --all) AREA_NAME="__all__" ;;
    *) AREA_NAME="$arg" ;;
  esac
done

if [[ -z "$AREA_NAME" ]]; then
  echo "Usage: /feature-status <name> [--post]"
  echo "       /feature-status --all [--post]"
  echo ""
  echo "Available features:"
  jq -r '.features[] | select(.enabled == true) | "  \(.name) — \(.jiraFeature) (\(.summary))"' "$CONFIG_FILE"
  exit 0
fi
```

### 2. Load Feature Configuration

Each feature in `config/features.json` defines its data sources:

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

```bash
if [[ "$AREA_NAME" == "__all__" ]]; then
  AREAS=$(jq -r '.features[] | select(.enabled == true) | .name' "$CONFIG_FILE")
else
  AREA_CONFIG=$(jq --arg name "$AREA_NAME" '.features[] | select(.name == $name)' "$CONFIG_FILE")
  if [[ -z "$AREA_CONFIG" ]]; then
    echo "ERROR: Feature '$AREA_NAME' not found in config"
    exit 1
  fi
  AREAS="$AREA_NAME"
fi
```

### 3. For Each Feature: Gather Data

#### 3.1. Load Previous State

```bash
STATE_DIR="$STATUS_ROOT/state/$AREA_NAME"
mkdir -p "$STATE_DIR"

LAST_RUN_FILE="$STATE_DIR/last-run.json"
if [[ -f "$LAST_RUN_FILE" ]]; then
  LAST_RUN_DATE=$(jq -r '.lastRun // "1970-01-01T00:00:00Z"' "$LAST_RUN_FILE")
else
  LAST_RUN_DATE=$(date -u -d '14 days ago' +"%Y-%m-%dT%H:%M:%SZ")
fi
```

#### 3.2. Fetch GitHub PR Data

For each repo defined in `sources.github.repos`:

```bash
# Search for open PRs matching branch patterns or search terms
gh pr list --repo "$OWNER/$REPO" --state open --limit 30 \
  --json number,title,author,state,headRefName,updatedAt,createdAt,isDraft,url,additions,deletions,changedFiles,reviews

# Search for recently merged PRs since last run
gh pr list --repo "$OWNER/$REPO" --state merged --limit 20 \
  --search "merged:>=$LAST_RUN_DATE" \
  --json number,title,author,state,headRefName,mergedAt,url,additions,deletions,changedFiles

# Filter PRs by branch patterns and search terms
# Classify: open-needing-review, open-in-progress, recently-merged, recently-closed
```

For each relevant PR, compute:
- Whether it was opened, merged, or updated since `LAST_RUN_DATE`
- Size category (small <100, medium <500, large <2000, extra-large 2000+)
- Review status (approved, changes-requested, pending)

#### 3.3. Fetch GitHub Projects Board Data

For each project in `sources.githubProjects`:

Query the GitHub Projects V2 GraphQL API to get items and their status column values:

```bash
gh api graphql -f query='
query($owner: String!, $number: Int!) {
  organization(login: $owner) {
    projectV2(number: $number) {
      title
      items(first: 100) {
        nodes {
          fieldValueByName(name: "Status") {
            ... on ProjectV2ItemFieldSingleSelectValue { name }
          }
          content {
            ... on Issue {
              number
              title
              assignees(first: 3) { nodes { login } }
              state
              updatedAt
              url
              labels(first: 5) { nodes { name } }
            }
            ... on PullRequest {
              number
              title
              author { login }
              state
              updatedAt
              url
            }
          }
          updatedAt
        }
      }
    }
  }
}' -f owner="$OWNER" -F number="$PROJECT_NUMBER"
```

If the project belongs to a user rather than an org, substitute `user(login: $owner)` for `organization(login: $owner)`.

Compute board summary (same shape as Jira boards):
- Group items by their Status column value (e.g., "Todo", "In Progress", "Done")
- Items that changed status since `LAST_RUN_DATE` (compare `updatedAt`)
- Blocked or stale items (status is active but `updatedAt` > 7 days ago)
- New items added since last run

The `statusField` config value (default: `"Status"`) determines which project field to read for column grouping. Override it if the project uses a custom field name.

#### 3.4. Fetch Jira Board Data

For each board in `sources.jiraBoards`:

Use `jira_get_board_issues` MCP tool with the configured boardId and JQL.

Compute board summary:
- Count by status category (To Do, In Progress, In Review, Done since last run)
- Items that changed status since `LAST_RUN_DATE`
- Blocked or stale items (in progress but no update in 7+ days)
- New items added since last run

#### 3.5. Fetch Jira Feature Data

For each issue in `sources.jiraIssues`:

Use `jira_get_issue` MCP tool to get:
- Current status, priority, assignee
- Comments since `LAST_RUN_DATE`
- Any status transitions since last run

#### 3.6. Gather Notes (Optional)

Check for `$STATE_DIR/notes.md` — a freeform file where the user can add manual context that should be included in the status update. If present, include its contents.

### 4. Synthesize Status Update

Produce a concise, structured status comment:

```markdown
## Status Update — {Feature Name} ({JIRA_FEATURE})
**Date:** {YYYY-MM-DD}
**Period:** Since {last_run_date}

### Progress
- {What moved forward: merged PRs, closed board items, key decisions from Jira comments}

### Active Work
- {Open PRs with status: who is working on what, review state}
- {In-progress board items: grouped by theme if possible}

### Risks / Blockers
- {Items stale >7 days, PRs with unresolved change requests, unassigned critical items}

### Upcoming
- {New/backlog items ready to start, items in review about to merge}

### Metrics
- PRs: {opened} opened, {merged} merged, {closed} closed since last update
- Board: {in_progress} in progress, {in_review} in review, {done_since_last} completed
```

Rules for synthesis:
- **Be topical**: Only include changes since last run. Don't repeat stable state.
- **Be concise**: Each bullet should be one line. Avoid restating Jira descriptions.
- **Name people**: Include assignee names so readers know who to follow up with.
- **Link issues**: Reference Jira keys and PR numbers inline.
- **Highlight decisions**: If Jira comments contain decisions or clarifications, call them out.
- **Flag risks**: Any item that hasn't moved in 7+ days while in-progress is a risk.

### 5. Output or Post

#### Dry-Run Mode (default)

```bash
echo "$STATUS_UPDATE"
echo ""
echo "---"
echo "To post this as a comment on $JIRA_FEATURE:"
echo "  /feature-status $AREA_NAME --post"
```

Also save to `$STATUS_ROOT/output/$AREA_NAME-$DATE-draft.md` for reference.

#### Post Mode (--post)

Use `jira_add_comment` MCP tool to post the status update as a comment on the Jira feature issue.

```bash
jira_add_comment "$JIRA_FEATURE" "$STATUS_UPDATE"

echo "Posted status to $JIRA_FEATURE"
```

### 6. Update State

```bash
CURRENT_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

cat > "$LAST_RUN_FILE" <<JSON
{
  "lastRun": "$CURRENT_TIME",
  "lastRunArea": "$AREA_NAME",
  "lastRunMode": "$(if $DRY_RUN; then echo 'dry-run'; else echo 'posted'; fi)",
  "lastRunFeature": "$JIRA_FEATURE",
  "prsSeen": { "open": $OPEN_COUNT, "merged": $MERGED_COUNT },
  "boardSeen": { "total": $BOARD_TOTAL, "inProgress": $IN_PROGRESS_COUNT }
}
JSON
```

## Error Handling

- **Jira MCP unavailable**: Fall back to cached data, warn user
- **GitHub rate limit**: Show warning, use cached PR data
- **No changes since last run**: Output "No significant changes since {date}" instead of empty update
- **Missing config**: Show helpful error with available feature names

## Usage Examples

```bash
# Preview status for a feature
/feature-status payments-v2

# Post the update to Jira
/feature-status payments-v2 --post

# Preview all features
/feature-status --all

# Post all
/feature-status --all --post
```
