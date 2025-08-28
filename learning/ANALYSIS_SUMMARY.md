# Crush Codebase Analysis Summary

## Initial Analysis Overview

This document captures the comprehensive analysis performed on the Crush codebase on August 28, 2025. This analysis forms the foundation for understanding the architecture, patterns, and learning approach for this sophisticated Go-based terminal AI assistant.

---

## Project Discovery

### What is Crush?

**Crush** is a terminal-based AI assistant for software development, built by Charm. It's designed to be your "coding bestie" that integrates with multiple Large Language Models (LLMs) and provides enhanced context through Language Server Protocol (LSP) integration.

### Key Characteristics Discovered

1. **Multi-Provider AI Support**: Supports OpenAI, Anthropic, Google Gemini, Groq, Azure OpenAI, AWS Bedrock, and more
2. **Session-Based Conversations**: Maintains context across multiple work sessions per project
3. **LSP-Enhanced Context**: Uses Language Server Protocols for deeper code understanding
4. **Extensible Architecture**: Supports Model Context Protocol (MCP) for adding capabilities
5. **Cross-Platform Terminal UI**: Built with Bubble Tea for consistent experience across platforms

---

## Architectural Analysis

### Core Technology Stack

```
Programming Language: Go 1.25+
├── UI Framework: Bubble Tea (Terminal UI)
├── CLI Framework: Cobra (Command-line interface)
├── Database: SQLite with go-sqlite3
├── Styling: Lipgloss (Terminal styling)
├── Query Generation: sqlc (Type-safe SQL)
├── Configuration: JSON-based hierarchical config
└── External APIs: Multiple LLM provider SDKs
```

### System Architecture

The application follows a layered architecture pattern:

```
┌─── Entry Point (main.go)
│    └── Profiling support (pprof)
│    └── Environment loading (.env)
│
├─── CLI Layer (internal/cmd/)
│    ├── root.go - Main command structure
│    ├── run.go - Interactive mode
│    ├── logs.go - Log management
│    └── schema.go - Configuration schema
│
├─── Application Layer (internal/app/)
│    ├── app.go - Core application orchestration
│    ├── lsp.go - LSP client management
│    └── lsp_events.go - LSP event handling
│
├─── Service Layer
│    ├── session/ - Conversation session management
│    ├── message/ - Message CRUD and processing
│    ├── history/ - File and command history
│    ├── permission/ - User permission handling
│    ├── llm/ - AI agent and provider management
│    ├── lsp/ - Language Server Protocol integration
│    ├── tui/ - Terminal user interface
│    └── config/ - Configuration management
│
└─── Data Layer
     ├── db/ - SQLite database with migrations
     ├── fsext/ - File system utilities
     └── External APIs - LLM providers
```

---

## Key Design Patterns Identified

### 1. Service Locator Pattern
The main `App` struct acts as a central service registry:

```go
type App struct {
    Sessions    session.Service
    Messages    message.Service
    History     history.Service
    Permissions permission.Service
    CoderAgent  agent.Service
    LSPClients  map[string]*lsp.Client
    // ... coordination and lifecycle management
}
```

### 2. Strategy Pattern for LLM Providers
Multiple AI providers implement a common interface, allowing runtime switching between different LLMs while preserving conversation context.

### 3. Observer/Event-Driven Pattern
Uses Bubble Tea's event system enhanced with custom pub/sub patterns for component communication.

### 4. Repository Pattern
Database access is abstracted through service interfaces, with sqlc providing type-safe query generation.

### 5. Builder Pattern for Configuration
Hierarchical configuration system that merges settings from multiple sources with validation.

---

## Core Components Deep Dive

### 1. Entry Point Analysis (`main.go`)

