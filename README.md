# Skematia Protocol Specification v1.0

**An open protocol for AI agents and IDEs to interact with real databases.**

**Status:** PUBLISHED
**Date:** 2026-04-26
**License:** Apache 2.0
**Authors:** Reyes Uscanga, Claude (Development Agent)

---

## 1. Overview

The Skematia Protocol is an open specification that enables **AI agents and IDEs** — Claude, Cursor, Windsurf, GitHub Copilot, custom autonomous agents, and any tool that wants to query, design, or operate on real databases — to connect to a production-grade backend with a single configuration step.

The protocol turns a database into a **first-class citizen of the agent ecosystem**: discoverable via Agent Card (A2A spec), operable via MCP (Model Context Protocol), and accessible via REST for any other client. An agent that has never seen a Skematia instance before can read its `/.well-known/agent.json`, learn what it can do, authenticate, and start working — without human assistance.

### Why this exists

The current AI tooling landscape has a gap. Agents are powerful at reasoning and code generation, but their interaction with persistent data is limited to flat files, fragile shell commands, or one-off database wrappers each developer has to build from scratch. There is no shared, standard way for an agent to:

1. **Discover** what a database service can do.
2. **Authenticate** with a single credential that works across protocols.
3. **Operate** on the data — query, mutate, design schemas, evolve them safely.
4. **Trust** the result — with versioning, audit, rollback, and safety guards built in.

The Skematia Protocol fills that gap. It is to AI agents what JDBC was to Java applications, or what ODBC was to desktop apps — a uniform way to talk to a database, but designed for the era of autonomous agents.

### Primary audience

The protocol is designed for, in order of priority:

1. **AI agents** — autonomous or semi-autonomous LLM-based systems that need persistent state.
2. **AI-powered IDEs** — Claude Code, Cursor, Windsurf, GitHub Copilot, Antigravity, and similar tools whose users want to point an agent at a real backend.
3. **Traditional applications** — desktop apps, web services, scripts, and any HTTP-capable client. These are first-class but not the primary driver.

### Architecture

```
                      ┌──────────────────────────┐
   AI Agent (Claude)  │                          │
   AI IDE (Cursor)    │   Skematia Protocol      │
   Custom Bot         │  ┌─────────────────────┐ │
   Desktop App     →  │  │ Agent Card (A2A)    │ │  →  Firebird 5.0
   Web Service        │  │ MCP (JSON-RPC)      │ │     (one .fdb per project)
   Shell / cURL       │  │ REST API            │ │     (never exposed)
                      │  └─────────────────────┘ │
                      │     Auth + Versioning    │
                      │     Schema AI + Audit    │
                      └──────────────────────────┘
```

The database engine is never exposed to the internet. All access flows through the protocol layer, which provides discovery, authentication, schema management, safety guards, audit trails, and AI-assisted operations.

---

## 2. Core Concepts

### 2.1 Project

A **project** is an isolated Firebird 5.0 database. Each project has its own `.fdb` file, its own API key, and its own data. It is equivalent to a separate database connection in any driver or ORM.

### 2.2 API Key

An **API key** (`sk_...`) is the credential that identifies a client and associates it with a project. It is sent with every HTTP request as a Bearer token — the equivalent of database credentials, but over HTTP.

### 2.3 Agent

An **agent** is any software that connects to the protocol: an AI bot, a desktop application, a Python script, a Node.js service, a PowerShell automation — anything that speaks HTTP.

### 2.4 Transport

The protocol supports two communication methods, both of which MUST be implemented by a conformant server:

- **MCP (Model Context Protocol)** — The primary transport for AI agents and IDEs. Uses JSON-RPC over HTTP in stateless mode. Designed for Claude, GPT, and any LLM-based agent to discover and call tools without prior coordination. This is the path that delivers the "single configuration step" promise.
- **REST API** — Standard HTTP endpoints for traditional clients (desktop apps, scripts, services, automation). Any language with an HTTP client can use it.

Both transports reach the same Firebird engine, share the same authentication, and respect the same permissions and safety guards. The difference is only in the request format. An agent operating via MCP and an application operating via REST against the same project see the same data with the same rules.

---

## 3. Discovery

### 3.1 Agent Card

Every server implementing the Skematia Protocol MUST publish an Agent Card at:

