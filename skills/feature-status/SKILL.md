---
name: feature-status
description: Gather data from GitHub PRs, Jira kanban boards, and Jira features to produce a concise status update. Supports dry-run mode (default) to preview the update before posting it as a Jira comment. Use when user asks for feature status, status update, progress report, or wants to update a Jira feature with current status.
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

Gathers status from multiple sources (GitHub PRs, Jira kanban boards, Jira features) and produces a concise, topical status update suitable for posting as a comment on the top-level Jira feature.

**Arguments**: `$ARGUMENTS`

Supported forms:
- `/feature-status <area-name>` — dry-run (default), outputs status to console
- `/feature-status <area-name> --post` — posts the status as a Jira comment on the feature
- `/feature-status --all` — dry-run for all enabled areas
- `/feature-status --all --post` — posts status for all enabled areas

Where `<area-name>` matches a name in `config/features.json` (e.g., "installer", "authz").

## Implementation

### 1. Setup and Configuration

```bash
CONFIG_FILE="${CLAUDE_SKILL_DIR}/config/features.json"
STATE_ROOT="${CLAUDE_SKILL_DIR}/state"
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
  echo "Usage: /feature-status <area-name> [--post]"
  echo "       /feature-status --all [--post]"
  echo ""
  echo "Available areas:"
  jq -r '.features[] | select(.enabled == true) | "  \(.name) — \(.jiraFeature) (\(.summary))"' "$CONFIG_FILE"
  exit 0
fi
```

### 2. Load Feature Configuration

Each feature in `config/features.json` defines:

```json
{
  "features": [
    {
      "name": "installer",
      "summary": "Installation and Reference Architectures",
      "jiraFeature": "ANSTRAT-1899",
      "enabled": true,
      "sources": {
        "github": {
          "repos": [
            { "owner": "automation-nexus", "repo": "nexus", "branchPatterns": ["035-*", "*ANSTRAT-1899*", "feat/prototype-operator"] },
            { "owner": "automation-nexus", "repo": "automation-orchestrator-operator" }
          ],
          "prSearchTerms": ["ANSTRAT-1899", "installer", "operator"]
        },
        "jiraBoards": [
          { "boardId": 11707, "project": "AAP", "jql": "status != Done ORDER BY updated DESC" }
        ],
        "jiraIssues": ["ANSTRAT-1899"]
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
    echo "ERROR: Area '$AREA_NAME' not found in config"
    exit 1
  fi
  AREAS="$AREA_NAME"
fi
```

### 3. For Each Feature Area: Gather Data

#### 3.1. Load Previous State

```bash
STATE_DIR="$STATE_ROOT/state/$AREA_NAME"
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

#### 3.3. Fetch Jira Board Data

For each board in `sources.jiraBoards`:

Use `jira_get_board_issues` MCP tool with the configured boardId and JQL.

Compute board summary:
- Count by status category (To Do, In Progress, In Review, Done since last run)
- Items that changed status since `LAST_RUN_DATE`
- Blocked or stale items (in progress but no update in 7+ days)
- New items added since last run

#### 3.4. Fetch Jira Feature Data

For each issue in `sources.jiraIssues`:

Use `jira_get_issue` MCP tool to get:
- Current status, priority, assignee
- Comments since `LAST_RUN_DATE`
- Any status transitions since last run

#### 3.5. Gather Notes (Optional)

Check for `$STATE_DIR/notes.md` — a freeform file where the user can add manual context that should be included in the status update. If present, include its contents.

### 4. Fetch Current Feature Metadata from Jira

Before prompting the user, read the current state of the ANSTRAT feature issue so the user can see what's currently set and decide what to change.

Fetch these fields from the ANSTRAT feature (e.g. `ANSTRAT-1972`):
- **labels** — look for health label: `green`, `yellow`, or `red` (there will be at most one)
- **customfield_10022** — Target start date (YYYY-MM-DD)
- **customfield_10023** — Target end date (YYYY-MM-DD)

Present a brief summary to the user:
```
Current feature state for ANSTRAT-XXXX:
  Health: Green (label)  |  or "not set"
  Target start: 2026-01-07  |  or "not set"
  Target end: 2026-06-12  |  or "not set"

Data collected:
  GitHub: X open PRs, Y merged since last update
  Board: Z items in progress, W completed
