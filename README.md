# PagerDuty Claude Code Plugins

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

PagerDuty's open source [Claude Code plugins](https://code.claude.com/docs/en/plugins) that bring operational intelligence into your development workflow.

## Available plugins

| Plugin                                                    | Description                                                                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [precommit-risk-scoring](plugins/precommit-risk-scoring/) | Pre-commit risk assessment using PagerDuty incident history, change event correlation, and git diff analysis  |

## Getting started

### Prerequisites

- [Claude Code](https://code.claude.com) v1.0.33 or later

### 1. Add the marketplace

Run this once in any Claude Code session:

```bash
/plugin marketplace add PagerDuty/claude-code-plugins
```

### 2. Browse and install plugins

Open the plugin manager and go to the **Discover** tab to browse available plugins:

```bash
/plugin
```

Or install a plugin directly:

```bash
/plugin install precommit-risk-scoring@pagerduty-claude-code-plugins
```

### 3. Configure

Each plugin may require additional configuration (API keys, environment variables, etc.). See the plugin's own README for setup details.

## Set up for your team

You can configure a repository so team members are automatically prompted to add this marketplace when they trust the project folder. Add the following to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "pagerduty-claude-code-plugins": {
      "source": {
        "source": "github",
        "repo": "PagerDuty/claude-code-plugins"
      }
    }
  }
}
```

To also auto-enable specific plugins for the project:

```json
{
  "extraKnownMarketplaces": {
    "pagerduty-claude-code-plugins": {
      "source": {
        "source": "github",
        "repo": "PagerDuty/claude-code-plugins"
      }
    }
  },
  "enabledPlugins": {
    "precommit-risk-scoring@pagerduty-claude-code-plugins": true
  }
}
```

## Managing plugins

Update the marketplace to get the latest plugins:

```bash
/plugin marketplace update pagerduty-claude-code-plugins
```

Disable a plugin without uninstalling:

```bash
/plugin disable precommit-risk-scoring@pagerduty-claude-code-plugins
```

Uninstall a plugin:

```bash
/plugin uninstall precommit-risk-scoring@pagerduty-claude-code-plugins
```

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