```
GET /.well-known/agent.json
```

The Agent Card follows the A2A Protocol specification v0.2.6 and describes the server's capabilities. Any agent reaching a Skematia instance can read this file to learn how to connect without human assistance.

**Required fields (MUST):**

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Service name |
| `description` | string | Capability description |
| `url` | string | Base URL |
| `version` | string | Service version |
| `protocolVersion` | string | A2A protocol version |
| `capabilities` | object | Supported capabilities |
| `skills` | array | Available operations |
| `defaultInputModes` | array | Accepted input formats |
| `defaultOutputModes` | array | Output formats |

**Recommended fields (SHOULD):**

| Field | Type | Description |
|-------|------|-------------|
| `integration` | object | Connection instructions per language/platform |
| `documentationUrl` | string | Documentation URL |
| `authentication` | object | Authentication scheme |

**Minimal example:**

```json
{
  "name": "SkematIA",
  "description": "AI-powered BaaS for Firebird 5.0",
  "url": "https://skematia.com",
  "version": "1.0.0",
  "protocolVersion": "0.2.6",
  "capabilities": { "streaming": false, "pushNotifications": false },
  "skills": [
    {
      "id": "execute_sql",
      "name": "Execute SQL",
      "description": "Execute SQL statements against the project's Firebird database"
    }
  ],
  "defaultInputModes": ["text/plain", "application/json"],
  "defaultOutputModes": ["application/json"],
  "authentication": {
    "schemes": ["bearer"],
    "description": "API Key (sk_...) as Bearer token"
  }
}
```

---

## 4. Authentication

### 4.1 API Keys

Every request to the protocol MUST include an API key in the HTTP header:

```
Authorization: Bearer sk_XXXXXXXX_XXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Key format: `sk_` + partial project_id + `_` + random token.

**Rules:**

- An API key is bound to exactly one project.
- The project_id MAY be omitted in requests if the key is already bound.
- If an explicit project_id is sent, it MUST match the key's project. Otherwise, the server responds `403 Forbidden`.

### 4.2 Permissions

Each API key has a permission level:

| Level | Allowed operations |
|-------|--------------------|
| `read_only` | SELECT only |
| `read_write` | SELECT + INSERT, UPDATE, DELETE |
| `full` | Everything including CREATE, ALTER, DROP |

Default is `full` for backward compatibility. `read_write` is RECOMMENDED for external agents. `full` SHOULD be reserved for the project owner's dashboard.

### 4.3 Rate Limiting

The server MAY implement rate limiting per API key. Recommended scheme is sliding window with the following response headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1712234567
```

When exceeded, the server responds `429 Too Many Requests`.

---

## 5. REST API

### 5.1 Base URL

All REST endpoints use the `/api/` prefix:

```
https://skematia.com/api/data/{project_id}/{table_name}
```

### 5.2 Data Endpoints

#### Get rows

```
GET /api/data/{project_id}/{table_name}
```

Response:
```json
{
  "rows": [
    {"ID": 1, "NAME": "Juan", "AGE": 35},
    {"ID": 2, "NAME": "Maria", "AGE": 28}
  ],
  "count": 2
}
```

#### Insert record

```
POST /api/data/{project_id}/{table_name}
Content-Type: application/json

{"NAME": "Pedro", "AGE": 42}
```

Response:
```json
{
  "row": {"ID": 3, "NAME": "Pedro", "AGE": 42}
}
```

#### Update record

```
PUT /api/data/{project_id}/{table_name}/{id}
Content-Type: application/json

{"AGE": 43}
```

#### Delete record

```
DELETE /api/data/{project_id}/{table_name}/{id}
```

#### Execute raw SQL

```
POST /api/data/{project_id}/raw-sql
Content-Type: application/json

{"sql": "SELECT * FROM CLIENTS WHERE AGE > 30"}
```

Response:
```json
{
  "results": [
    {"ID": 1, "NAME": "Juan", "AGE": 35}
  ]
}
```

**SQL Safety Guards:**

The following statements are blocked by default via REST and MCP:

- `DROP TABLE`
- `DROP DATABASE`
- `DROP INDEX`
- `TRUNCATE TABLE`
- `ALTER TABLE ... DROP`
- `DELETE` without `WHERE` clause

