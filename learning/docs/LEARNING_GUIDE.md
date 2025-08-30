# Crush Codebase Learning Guide

## Overview

**Crush** is a terminal-based AI assistant for software development built with Go. It provides an interactive chat interface with AI capabilities, code analysis, and Language Server Protocol (LSP) integration to assist developers directly from the terminal.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [Learning Plan](#learning-plan)
4. [Core Components Deep Dive](#core-components-deep-dive)
5. [Key Design Patterns](#key-design-patterns)
6. [Data Flow](#data-flow)
7. [External Integrations](#external-integrations)
8. [Development Workflow](#development-workflow)

---

## Project Overview

### What is Crush?
- **Purpose**: Terminal-based AI coding assistant that integrates with multiple LLM providers
- **Key Features**:
  - Multi-model LLM support (OpenAI, Anthropic, Groq, etc.)
  - Session-based conversations with context preservation
  - LSP integration for enhanced code understanding
  - Extensible via Model Context Protocol (MCP)
  - Cross-platform terminal UI using Bubble Tea

### Technology Stack
- **Language**: Go 1.25+
- **UI Framework**: [Bubble Tea](https://github.com/charmbracelet/bubbletea) (Terminal UI)
- **Database**: SQLite with [go-sqlite3](https://github.com/ncruces/go-sqlite3)
- **CLI Framework**: [Cobra](https://github.com/spf13/cobra)
- **Styling**: [Lipgloss](https://github.com/charmbracelet/lipgloss)
- **Text Processing**: Various markdown and syntax highlighting libraries

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         Main Entry                          │
│                      (main.go)                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    CLI Layer                                │
│              (internal/cmd/)                               │
│  ┌─────────────┬─────────────┬─────────────┬──────────────┐ │
│  │ root.go     │ run.go      │ logs.go     │ schema.go    │ │
│  └─────────────┴─────────────┴─────────────┴──────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                Application Layer                            │
│                (internal/app/)                             │
│  ┌─────────────────┬─────────────────┬─────────────────────┐ │
│  │ app.go          │ lsp.go          │ lsp_events.go       │ │
│  │ (Core App)      │ (LSP Manager)   │ (Event Handling)    │ │
│  └─────────────────┴─────────────────┴─────────────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   Service Layer                             │
│  ┌──────────────┬──────────────┬──────────────┬───────────┐ │
│  │ Sessions     │ Messages     │ History      │ LSP       │ │
│  │ (session/)   │ (message/)   │ (history/)   │ (lsp/)    │ │
│  └──────────────┼──────────────┼──────────────┼───────────┘ │
│  ┌──────────────┼──────────────┼──────────────┼───────────┐ │
│  │ Permissions  │ LLM Agents   │ TUI          │ Config    │ │
│  │ (permission/)│ (llm/)       │ (tui/)       │ (config/) │ │
│  └──────────────┴──────────────┴──────────────┴───────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                  Data Layer                                 │
│  ┌─────────────────┬─────────────────┬─────────────────────┐ │
│  │ SQLite Database │ File System     │ External APIs       │ │
│  │ (db/)           │ (fsext/)        │ (llm/provider/)     │ │
│  └─────────────────┴─────────────────┴─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Learning Plan

### Phase 1: Foundation (Week 1)
**Goal**: Understand the basic structure and entry points

1. **Entry Point Analysis**
   - [ ] Study `main.go` - application bootstrap
   - [ ] Examine `internal/cmd/root.go` - CLI structure
   - [ ] Understand command parsing and flag handling

2. **Configuration System**
   - [ ] Read `internal/config/` - how configuration is loaded and merged
   - [ ] Study `crush.json` schema and configuration priority
   - [ ] Understand environment variable handling

3. **Database Layer**
   - [ ] Explore `internal/db/` - SQLite integration
   - [ ] Study migration system in `internal/db/migrations/`
   - [ ] Understand the query layer using sqlc

### Phase 2: Core Services (Week 2)
**Goal**: Understand the main business logic components

1. **Application Core**
   - [ ] Deep dive into `internal/app/app.go` - main application struct
   - [ ] Study service initialization and dependency injection
   - [ ] Understand the event system and pub/sub pattern

2. **Session Management**
   - [ ] Study `internal/session/` - how conversations are managed
   - [ ] Understand session persistence and context preservation
   - [ ] Learn about session switching and state management

3. **Message System**
   - [ ] Explore `internal/message/` - message structure and handling
   - [ ] Study attachments and content processing
   - [ ] Understand message persistence and retrieval

### Phase 3: AI Integration (Week 3)
**Goal**: Understand how AI/LLM integration works

1. **LLM Provider System**
   - [ ] Study `internal/llm/provider/` - multiple LLM backends
   - [ ] Understand provider abstraction and configuration
   - [ ] Examine API integration patterns

2. **Agent System**
   - [ ] Explore `internal/llm/agent/` - AI agent orchestration
   - [ ] Study tool integration and MCP support
   - [ ] Understand prompt engineering and templates

3. **Prompt Management**
   - [ ] Study `internal/llm/prompt/` - prompt templates and generation
   - [ ] Understand context building and summarization
   - [ ] Learn about different prompt strategies

### Phase 4: UI and Interaction (Week 4)
**Goal**: Understand the terminal user interface

1. **TUI Framework**
   - [ ] Study `internal/tui/` - Bubble Tea integration
   - [ ] Understand component architecture
   - [ ] Learn about state management and updates

2. **Terminal Components**
   - [ ] Explore `internal/tui/components/` - reusable UI components
   - [ ] Study styling with Lipgloss
   - [ ] Understand input handling and key bindings

3. **Event System**
   - [ ] Study the pub/sub event system
   - [ ] Understand message passing between components
   - [ ] Learn about asynchronous updates

### Phase 5: Advanced Features (Week 5)
**Goal**: Understand advanced integrations and features

1. **LSP Integration**
   - [ ] Deep dive into `internal/lsp/` - Language Server Protocol
   - [ ] Understand how code context is gathered
   - [ ] Study LSP client management and communication

2. **File System Integration**
   - [ ] Study `internal/fsext/` - file system utilities
   - [ ] Understand file watching and change detection
   - [ ] Learn about ignore patterns and filtering

3. **Shell Integration**
   - [ ] Explore `internal/shell/` - shell command execution
   - [ ] Understand command parsing and execution
   - [ ] Study output capturing and processing

### Phase 6: Testing and Deployment (Week 6)
**Goal**: Understand testing patterns and deployment

1. **Testing Strategy**
   - [ ] Study test files throughout the codebase
   - [ ] Understand mocking and dependency injection for tests
   - [ ] Learn about integration vs unit testing approaches

2. **Build and Release**
   - [ ] Study `Taskfile.yaml` - build automation
   - [ ] Understand Go module management
   - [ ] Learn about cross-platform compilation

---

## Core Components Deep Dive

### 1. Application Structure (`internal/app/`)

The `App` struct is the heart of the application:

```go
type App struct {
    Sessions    session.Service
    Messages    message.Service
    History     history.Service
    Permissions permission.Service
    CoderAgent  agent.Service
    LSPClients  map[string]*lsp.Client
    // ... other fields
}
```

**Key Responsibilities**:
- Service coordination and dependency injection
- LSP client management
- Event system orchestration
- Global context and lifecycle management

### 2. Service Pattern

The codebase follows a service-oriented pattern where each major feature is encapsulated in a service:

- **Session Service**: Manages conversation sessions
- **Message Service**: Handles message CRUD operations
- **History Service**: Manages file and command history
- **Permission Service**: Handles user permission requests
- **Agent Service**: Orchestrates AI interactions

### 3. Database Layer (`internal/db/`)

Uses SQLite with sqlc for type-safe queries:
- **Models**: Generated structs from SQL schema
- **Queries**: Type-safe query methods
- **Migrations**: Database schema evolution

### 4. Configuration System (`internal/config/`)

Hierarchical configuration with multiple sources:
1. Project-local `.crush.json`
2. Global `~/.config/crush/crush.json`
3. Environment variables
4. Default values

---

## Key Design Patterns

### 1. Service Locator Pattern
The `App` struct acts as a service locator, providing access to all major services.

### 2. Observer Pattern
Event-driven architecture using channels and pub/sub for component communication.

### 3. Strategy Pattern
Multiple LLM providers implementing a common interface.

### 4. Builder Pattern
Configuration building with merging and validation.

### 5. Repository Pattern
Database access abstracted through service interfaces.

---

## Data Flow

### 1. User Input Flow
```
Terminal Input → TUI Component → Event Channel → App Handler → Service Layer → Database/API
```

### 2. AI Response Flow
```
User Message → Agent Service → LLM Provider → Response Processing → Message Service → TUI Update
```

### 3. LSP Integration Flow
```
File Change → LSP Client → Context Gathering → Agent Service → Enhanced AI Response
```

---

## External Integrations

### 1. LLM Providers
- OpenAI GPT models
- Anthropic Claude
- Google Gemini
- Groq
- Azure OpenAI
- AWS Bedrock

### 2. Language Servers
- Support for any LSP-compliant language server
- Dynamic client management
- Protocol abstraction

### 3. Model Context Protocol (MCP)
- Extensible tool system
- HTTP, stdio, and SSE support
- Community plugin ecosystem

---

## Development Workflow

### 1. Key Files to Modify for Features

**Adding a new LLM provider**:
- `internal/llm/provider/` - New provider implementation
- `internal/config/` - Configuration support
- `internal/llm/agent/` - Agent integration

**Adding UI components**:
- `internal/tui/components/` - New component
- `internal/tui/` - Integration with main TUI
- `internal/tui/styles/` - Styling

**Adding database features**:
- `internal/db/sql/` - New queries
- `internal/db/migrations/` - Schema changes
- Service layer updates

### 2. Testing Approach
- Unit tests for individual components
- Integration tests for service interactions
- Manual testing with different LLM providers

### 3. Common Development Tasks
- Adding new commands: `internal/cmd/`
- Modifying AI behavior: `internal/llm/prompt/`
- UI changes: `internal/tui/`
- Configuration updates: `internal/config/`

---

## Next Steps

Once you've completed this learning plan, you'll have a comprehensive understanding of the Crush codebase. Key areas for contribution might include:

1. **New LLM Provider Integration**
2. **Enhanced TUI Components**
3. **Additional LSP Features**
4. **Performance Optimizations**
5. **Testing Coverage Improvements**

---

## Resources

- [Bubble Tea Documentation](https://github.com/charmbracelet/bubbletea)
- [Cobra CLI Documentation](https://github.com/spf13/cobra)
- [LSP Specification](https://microsoft.github.io/language-server-protocol/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Go SQLite Driver](https://github.com/ncruces/go-sqlite3)

---

*This guide will be updated as we discover new patterns and architectural decisions in the codebase.*
