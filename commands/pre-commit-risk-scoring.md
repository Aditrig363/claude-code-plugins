---
description: "Assess pre-commit risk by correlating PagerDuty incidents with current code changes"
argument-hint: "[service-name-override]"
allowed-tools: ["Read", "Glob", "Grep", "Bash", "Write", "AskUserQuestion", "mcp__pagerduty__list_services", "mcp__pagerduty__list_incidents", "mcp__pagerduty__list_incident_notes", "mcp__pagerduty__list_service_change_events"]
---

You are performing a pre-commit risk assessment. Follow these steps precisely and in order. Do not skip steps. Do not silently degrade -- if a required tool is unavailable, stop and tell the user.

## Step 0: Pre-flight Checks

Before doing anything else, verify that the required MCP tools are available.

### 0a: Verify PagerDuty MCP

Call `mcp__pagerduty__list_services` with query `"test"`. This is a connectivity check.

If the call fails or the tool is not available, STOP and tell the user:

```
PagerDuty MCP server is not available. This plugin requires it.

To fix:
1. Set the PAGERDUTY_API_KEY environment variable with a valid PagerDuty API token
2. Restart Claude Code so the plugin's MCP server configuration is loaded
3. Re-run /pre-commit-risk-scoring
```

Do NOT proceed without PagerDuty MCP. Do NOT fall back to a degraded assessment. The entire point of this plugin is PagerDuty incident correlation.

### 0b: Verify changes exist

Run `git diff --stat` and `git diff --cached --stat` to check for uncommitted changes (unstaged and staged).

If both are empty, run `git log -1 --oneline` and tell the user:

```
No uncommitted changes detected. The most recent commit is:
<commit hash and message>

/pre-commit-risk-scoring analyzes uncommitted changes. Make some changes first, or if you want to assess the last commit, let me know.
```

Then STOP. Do not proceed to analyze committed history on your own.

## Step 1: Resolve Service Mapping

Determine the PagerDuty service ID for this repository.

### 1a: Check cached configuration

Read `.claude/risk-config.json`. If it exists and contains `pagerduty.serviceId`, use that value and skip to Step 2. Display the cached service name to the user.

### 1b: Check Backstage catalog

If no cached config, check for `catalog-info.yaml` in the repository root. Look for the `pagerduty.com/service-id` annotation under `metadata.annotations`. If found, use that service ID.

Validate it by calling `mcp__pagerduty__list_services` with a query matching the service ID. Extract the service name from the response.

### 1c: Auto-detect from repository name

If neither config nor catalog exists:

1. Get the repository name from the current directory basename, or parse it from `git remote -v` output.
2. Call `mcp__pagerduty__list_services` with a query matching the repository name.
3. If exactly one service matches: use it. Tell the user which service was detected and ask them to confirm.
4. If multiple services match: present the options to the user via `AskUserQuestion` and let them pick.
5. If no services match and `$ARGUMENTS` is provided: call `mcp__pagerduty__list_services` with `$ARGUMENTS` as the query. If that matches, use it.
6. If still no match: use `AskUserQuestion` to ask the user for the PagerDuty service name or ID. Search for it with `mcp__pagerduty__list_services` to validate and get the canonical service ID.

### 1d: Persist configuration

Once a service ID is resolved, write `.claude/risk-config.json`:

```json
{
  "version": "1.0",
  "pagerduty": {
    "serviceId": "<resolved-service-id>",
    "serviceName": "<resolved-service-name>"
  }
}
```

Create the `.claude/` directory first if it does not exist.

## Step 2: Check Ongoing Incidents

Call `mcp__pagerduty__list_incidents` with:

- `statuses`: `["triggered", "acknowledged"]`
- `service_ids`: `["<serviceId>"]`

For each active incident returned, call `mcp__pagerduty__list_incident_notes` to get responder context.

If there are active incidents, report them prominently -- these are the highest-priority risk signal. Use this format:

```
ACTIVE INCIDENTS

[TRIGGERED] INC-12345: Database connection pool exhaustion (P1)
  Triggered 2h ago. Notes: "Scaling up read replicas, ETA 30min"

[ACKNOWLEDGED] INC-12346: Elevated error rate on /api/checkout (P2)
  Acknowledged 45min ago. Notes: "Investigating correlation with deploy at 14:30"
```

If there are no active incidents, note that briefly and continue.

## Step 3: Fetch Recent Incident History

Call `mcp__pagerduty__list_incidents` for the service over the last 90 days (use `since` parameter set to 90 days ago from today, and `until` set to today). Include all statuses.

For any P1 or P2 severity incidents (urgency "high"), call `mcp__pagerduty__list_incident_notes` to get root cause and remediation context.