The server MUST respond `403 Forbidden` with a descriptive message when any of these are detected.

### 5.3 Schema Endpoints

```
GET  /api/explorer/{project_id}/tables                          — List tables
GET  /api/explorer/{project_id}/tables/{table_name}/structure   — Table structure
GET  /api/explorer/{project_id}/schema/history                  — DDL change history
GET  /api/explorer/{project_id}/schema/suggestions              — Improvement suggestions
```

### 5.4 Project Endpoints

```
GET  /api/projects          — List user's projects
POST /api/projects          — Create project
GET  /api/projects/{id}     — Project details
```

### 5.5 Error Format

All errors MUST follow this structure:

```json
{
  "error": "Descriptive message",
  "error_type": "error_type_code",
  "hint": "How to fix it",
  "details": {}
}
```

**Defined error types:**

| error_type | HTTP Status | Description |
|-----------|-------------|-------------|
| `table_not_found` | 400 | Table does not exist |
| `column_not_found` | 400 | Column does not exist |
| `string_truncation` | 400 | Data exceeds field size |
| `not_null_violation` | 400 | Required field missing |
| `duplicate_key` | 400 | Duplicate value in unique field |
| `foreign_key_violation` | 400 | Reference to nonexistent record |
| `type_mismatch` | 400 | Incorrect data type |
| `numeric_overflow` | 400 | Number out of range |
| `permission_denied` | 403 | No permission for this operation |
| `sql_blocked` | 403 | SQL statement blocked by safety guards |
| `rate_limited` | 429 | Request limit exceeded |
| `connection_error` | 503 | Firebird connection error |

---

## 6. MCP (Model Context Protocol)

### 6.1 Transport

The MCP server operates in **stateless** mode over HTTP:

```
POST /mcp/
Content-Type: application/json
Authorization: Bearer sk_...
```

Each request is independent. No session-id or prior state is required. This allows horizontal scaling without shared state.

### 6.2 Available Tools

| Tool | Description | Required params |
|------|-------------|-----------------|
| `list_projects` | List accessible projects | — |
| `list_tables` | List tables with counts | `project_id` |
| `get_schema` | Full project schema | `project_id` |
| `execute_sql` | Execute arbitrary SQL | `project_id`, `sql` |
| `query` | Natural language query | `project_id`, `question` |
| `insert_data` | Insert records | `table`, `records` |
| `update_data` | Update by primary key | `table`, `id_value`, `data` |
| `delete_data` | Delete by primary key | `table`, `id_value` |
| `bulk_update` | Bulk update with WHERE | `table`, `where`, `data` |
| `design_schema` | AI-assisted schema design | `description` |
| `modify_schema` | AI-assisted schema modification | `project_id`, `instruction` |
| `schema_history` | DDL change history | `project_id` |
| `schema_suggestions` | Schema improvement suggestions | `project_id` |
| `list_backups` | List available backups | `project_id` |

### 6.3 Implicit API Key

If the API key is sent in the `Authorization` header, it does not need to be included as a parameter in each tool call. The server extracts it automatically from the header.

Priority: explicit API key in parameters > API key from header.