```go
func main() {
    defer log.RecoverPanic("main", func() {
        slog.Error("Application terminated due to unhandled panic")
    })

    // Optional profiling support
    if os.Getenv("CRUSH_PROFILE") != "" {
        go func() {
            slog.Info("Serving pprof at localhost:6060")
            if httpErr := http.ListenAndServe("localhost:6060", nil); httpErr != nil {
                slog.Error("Failed to pprof listen", "error", httpErr)
            }
        }()
    }

    cmd.Execute()
}
```

**Key Insights**:
- Includes panic recovery with structured logging
- Optional profiling support for performance analysis
- Clean separation between bootstrap and application logic
- Automatic .env file loading via godotenv

### 2. Configuration System (`internal/config/`)

The configuration system follows a priority hierarchy:
1. `.crush.json` (project-local)
2. `crush.json` (project-local alternative)
3. `$HOME/.config/crush/crush.json` (global user config)
4. Environment variables
5. Default values

**Features**:
- JSON Schema validation
- Hierarchical merging
- Environment variable integration
- LSP configuration support
- Provider-specific settings

### 3. Database Layer (`internal/db/`)

Uses SQLite with modern Go patterns:
- **sqlc integration**: Type-safe query generation
- **Migration system**: Versioned schema evolution
- **Embedded queries**: SQL files embedded in binary
- **Connection management**: Proper connection lifecycle

**Schema Overview**:
- `sessions` - Conversation sessions
- `messages` - Individual messages with attachments
- `files` - File history and tracking
- Indexes for performance optimization

### 4. LLM Integration (`internal/llm/`)

Multi-layered AI integration:

```
Agent Layer (agent/)
├── Tool orchestration
├── MCP integration
└── Conversation management

Provider Layer (provider/)
├── OpenAI integration
├── Anthropic Claude
├── Google Gemini
├── Groq and others
└── Provider abstraction

Prompt Layer (prompt/)
├── Template management
├── Context building
├── Summarization
└── Task-specific prompts
```

### 5. LSP Integration (`internal/lsp/`)

Sophisticated Language Server Protocol integration:
- **Dynamic client management**: Spawns LSP clients per language
- **Protocol abstraction**: Clean interface over LSP complexity
- **Context gathering**: Extracts relevant code context for AI
- **File watching**: Monitors changes for real-time updates

### 6. Terminal UI (`internal/tui/`)

Built on Bubble Tea with component architecture:
- **Component system**: Reusable UI components
- **State management**: Proper state flow and updates
- **Styling**: Consistent theming with Lipgloss
- **Event handling**: Keyboard and terminal event processing

---

## Data Flow Analysis

### 1. User Interaction Flow

```
User Input (Terminal)
    ↓
TUI Component (Bubble Tea)
    ↓
Event Channel (Message passing)
    ↓
App Handler (Central coordination)
    ↓
Service Layer (Business logic)
    ↓
Database/External APIs (Persistence/AI)
```

### 2. AI Response Generation

```
User Message
    ↓
Agent Service (Context preparation)
    ↓
LLM Provider (AI processing)
    ↓
Response Processing (Formatting/validation)
    ↓
Message Service (Persistence)
    ↓
TUI Update (Display to user)
```

### 3. LSP Context Integration

```
File Change Detection
    ↓
LSP Client (Language-specific analysis)
    ↓
Context Gathering (Relevant code extraction)
    ↓
Agent Service (Enhanced prompt context)
    ↓
Improved AI Response (Context-aware assistance)
```

---

## External Integration Points

### 1. LLM Providers Supported

| Provider | Environment Variable | Features |
|----------|---------------------|----------|
| OpenAI | `OPENAI_API_KEY` | GPT models, function calling |
| Anthropic | `ANTHROPIC_API_KEY` | Claude models, tool use |
| Google Gemini | `GEMINI_API_KEY` | Gemini models |
| Groq | `GROQ_API_KEY` | Fast inference |
| Azure OpenAI | `AZURE_OPENAI_*` | Enterprise integration |
| AWS Bedrock | `AWS_*` | Claude on AWS |

### 2. Language Server Support

Configurable LSP integration for any language:

