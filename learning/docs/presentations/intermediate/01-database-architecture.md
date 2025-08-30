# Intermediate 01: Database Architecture Deep Dive

**Duration**: 30-35 minutes
**Level**: 🔧 Intermediate
**Goal**: Master the data layer, SQLite integration, and query patterns

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand the complete database architecture and design decisions
- Master the sqlc-based query generation system
- Learn migration strategies and schema evolution
- Understand performance characteristics and optimization opportunities
- Be able to add new database features and queries

---

## 📋 Prerequisites

- ✅ Completed all Beginner level tutorials
- ✅ Solid understanding of SQL and database design
- ✅ Experience with Go database programming (`database/sql`)
- ✅ Familiarity with migration systems

---

## 🏗️ Architecture Overview

Crush uses a sophisticated data layer built on SQLite with modern Go patterns:

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Layer                            │
│  ┌──────────────┬──────────────┬──────────────┬───────────┐ │
│  │ session.Service │ message.Service │ history.Service │ etc... │ │
│  └──────────────┴──────────────┴──────────────┴───────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │ (uses)
┌─────────────────────▼───────────────────────────────────────┐
│                Generated Query Layer (sqlc)                 │
│  ┌─────────────────┬─────────────────┬─────────────────────┐ │
│  │ SessionQueries  │ MessageQueries  │ FileQueries         │ │
│  └─────────────────┴─────────────────┴─────────────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │ (generates from)
┌─────────────────────▼───────────────────────────────────────┐
│                   SQL Query Files                           │
│  ┌─────────────────┬─────────────────┬─────────────────────┐ │
│  │ sessions.sql    │ messages.sql    │ files.sql           │ │
│  └─────────────────┴─────────────────┴─────────────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │ (executed against)
┌─────────────────────▼───────────────────────────────────────┐
│                    SQLite Database                          │
│              (with migrations applied)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema Deep Dive

Let's examine the current schema and understand the design decisions:

### Core Tables

```bash
# Look at the initial migration
cat internal/db/migrations/20250424200609_initial.sql
```

**Table Design Analysis**:

#### 1. **Sessions Table**
```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Design Decisions**:
- `TEXT PRIMARY KEY`: Uses UUID strings for globally unique IDs
- `provider` and `model`: Allows switching AI providers per session
- `created_at`/`updated_at`: Audit trail and sorting capabilities

#### 2. **Messages Table**
```sql
CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    provider TEXT,
    model TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id) ON DELETE CASCADE
);
```

**Design Decisions**:
- `CHECK` constraint: Enforces valid message roles at database level
- `CASCADE DELETE`: When session is deleted, all messages go with it
- `provider`/`model` on messages: Tracks which AI generated each response
- No `updated_at`: Messages are immutable once created

#### 3. **Files Table**
```sql
CREATE TABLE files (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    size INTEGER NOT NULL,
    modified_at DATETIME NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(path, content_hash)
);
```

**Design Decisions**:
- `AUTOINCREMENT`: Simple integer IDs for file tracking
- `content_hash`: Detects file changes without re-reading content
- `UNIQUE(path, content_hash)`: Prevents duplicate entries
- File versioning through hash changes

---

## ⚡ sqlc Integration Mastery

Crush uses [sqlc](https://github.com/kyleconroy/sqlc) for type-safe query generation:

### Configuration

```bash
# Look at sqlc configuration
cat sqlc.yaml
```

```yaml
version: "2"
sql:
  - engine: "sqlite"
    queries: "internal/db/sql"
    schema: "internal/db/migrations"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_prepared_queries: false
        emit_interface: true
        emit_exact_table_func: false
```

**Key Configuration Decisions**:
- `emit_interface: true`: Generates interfaces for easy mocking
- `emit_json_tags: true`: Struct fields can be JSON serialized
- `emit_prepared_queries: false`: Simpler API for SQLite use case

### Query Files Structure

```bash
# Look at the query organization
tree internal/db/sql/
```

Each domain gets its own `.sql` file:
- `sessions.sql` - Session CRUD operations
- `messages.sql` - Message operations with complex joins
- `files.sql` - File tracking and history

### Generated Code Analysis

```bash
# Look at generated query methods
head -50 internal/db/sessions.sql.go
```

**Generated Patterns**:

```go
// Generated struct (from schema)
type Session struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Provider  string    `json:"provider"`
    Model     string    `json:"model"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// Generated query interface
type Querier interface {
    CreateSession(ctx context.Context, arg CreateSessionParams) error
    GetSession(ctx context.Context, id string) (Session, error)
    ListSessions(ctx context.Context) ([]Session, error)
    UpdateSession(ctx context.Context, arg UpdateSessionParams) error
    DeleteSession(ctx context.Context, id string) error
}

// Generated query implementation
func (q *Queries) CreateSession(ctx context.Context, arg CreateSessionParams) error {
    _, err := q.db.ExecContext(ctx, createSession,
        arg.ID,
        arg.Name,
        arg.Provider,
        arg.Model,
    )
    return err
}
```

