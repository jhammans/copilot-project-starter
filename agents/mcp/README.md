# MCP Servers

This directory documents the Model Context Protocol (MCP) servers available to Copilot agents in this project. Each `.mcp.md` file describes a server's purpose, configuration, and the tools it exposes.

## What is MCP?

MCP (Model Context Protocol) servers extend what agents can *do* by exposing external tools — such as querying GitHub, reading the filesystem, or running SQL — directly within Copilot Chat. Persona agents (in `../personas/`) declare which MCP tools they use via the `tools:` frontmatter field.

## Available Servers

| File | Server | Purpose |
|------|--------|---------|
| `github.mcp.md` | `github` | GitHub API — repos, issues, PRs, branches |
| `filesystem.mcp.md` | `filesystem` | Scoped read/write access to local files |
| `database.mcp.md` | `database` | SQL query execution and schema inspection |

## Configuration

MCP servers are registered in `.vscode/mcp.json` at the workspace root. Use `mcp.json.template` in this directory as a starting point — copy it to `.vscode/mcp.json` and fill in the required environment variables.

## Adding a New MCP Server

1. Create a `<name>.mcp.md` file in this directory following the existing format.
2. Add the server entry to `mcp.json.template`.
3. Register any tools the server exposes in the `tools:` list of the relevant persona agents.
4. Document required environment variables and setup steps in the `.mcp.md` file.