```json
{
  "lsp": {
    "go": {
      "command": "gopls",
      "env": { "GOTOOLCHAIN": "go1.24.5" }
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"]
    }
  }
}
```

### 3. Model Context Protocol (MCP)

Extensible tool system supporting:
- HTTP-based tools
- stdio communication
- Server-Sent Events (SSE)
- Community plugin ecosystem

---

## Development Insights

### Code Quality Patterns

1. **Error Handling**: Consistent error wrapping and structured logging
2. **Concurrency**: Proper goroutine management with context cancellation
3. **Testing**: Mix of unit and integration tests
4. **Documentation**: Good inline documentation and examples
5. **Modularity**: Clear separation of concerns between packages

### Build and Deployment

- **Task Runner**: Uses Taskfile.yaml for build automation
- **Cross-Platform**: Supports multiple operating systems
- **Package Management**: Go modules with dependency management
- **Release Process**: Automated builds and package distribution

### Notable Dependencies

```go
// Core dependencies from go.mod analysis
github.com/charmbracelet/bubbletea/v2  // Terminal UI framework
github.com/charmbracelet/lipgloss/v2   // Terminal styling
github.com/spf13/cobra                 // CLI framework
github.com/ncruces/go-sqlite3          // SQLite driver
github.com/anthropics/anthropic-sdk-go // Anthropic API
github.com/openai/openai-go           // OpenAI API
github.com/mark3labs/mcp-go           // Model Context Protocol
```

---

## Learning Strategy Recommendations

Based on the analysis, the recommended learning approach is:

### Phase 1: Foundation (Week 1)
- Start with `main.go` and `internal/cmd/`
- Understand configuration loading and CLI structure
- Explore database setup and migration system

### Phase 2: Core Services (Week 2)
- Deep dive into `internal/app/app.go`
- Study service initialization and coordination
- Understand session and message management

### Phase 3: AI Integration (Week 3)
- Explore `internal/llm/` thoroughly
- Understand provider abstraction and agent system
- Study prompt engineering and context building

### Phase 4: User Interface (Week 4)
- Study Bubble Tea integration in `internal/tui/`
- Understand component architecture and styling
- Learn event handling and state management

### Phase 5: Advanced Features (Week 5)
- Deep dive into LSP integration
- Study file system utilities and shell integration
- Understand permission system and security

### Phase 6: Contribution Readiness (Week 6)
- Study testing patterns and practices
- Understand build and release processes
- Identify contribution opportunities

---

## Potential Contribution Areas

Based on the codebase analysis, potential areas for contribution include:

1. **New LLM Provider Integration** - Adding support for new AI services
2. **Enhanced TUI Components** - Improving user interface elements
3. **Additional LSP Features** - Expanding language server capabilities
4. **Performance Optimizations** - Database queries, memory usage, startup time
5. **Testing Coverage** - Adding tests for uncovered areas
6. **Documentation** - API documentation, user guides, developer docs
7. **Cross-Platform Features** - Platform-specific integrations
8. **Security Enhancements** - Permission system improvements

---

## Architecture Strengths

1. **Modularity**: Clean separation of concerns across packages
2. **Extensibility**: Plugin system via MCP and configurable providers
3. **Maintainability**: Clear interfaces and dependency injection
4. **Performance**: Efficient SQLite usage and goroutine management
5. **User Experience**: Polished terminal UI with consistent styling
6. **Flexibility**: Support for multiple LLMs and development workflows

---

## Questions for Future Investigation

1. How does the session context preservation work across restarts?
2. What are the performance characteristics under heavy usage?
3. How does the LSP integration handle multiple language servers simultaneously?
4. What security measures are in place for API key management?
5. How does the MCP tool system handle sandboxing and permissions?
6. What are the scaling limits for large codebases and long conversations?

---

*This analysis provides the foundation for deep learning of the Crush codebase. Use this in conjunction with the Learning Guide for a structured approach to understanding this sophisticated terminal AI assistant.*
