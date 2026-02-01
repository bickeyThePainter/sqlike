# Feature Specification: SQLlike MCP Server

**Feature Branch**: `001-postgres-mcp-server`  
**Created**: 2026-02-01  
**Status**: Draft  
**Input**: User description: "Build an MCP server for SQL databases that provides tools for agents to query against"

> **Note**: Project rebranded from "PostgreSQL MCP Server" to "SQLlike" to reflect future multi-database support (MySQL, SQLite, etc.)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Query Database with Natural Language (Priority: P1)

As an AI agent, I want to query a PostgreSQL database using natural language so that I can retrieve data without writing raw SQL.

**Why this priority**: This is the core value proposition - enabling agents to interact with databases naturally. Without this, the MCP server has no primary purpose.

**Independent Test**: Can be fully tested by sending a natural language query like "show me all users created this month" and receiving structured data results. Delivers immediate value for agent-database interactions.

**Acceptance Scenarios**:

1. **Given** a connected PostgreSQL database with a users table, **When** the agent sends "show me all active users", **Then** the system returns a structured result set containing active user records.
2. **Given** a connected database, **When** the agent sends a valid SQL query directly, **Then** the system executes the SQL and returns the results.
3. **Given** a connected database, **When** the agent sends an invalid or malformed query, **Then** the system returns a clear error message explaining why the query failed.
4. **Given** a natural language query that cannot be translated to valid SQL, **When** processed, **Then** the system returns an error with suggestions for clarification.

---

### User Story 2 - Explore Database Schema (Priority: P1)

As an AI agent, I want to retrieve the schema of a database so that I can understand its structure before querying.

**Why this priority**: Understanding schema is essential for generating accurate queries. Agents need this context to formulate meaningful natural language requests.

**Independent Test**: Can be tested by requesting schema for a database and receiving a clear, readable representation of tables, columns, relationships, and descriptions.

**Acceptance Scenarios**:

1. **Given** a connected PostgreSQL database, **When** the agent requests the schema, **Then** the system returns a human-readable representation of all tables and their columns.
2. **Given** a database with multiple tables, **When** schema is requested, **Then** the response includes table names, column names, data types, and basic descriptions.
3. **Given** a database with foreign key relationships, **When** schema is requested, **Then** the relationships between tables are clearly indicated.

---

### User Story 3 - Get Help and Usage Information (Priority: P2)

As an AI agent, I want to access help documentation so that I understand how to use the MCP server tools effectively.

**Why this priority**: Helps agents self-discover capabilities and use the tools in the correct order, improving autonomous operation.

**Independent Test**: Can be tested by requesting help and receiving clear documentation on available tools and recommended usage patterns.

**Acceptance Scenarios**:

1. **Given** the MCP server is running, **When** the agent requests help, **Then** the system returns documentation describing available tools.
2. **Given** help is requested, **When** documentation is returned, **Then** it includes the recommended order of tool calls (e.g., schema first, then query).
3. **Given** help is requested, **When** documentation is returned, **Then** it includes examples of how to use each tool.

---

### User Story 4 - Connect to Multiple Databases (Priority: P2)

As a system administrator, I want to configure multiple database connections so that agents can query different databases as needed.

**Why this priority**: Multi-database support expands the utility of the server for organizations with multiple data sources.

**Independent Test**: Can be tested by configuring two database connections and successfully querying each one independently.

**Acceptance Scenarios**:

1. **Given** multiple databases are configured, **When** an agent specifies a database identifier, **Then** the query executes against the correct database.
2. **Given** database connections are defined via configuration, **When** the server starts, **Then** all configured databases are available for querying.
3. **Given** an agent requests schema without specifying a database, **When** multiple databases exist, **Then** the system prompts for database selection or returns schemas for all.

---

### Edge Cases