```

### 5. Prompt for Human-Authored Status and Feature Metadata

After showing the collected data and current feature state, use AskUserQuestion to gather the following from the user. Ask all questions in a single AskUserQuestion call:

1. **Health status** (single-select): "What is the current health of this feature?"
   - Options: Green, Yellow, Red
   - Pre-select the current label if one exists

2. **Status narrative** (free-text via "Other"): "Provide your status summary for this feature. This is your human-authored narrative that will appear at the top of the report."

3. **Milestones** (free-text via "Other"): "List key milestones with dates (or say 'skip' to omit). Example: 'Design complete - 2026-06-15, Implementation done - 2026-07-30, QE signoff - 2026-08-15'"

4. **Target end date update** (free-text via "Other"): "Update the target end date? Enter a date (YYYY-MM-DD) or 'keep' to leave unchanged. Current: {current_target_end or 'not set'}"

The user's responses become:
- `HEALTH_LABEL` — green, yellow, or red
- `HUMAN_STATUS` — the narrative text
- `MILESTONES` — parsed into a list of milestone + date pairs
- `TARGET_END_UPDATE` — a date string or null

### 6. Synthesize Status Update

Produce a concise, structured status comment. The format:

```markdown
## Status Update — {Feature Name} ({JIRA_FEATURE})
**Date:** {YYYY-MM-DD}
**Health:** {HEALTH_LABEL — Green / Yellow / Red}
**Period:** Since {last_run_date}
**Target end:** {target_end_date}

### Summary
{HUMAN_STATUS — the user's own narrative, inserted verbatim}

### Milestones
| Milestone | Target Date | Status |
|-----------|-------------|--------|
| {milestone_1} | {date_1} | {On track / At risk / Complete} |
| {milestone_2} | {date_2} | {On track / At risk / Complete} |

### Risks / Blockers
- {Items stale >7 days, PRs with unresolved change requests, unassigned critical items}
- {User-called-out risks from their narrative}

### Progress
- {Bullet points summarizing what moved forward: merged PRs, closed board items, key decisions from Jira comments}

### Active Work
- {Open PRs with status: who is working on what, review state}
- {In-progress board items: grouped by theme if possible}

### Upcoming
- {New/backlog items that appear ready to start, items in review about to merge}

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
- **Flag risks early**: Any item that hasn't moved in 7+ days while in-progress is a risk. Surface risks prominently — do not wait for a sync.
- **Milestones table**: Omit the section if the user skipped milestones. If included, infer status from dates (past due = "At risk", future = "On track", done items = "Complete").

### 7. Output or Post

#### Dry-Run Mode (default)

```bash
echo "$STATUS_UPDATE"
echo ""
echo "---"
echo "To post this as a comment on $JIRA_FEATURE:"
echo "  /feature-status $AREA_NAME --post"
```

Also save to `$STATE_ROOT/output/$AREA_NAME-$DATE-draft.md` for reference.

#### Post Mode (--post)

Perform all three Jira updates:

**a) Update health label on the ANSTRAT feature**

Remove any existing health label (`green`, `yellow`, `red`) and add the new one using `editJiraIssue`:
- Read current labels, filter out any of `green`, `yellow`, `red`
- Add the user's selected `HEALTH_LABEL`
- cloudId: `redhat.atlassian.net`

**b) Update target end date if changed**

If the user provided a new target end date, update `customfield_10023` on the ANSTRAT feature using `editJiraIssue`.

**c) Post status comment**

Use `addCommentToJiraIssue` MCP tool to post the status update as a comment on the Jira feature issue (cloudId: `redhat.atlassian.net`).

```
echo "✓ Health label set to $HEALTH_LABEL on $JIRA_FEATURE"
echo "✓ Target end date updated to $TARGET_END on $JIRA_FEATURE"  # if changed
echo "✓ Status comment posted to $JIRA_FEATURE"
echo "  View: https://redhat.atlassian.net/browse/$JIRA_FEATURE"
```

### 8. Update State

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
- **Missing config**: Show helpful error with available area names

## Usage Examples

```bash
# Preview status for installer feature
/feature-status installer

# Post status to ANSTRAT-1899
/feature-status installer --post

# Preview all features
/feature-status --all

# Post all
/feature-status --all --post
```
