# claude-plugins

Claude Code plugins by pdbdnt.

## Plugins

### flow-gc-visualizer
Group-chat style architecture/flow visualization where system components talk to each other like friends in a messaging app.

Turn your system architecture into an animated group chat — each service is an "actor" with a colored avatar, and they exchange messages explaining what they do in plain language. Great for "How It Works" pages or onboarding flows.

### codex-broker
Connects to the Codex broker MCP server for cross-AI plan handoffs. Requires a local Codex broker server running — see `.mcp.json` template and set your own port.

### git-cp
Git commit and push workflow that aggregates all uncommitted changes, reads plan files from the project root `plans/` directory to build detailed commit messages, then commits and pushes. Invoke with `/git-cp`.

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add pdbdnt/claude-plugins
```

Then install individual plugins:

```
/plugin install flow-gc-visualizer
/plugin install codex-broker
/plugin install git-cp
```

## codex-broker Setup

The `codex-broker` plugin requires a local MCP server. Edit `plugins/codex-broker/.mcp.json` and replace `${PORT}` with your actual port number (e.g., `8765`).
