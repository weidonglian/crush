# Tutorial 2: Code Structure Tour

**Duration**: 20-25 minutes
**Goal**: Navigate the codebase and understand the organization

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand the Go project layout used by Crush
- Know what each major package does
- Recognize common Go patterns and conventions
- Be able to navigate the codebase confidently

---

## 🗺️ The Big Picture

Crush follows the **Standard Go Project Layout** with a focus on clarity and modularity.

```
crush/
├── main.go              # Entry point
├── go.mod              # Go module definition
├── internal/           # Private application code
├── scripts/            # Build and utility scripts
├── learning/           # Our tutorial materials
└── config files        # Various config files
```

### Why `internal/`?

In Go, the `internal/` directory has special meaning:
- Code in `internal/` cannot be imported by external packages
- This enforces encapsulation and API boundaries
- Everything in here is "private" to the Crush application

---

## 🏗️ Architecture Overview

Let's explore the `internal/` directory structure:

```bash
# Take a look at the internal structure
tree internal/ -L 2 -d
```

You'll see something like:

```
internal/
├── ansiext/           # ANSI terminal extensions
├── app/               # Core application orchestration
├── cmd/               # CLI command definitions
├── config/            # Configuration management
├── csync/             # Concurrency synchronization utilities
├── db/                # Database layer and models
├── diff/              # Text diffing utilities
├── env/               # Environment variable handling
├── format/            # Output formatting (spinners, etc.)
├── fsext/             # File system extensions
├── history/           # File and command history
├── home/              # Home directory utilities
├── llm/               # Large Language Model integration
├── log/               # Logging utilities
├── lsp/               # Language Server Protocol
├── message/           # Message handling and storage
├── permission/        # Permission and security
├── pubsub/            # Event publishing and subscription
├── session/           # Session management
├── shell/             # Shell command execution
├── slicesext/         # Slice utility extensions
├── tui/               # Terminal User Interface
└── version/           # Version information
```

---

## 📦 Package-by-Package Breakdown

### 🏛️ **Core Foundation Packages**

#### `cmd/` - Command Line Interface
```bash
ls internal/cmd/
# Output: logs.go  root.go  run.go  schema.go
```

**Purpose**: Defines all CLI commands and their behavior
- `root.go` - Main command definition and flags
- `run.go` - Interactive mode command
- `logs.go` - Log viewing command
- `schema.go` - Configuration schema command

#### `app/` - Application Core
```bash
ls internal/app/
# Output: app.go  lsp.go  lsp_events.go
```

**Purpose**: Central application orchestration
- `app.go` - Main application struct and lifecycle
- `lsp.go` - LSP client management
- `lsp_events.go` - LSP event handling

#### `config/` - Configuration Management
```bash
ls internal/config/
# Output: config.go  init.go  load.go  merge.go  provider.go  resolve.go
# Plus test files: *_test.go
```

**Purpose**: Handles all configuration loading and merging
- Hierarchical configuration (project → global → defaults)
- JSON schema validation
- Environment variable integration

### 🤖 **AI/LLM Integration**

#### `llm/` - Large Language Model Integration
```bash
tree internal/llm/ -L 2 -d
# Output:
# internal/llm/
# ├── agent/      # AI agent orchestration
# ├── prompt/     # Prompt management and templates
# ├── provider/   # Multiple LLM provider implementations
# └── tools/      # AI tools and capabilities
```

**This is a BIG package!** Let's break it down:

- **`agent/`**: Orchestrates AI conversations and tool usage
- **`prompt/`**: Manages prompt templates and context building
- **`provider/`**: Implementations for different AI services (OpenAI, Anthropic, etc.)
- **`tools/`**: Extensible tool system for AI capabilities

### 🗄️ **Data Management**

#### `db/` - Database Layer
```bash
ls internal/db/
# Output: connect.go  db.go  embed.go  models.go  querier.go
#         files.sql.go  messages.sql.go  sessions.sql.go
#         migrations/  sql/
```

**Purpose**: SQLite database management with type-safe queries
- Uses `sqlc` for generating type-safe Go code from SQL
- Migration system for schema evolution
- Separate query files for different domains

#### `message/` - Message Management
```bash
ls internal/message/
# Output: attachment.go  content.go  message.go
```

**Purpose**: Handles chat messages and attachments
- Message structure and validation
- Attachment handling (files, images, etc.)
- Content processing and formatting

#### `session/` - Session Management
```bash
ls internal/session/
# Output: session.go
```

**Purpose**: Manages conversation sessions
- Session lifecycle and persistence
- Context preservation across conversations
- Session switching and management

### 🎨 **User Interface**

#### `tui/` - Terminal User Interface
```bash
tree internal/tui/ -L 2 -d
# Output:
# internal/tui/
# ├── components/    # Reusable UI components
# ├── exp/          # Experimental UI features
# ├── highlight/    # Syntax highlighting
# ├── page/         # Page-based navigation
# ├── styles/       # Styling and themes
# └── util/         # UI utilities
```

