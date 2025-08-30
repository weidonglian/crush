# Tutorial 1: First Look at Crush

**Duration**: 15-20 minutes
**Goal**: Understand what Crush is and see it in action

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand what Crush does
- Know the key features and capabilities
- See Crush in action
- Understand the basic user workflow

---

## 📖 What is Crush?

Crush is a **terminal-based AI assistant** for software development. Think of it as your coding buddy that:

- Lives in your terminal
- Talks to multiple AI models (OpenAI, Anthropic, etc.)
- Understands your code through Language Servers
- Maintains conversation history and context
- Can be extended with tools and plugins

### 🌟 Key Features

1. **Multi-Model Support**
   - OpenAI GPT models
   - Anthropic Claude
   - Google Gemini
   - Groq, Azure OpenAI, AWS Bedrock

2. **Smart Context**
   - Uses Language Server Protocol (LSP)
   - Understands your project structure
   - Maintains conversation sessions

3. **Terminal Native**
   - Beautiful terminal UI
   - Works everywhere (macOS, Linux, Windows)
   - Keyboard-driven workflow

4. **Extensible**
   - Model Context Protocol (MCP) support
   - Custom tools and integrations

---

## 🚀 Let's See It in Action

### Step 1: Basic Overview

Open the Crush repository and look at the README:

```bash
# If you haven't already, clone or examine the README.md
cat README.md | head -20
```

**What to notice**:
- It's built by Charm (known for beautiful terminal tools)
- Supports many installation methods
- Has a clean, friendly presentation

### Step 2: Check the Entry Point

Look at the main application entry:

```bash
# Examine the main.go file
cat main.go
```

**Key observations**:
```go
package main

import (
    "log/slog"
    "net/http"
    "os"
    _ "net/http/pprof" // profiling support
    _ "github.com/joho/godotenv/autoload" // auto-load .env
    "github.com/charmbracelet/crush/internal/cmd"
    "github.com/charmbracelet/crush/internal/log"
)

func main() {
    defer log.RecoverPanic("main", func() {
        slog.Error("Application terminated due to unhandled panic")
    })

    // Optional profiling
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

**What this tells us**:
- Clean, simple entry point
- Built-in panic recovery
- Optional profiling for performance analysis
- Automatic environment loading
- Delegates to CLI command system

### Step 3: Examine the Project Structure

```bash
# Look at the high-level structure
tree -L 2 -d
```

**Key directories**:
```
.
├── internal/           # All Go application code
│   ├── app/           # Core application logic
│   ├── cmd/           # CLI commands
│   ├── config/        # Configuration management
│   ├── db/            # Database layer
│   ├── llm/           # AI/LLM integration
│   ├── lsp/           # Language Server Protocol
│   ├── tui/           # Terminal User Interface
│   └── ...            # Many other specialized packages
├── learning/          # Our tutorial materials
└── scripts/           # Build and utility scripts
```

---

## 🔍 Understanding the User Experience

### How Someone Uses Crush

1. **Installation**: User installs via package manager or Go
2. **Configuration**: Sets up API keys for AI providers
3. **Launch**: Runs `crush` in their project directory
4. **Interaction**: Chats with AI about their code
5. **Context**: AI understands the project via LSP integration

### Example User Workflow

```bash
# User starts crush in their Go project
cd /my-go-project
crush

# Crush starts up and shows:
# - Beautiful terminal interface
# - Configuration prompts (if needed)
# - Chat interface ready for questions

# User can ask:
# "Explain this function"
# "Help me optimize this code"
# "Write tests for this package"
# "What does this error mean?"
```

---

## 💡 Key Insights from This First Look

### 1. Professional Quality
- Clean code structure
- Proper error handling
- Performance considerations (profiling)
- Production-ready patterns

### 2. Developer-Friendly
- Well-organized packages
- Clear separation of concerns
- Standard Go project layout
- Good documentation

### 3. Sophisticated Features
- Multiple AI provider support
- Language server integration
- Session management
- Terminal UI framework

### 4. Extensible Design
- Modular architecture
- Plugin system (MCP)
- Configurable everything
- Clean interfaces

---

## 🤔 Questions to Think About

1. **Why terminal-based?** What advantages does this have over a web or desktop app?

2. **Multiple AI providers?** Why support so many instead of just one?

3. **LSP integration?** How does this make the AI more helpful?

4. **Session management?** Why is conversation history important?

---

## 🎯 What's Next

In the next tutorial, we'll take a **Code Structure Tour** where we'll:
- Navigate through the `internal/` directory
- Understand the package organization
- Identify key patterns and conventions
- See how different components relate to each other

---

## 📝 Try This Yourself

Before moving to the next tutorial:

1. **Read the README**: Get familiar with installation and basic usage
2. **Browse the structure**: Look at the directory layout
3. **Examine main.go**: Understand the entry point
4. **Think about use cases**: When would you use a tool like this?

---

**Next**: [Tutorial 2: Code Structure Tour](02-code-structure.md)

---

*💡 **Tip**: Keep the Crush codebase open in your editor while going through these tutorials. The best way to learn code is to read it!*
