# Pre-Commit Risk Scoring Plugin

A Claude Code plugin that assesses pre-commit risk by correlating PagerDuty incident history with current code changes.

The `/risk-score` command gathers PagerDuty incident data for your service, analyzes your current git diff, and looks for correlations between the areas you are changing and areas that have historically caused incidents. It surfaces active incidents, recent incident patterns, structural risk signals in the diff, and actionable recommendations.

## Installation

### 1. Add the marketplace

```
/plugin marketplace add pagerduty/claude-code-plugins
```

### 2. Install the plugin

```
/plugin install precommit-risk-scoring@pagerduty-claude-code-plugins
```

Or browse available plugins interactively with `/plugin` and go to the **Discover** tab.

### 3. Configure API key

The plugin bundles a PagerDuty MCP server that needs an API token. Git commit history is read directly from the local repository.

**Option A: Per-repo via `.claude/settings.local.json`** (recommended)

This file is gitignored by default, so secrets stay local:

```json
{
  "env": {
    "PAGERDUTY_API_KEY": "your-pagerduty-token"
  }
}
```

If the file already exists, merge the `env` key into it.

**Option B: Shell profile**

```bash
# In ~/.zshrc or ~/.bashrc
export PAGERDUTY_API_KEY=your-pagerduty-token
```

### Where to get the token

PagerDuty > User Profile > User Settings > API Access > Create API User Token.

Restart Claude Code after setting the key so the MCP server picks it up.

## Usage

With uncommitted changes in your working tree:

```
/risk-score
```

On first run, the plugin resolves your repository to a PagerDuty service through a fallback chain:

1. Cached config in `.claude/risk-config.json`
2. Backstage `catalog-info.yaml` annotation (`pagerduty.com/service-id`)
3. Auto-detection by matching the repository name against PagerDuty services
4. Manual input via interactive prompt

The resolved mapping is saved to `.claude/risk-config.json` for subsequent runs.

You can pass a service name hint as an argument:

```
/risk-score my-service-name
```

## Output

The command produces a structured risk assessment containing:

- **Active incidents** for the mapped service (highest-priority signal)
- **Incident history** summary over the last 90 days
- **Change analysis** with file-level and structural risk signals
- **Correlation findings** between current changes and past incidents
- **Risk score** from 0 (no risk) to 5 (critical), based on incident correlation and change analysis
- **Recommendations** based on identified risk factors

## Changing the mapped service

Delete `.claude/risk-config.json` and re-run `/risk-score` to pick a different service.

## Plugin structure

```
precommit-risk-scoring/
  .claude-plugin/
    plugin.json          # Plugin metadata
  commands/
    risk-score.md        # Slash command definition
  .mcp.json              # PagerDuty MCP server declaration
  README.md              # This file
```

## MCP servers

| Server | Declared in | Tools used |
|--------|-------------|------------|
| PagerDuty | `.mcp.json` (plugin-local) | `list_services`, `list_incidents`, `list_incident_notes`, `list_service_change_events` |
