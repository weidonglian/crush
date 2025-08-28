# Tutorial 3: Entry Points and Bootstrap

**Duration**: 15-20 minutes
**Goal**: Understand how the application starts and initializes

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Follow the complete application startup sequence
- Understand how dependencies are initialized
- See how error handling and logging work
- Learn about the application lifecycle

---

## 🚀 The Journey from `main()` to Running App

Let's trace the path from when you type `crush` to when the TUI is running.

### Step 1: The Entry Point (`main.go`)

```bash
# Let's examine the entry point again
cat main.go
```

Here's what happens in `main()`:

```go
func main() {
    // 1. Set up panic recovery
    defer log.RecoverPanic("main", func() {
        slog.Error("Application terminated due to unhandled panic")
    })

    // 2. Optional profiling server (if CRUSH_PROFILE is set)
    if os.Getenv("CRUSH_PROFILE") != "" {
        go func() {
            slog.Info("Serving pprof at localhost:6060")
            if httpErr := http.ListenAndServe("localhost:6060", nil); httpErr != nil {
                slog.Error("Failed to pprof listen", "error", httpErr)
            }
        }()
    }

    // 3. Execute the CLI command system
    cmd.Execute()
}
```

**Key Points**:
- **Panic Recovery**: The `defer` ensures crashes are logged properly
- **Optional Profiling**: Performance monitoring for development
- **CLI Delegation**: All the real work happens in `cmd.Execute()`

### Step 2: CLI Command System (`internal/cmd/root.go`)

```bash
# Let's look at the command system
cat internal/cmd/root.go | head -50
```

The CLI is built with Cobra:

```go
var rootCmd = &cobra.Command{
    Use:   "crush",
    Short: "Terminal-based AI assistant for software development",
    Long: `Crush is a powerful terminal-based AI assistant...`,
    RunE: func(cmd *cobra.Command, args []string) error {
        return runInteractive(cmd.Context(), cmd)
    },
}

func Execute() {
    // Set up context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle interrupt signals gracefully
    go func() {
        sigChan := make(chan os.Signal, 1)
        signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
        <-sigChan
        cancel()
    }()

    // Execute the root command
    if err := rootCmd.ExecuteContext(ctx); err != nil {
        slog.Error("Command execution failed", "error", err)
        os.Exit(1)
    }
}
```

**What's happening**:
1. **Context Setup**: Creates a cancellable context for the entire app
2. **Signal Handling**: Gracefully handles Ctrl+C and SIGTERM
3. **Command Execution**: Runs the appropriate command (usually interactive mode)

### Step 3: Interactive Mode (`internal/cmd/run.go`)

```bash
# Examine the run command
head -50 internal/cmd/run.go
```

This is where the real application startup begins:

```go
func runInteractive(ctx context.Context, cmd *cobra.Command) error {
    // 1. Parse command-line flags
    cwd, _ := cmd.Flags().GetString("cwd")
    dataDir, _ := cmd.Flags().GetString("data-dir")
    debug, _ := cmd.Flags().GetBool("debug")
    yolo, _ := cmd.Flags().GetBool("yolo")

    // 2. Set up logging
    if debug {
        slog.SetLogLoggerLevel(slog.LevelDebug)
    }

    // 3. Determine working directory
    if cwd == "" {
        if wd, err := os.Getwd(); err == nil {
            cwd = wd
        }
    }

    // 4. Load configuration
    cfg, err := config.Load(cwd, dataDir)
    if err != nil {
        return fmt.Errorf("failed to load config: %w", err)
    }

    // 5. Set up database
    dbPath := filepath.Join(cfg.DataDir(), "crush.db")
    conn, err := db.Connect(ctx, dbPath)
    if err != nil {
        return fmt.Errorf("failed to connect to database: %w", err)
    }
    defer conn.Close()

    // 6. Create and start the application
    app, err := app.New(ctx, conn, cfg)
    if err != nil {
        return fmt.Errorf("failed to create app: %w", err)
    }

    // 7. Run the TUI
    return app.Run(ctx, yolo)
}
```

---

## 🔧 Configuration Loading Deep Dive

The configuration system is quite sophisticated. Let's explore it:

### Configuration Sources (in priority order):

1. **Project-local**: `.crush.json` or `crush.json`
2. **User-global**: `~/.config/crush/crush.json`
3. **Environment variables**
4. **Default values**

```bash
# Look at the config loading logic
head -30 internal/config/load.go
```

**Key Configuration Steps**:

```go
func Load(workingDir, dataDir string) (*Config, error) {
    // 1. Initialize with defaults
    cfg := &Config{}

    // 2. Load global config
    globalPath := globalConfigPath()
    if exists(globalPath) {
        if err := loadFromFile(cfg, globalPath); err != nil {
            return nil, err
        }
    }

    // 3. Load project config
    projectPaths := []string{
        filepath.Join(workingDir, ".crush.json"),
        filepath.Join(workingDir, "crush.json"),
    }
    for _, path := range projectPaths {
        if exists(path) {
            if err := loadFromFile(cfg, path); err != nil {
                return nil, err
            }
            break // Only load the first one found
        }
    }

    // 4. Apply environment variables
    applyEnvVars(cfg)

    // 5. Resolve and validate
    return resolve(cfg, workingDir, dataDir)
}
```

---

## 🗄️ Database Initialization

Crush uses SQLite with automatic migrations:

```bash
# Look at database connection setup
head -30 internal/db/connect.go
```

**Database Setup Process**:

