# Filesystem MCP Server

## Purpose

Provides Copilot agents with scoped read and write access to the local filesystem. This allows agents to create, edit, move, and delete files within explicitly allowed directories without requiring a terminal.

## Tools Exposed

| Tool | Description |
|------|-------------|
| `read_file` | Read the contents of a file |
| `write_file` | Write or overwrite a file |
| `list_directory` | List the contents of a directory |
| `create_directory` | Create a new directory |
| `move_file` | Move or rename a file |
| `delete_file` | Delete a file |
| `search_files` | Search for files by name pattern within allowed dirs |

## Setup

### Prerequisites
- The `@modelcontextprotocol/server-filesystem` package (Node.js).

### Environment Variables

None required; allowed directories are configured in `mcp.json`.

### mcp.json entry

```json
{
  "filesystem": {
    "command": "npx",
    "args": [
      "-y",
      "@modelcontextprotocol/server-filesystem",
      "/path/to/allowed/dir1",
      "/path/to/allowed/dir2"
    ]
  }
}
```

Replace the paths with the directories agents are permitted to access. Do **not** pass `/` or other root paths.

## Persona Agents That Use This Server

- `senior-developer` — reads and writes source files
- `devops-engineer` — reads and writes CI/CD and infrastructure files
- `code-reviewer` — reads source files for review

## Security Notes

- Only include directories that agents legitimately need to access.
- Never include directories containing secrets, credentials, or sensitive data.
- Prefer the narrowest possible scope (e.g., the project root, not the entire home directory).
