# Research: SQLlike MCP Server

**Date**: 2026-02-01  
**Feature**: 001-postgres-mcp-server

> **Note**: Project rebranded to "SQLlike" to reflect future multi-database support.

## Technology Decisions

### 1. MCP Framework: FastMCP

**Decision**: Use FastMCP 2.x (stable) for MCP server implementation

**Rationale**:
- FastMCP is the de-facto standard Python MCP framework with 22.4k GitHub stars
- Version 2.14.4 is the latest stable release (Jan 22, 2026)
- Provides native stdin/stdout transport support (MCP protocol requirement)
- Clean decorator-based API for defining tools
- Active maintenance by Prefect team

**Alternatives Considered**:
- FastMCP 3.0 (beta): Too new, still in beta - not production ready
- Raw MCP implementation: Unnecessary complexity when FastMCP handles protocol details

**Version**: `fastmcp>=2.14.0,<3.0`

---

### 2. SQL Validation: SQLGlot

**Decision**: Use SQLGlot for SQL parsing, validation, and dialect support

**Rationale**:
- No external dependencies (pure Python)
- Supports 31 SQL dialects including PostgreSQL
- Production-stable (v28.7.0, Jan 30, 2026)
- Can validate SQL syntax before execution
- Enables future multi-database support through dialect transpilation
- 8.8k GitHub stars, actively maintained

**Alternatives Considered**:
- sqlparse: Parsing only, no validation or dialect awareness
- Direct PostgreSQL EXPLAIN: Requires database connection, slower feedback loop

**Version**: `sqlglot>=28.0.0`

---

### 3. PostgreSQL Driver: asyncpg

**Decision**: Use asyncpg for async PostgreSQL connectivity

**Rationale**:
- Purpose-built for asyncio and PostgreSQL
- Binary protocol implementation (faster than text protocol)
- Built-in connection pooling
- Automatic type conversion for PostgreSQL types
- Supports PostgreSQL 9.5-18
- Well-suited for MCP's async nature

**Alternatives Considered**:
- psycopg3: More versatile (sync+async), but we only need async; asyncpg is more performant for pure async
- psycopg2: Sync-only, not suitable for async MCP server

**Version**: `asyncpg>=0.29.0`

---

### 4. AI Provider: Abstracted Multi-Provider Support

**Decision**: Create abstract AI provider interface supporting Anthropic, Bedrock, and OpenAI

**Rationale**:
- FR-009 requires support for multiple AI providers
- Abstract interface allows easy addition of future providers
- Each provider has different strengths (cost, latency, availability)
- Organizations may have existing contracts with specific providers

**Supported Providers**:

1. **Anthropic (Direct API)**
   - SDK: `anthropic>=0.40.0`
   - Config: `ANTHROPIC_API_KEY`
   - Model: claude-3-5-sonnet (default)

2. **AWS Bedrock**
   - SDK: `anthropic>=0.40.0` (includes Bedrock support)
   - Config: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
   - Model: anthropic.claude-3-5-sonnet (default)

3. **OpenAI**
   - SDK: `openai>=1.50.0`
   - Config: `OPENAI_API_KEY`
   - Model: gpt-4o (default)

**Configuration Strategy**:
- Environment variable `SQLLIKE_AI_PROVIDER` selects provider: `anthropic`, `bedrock`, or `openai`
- Auto-detect mode if not specified (checks for available credentials in order: Anthropic → OpenAI → Bedrock)
- Provider-specific model override via `SQLLIKE_AI_MODEL`

**Version**: `anthropic>=0.40.0`, `openai>=1.50.0`

---

### 5. Database Abstraction Layer

**Decision**: Create abstract `DatabaseAdapter` protocol with PostgreSQL implementation

**Rationale**:
- FR-010 requires architecture supporting future database types
- Protocol-based abstraction allows MySQL, SQLite implementations later
- SQLGlot's dialect support enables SQL translation between databases
- Clean separation of concerns

**Structure**:
```
src/
├── adapters/
│   ├── base.py          # DatabaseAdapter protocol
│   └── postgres.py      # PostgreSQL implementation (asyncpg)
```

---

### 6. Logging: Python stdlib logging

**Decision**: Use Python's built-in logging module with structured output

**Rationale**:
- FR-013 requires query logging without result data
- No additional dependencies needed
- MCP stdout is reserved for protocol; logs go to stderr
- Configurable levels (DEBUG, INFO, WARNING, ERROR)

**Format**: JSON structured logging for machine parsing

---

### 7. Configuration: Environment Variables + Pydantic

**Decision**: Use Pydantic Settings for configuration management

**Rationale**:
- FR-004 requires environment variable configuration
- Pydantic provides validation, type coercion, and documentation
- Supports `.env` files for local development
- Already a transitive dependency of FastMCP

**Configuration Schema**:
```
SQLLIKE_DATABASES=db1:postgresql://...,db2:postgresql://...
SQLLIKE_DEFAULT_DATABASE=db1
SQLLIKE_ROW_LIMIT=1000
SQLLIKE_AI_PROVIDER=anthropic  (or openai, bedrock)
ANTHROPIC_API_KEY=sk-...  (or OPENAI_API_KEY, or AWS_* for Bedrock)
```

---

## Best Practices Applied

### MCP Tool Design
- Each tool returns structured data (not just strings)
- Schema tool should be called before query tool (documented in help)
- Error responses include actionable suggestions

### Security
- Read-only queries only (SELECT)
- SQL validation before execution prevents injection via malformed SQL
- No result data in logs (credential/PII protection)
- Connection strings validated at startup

### Performance
- Connection pooling via asyncpg
- Row limit prevents memory exhaustion
- Async throughout for concurrent requests

---

## Unresolved Items

None - all technical decisions made based on spec requirements and research.