### 6.4 Tool Call Example

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "execute_sql",
    "arguments": {
      "project_id": "abc123",
      "sql": "SELECT * FROM CLIENTS ROWS 10"
    }
  },
  "id": 1
}
```

---

## 7. Schema Management

### 7.1 Versioning

Every DDL change (CREATE, ALTER, DROP) is automatically recorded with:

- Timestamp
- SQL executed
- Source (ai_studio, explorer, mcp, rest_api)
- Agent that executed it

The history is immutable. It cannot be edited or deleted.

### 7.2 Schema Advisor

The server MAY offer automatic schema analysis with improvement suggestions:

- Fields that should be NOT NULL
- Fields that should be UNIQUE
- Foreign keys without indexes
- Tables without primary keys

Each suggestion includes the SQL to apply it.

### 7.3 Assisted Migrations

Advisor suggestions can be applied with automatic rollback:

- Each migration generates an `up` (apply) and `down` (revert) script
- The `down` script is calculated automatically from the `up` SQL

---

## 8. Firebird Notes

### 8.1 Syntax Differences

Firebird 5.0 has differences from other engines that implementors MUST be aware of:

| Concept | Firebird 5.0 | MySQL / PostgreSQL |
|---------|-------------|-------------------|
| Row limit | `ROWS 10` | `LIMIT 10` |
| Table alias | `AS` required | `AS` optional |
| Column names | Always UPPERCASE from RDB$ | Case-sensitive |
| INSERT with return | `RETURNING` supported since v2.1 | Supported |
| Reserved word | `PRECISION` is reserved | Not in all engines |

### 8.2 System Tables

Schema introspection uses Firebird's `RDB$` tables:

- `RDB$RELATIONS` — Tables and views
- `RDB$RELATION_FIELDS` — Fields per table
- `RDB$INDEX_SEGMENTS` — Index segments
- `RDB$REF_CONSTRAINTS` — Foreign keys (use `RDB$CONST_NAME_UQ`, not `RDB$CONST_NAME_UKD`)
- `RDB$VIEW_SOURCE` — View source (not `RDB$VIEW_BLN` which does not exist in v5)

### 8.3 One File Per Project

Each project is an independent `.fdb` file on the server's filesystem. This enables:

- Individual backup with `gbak`
- Restore without affecting other projects
- Portability: the file can be moved to another server

---

## 9. Security

### 9.1 Isolation

- Each project has its own Firebird database (separate `.fdb` file).
- An API key can only access the project it is bound to.
- Firebird is not exposed to the internet — accessible only internally via Docker network.

### 9.2 SQL Safety Guards

Destructive statements are blocked by default (see section 5.2). An implementor MAY disable this for `full` permission API keys, but it is RECOMMENDED to keep it always active.

### 9.3 Audit Trail

All DDL operations are recorded in the schema history (section 7.1). DML operations (INSERT/UPDATE/DELETE) MAY be recorded optionally via audit log.

### 9.4 Backups

The server MUST implement automatic backups for each project:

- Minimum frequency: daily
- Minimum retention: 7 days
- Tool: `gbak` (Firebird native)
- Pre-restore backup required before any restoration

---

## 10. Connection Examples

### 10.1 Python

```python
import requests

API_URL = "https://skematia.com/api"
API_KEY = "sk_abc123_xxxxxxx"
PROJECT = "abc123-def456"
HEADERS = {"Authorization": f"Bearer {API_KEY}"}

# SQL query
r = requests.post(
    f"{API_URL}/data/{PROJECT}/raw-sql",
    headers=HEADERS,
    json={"sql": "SELECT * FROM CLIENTS ROWS 10"}
)
print(r.json()["results"])

# Insert record
r = requests.post(
    f"{API_URL}/data/{PROJECT}/CLIENTS",
    headers=HEADERS,
    json={"NAME": "Juan", "PHONE": "555-1234"}
)
print(r.json()["row"])
```

### 10.2 Node.js

```javascript
const API_URL = "https://skematia.com/api";
const API_KEY = "sk_abc123_xxxxxxx";
const PROJECT = "abc123-def456";

const res = await fetch(`${API_URL}/data/${PROJECT}/raw-sql`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${API_KEY}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ sql: "SELECT * FROM CLIENTS ROWS 10" })
});
const data = await res.json();
console.log(data.results);
```

### 10.3 cURL

```bash
curl -X POST https://skematia.com/api/data/PROJECT_ID/raw-sql \
  -H "Authorization: Bearer sk_abc123_xxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT * FROM CLIENTS ROWS 10"}'
```

### 10.4 PowerShell

```powershell
$headers = @{
    "Authorization" = "Bearer sk_abc123_xxxxxxx"
    "Content-Type"  = "application/json"
}

$body = @{ sql = "SELECT * FROM CLIENTS ROWS 10" } | ConvertTo-Json

$response = Invoke-RestMethod `
    -Uri "https://skematia.com/api/data/PROJECT_ID/raw-sql" `
    -Method POST `
    -Headers $headers `
    -Body $body

$response.results
```

### 10.5 Delphi / Lazarus

```pascal
uses System.Net.HttpClient, System.JSON;

function QuerySkematia(const AURL, AApiKey, ASQL: string): TJSONObject;
var
  Client: THTTPClient;
  Response: IHTTPResponse;
  Body: TStringStream;
  ReqJSON: TJSONObject;
