# GitHub MCP Server

## Purpose

Provides Copilot agents with direct access to the GitHub API, enabling actions such as reading repositories, creating issues, managing pull requests, and inspecting branches — without leaving Copilot Chat.

## Tools Exposed

| Tool | Description |
|------|-------------|
| `github_repo` | Clone or browse a repository's file tree |
| `github_create_issue` | Open a new issue on a repository |
| `github_list_issues` | List open/closed issues with filters |
| `github_create_pull_request` | Open a pull request |
| `github_get_pull_request` | Read PR details, diff, and review comments |
| `github_merge_pull_request` | Merge an approved PR |
| `github_list_branches` | List branches for a repo |
| `github_create_branch` | Create a new branch |

## Setup

### Prerequisites
- A GitHub Personal Access Token (PAT) or GitHub App installation token with the required scopes.
- The `@modelcontextprotocol/server-github` package (Node.js).

### Required Scopes (PAT)
- `repo` — full repository access
- `read:org` — read org membership (if using org repos)

### Environment Variables

| Variable | Description |
|----------|-------------|
| `GITHUB_TOKEN` | GitHub PAT or App installation token |
| `GITHUB_HOST` | Override for GitHub Enterprise (optional, e.g. `https://github.example.com`) |

### mcp.json entry

```json
{
  "github": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
    }
  }
}
```

## Persona Agents That Use This Server

- `requirements-analyst` — reads existing issues and specs
- `solution-architect` — reads repo structure and ADRs
- `senior-developer` — reads/writes branches and files
- `code-reviewer` — reads PRs and diff content
- `devops-engineer` — manages branches and CI configuration

## Security Notes

- Store `GITHUB_TOKEN` in your environment or a secrets manager — never hard-code it.
- Use fine-grained PATs scoped to specific repositories where possible.
- Rotate tokens on a schedule consistent with your org's security policy.
