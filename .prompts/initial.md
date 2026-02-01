## Task Description
Basically, we want to build a **mcp sever** for PostgreSQL database that can provide tools for agent to query against.

## Requirements
1. The sever should support multi-databases, and can configured through environment variables.
2. There are three main APIs,
   - `query`: take natural language/SQL as input and return the result. Better call `schema` first to grasp the schema of the database.
   - `schema`: return the schema of the database. Good ascii illustration of the database schema. rough description of each table and columns.
   - `help`: return the help message of the server. Like how to use, what's the order of the tool-calls.


## Tech Concerns
1. Python & uv is the preferred language for the project.
2. fastmcp for mcp framework.
3. make sure the input sql can compile to valid sql.if not, return the error message.
4. use anthropic api for sql generation. (support bedrock config and straight anthropic api key both)
5. use stdin and stdout for communication between the server and the agent.
6. certain level of abstraction is in need, incase we want to support other SQL databases like mysql, sqlite, etc.
7. explore the internet, make sure we leverage the most popular and modern deps for the project.