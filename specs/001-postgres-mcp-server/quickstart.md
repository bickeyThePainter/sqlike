# Quickstart: SQLlike MCP Server

**Date**: 2026-02-01  
**Feature**: 001-postgres-mcp-server

> **Note**: Project rebranded to "SQLlike" to reflect future multi-database support.

## Prerequisites

- Python 3.11+
- uv package manager
- SQL database (PostgreSQL 9.5+ initially, MySQL/SQLite coming soon)
- AI provider credentials: Anthropic API key, OpenAI API key, OR AWS credentials (for Bedrock)

---

## Installation

```bash
# Clone and install
git clone <repository>
cd sqllike

# Install dependencies with uv
uv sync

# Or install as a tool
uv tool install .
```

---

## Configuration

### Environment Variables

Create a `.env` file or export environment variables:

```bash
# Database Configuration (required)
# Format: identifier:connection_string,identifier2:connection_string2
SQLLIKE_DATABASES="main:postgresql://user:pass@localhost:5432/mydb"

# Optional: Default database when not specified in query
SQLLIKE_DEFAULT_DATABASE="main"

# Optional: Row limit (default: 1000)
SQLLIKE_ROW_LIMIT="1000"

# Optional: Log level (default: INFO)
SQLLIKE_LOG_LEVEL="INFO"

# AI Provider Selection (optional - auto-detects based on available credentials)
# Options: anthropic, openai, bedrock
SQLLIKE_AI_PROVIDER="anthropic"

# AI Provider - Option 1: Direct Anthropic API
ANTHROPIC_API_KEY="sk-ant-..."

# AI Provider - Option 2: OpenAI
OPENAI_API_KEY="sk-..."

# AI Provider - Option 3: AWS Bedrock
AWS_ACCESS_KEY_ID="AKIA..."
AWS_SECRET_ACCESS_KEY="..."
AWS_REGION="us-east-1"

# Optional: Override default model
SQLLIKE_AI_MODEL="claude-3-5-sonnet"  # or "gpt-4o" for OpenAI
```

### Multiple Databases

```bash
SQLLIKE_DATABASES="prod:postgresql://prod-user:pass@prod-host:5432/app,analytics:postgresql://analytics:pass@analytics-host:5432/warehouse"
SQLLIKE_DEFAULT_DATABASE="prod"
```

---

## Running the Server

### Standalone (for testing)

```bash
# Run directly
uv run python -m sqllike

# Or if installed as tool
sqllike
```

### With Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "sqllike": {
      "command": "uv",
      "args": ["run", "python", "-m", "sqllike"],
      "cwd": "/path/to/sqllike",
      "env": {
        "SQLLIKE_DATABASES": "main:postgresql://user:pass@localhost:5432/mydb",
        "ANTHROPIC_API_KEY": "sk-ant-..."
      }
    }
  }
}
```

### With Other MCP Clients

The server uses stdin/stdout for MCP protocol communication. Any MCP-compatible client can connect by spawning the process and communicating via stdio.

---

## Usage Examples

### Workflow: Schema First, Then Query

**Recommended order:**
1. Call `help` to understand available tools and databases
2. Call `schema` to see database structure
3. Call `query` with natural language or SQL

### Example 1: Explore Database

```json
// Step 1: Get help
{"tool": "help", "input": {}}

// Step 2: Get schema
{"tool": "schema", "input": {"database": "main"}}

// Step 3: Query with natural language
{"tool": "query", "input": {"input": "show me all users created in the last 7 days"}}
```

### Example 2: Direct SQL Query

```json
{"tool": "query", "input": {
  "input": "SELECT id, name, email FROM users WHERE status = 'active' ORDER BY created_at DESC",
  "database": "main",
  "limit": 50
}}
```

### Example 3: Multi-Database

```json
// Query production database
{"tool": "query", "input": {"input": "count all orders", "database": "prod"}}

// Query analytics database
{"tool": "query", "input": {"input": "show revenue by month", "database": "analytics"}}
```

---

## Error Handling

### SQL-Only Mode (AI Unavailable)

If the AI translation service is unavailable, the server continues to accept raw SQL:

```json
// This works even without AI
{"tool": "query", "input": {"input": "SELECT * FROM users LIMIT 10"}}

// This returns an error explaining AI is unavailable
{"tool": "query", "input": {"input": "show me all users"}}
// Response: {"error": "ai_unavailable", "message": "Natural language queries require AI service. Please use SQL directly or try again later.", "suggestions": ["Use raw SQL instead", "Check AI provider configuration"]}
```

### Ambiguous Queries

When a natural language query could mean multiple things:

```json
{"tool": "query", "input": {"input": "show me the sales"}}
// Response:
{
  "type": "clarification_needed",
  "message": "Your query could mean several things. Please clarify:",
  "options": [
    {"id": "A", "description": "All sales records", "sql": "SELECT * FROM sales"},
    {"id": "B", "description": "Sales total amount", "sql": "SELECT SUM(amount) FROM sales"},
    {"id": "C", "description": "Sales by product", "sql": "SELECT product_id, COUNT(*) FROM sales GROUP BY product_id"}
  ]
}
```

---

## Troubleshooting

### Connection Issues

```bash
# Test database connection
uv run python -c "import asyncpg; import asyncio; asyncio.run(asyncpg.connect('postgresql://...'))"
```

### View Logs

```bash
# Set debug logging
export SQLLIKE_LOG_LEVEL=DEBUG
uv run python -m sqllike 2>debug.log
```

### Verify Configuration

```bash
# Print parsed configuration (without secrets)
uv run python -m sqllike --check-config
```

---

## Integration Testing

### Manual Test

```bash
# Start server and send test request
echo '{"jsonrpc": "2.0", "method": "tools/list", "id": 1}' | uv run python -m sqllike
```

### Automated Tests

```bash
uv run pytest tests/integration/ -v
```
