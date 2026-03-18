# Agents

This directory contains custom GitHub Copilot agent mode definitions (`.agent.md` files). Each agent represents a specialized role in the software development lifecycle.

## Available Agents

| Agent | Role | When to Use |
|-------|------|-------------|
| `requirements-analyst` | Elicits and validates requirements | Start of every project or feature |
| `solution-architect` | Designs system architecture | After requirements are finalized |
| `senior-developer` | Implements features to standards | During implementation stage |
| `security-engineer` | Reviews for vulnerabilities and enforces security controls | Before implementation; ongoing |
| `code-reviewer` | Reviews PRs for quality, security, and standards | Before merging any code |
| `devops-engineer` | Handles CI/CD, infrastructure, and deployment | Deployment stage |

## How to Use Agents

In VS Code with GitHub Copilot Chat, switch to an agent by selecting **"Use Agent"** and choosing the agent name, or by prefixing your prompt with the agent's name.

Each agent file follows the `.agent.md` format with:
- `name`: Agent identifier
- `description`: When Copilot should suggest this agent
- `model`: Preferred model (e.g., `gpt-4o`, `claude-sonnet`)
- `tools`: List of enabled MCP or built-in tools
- `instructions`: Detailed behavioral instructions