---

## 🔄 Migration System Deep Dive

Crush uses [goose](https://github.com/pressly/goose) for database migrations:

### Migration Strategy

```bash
# Look at migration files
ls -la internal/db/migrations/
```

**Naming Convention**: `YYYYMMDDHHMMSS_description.sql`
- Timestamps ensure chronological ordering
- Descriptive names explain the change purpose

### Migration Evolution Example

Let's trace how the schema evolved:

#### 1. **Initial Schema** (20250424200609_initial.sql)
```sql
-- Basic tables: sessions, messages, files
-- Core functionality only
```

#### 2. **Add Summary Feature** (20250515105448_add_summary_message_id.sql)
```sql
-- Add summary_message_id to sessions
-- Enables conversation summarization
```

#### 3. **Performance Optimization** (20250624000000_add_created_at_indexes.sql)
```sql
-- Add indexes on created_at columns
-- Improves query performance for time-based operations
```

#### 4. **Provider Tracking** (20250627000000_add_provider_to_messages.sql)
```sql
-- Add provider column to messages
-- Track which AI service generated each response
```

### Migration Best Practices Used

1. **Additive Changes**: New columns with DEFAULT values
2. **Backward Compatibility**: Old code continues working
3. **Performance Aware**: Indexes added separately
4. **Atomic Changes**: Each migration does one logical thing

---

## 🚀 Query Patterns and Performance

### Complex Query Examples

Let's examine sophisticated queries in the codebase:

#### 1. **Session with Message Count**
```sql
-- name: GetSessionWithMessageCount :one
SELECT
    s.*,
    COALESCE(COUNT(m.id), 0) as message_count
FROM sessions s
LEFT JOIN messages m ON s.id = m.session_id
WHERE s.id = $1
GROUP BY s.id;
```

#### 2. **Recent Sessions with Last Message**
```sql
-- name: GetRecentSessionsWithLastMessage :many
SELECT
    s.*,
    m.content as last_message,
    m.created_at as last_message_at
FROM sessions s
LEFT JOIN (
    SELECT DISTINCT session_id,
           FIRST_VALUE(content) OVER (
               PARTITION BY session_id
               ORDER BY created_at DESC
           ) as content,
           FIRST_VALUE(created_at) OVER (
               PARTITION BY session_id
               ORDER BY created_at DESC
           ) as created_at
    FROM messages
) m ON s.id = m.session_id
ORDER BY COALESCE(m.created_at, s.updated_at) DESC
LIMIT $1;
```

### Performance Characteristics

**SQLite Optimizations Used**:
1. **WAL Mode**: Better concurrency for read-heavy workloads
2. **Foreign Keys**: Enabled for referential integrity
3. **Strategic Indexes**: On commonly queried columns
4. **Query Planning**: EXPLAIN QUERY PLAN for optimization

```bash
# Look at connection setup
grep -A 10 -B 5 "foreign_keys" internal/db/connect.go
```

---

## 🔧 Service Layer Integration

### Service Pattern Implementation

Each domain service wraps the generated queries:

```go
// Session service example
type Service struct {
    queries *db.Queries
}

func NewService(queries *db.Queries) *Service {
    return &Service{queries: queries}
}

func (s *Service) Create(ctx context.Context, session Session) error {
    return s.queries.CreateSession(ctx, db.CreateSessionParams{
        ID:       session.ID,
        Name:     session.Name,
        Provider: session.Provider,
        Model:    session.Model,
    })
}
```

**Benefits of This Pattern**:
- **Clean Separation**: Business logic separate from data access
- **Easy Testing**: Services can be mocked
- **Type Safety**: sqlc ensures query/struct compatibility
- **Transaction Support**: Can wrap multiple queries in transactions

### Transaction Handling

```go
func (s *Service) CreateSessionWithInitialMessage(ctx context.Context, session Session, message Message) error {
    return s.db.WithTx(ctx, func(tx *sql.Tx) error {
        queries := s.queries.WithTx(tx)

        if err := queries.CreateSession(ctx, sessionParams); err != nil {
            return err
        }

        return queries.CreateMessage(ctx, messageParams)
    })
}
```

---

## 🔍 Advanced Database Features

### 1. **JSON Storage and Querying**

Some fields store JSON for flexibility:

```sql
-- Store structured data as JSON
INSERT INTO messages (content) VALUES (json('{"text": "Hello", "attachments": [...]}'));

-- Query JSON fields
SELECT * FROM messages WHERE json_extract(content, '$.attachments') IS NOT NULL;
```

### 2. **Full-Text Search** (Future Enhancement)

```sql
-- Virtual table for full-text search
CREATE VIRTUAL TABLE messages_fts USING fts5(content, session_id);

-- Trigger to keep FTS table updated
CREATE TRIGGER messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(content, session_id) VALUES (new.content, new.session_id);
END;
```

### 3. **Data Retention Policies**

```sql
-- Clean up old sessions (could be a background job)
DELETE FROM sessions
WHERE created_at < datetime('now', '-30 days')
  AND id NOT IN (SELECT DISTINCT session_id FROM messages WHERE created_at > datetime('now', '-7 days'));
```

---

## 🎯 Common Database Tasks

### Adding a New Table

1. **Create Migration**:
```bash
# Create new migration file
touch internal/db/migrations/$(date +%Y%m%d%H%M%S)_add_user_preferences.sql
```

2. **Define Schema**:
```sql
CREATE TABLE user_preferences (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, key)
);
```

3. **Create Query File**:
```sql
-- internal/db/sql/preferences.sql

-- name: CreatePreference :exec
INSERT INTO user_preferences (id, user_id, key, value)
VALUES ($1, $2, $3, $4);

-- name: GetPreference :one
SELECT * FROM user_preferences
WHERE user_id = $1 AND key = $2;

-- name: UpdatePreference :exec
UPDATE user_preferences
SET value = $1, updated_at = CURRENT_TIMESTAMP
WHERE user_id = $2 AND key = $3;
```

4. **Generate Code**:
```bash
sqlc generate
```

5. **Create Service**:
```go
type PreferencesService struct {
    queries *db.Queries
}

func (s *PreferencesService) Set(ctx context.Context, userID, key, value string) error {
    return s.queries.CreatePreference(ctx, db.CreatePreferenceParams{
        ID:     uuid.New().String(),
        UserID: userID,
        Key:    key,
        Value:  value,
    })
}
```

---

## 🚨 Common Pitfalls and Solutions

### 1. **Migration Ordering**
❌ **Problem**: Migrations run out of order on different environments
✅ **Solution**: Use timestamp-based naming, never edit existing migrations

### 2. **Query Performance**
❌ **Problem**: N+1 queries when loading related data
✅ **Solution**: Use JOINs or batch loading patterns

### 3. **Transaction Scope**
❌ **Problem**: Long-running transactions causing locks
✅ **Solution**: Keep transactions short, defer non-critical operations

### 4. **Schema Changes**
❌ **Problem**: Breaking changes in production
✅ **Solution**: Additive migrations, feature flags for new columns

---

## 🎯 What's Next

In the next intermediate tutorial, we'll explore **AI Integration Deep Dive** where we'll:
- Understand the provider abstraction layer
- Learn about agent orchestration and tool management
- Master prompt engineering and context building
- See how multiple AI models are coordinated

---

## 📝 Hands-On Exercises

1. **Query Analysis**: Write a complex query to find sessions with the most user messages
2. **Migration Practice**: Design a migration to add user authentication
3. **Performance Testing**: Use EXPLAIN QUERY PLAN to optimize a slow query
4. **Service Implementation**: Create a new service for managing user preferences

---

**Previous**: [Beginner Level Tutorials](../beginner/)
**Next**: [Intermediate 02: AI Integration Deep Dive](02-ai-integration.md)

---

*💡 **Expert Tip**: The database layer is the foundation of any serious application. Master this, and you'll understand how to build scalable, maintainable data systems in Go.*
