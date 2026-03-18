# Agents

This directory is organized into two subfolders:

- **`personas/`** — Custom GitHub Copilot agent mode definitions (`.agent.md`). Each persona represents a specialized SDLC role with its own model, tool list, and behavioral instructions.
- **`mcp/`** — MCP (Model Context Protocol) server documentation and configuration. Each `.mcp.md` file describes an external tool server that persona agents can call.

---

## Personas (`personas/`)

| Agent | Role | When to Use |
|-------|------|-------------|
| `requirements-analyst` | Elicits and validates requirements | Start of every project or feature |
| `solution-architect` | Designs system architecture | After requirements are finalized |
| `senior-developer` | Implements features to standards | During implementation stage |
| `security-engineer` | Reviews for vulnerabilities and enforces security controls | Before implementation; ongoing |
| `code-reviewer` | Reviews PRs for quality, security, and standards | Before merging any code |
| `devops-engineer` | Handles CI/CD, infrastructure, and deployment | Deployment stage |

In VS Code with GitHub Copilot Chat, switch to a persona by selecting **"Use Agent"** and choosing the agent name, or by prefixing your prompt with the agent's name.

Each `.agent.md` file includes:
- `name`: Agent identifier
- `description`: When Copilot should suggest this agent
- `model`: Preferred model (e.g., `claude-sonnet`)
- `tools`: List of enabled MCP or built-in tools
- `instructions`: Detailed behavioral instructions

---

## MCP Servers (`mcp/`)

| File | Server | Purpose |
|------|--------|---------|
| `github.mcp.md` | `github` | GitHub API — repos, issues, PRs, branches |
| `filesystem.mcp.md` | `filesystem` | Scoped read/write access to local files |
| `database.mcp.md` | `database` | SQL query execution and schema inspection |

MCP servers are registered in `.vscode/mcp.json`. Copy `mcp/mcp.json.template` to `.vscode/mcp.json` to get started.

See `mcp/README.md` for full setup instructions and guidance on adding new servers.