Call `mcp__pagerduty__list_service_change_events` for the service to get recent change events that PagerDuty has tracked.

From the collected data, summarize:
- Total incident count over 90 days
- Severity distribution (how many high-urgency vs low-urgency)
- Recency of the most recent resolved incident
- Common patterns in incident titles or notes (repeated keywords, affected components)
- Change events and their timing relative to incidents

## Step 4: Analyze Current Changes and Correlate

### 4a: Gather current changes

Run the following commands to understand the current diff:
- `git diff --stat` for a summary of changes (unstaged)
- `git diff --cached --stat` for staged changes
- `git diff --name-only` and `git diff --cached --name-only` for the list of changed files
- `git diff` and `git diff --cached` for the full diff content (if the diff is very large, focus on `--stat` and file names)

### 4b: Gather recent commit history

Run `git log --format="%h %s (%an, %ar)" -20` to get the last 20 commits. This provides context on what has been changing recently.

### 4c: Correlate changes with incidents

Analyze the data gathered in Steps 2-4b and look for correlations:

1. **File/directory overlap**: Compare the files and directories in the current diff with areas mentioned in incident notes or titles. Are we touching components that have been involved in recent incidents?

2. **Change event correlation**: Compare PagerDuty change events with the current diff paths. Have previous changes to these same areas preceded incidents?

3. **Structural risk patterns**: Identify high-risk file types in the current diff:
   - Authentication/authorization files (auth, login, session, token, permission)
   - Database migrations (migrate, schema, alembic, flyway)
   - Configuration files (config, settings, env, infrastructure)
   - Dependency files (requirements.txt, package.json, go.mod, Gemfile, build.gradle)
   - API contracts (openapi, swagger, proto, graphql schema)
   - Infrastructure (terraform, cloudformation, kubernetes, helm, dockerfile)

4. **Change magnitude**: Assess the size and spread of changes -- number of files, lines changed, number of distinct directories affected.

5. **Pattern similarity**: Does the current change resemble (in nature, scope, or affected area) changes that preceded past incidents?

## Step 5: Assign Risk Score

Based on all the data gathered, assign a risk score from 0 to 5:

- **0** -- No risk signals at all. Pure documentation, comments, or whitespace.
- **1** -- Trivial changes with no incident correlation. Test cleanup, minor refactors, no structural risk signals.
- **2** -- Some structural signals (config changes, dependency updates) or recent incidents exist, but no correlation with current changes.
- **3** -- Moderate risk. Touching areas related to recent incidents, or high-risk file types (auth, migrations, infra) with no active incidents.
- **4** -- High risk. Active incidents on the service AND changes correlate with incident-affected areas, OR large changes to critical paths with recent incident history.
- **5** -- Critical. Active P1/P2 incident AND current changes directly touch the code involved in the incident.

Build the score bar using filled and empty blocks. For score N, use N filled blocks (█) and (5-N) empty blocks (░):
- 0/5: `░░░░░`
- 1/5: `█░░░░`
- 2/5: `██░░░`
- 3/5: `███░░`
- 4/5: `████░`
- 5/5: `█████`

## Step 6: Present Risk Assessment

Output a compact, structured risk assessment. Be concise -- one line per finding, no filler. Use this format exactly:

```
PRE-COMMIT RISK ASSESSMENT
Service: <service-name> (<service-id>) | Changes: <N> files (+<additions>, -<deletions>)

RISK SCORE: <N>/5 [<LEVEL>] <score-bar>

Active incidents: <"None" or one-line summary per incident>
Incident history (90d): <count> incidents. <one-line summary of severity, recency, patterns>

CHANGE ANALYSIS
- <file-or-group> -- <what changed, one line>
- <file-or-group> -- <what changed, one line>
- Structural risk signals: <"None" or list>
- Incident correlation: <"None" or one-line description of correlation found>

RISK FACTORS
<numbered list, one line each. If none, say "No significant risk factors identified.">

RECOMMENDATION
<one to two sentences. Match the tone to the score level.>
```

Where `<LEVEL>` corresponds to the score:
- 0-1: `LOW`
- 2: `MODERATE`
- 3: `ELEVATED`
- 4: `HIGH`
- 5: `CRITICAL`

Guidelines:
- Be concise. Each bullet should be one line. Do not repeat information across sections.
- Do not pad findings. If incident history is sparse, say so briefly. If there are no correlations, say "None".
- Do not invent risk factors that are not supported by the data gathered.
- If there are active incidents that relate to the areas being changed, this is the most critical finding -- call it out and recommend coordinating with incident responders.
- For scores 0-2, the recommendation can be a single sentence.
- For scores 3+, include specific actionable guidance.