begin
  Client := THTTPClient.Create;
  try
    Client.CustomHeaders['Authorization'] := 'Bearer ' + AApiKey;
    Client.ContentType := 'application/json';

    ReqJSON := TJSONObject.Create;
    ReqJSON.AddPair('sql', ASQL);

    Body := TStringStream.Create(ReqJSON.ToJSON, TEncoding.UTF8);
    try
      Response := Client.Post(
        AURL + '/api/data/PROJECT_ID/raw-sql',
        Body
      );
      Result := TJSONObject.ParseJSONValue(
        Response.ContentAsString
      ) as TJSONObject;
    finally
      Body.Free;
      ReqJSON.Free;
    end;
  finally
    Client.Free;
  end;
end;
```

### 10.6 MCP via Claude Desktop

`claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "skematia": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://skematia.com/mcp/",
        "--header",
        "Authorization: Bearer sk_abc123_xxxxxxx"
      ]
    }
  }
}
```

---

## 11. Protocol Versioning

### 11.1 Version Scheme

The protocol uses semantic versioning: `MAJOR.MINOR.PATCH`

- **MAJOR** — Breaking changes (e.g., changing auth format, renaming endpoints)
- **MINOR** — New backward-compatible features (e.g., new endpoint, new MCP tool)
- **PATCH** — Non-functional fixes

### 11.2 Compatibility Guarantee

A server declaring support for v1.x MUST maintain compatibility with all v1.0 clients:

- Existing endpoints do not change their URL or response format.
- Existing MCP tools do not change their required parameters.
- New features are added as additional endpoints/tools, never replacing existing ones.
- Optional fields MAY be added to JSON responses without breaking existing clients.

### 11.3 Deprecation

When an endpoint or tool needs to be replaced:

1. The new endpoint/tool is published in a MINOR version.
2. The old one is marked deprecated with a `Deprecated: true` header in responses.
3. It remains functional for at least 6 months or 2 MINOR versions.
4. It is removed in the next MAJOR version.

---

## 12. Conformance

### 12.1 Implementation Levels

| Level | Requirements |
|-------|-------------|
| **Core** | Agent Card + Auth + REST CRUD + Raw SQL |
| **Schema** | Core + versioning + advisor + migrations |
| **AI** | Schema + design_schema + modify_schema + natural language query |
| **Full** | AI + MCP transport + backups + rate limiting + audit |

An implementor declares their level in the Agent Card's `capabilities` field.

### 12.2 RFC Terminology

In this specification:

- **MUST** — Absolute requirement
- **SHOULD** — Recommended, with justification for exceptions
- **MAY** — Optional

---

## Appendix A: Design Decisions

### A.1 Why Firebird

Firebird has over 25 years of development. It is ACID compliant, works offline with a single `.fdb` file, requires no per-use licensing, and has native replication in v5.0. These characteristics make it ideal for scenarios where connectivity is intermittent — rural clinics, retail stores, remote branches.

### A.2 Why Stateless MCP

Stateless mode (`stateless_http=True`) eliminates the Redis dependency for MCP sessions. Each request is independent, allowing horizontal scaling without coordination between instances. The trade-off is no conversational state between calls — but for data operations, this is not needed.

### A.3 Why API Keys (not OAuth)

OAuth adds significant complexity for the primary use case (AI agents and desktop applications). API keys are simple, predictable, and work identically in Python, Node.js, cURL, PowerShell, and Delphi. OAuth can be added as an option in a future version without breaking existing authentication.

### A.4 Why REST + MCP (not just one)

REST is universal — every language speaks it. MCP is the emerging standard for AI agents. Supporting both allows a legacy desktop application and a modern AI agent to connect with equal ease.

---

## Appendix B: Roadmap (future extensions)

The following extensions are planned as MINOR protocol versions. All are backward-compatible with v1.0:

- **v1.1** — Offline replication (local `.fdb` file sync with cloud)
- **v1.2** — Row-Level Security (automatic filtering by user context)
- **v1.3** — Realtime (push notifications via Firebird POST_EVENT + WebSockets)
- **v1.4** — Schema marketplace (pre-built schemas by industry)
- **v1.5** — Domain plugins (domain-specific validation and business logic)

---

*Skematia Protocol Specification v1.0 — PUBLISHED*
*"The database layer for AI agents and IDEs."*