- What happens when the database connection is lost mid-query? → Return clear error with connection status
- How does the system handle queries that would return extremely large result sets? → **Resolved**: Enforce row limit with truncation indicator
- What happens when a natural language query is ambiguous and could map to multiple SQL interpretations? → **Resolved**: Ask for clarification with options
- How does the system handle database timeout scenarios? → Return timeout error with suggested actions
- What happens when configured database credentials are invalid? → Return authentication error at startup or first connection attempt
- How does the system handle SQL injection attempts in natural language queries? → Read-only mode mitigates risk; generated SQL is validated before execution
- What happens if the AI generates a non-SELECT query (e.g., DELETE, UPDATE)? → **Resolved**: All AI-generated SQL is validated; non-SELECT statements are rejected with error before execution

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a `query` tool that accepts natural language or SQL input and returns query results. Queries are restricted to read-only operations (SELECT only); write operations (INSERT, UPDATE, DELETE) are rejected. This restriction applies to ALL SQL—including AI-generated SQL—which MUST be validated before execution.
- **FR-002**: System MUST provide a `schema` tool that returns a readable representation of database structure including tables, columns, types, and relationships.
- **FR-003**: System MUST provide a `help` tool that returns usage documentation including available tools and recommended call order.
- **FR-004**: System MUST support multiple database connections configurable via environment variables.
- **FR-005**: System MUST validate that ALL SQL (user-provided AND AI-generated) is syntactically correct AND is a SELECT statement before execution. Non-SELECT statements MUST be rejected with a clear error message.
- **FR-006**: System MUST return clear, actionable error messages when queries fail.
- **FR-007**: System MUST translate natural language queries to valid SQL using AI-powered translation.
- **FR-008**: System MUST communicate via stdin/stdout for MCP protocol compatibility.
- **FR-009**: System MUST support multiple AI providers through an abstracted provider interface: Anthropic (direct API), AWS Bedrock, and OpenAI. Provider selection is configurable via environment variables.
- **FR-010**: System MUST be designed with abstraction to support additional SQL database types in the future.
- **FR-011**: System MUST enforce a configurable row limit on query results (default: 1000 rows) and include a truncation indicator when results exceed the limit.
- **FR-012**: System MUST gracefully degrade to SQL-only mode when the AI translation service is unavailable, accepting raw SQL queries while rejecting natural language input with a clear error message.
- **FR-013**: System MUST log all queries (input, translated SQL, execution status, timing) without logging result data, for debugging and security auditing purposes.
- **FR-014**: System MUST detect ambiguous natural language queries and return clarification options to the agent rather than executing a potentially incorrect interpretation.

### Key Entities

- **Database Connection**: Represents a configured database with connection parameters, identifier, and optional description.
- **Query Request**: Contains the input (natural language or SQL), target database identifier, and any query options.
- **Query Result**: Contains the returned data, column metadata, row count, and execution status.
- **Schema Representation**: Contains tables, columns, data types, constraints, and relationship information for a database.
- **Tool Documentation**: Contains help text, usage examples, and recommended workflows for each MCP tool.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Agents can successfully query a database using natural language within 5 seconds for typical queries.
- **SC-002**: Schema representation is clear enough that an agent can formulate valid queries without prior database knowledge.
- **SC-003**: 95% of syntactically valid natural language queries produce correct SQL translations.
- **SC-004**: System handles database connection failures gracefully with clear error messages within 10 seconds.
- **SC-005**: Agents can successfully switch between multiple configured databases within a single session.
- **SC-006**: Help documentation enables agents to use tools correctly on first attempt in 90% of cases.
- **SC-007**: Invalid SQL is detected and rejected before execution 100% of the time.

## Clarifications

### Session 2026-02-01

- Q: Should the query tool allow write operations (INSERT, UPDATE, DELETE) or be restricted to read-only queries? → A: Read-only (SELECT queries only)
- Q: How should the system handle queries that return very large result sets? → A: Row limit with truncation indicator (return up to limit with message indicating truncation)
- Q: How should the system behave when the AI translation service is unavailable? → A: Fallback to SQL-only mode (accept raw SQL, reject natural language with clear message)
- Q: What level of logging/observability should the system provide? → A: Query logging (log all queries and their status, but not result data)
- Q: How should the system handle ambiguous natural language queries? → A: Ask for clarification (return options and ask agent to specify)

### Round 1 Updates (2026-02-01)

- **Reinforced SELECT-only enforcement**: AI-generated SQL MUST be validated before execution; non-SELECT queries from AI are rejected
- **Added OpenAI provider support**: AI provider abstraction now includes Anthropic, AWS Bedrock, AND OpenAI as supported backends

## Assumptions

- Database credentials and connection strings will be provided via environment variables following standard patterns.
- The AI translation service will be available and responsive during operation.
- Target databases will be PostgreSQL-compatible initially, with the architecture supporting future database types.
- Result sets are limited to a configurable maximum (default 1000 rows); when exceeded, results are truncated and a clear indicator is included in the response.
- The MCP protocol's stdin/stdout communication pattern is well-defined and stable.
