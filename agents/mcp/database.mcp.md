# Database MCP Server

## Purpose

Provides Copilot agents with the ability to inspect database schemas and execute read-only SQL queries. Useful during architecture, implementation, and debugging stages.

## Tools Exposed

| Tool | Description |
|------|-------------|
| `list_tables` | List all tables in the connected database |
| `describe_table` | Return column names, types, and constraints for a table |
| `run_query` | Execute a SQL query and return results (read-only by default) |
| `list_schemas` | List available schemas/databases |

## Setup

### Prerequisites
- A running database instance accessible from the local machine.
- The appropriate MCP database server package for your engine:
  - PostgreSQL: `@modelcontextprotocol/server-postgres`
  - SQLite: `@modelcontextprotocol/server-sqlite`

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DB_CONNECTION_STRING` | Full connection string for the target database |

### mcp.json entry (PostgreSQL example)

```json
{
  "database": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-postgres"],
    "env": {
      "POSTGRES_CONNECTION_STRING": "${env:DB_CONNECTION_STRING}"
    }
  }
}
```

### mcp.json entry (SQLite example)

```json
{
  "database": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sqlite", "--db-path", "/path/to/database.db"]
  }
}
```

## Persona Agents That Use This Server

- `solution-architect` — inspects schema during design
- `senior-developer` — reads schema when generating queries or migrations
- `security-engineer` — inspects schema for sensitive data exposure risks

## Security Notes

- Use a **read-only** database user for MCP access. Never connect with an admin or write-capable account.
- Store `DB_CONNECTION_STRING` in environment variables or a secrets manager — never commit it to source control.
- Avoid connecting to production databases. Use a staging or read replica instead.
- Ensure the connection string does not appear in logs or error messages.
