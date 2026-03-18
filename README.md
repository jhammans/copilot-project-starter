# Copilot Project Starter

A reusable GitHub Copilot instruction set for bootstrapping new projects with standardized agents, skills, tasks, and coding/security standards.

## Purpose

This repository provides a structured library of GitHub Copilot customization files that can be:

- **Referenced directly** by including this repo as a Git submodule or copying files into your project
- **Used as templates** to generate consistent, high-quality projects across your organization
- **Extended** by adding new skill definitions, agent configurations, and standards for additional languages or frameworks

## Directory Structure

```
copilot-project-starter/
├── .github/
│   └── copilot-instructions.md        # Root-level Copilot instructions (default for all interactions)
├── agents/
│   ├── personas/                      # SDLC role agents (.agent.md) — define model, tools, behavior
│   │   ├── requirements-analyst.agent.md
│   │   ├── solution-architect.agent.md
│   │   ├── senior-developer.agent.md
│   │   ├── security-engineer.agent.md
│   │   ├── code-reviewer.agent.md
│   │   └── devops-engineer.agent.md
│   └── mcp/                           # MCP server documentation and configuration (.mcp.md)
│       ├── github.mcp.md
│       ├── filesystem.mcp.md
│       ├── database.mcp.md
│       └── mcp.json.template          # Copy to .vscode/mcp.json to register servers
├── skills/                            # Reusable skill definitions (.instructions.md / SKILL.md)
│   ├── requirements-gathering/        # Requirements elicitation & SDLC gating
│   ├── security/                      # OWASP, IAM, RBAC, threat modeling
│   ├── languages/                     # Per-language coding standards
│   └── frameworks/                    # Per-framework conventions
├── tasks/                             # Prompt files for common SDLC tasks (.prompt.md)
└── standards/                         # Org-wide coding, docs, testing, and API standards
```

## Quick Start

### Option 1 — Git Submodule

```bash
git submodule add https://github.com/<org>/copilot-project-starter .copilot-starter
```

Then copy or symlink the files you need into your project's `.github/` and `.copilot/` directories.

### Option 2 — Copy into your project

```bash
cp -r copilot-project-starter/.github/copilot-instructions.md  your-project/.github/
cp -r copilot-project-starter/skills/security/                 your-project/.github/skills/security/
```

### Option 3 — Template Repository

Click **"Use this template"** on GitHub to create a new repo pre-populated with all files.

## How the Instruction Files Work

| File type | Location pattern | Purpose |
|-----------|-----------------|---------|
| `copilot-instructions.md` | `.github/copilot-instructions.md` | Always-active instructions for every Copilot interaction |
| `*.instructions.md` | `.github/instructions/*.instructions.md` | Scoped instructions applied via `applyTo` glob patterns |
| `*.agent.md` | `agents/personas/*.agent.md` | Named custom agent modes (personas) with specific model, tool & behavior configs |
| `*.mcp.md` | `agents/mcp/*.mcp.md` | MCP server documentation — setup, exposed tools, and security requirements |
| `mcp.json.template` | `agents/mcp/mcp.json.template` | Template for `.vscode/mcp.json`; registers MCP servers in VS Code |
| `*.prompt.md` | `.github/prompts/*.prompt.md` | Reusable prompt templates for common tasks |
| `SKILL.md` | Any directory | Skill index files used by parent agents |

## SDLC Workflow

This starter enforces a **gate-based SDLC**:

```
Requirements → Architecture → Security Review → Implementation → Code Review → Deployment
     ↑               ↑               ↑                ↑               ↑            ↑
  No code        No code         Must pass        Standards       Must pass     Checklist
  until done    until done      before impl.      enforced        before merge  required
```

Each gate is controlled by task prompts in `tasks/` and gated by the requirements-gathering skill.

## MCP Servers

MCP (Model Context Protocol) servers extend persona agents with access to external tools. Server documentation lives in `agents/mcp/`:

| Server | File | Purpose |
|--------|------|---------|
| `github` | `agents/mcp/github.mcp.md` | GitHub API — repos, issues, PRs, branches |
| `filesystem` | `agents/mcp/filesystem.mcp.md` | Scoped read/write access to local files |
| `database` | `agents/mcp/database.mcp.md` | SQL query execution and schema inspection |

To activate servers, copy `agents/mcp/mcp.json.template` to `.vscode/mcp.json` and set the required environment variables.

## Diagrams

All diagrams must be authored in **Mermaid** and embedded in Markdown using a fenced code block. Supported types:

| Type | Keyword | Use for |
|------|---------|---------|
| Flowchart | `graph TD` / `graph LR` | Architecture overviews, decision flows |
| Sequence | `sequenceDiagram` | API and auth flows |
| Entity-Relationship | `erDiagram` | Data models |
| C4 Context | `C4Context` | System context (C4 Level 1) |
| C4 Container | `C4Container` | Container diagrams (C4 Level 2) |

See `standards/documentation-standards.md` for the full diagrams standard.

## Security Model

All generated code and architecture recommendations follow:

- **OWASP Top 10** vulnerability prevention (see `skills/security/owasp-top10.instructions.md`)
- **IAM least-privilege** principle (see `skills/security/iam.instructions.md`)
- **RBAC patterns** (see `skills/security/rbac.instructions.md`)
- **Threat modeling** (see `skills/security/threat-modeling.instructions.md`)

## Contributing

1. Fork this repository
2. Add or update instruction files following the patterns in `standards/`
3. Open a pull request — the `code-reviewer` agent will guide review

## License

MIT