**Purpose**: Beautiful terminal interface using Bubble Tea
- Component-based architecture
- Consistent styling and theming
- Keyboard-driven interaction

### 🔗 **Integration Layers**

#### `lsp/` - Language Server Protocol
```bash
tree internal/lsp/ -L 2 -d
# Output:
# internal/lsp/
# ├── protocol/     # LSP protocol implementation
# ├── util/         # LSP utilities
# └── watcher/      # File watching for LSP
```

**Purpose**: Integrates with language servers for code understanding
- Protocol-compliant LSP client
- Multi-language support
- Real-time code analysis

#### `permission/` - Permission System
```bash
ls internal/permission/
# Output: permission.go  permission_test.go
```

**Purpose**: Manages user permissions and security
- Tool execution permissions
- User confirmation workflows
- Security boundaries

### 🛠️ **Utility Packages**

#### `fsext/` - File System Extensions
```bash
ls internal/fsext/
# Output: expand.go  fileutil.go  ignore.go  ls.go  owner_*.go  parent.go
#         *_test.go
```

**Purpose**: Enhanced file system operations
- File globbing and filtering
- Ignore pattern handling (like .gitignore)
- Cross-platform file utilities

#### `shell/` - Shell Integration
```bash
ls internal/shell/
# Output: command_block.go  comparison.go  doc.go  persistent.go  shell.go
#         *_test.go
```

**Purpose**: Execute and manage shell commands
- Command parsing and execution
- Output capturing and processing
- Shell session management

---

## 🎯 Key Architecture Patterns

### 1. **Service-Oriented Architecture**
Each major feature is encapsulated in its own service:
```go
type App struct {
    Sessions    session.Service     // Session management
    Messages    message.Service     // Message handling
    History     history.Service     // File history
    Permissions permission.Service  // Permission handling
    CoderAgent  agent.Service      // AI agent
    // ...
}
```

### 2. **Interface-Based Design**
Services are defined by interfaces, making them testable and swappable:
```go
type Service interface {
    Create(ctx context.Context, session Session) error
    Get(ctx context.Context, id string) (Session, error)
    // ...
}
```

### 3. **Event-Driven Communication**
Components communicate through events and pub/sub:
```go
// From pubsub package
type Broker interface {
    Subscribe(topic string) <-chan Event
    Publish(topic string, event Event)
}
```

### 4. **Configuration-Driven Behavior**
Almost everything can be configured through JSON config files:
```json
{
  "lsp": {
    "go": {"command": "gopls"},
    "typescript": {"command": "typescript-language-server"}
  },
  "providers": {
    "openai": {"model": "gpt-4"}
  }
}
```

---

## 🔍 Code Quality Observations

### ✅ **What's Done Well**

1. **Clear Package Purpose**: Each package has a focused responsibility
2. **Consistent Naming**: Follows Go conventions throughout
3. **Test Coverage**: Most packages have corresponding test files
4. **Documentation**: Good inline documentation and examples
5. **Error Handling**: Consistent error wrapping and structured logging

### 📚 **Go Conventions Used**

1. **Package Organization**: Standard Go project layout
2. **Interface Definitions**: Small, focused interfaces
3. **Error Handling**: Explicit error returns and wrapping
4. **Context Usage**: Proper context propagation for cancellation
5. **Dependency Injection**: Services injected through constructors

---

## 🤔 Questions to Explore

As you navigate the code, think about:

1. **Why separate `agent` from `provider`?** What's the difference in responsibility?

2. **How does the TUI stay responsive?** What patterns enable smooth interaction?

3. **Why use SQLite instead of just files?** What benefits does this provide?

4. **How are different language servers managed?** What happens when you have multiple languages in one project?

---

## 🎯 What's Next

In the next tutorial, we'll dive into **Entry Points and Bootstrap** where we'll:
- Follow the application startup sequence
- Understand dependency initialization
- See how configuration is loaded
- Learn about the application lifecycle

---

## 📝 Try This Yourself

Before moving to the next tutorial:

1. **Navigate the packages**: Use your editor to browse through each major package
2. **Read package documentation**: Look at the `doc.go` files where they exist
3. **Identify patterns**: Notice the consistent structure across packages
4. **Find interfaces**: Look for interface definitions in each service package

**Bonus**: Pick one utility package (like `fsext` or `slicesext`) and read through it completely. These smaller packages are great for understanding Go patterns!

---

**Previous**: [Tutorial 1: First Look at Crush](01-first-look.md)
**Next**: [Tutorial 3: Entry Points and Bootstrap](03-entry-points.md)

---

*💡 **Tip**: Use your editor's "Go to Definition" feature to jump between related files. This is a great way to understand how packages interact!*
