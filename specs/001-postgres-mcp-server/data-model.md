# Data Model: SQLlike MCP Server

**Date**: 2026-02-01  
**Feature**: 001-postgres-mcp-server

> **Note**: Project rebranded to "SQLlike" to reflect future multi-database support.

## Overview

This MCP server is stateless - it maintains no persistent data of its own. The data model describes the runtime structures used for request/response handling and configuration.

---

## Configuration Entities

### DatabaseConfig

Represents a single database connection configuration.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| identifier | string | Yes | Unique name for this database (e.g., "main", "analytics") |
| connection_string | string | Yes | PostgreSQL connection URI |
| description | string | No | Human-readable description for help output |
| pool_min_size | int | No | Minimum pool connections (default: 1) |
| pool_max_size | int | No | Maximum pool connections (default: 10) |

**Validation Rules**:
- `identifier` must be alphanumeric with hyphens/underscores, 1-50 chars
- `connection_string` must be valid PostgreSQL URI format
- Pool sizes must be positive integers, min <= max

---

### ServerConfig

Top-level server configuration.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| databases | list[DatabaseConfig] | Yes | List of database configurations |
| default_database | string | No | Default database identifier when not specified |
| row_limit | int | No | Maximum rows returned (default: 1000) |
| log_level | string | No | Logging level: DEBUG, INFO, WARNING, ERROR (default: INFO) |
| ai_provider | AIProviderConfig | Yes | AI provider configuration |

---

### AIProviderConfig

Configuration for AI-powered SQL translation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| provider | enum | Yes | "anthropic", "bedrock", or "openai" |
| api_key | string | Conditional | Required if provider is "anthropic" or "openai" |
| aws_region | string | Conditional | Required if provider is "bedrock" |
| aws_access_key_id | string | Conditional | Required if provider is "bedrock" |
| aws_secret_access_key | string | Conditional | Required if provider is "bedrock" |
| model | string | No | Model identifier (defaults: claude-3-5-sonnet for Anthropic/Bedrock, gpt-4o for OpenAI) |

**Provider-specific environment variables**:
- Anthropic: `ANTHROPIC_API_KEY`
- OpenAI: `OPENAI_API_KEY`
- Bedrock: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
- Provider selection: `SQLLIKE_AI_PROVIDER` (auto-detects if not set)

---

## Request/Response Entities

### QueryRequest

Input to the `query` tool.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| input | string | Yes | Natural language query or raw SQL |
| database | string | No | Target database identifier (uses default if omitted) |
| limit | int | No | Override row limit for this query |

---

### QueryResult

Output from the `query` tool on success.

| Field | Type | Description |
|-------|------|-------------|
| columns | list[ColumnInfo] | Column metadata |
| rows | list[list[any]] | Result data (list of row values) |
| row_count | int | Number of rows returned |
| truncated | bool | True if results exceeded limit |
| total_available | int | Total rows available (if known), null otherwise |
| executed_sql | string | The SQL that was executed |
| execution_time_ms | float | Query execution time in milliseconds |

---

### ColumnInfo

Metadata for a result column.

| Field | Type | Description |
|-------|------|-------------|
| name | string | Column name |
| type | string | PostgreSQL data type |
| nullable | bool | Whether column allows NULL |

---

### SchemaRequest

Input to the `schema` tool.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| database | string | No | Target database identifier (uses default if omitted) |
| table | string | No | Specific table name (all tables if omitted) |

---

### SchemaResult

Output from the `schema` tool.

| Field | Type | Description |
|-------|------|-------------|
| database | string | Database identifier |
| tables | list[TableSchema] | List of table schemas |
| ascii_diagram | string | ASCII representation of schema relationships |

---

### TableSchema

Schema information for a single table.

| Field | Type | Description |
|-------|------|-------------|
| name | string | Table name |
| description | string | Table comment/description (if available) |
| columns | list[ColumnSchema] | Column definitions |
| primary_key | list[string] | Primary key column names |
| foreign_keys | list[ForeignKey] | Foreign key relationships |

---

### ColumnSchema

Schema information for a single column.

| Field | Type | Description |
|-------|------|-------------|
| name | string | Column name |
| type | string | PostgreSQL data type |
| nullable | bool | Whether column allows NULL |
| default | string | Default value expression (if any) |
| description | string | Column comment/description (if available) |

---

### ForeignKey

Foreign key relationship.

| Field | Type | Description |
|-------|------|-------------|
| columns | list[string] | Local column names |
| references_table | string | Referenced table name |
| references_columns | list[string] | Referenced column names |

---

### HelpResult

Output from the `help` tool.

| Field | Type | Description |
|-------|------|-------------|
| tools | list[ToolHelp] | Available tools with descriptions |
| workflow | string | Recommended workflow/order of operations |
| examples | list[Example] | Usage examples |
| databases | list[DatabaseSummary] | Available databases |

---

### ToolHelp

Help information for a single tool.

| Field | Type | Description |
|-------|------|-------------|
| name | string | Tool name |
| description | string | What the tool does |
| parameters | list[ParameterHelp] | Parameter documentation |

---

### ErrorResult

Standard error response.

| Field | Type | Description |
|-------|------|-------------|
| error | string | Error type (e.g., "validation_error", "database_error") |
| message | string | Human-readable error description |
| details | dict | Additional error context |
| suggestions | list[string] | Actionable suggestions for resolution |

---

### ClarificationRequest

Returned when natural language query is ambiguous (FR-014).

| Field | Type | Description |
|-------|------|-------------|
| type | string | Always "clarification_needed" |
| message | string | Explanation of ambiguity |
| options | list[ClarificationOption] | Possible interpretations |

---

### ClarificationOption

A possible interpretation of an ambiguous query.

| Field | Type | Description |
|-------|------|-------------|
| id | string | Option identifier (e.g., "A", "B", "C") |
| description | string | Natural language description |
| sql | string | The SQL this option would execute |

---

## State Transitions

### Query Processing Flow

```
Input Received
    │
    ▼
┌─────────────────┐
│ Detect Input    │──▶ Raw SQL ──▶ Validate SQL ──▶ Execute
│ Type            │
└─────────────────┘
    │
    ▼ Natural Language
┌─────────────────┐
│ AI Translation  │──▶ Unavailable ──▶ Return SQL-only mode error
└─────────────────┘
    │
    ▼ Available
┌─────────────────┐
│ Generate SQL    │──▶ Ambiguous ──▶ Return ClarificationRequest
└─────────────────┘
    │
    ▼ Clear
┌─────────────────┐
│ Validate SQL    │──▶ Invalid ──▶ Return ErrorResult
└─────────────────┘
    │
    ▼ Valid + Read-only
┌─────────────────┐
│ Execute Query   │──▶ Error ──▶ Return ErrorResult
└─────────────────┘
    │
    ▼ Success
Return QueryResult (with truncation if needed)
```

---

## Relationships Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     ServerConfig                             │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │ databases[] │  │ row_limit   │  │ ai_provider      │    │
│  └──────┬──────┘  └─────────────┘  └──────────────────┘    │
│         │                                                    │
└─────────┼────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────┐
│ DatabaseConfig  │ 1..* per server
│  - identifier   │
│  - conn_string  │
│  - pool config  │
└─────────────────┘
          │
          │ connects to
          ▼
┌─────────────────┐
│ PostgreSQL DB   │ (external)
│  - tables       │
│  - schemas      │
└─────────────────┘
```