```go
func Connect(ctx context.Context, dbPath string) (*sql.DB, error) {
    // 1. Ensure directory exists
    if err := os.MkdirAll(filepath.Dir(dbPath), 0755); err != nil {
        return nil, err
    }

    // 2. Open SQLite connection
    conn, err := sql.Open("sqlite3", dbPath+"?_pragma=foreign_keys(1)")
    if err != nil {
        return nil, err
    }

    // 3. Run migrations
    if err := runMigrations(ctx, conn); err != nil {
        conn.Close()
        return nil, err
    }

    return conn, nil
}
```

**Migration System**:
```bash
# Look at the migrations
ls internal/db/migrations/
```

You'll see files like:
- `20250424200609_initial.sql` - Initial schema
- `20250515105448_add_summary_message_id.sql` - Add summary feature
- `20250624000000_add_created_at_indexes.sql` - Performance optimization

---

## 🏗️ Application Creation (`internal/app/app.go`)

This is where all the services come together:

```bash
# Look at the New function
grep -A 30 "func New" internal/app/app.go
```

**Service Initialization**:

```go
func New(ctx context.Context, conn *sql.DB, cfg *config.Config) (*App, error) {
    // 1. Create database query interface
    q := db.New(conn)

    // 2. Initialize core services
    sessions := session.NewService(q)
    messages := message.NewService(q)
    files := history.NewService(q, conn)

    // 3. Set up permissions
    skipPermissionsRequests := cfg.Permissions != nil && cfg.Permissions.SkipRequests
    allowedTools := []string{}
    if cfg.Permissions != nil && cfg.Permissions.AllowedTools != nil {
        allowedTools = cfg.Permissions.AllowedTools
    }

    // 4. Create the main app struct
    app := &App{
        Sessions:    sessions,
        Messages:    messages,
        History:     files,
        Permissions: permission.NewPermissionService(cfg.WorkingDir(), skipPermissionsRequests, allowedTools),
        LSPClients:  make(map[string]*lsp.Client),

        globalCtx: ctx,
        config:    cfg,

        watcherCancelFuncs: csync.NewSlice[context.CancelFunc](),
        events:             make(chan tea.Msg, 100),
        serviceEventsWG:    &sync.WaitGroup{},
        tuiWG:              &sync.WaitGroup{},
    }

    // 5. Set up event system
    app.setupEvents()

    // 6. Initialize LSP clients in background
    app.initLSPClients(ctx)

    // 7. Initialize AI agent (if configured)
    if cfg.IsConfigured() {
        if err := app.InitCoderAgent(); err != nil {
            return nil, fmt.Errorf("failed to initialize coder agent: %w", err)
        }
    }

    return app, nil
}
```

---

## ⚡ Event System Setup

Crush uses an event-driven architecture:

```go
func (a *App) setupEvents() {
    // Start the event loop
    a.serviceEventsWG.Add(1)
    go func() {
        defer a.serviceEventsWG.Done()
        for {
            select {
            case <-a.eventsCtx.Done():
                return
            case event := <-a.events:
                // Process events
                a.handleEvent(event)
            }
        }
    }()
}
```

**Event Types**:
- LSP notifications (code changes, diagnostics)
- AI response updates
- File system changes
- User interface updates

---

## 🔄 Application Lifecycle

The complete lifecycle looks like this:

```
1. main()
   ↓
2. cmd.Execute() (signal handling, context)
   ↓
3. runInteractive() (flags, logging setup)
   ↓
4. config.Load() (configuration merging)
   ↓
5. db.Connect() (database + migrations)
   ↓
6. app.New() (service initialization)
   ↓
7. app.Run() (start TUI)
   ↓
8. [Running application with event loops]
   ↓
9. Graceful shutdown (context cancellation)
```

---

## 🛡️ Error Handling Strategy

Notice the consistent error handling pattern throughout:

```go
// Wrap errors with context
if err != nil {
    return fmt.Errorf("failed to load config: %w", err)
}

// Use structured logging
slog.Error("Command execution failed", "error", err)

// Graceful degradation
if cfg.IsConfigured() {
    // Initialize AI features
} else {
    slog.Warn("No agent configuration found")
    // Continue without AI features
}
```

---

## 🤔 Key Design Decisions

### 1. **Graceful Shutdown**
The app properly handles interrupts and cleans up resources:
- Context cancellation propagates through all goroutines
- Database connections are closed
- LSP clients are terminated

### 2. **Fail-Fast vs. Graceful Degradation**
- Configuration errors: Fail-fast (can't run without config)
- AI provider errors: Graceful degradation (warn but continue)
- LSP errors: Graceful degradation (works without language servers)

### 3. **Separation of Concerns**
- CLI layer handles command-line interface
- App layer handles business logic
- Service layer handles specific domains

---

## 🎯 What's Next

In the next tutorial, we'll dive deep into the **Configuration System** where we'll:
- Understand the JSON schema system
- Learn how hierarchical merging works
- See how environment variables are handled
- Explore provider-specific configuration

---

## 📝 Try This Yourself

Before moving to the next tutorial:

1. **Trace the startup**: Follow the code path from `main()` to `app.Run()`
2. **Find the error handling**: Notice how errors are wrapped and logged
3. **Identify the services**: See what services are created in `app.New()`
4. **Check the graceful shutdown**: Look for context cancellation patterns

**Experiment**: Try adding a `slog.Info()` statement in `main.go` and see where it appears when you run the app.

---

**Previous**: [Tutorial 2: Code Structure Tour](02-code-structure.md)
**Next**: [Tutorial 4: Configuration System](04-configuration.md)

---

*💡 **Tip**: The startup sequence is crucial for understanding how complex applications work. Take time to really understand each step!*
