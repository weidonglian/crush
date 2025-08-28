# Advanced 02: Extension Architecture & MCP System

**Duration**: 40-45 minutes
**Level**: 🚀 Advanced
**Goal**: Master the Model Context Protocol (MCP) system, design plugin architectures, and build custom extensions

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Master the Model Context Protocol (MCP) specification and implementation
- Design and build custom tools and extensions
- Understand security boundaries and sandboxing strategies
- Learn plugin system architecture patterns
- Be able to create production-ready extensions for Crush

---

## 📋 Prerequisites

- ✅ Completed Intermediate level tutorials
- ✅ Expert Go programming with interface design
- ✅ Understanding of distributed systems and protocols
- ✅ Experience with JSON-RPC, WebSockets, or similar protocols
- ✅ Knowledge of security principles and sandboxing

---

## 🏗️ Model Context Protocol (MCP) Architecture

MCP is a standardized protocol for AI systems to interact with external tools and data sources. Crush implements a sophisticated MCP system:

```
┌─────────────────────────────────────────────────────────────┐
│                    Crush Core                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ MCP Client Manager                                   │   │
│  │ - Server discovery and lifecycle                     │   │
│  │ - Protocol negotiation                               │   │
│  │ - Tool registration and validation                   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │ (JSON-RPC 2.0)
┌─────────────────────▼───────────────────────────────────────┐
│                  MCP Servers                                │
│  ┌──────────────┬──────────────┬──────────────┬──────────┐  │
│  │ File System  │ Git Tools    │ Web Search   │ Database │  │
│  │ Server       │ Server       │ Server       │ Server   │  │
│  │ (stdio)      │ (HTTP)       │ (SSE)        │ (stdio)  │  │
│  └──────────────┴──────────────┴──────────────┴──────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │ (implements)
┌─────────────────────▼───────────────────────────────────────┐
│                MCP Protocol Spec                            │
│  - Tool definitions and capabilities                        │
│  - Resource access patterns                                 │
│  - Security and permission models                           │
│  - Transport layer abstractions                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔌 MCP Protocol Implementation

### Core Protocol Structures

```bash
# Examine MCP implementation
head -50 internal/llm/agent/mcp-tools.go
```

**MCP Message Types**:

```go
// Core MCP protocol structures
type MCPMessage struct {
    JSONRPC string      `json:"jsonrpc"`
    ID      interface{} `json:"id,omitempty"`
    Method  string      `json:"method,omitempty"`
    Params  interface{} `json:"params,omitempty"`
    Result  interface{} `json:"result,omitempty"`
    Error   *MCPError   `json:"error,omitempty"`
}

type MCPError struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

// Tool definition in MCP
type MCPTool struct {
    Name        string                 `json:"name"`
    Description string                 `json:"description"`
    InputSchema map[string]interface{} `json:"inputSchema"`
}

// Resource definition
type MCPResource struct {
    URI         string            `json:"uri"`
    Name        string            `json:"name"`
    Description string            `json:"description,omitempty"`
    MimeType    string            `json:"mimeType,omitempty"`
    Metadata    map[string]string `json:"metadata,omitempty"`
}

// Server capabilities
type MCPServerCapabilities struct {
    Tools     *MCPToolsCapability     `json:"tools,omitempty"`
    Resources *MCPResourcesCapability `json:"resources,omitempty"`
    Prompts   *MCPPromptsCapability   `json:"prompts,omitempty"`
    Logging   *MCPLoggingCapability   `json:"logging,omitempty"`
}
```

### Transport Layer Abstraction

```go
// Generic transport interface for different MCP server types
type MCPTransport interface {
    // Connection management
    Connect(ctx context.Context) error
    Disconnect() error
    IsConnected() bool

    // Message passing
    Send(ctx context.Context, msg *MCPMessage) error
    Receive(ctx context.Context) (*MCPMessage, error)

    // Streaming support
    StartStream(ctx context.Context) (<-chan *MCPMessage, error)

    // Health checking
    Ping(ctx context.Context) error
}

// Stdio transport (for subprocess-based servers)
type StdioTransport struct {
    cmd    *exec.Cmd
    stdin  io.WriteCloser
    stdout io.ReadCloser
    stderr io.ReadCloser

    encoder *json.Encoder
    decoder *json.Decoder

    // Process management
    done    chan struct{}
    errChan chan error
}

func (t *StdioTransport) Connect(ctx context.Context) error {
    // Start the subprocess
    t.cmd = exec.CommandContext(ctx, t.config.Command, t.config.Args...)

    // Set up pipes
    stdin, err := t.cmd.StdinPipe()
    if err != nil {
        return fmt.Errorf("creating stdin pipe: %w", err)
    }
    t.stdin = stdin

    stdout, err := t.cmd.StdoutPipe()
    if err != nil {
        return fmt.Errorf("creating stdout pipe: %w", err)
    }
    t.stdout = stdout

    stderr, err := t.cmd.StderrPipe()
    if err != nil {
        return fmt.Errorf("creating stderr pipe: %w", err)
    }
    t.stderr = stderr

    // Start the process
    if err := t.cmd.Start(); err != nil {
        return fmt.Errorf("starting MCP server: %w", err)
    }

    // Set up JSON encoder/decoder
    t.encoder = json.NewEncoder(t.stdin)
    t.decoder = json.NewDecoder(t.stdout)

    // Start error monitoring
    go t.monitorProcess()

    // Perform initial handshake
    return t.handshake(ctx)
}

func (t *StdioTransport) Send(ctx context.Context, msg *MCPMessage) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-t.done:
        return fmt.Errorf("transport closed")
    default:
        return t.encoder.Encode(msg)
    }
}

func (t *StdioTransport) Receive(ctx context.Context) (*MCPMessage, error) {
    var msg MCPMessage

    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    case <-t.done:
        return nil, fmt.Errorf("transport closed")
    default:
        if err := t.decoder.Decode(&msg); err != nil {
            return nil, fmt.Errorf("decoding message: %w", err)
        }
        return &msg, nil
    }
}

// HTTP transport (for HTTP-based servers)
type HTTPTransport struct {
    client  *http.Client
    baseURL string
    headers map[string]string

    // Connection pooling
    transport *http.Transport
}

func (t *HTTPTransport) Send(ctx context.Context, msg *MCPMessage) error {
    payload, err := json.Marshal(msg)
    if err != nil {
        return fmt.Errorf("marshaling message: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, "POST", t.baseURL, bytes.NewReader(payload))
    if err != nil {
        return fmt.Errorf("creating request: %w", err)
    }

    req.Header.Set("Content-Type", "application/json")
    for k, v := range t.headers {
        req.Header.Set(k, v)
    }

    resp, err := t.client.Do(req)
    if err != nil {
        return fmt.Errorf("sending request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("server returned status %d", resp.StatusCode)
    }

    return nil
}
```

---

## 🛠️ MCP Server Management

### Server Lifecycle Management

```go
type MCPServerManager struct {
    servers   map[string]*MCPServer
    configs   map[string]*MCPServerConfig
    registry  *ToolRegistry

    // Security
    sandbox   *SecuritySandbox

    // Monitoring
    metrics   *ServerMetrics
    healthChk *HealthChecker

    mu sync.RWMutex
}

type MCPServer struct {
    // Identity
    id       string
    name     string
    version  string

    // Transport
    transport MCPTransport
    config    *MCPServerConfig

    // Capabilities
    capabilities MCPServerCapabilities
    tools        map[string]*MCPTool
    resources    map[string]*MCPResource

    // State
    connected bool
    lastSeen  time.Time

    // Communication
    requestID int64
    pending   map[string]chan *MCPMessage

    mu sync.RWMutex
}

func (m *MCPServerManager) StartServer(config *MCPServerConfig) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    // Validate configuration
    if err := m.validateConfig(config); err != nil {
        return fmt.Errorf("invalid config: %w", err)
    }

    // Create transport based on config
    transport, err := m.createTransport(config)
    if err != nil {
        return fmt.Errorf("creating transport: %w", err)
    }

    // Create server instance
    server := &MCPServer{
        id:        uuid.New().String(),
        name:      config.Name,
        transport: transport,
        config:    config,
        tools:     make(map[string]*MCPTool),
        resources: make(map[string]*MCPResource),
        pending:   make(map[string]chan *MCPMessage),
    }

    // Connect to server
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Connect(ctx); err != nil {
        return fmt.Errorf("connecting to server: %w", err)
    }

    // Discover capabilities
    if err := server.DiscoverCapabilities(ctx); err != nil {
        server.Disconnect()
        return fmt.Errorf("discovering capabilities: %w", err)
    }

    // Register tools
    for _, tool := range server.tools {
        if err := m.registry.RegisterTool(server.id, tool); err != nil {
            slog.Warn("Failed to register tool", "tool", tool.Name, "error", err)
        }
    }

    // Start monitoring
    go server.StartMonitoring()

    m.servers[server.id] = server
    return nil
}

func (s *MCPServer) Connect(ctx context.Context) error {
    // Connect transport
    if err := s.transport.Connect(ctx); err != nil {
        return err
    }

    // Send initialize request
    initMsg := &MCPMessage{
        JSONRPC: "2.0",
        ID:      s.nextRequestID(),
        Method:  "initialize",
        Params: map[string]interface{}{
            "protocolVersion": "2024-11-05",
            "capabilities": map[string]interface{}{
                "roots": map[string]interface{}{
                    "listChanged": true,
                },
                "sampling": map[string]interface{}{},
            },
            "clientInfo": map[string]interface{}{
                "name":    "crush",
                "version": version.Version,
            },
        },
    }

    if err := s.transport.Send(ctx, initMsg); err != nil {
        return fmt.Errorf("sending initialize: %w", err)
    }

    // Wait for initialize response
    response, err := s.waitForResponse(ctx, initMsg.ID)
    if err != nil {
        return fmt.Errorf("waiting for initialize response: %w", err)
    }

    if response.Error != nil {
        return fmt.Errorf("initialize failed: %s", response.Error.Message)
    }

    // Parse server capabilities
    if err := s.parseCapabilities(response.Result); err != nil {
        return fmt.Errorf("parsing capabilities: %w", err)
    }

    s.connected = true
    return nil
}
```

### Tool Registration and Discovery

```go
func (s *MCPServer) DiscoverTools(ctx context.Context) error {
    if !s.capabilities.Tools.ListChanged {
        return fmt.Errorf("server does not support tool listing")
    }

    // Send tools/list request
    listMsg := &MCPMessage{
        JSONRPC: "2.0",
        ID:      s.nextRequestID(),
        Method:  "tools/list",
        Params:  map[string]interface{}{},
    }

    if err := s.transport.Send(ctx, listMsg); err != nil {
        return fmt.Errorf("sending tools/list: %w", err)
    }

    response, err := s.waitForResponse(ctx, listMsg.ID)
    if err != nil {
        return fmt.Errorf("waiting for tools/list response: %w", err)
    }

    if response.Error != nil {
        return fmt.Errorf("tools/list failed: %s", response.Error.Message)
    }

    // Parse tools from response
    var toolsResult struct {
        Tools []MCPTool `json:"tools"`
    }

    if err := mapstructure.Decode(response.Result, &toolsResult); err != nil {
        return fmt.Errorf("decoding tools response: %w", err)
    }

    // Register discovered tools
    s.mu.Lock()
    defer s.mu.Unlock()

    for _, tool := range toolsResult.Tools {
        // Validate tool definition
        if err := s.validateTool(&tool); err != nil {
            slog.Warn("Invalid tool definition", "tool", tool.Name, "error", err)
            continue
        }

        s.tools[tool.Name] = &tool
    }

    return nil
}

func (s *MCPServer) CallTool(ctx context.Context, toolName string, arguments map[string]interface{}) (*ToolResult, error) {
    s.mu.RLock()
    tool, exists := s.tools[toolName]
    s.mu.RUnlock()

    if !exists {
        return nil, fmt.Errorf("tool %s not found", toolName)
    }

    // Validate arguments against schema
    if err := s.validateArguments(tool, arguments); err != nil {
        return nil, fmt.Errorf("invalid arguments: %w", err)
    }

    // Send tools/call request
    callMsg := &MCPMessage{
        JSONRPC: "2.0",
        ID:      s.nextRequestID(),
        Method:  "tools/call",
        Params: map[string]interface{}{
            "name":      toolName,
            "arguments": arguments,
        },
    }

    if err := s.transport.Send(ctx, callMsg); err != nil {
        return nil, fmt.Errorf("sending tools/call: %w", err)
    }

    response, err := s.waitForResponse(ctx, callMsg.ID)
    if err != nil {
        return nil, fmt.Errorf("waiting for tools/call response: %w", err)
    }

    if response.Error != nil {
        return nil, fmt.Errorf("tool call failed: %s", response.Error.Message)
    }

    // Parse result
    var result ToolResult
    if err := mapstructure.Decode(response.Result, &result); err != nil {
        return nil, fmt.Errorf("decoding tool result: %w", err)
    }

    return &result, nil
}
```

---

## 🔒 Security and Sandboxing

### Permission System for Tools

```go
type SecuritySandbox struct {
    // Permission policies
    policies map[string]*PermissionPolicy

    // Resource access control
    allowedPaths   []string
    blockedPaths   []string
    maxFileSize    int64

    // Network restrictions
    allowedHosts   []string
    blockedPorts   []int

    // Execution limits
    maxExecutionTime time.Duration
    maxMemoryUsage   int64
    maxProcesses     int

    // Audit logging
    auditLogger *AuditLogger
}

type PermissionPolicy struct {
    // Tool permissions
    AllowedTools   []string `json:"allowed_tools"`
    BlockedTools   []string `json:"blocked_tools"`

    // File system permissions
    ReadPaths      []string `json:"read_paths"`
    WritePaths     []string `json:"write_paths"`

    // Network permissions
    AllowNetwork   bool     `json:"allow_network"`
    AllowedDomains []string `json:"allowed_domains"`

    // Process permissions
    AllowExecution bool     `json:"allow_execution"`
    AllowedCommands []string `json:"allowed_commands"`
}

func (s *SecuritySandbox) ValidateToolCall(serverID, toolName string, arguments map[string]interface{}) error {
    policy, exists := s.policies[serverID]
    if !exists {
        return fmt.Errorf("no policy defined for server %s", serverID)
    }

    // Check if tool is allowed
    if !s.isToolAllowed(policy, toolName) {
        return fmt.Errorf("tool %s not allowed by policy", toolName)
    }

    // Validate arguments based on tool type
    switch toolName {
    case "file_read", "file_write":
        return s.validateFileAccess(policy, toolName, arguments)
    case "shell_exec":
        return s.validateShellExecution(policy, arguments)
    case "http_request":
        return s.validateNetworkAccess(policy, arguments)
    default:
        // Custom validation for extension tools
        return s.validateCustomTool(policy, toolName, arguments)
    }
}

func (s *SecuritySandbox) validateFileAccess(policy *PermissionPolicy, operation string, args map[string]interface{}) error {
    path, ok := args["path"].(string)
    if !ok {
        return fmt.Errorf("invalid path argument")
    }

    // Resolve absolute path
    absPath, err := filepath.Abs(path)
    if err != nil {
        return fmt.Errorf("resolving path: %w", err)
    }

    // Check against blocked paths
    for _, blocked := range s.blockedPaths {
        if strings.HasPrefix(absPath, blocked) {
            return fmt.Errorf("path %s is blocked", absPath)
        }
    }

    // Check operation-specific permissions
    switch operation {
    case "file_read":
        if !s.isPathAllowed(policy.ReadPaths, absPath) {
            return fmt.Errorf("read access to %s not allowed", absPath)
        }
    case "file_write":
        if !s.isPathAllowed(policy.WritePaths, absPath) {
            return fmt.Errorf("write access to %s not allowed", absPath)
        }

        // Check file size limits for writes
        if content, ok := args["content"].(string); ok {
            if int64(len(content)) > s.maxFileSize {
                return fmt.Errorf("content size exceeds limit")
            }
        }
    }

    return nil
}

// Execution sandbox using containers or chroot
type ProcessSandbox struct {
    // Container runtime (Docker, Podman, etc.)
    runtime string

    // Resource limits
    cpuLimit    string
    memoryLimit string
    diskLimit   string

    // Network isolation
    networkMode string

    // File system isolation
    rootFS      string
    readOnlyFS  bool
    tmpFS       []string
}

func (ps *ProcessSandbox) ExecuteInSandbox(ctx context.Context, command string, args []string) (*ExecResult, error) {
    // Create temporary container
    containerID := uuid.New().String()

    // Build container command
    containerArgs := []string{
        "run",
        "--rm",
        "--name", containerID,
        "--cpus", ps.cpuLimit,
        "--memory", ps.memoryLimit,
        "--network", ps.networkMode,
        "--read-only=" + strconv.FormatBool(ps.readOnlyFS),
    }

    // Add temporary filesystems
    for _, tmpPath := range ps.tmpFS {
        containerArgs = append(containerArgs, "--tmpfs", tmpPath)
    }

    // Add the command
    containerArgs = append(containerArgs, ps.rootFS, command)
    containerArgs = append(containerArgs, args...)

    // Execute with timeout
    execCtx, cancel := context.WithTimeout(ctx, ps.maxExecutionTime)
    defer cancel()

    cmd := exec.CommandContext(execCtx, ps.runtime, containerArgs...)

    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr

    start := time.Now()
    err := cmd.Run()
    duration := time.Since(start)

    result := &ExecResult{
        ExitCode: cmd.ProcessState.ExitCode(),
        Stdout:   stdout.String(),
        Stderr:   stderr.String(),
        Duration: duration,
    }

    if err != nil {
        result.Error = err.Error()
    }

    return result, nil
}
```

---

## 🧩 Custom Extension Development

### Building a Git MCP Server

```go
// Example: Git operations MCP server
type GitMCPServer struct {
    repoPath string
    repo     *git.Repository
}

func (g *GitMCPServer) GetTools() []MCPTool {
    return []MCPTool{
        {
            Name:        "git_status",
            Description: "Get the current git status of the repository",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "include_untracked": {
                        "type":        "boolean",
                        "description": "Include untracked files in status",
                        "default":     true,
                    },
                },
            },
        },
        {
            Name:        "git_diff",
            Description: "Get diff of changes in the repository",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "file": {
                        "type":        "string",
                        "description": "Specific file to diff (optional)",
                    },
                    "staged": {
                        "type":        "boolean",
                        "description": "Show staged changes only",
                        "default":     false,
                    },
                },
            },
        },
        {
            Name:        "git_commit",
            Description: "Create a new commit with the given message",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "message": {
                        "type":        "string",
                        "description": "Commit message",
                    },
                    "files": {
                        "type":        "array",
                        "items":       map[string]interface{}{"type": "string"},
                        "description": "Files to stage and commit (optional)",
                    },
                },
                "required": []string{"message"},
            },
        },
    }
}

func (g *GitMCPServer) ExecuteTool(ctx context.Context, toolName string, arguments map[string]interface{}) (*ToolResult, error) {
    switch toolName {
    case "git_status":
        return g.gitStatus(ctx, arguments)
    case "git_diff":
        return g.gitDiff(ctx, arguments)
    case "git_commit":
        return g.gitCommit(ctx, arguments)
    default:
        return nil, fmt.Errorf("unknown tool: %s", toolName)
    }
}

func (g *GitMCPServer) gitStatus(ctx context.Context, args map[string]interface{}) (*ToolResult, error) {
    includeUntracked := true
    if val, ok := args["include_untracked"].(bool); ok {
        includeUntracked = val
    }

    worktree, err := g.repo.Worktree()
    if err != nil {
        return nil, fmt.Errorf("getting worktree: %w", err)
    }

    status, err := worktree.Status()
    if err != nil {
        return nil, fmt.Errorf("getting status: %w", err)
    }

    var statusLines []string
    for file, fileStatus := range status {
        if !includeUntracked && fileStatus.Worktree == git.Untracked {
            continue
        }

        statusLines = append(statusLines, fmt.Sprintf("%c%c %s",
            fileStatus.Staging, fileStatus.Worktree, file))
    }

    sort.Strings(statusLines)

    return &ToolResult{
        Content: []map[string]interface{}{
            {
                "type": "text",
                "text": strings.Join(statusLines, "\n"),
            },
        },
    }, nil
}

func (g *GitMCPServer) gitDiff(ctx context.Context, args map[string]interface{}) (*ToolResult, error) {
    var file string
    if val, ok := args["file"].(string); ok {
        file = val
    }

    staged := false
    if val, ok := args["staged"].(bool); ok {
        staged = val
    }

    worktree, err := g.repo.Worktree()
    if err != nil {
        return nil, fmt.Errorf("getting worktree: %w", err)
    }

    var patch *object.Patch

    if staged {
        // Get staged changes
        head, err := g.repo.Head()
        if err != nil {
            return nil, fmt.Errorf("getting HEAD: %w", err)
        }

        headCommit, err := g.repo.CommitObject(head.Hash())
        if err != nil {
            return nil, fmt.Errorf("getting HEAD commit: %w", err)
        }

        headTree, err := headCommit.Tree()
        if err != nil {
            return nil, fmt.Errorf("getting HEAD tree: %w", err)
        }

        // Compare HEAD with index
        patch, err = headTree.Patch(ctx, worktree)
        if err != nil {
            return nil, fmt.Errorf("creating patch: %w", err)
        }
    } else {
        // Get working directory changes
        // Implementation depends on git library capabilities
    }

    var diffContent string
    if patch != nil {
        diffContent = patch.String()

        // Filter by file if specified
        if file != "" {
            diffContent = g.filterDiffByFile(diffContent, file)
        }
    }

    return &ToolResult{
        Content: []map[string]interface{}{
            {
                "type": "text",
                "text": diffContent,
            },
        },
    }, nil
}
```

### Database Query MCP Server

```go
// Example: Database query MCP server with safety checks
type DatabaseMCPServer struct {
    db           *sql.DB
    allowedTables []string
    readOnly      bool
    queryTimeout  time.Duration
}

func (d *DatabaseMCPServer) GetTools() []MCPTool {
    tools := []MCPTool{
        {
            Name:        "db_query",
            Description: "Execute a SELECT query against the database",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "query": {
                        "type":        "string",
                        "description": "SQL SELECT query to execute",
                    },
                    "limit": {
                        "type":        "integer",
                        "description": "Maximum number of rows to return",
                        "default":     100,
                        "maximum":     1000,
                    },
                },
                "required": []string{"query"},
            },
        },
        {
            Name:        "db_schema",
            Description: "Get schema information for tables",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "table": {
                        "type":        "string",
                        "description": "Specific table name (optional)",
                    },
                },
            },
        },
    }

    // Add write operations only if not read-only
    if !d.readOnly {
        tools = append(tools, MCPTool{
            Name:        "db_execute",
            Description: "Execute an INSERT, UPDATE, or DELETE statement",
            InputSchema: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "statement": {
                        "type":        "string",
                        "description": "SQL statement to execute",
                    },
                    "confirm": {
                        "type":        "boolean",
                        "description": "Confirmation that you want to execute this statement",
                        "default":     false,
                    },
                },
                "required": []string{"statement", "confirm"},
            },
        })
    }

    return tools
}

func (d *DatabaseMCPServer) ExecuteTool(ctx context.Context, toolName string, arguments map[string]interface{}) (*ToolResult, error) {
    switch toolName {
    case "db_query":
        return d.executeQuery(ctx, arguments)
    case "db_schema":
        return d.getSchema(ctx, arguments)
    case "db_execute":
        if d.readOnly {
            return nil, fmt.Errorf("database server is in read-only mode")
        }
        return d.executeStatement(ctx, arguments)
    default:
        return nil, fmt.Errorf("unknown tool: %s", toolName)
    }
}

func (d *DatabaseMCPServer) executeQuery(ctx context.Context, args map[string]interface{}) (*ToolResult, error) {
    query, ok := args["query"].(string)
    if !ok {
        return nil, fmt.Errorf("query must be a string")
    }

    limit := 100
    if val, ok := args["limit"].(float64); ok {
        limit = int(val)
    }

    // Validate query safety
    if err := d.validateQuery(query); err != nil {
        return nil, fmt.Errorf("query validation failed: %w", err)
    }

    // Execute with timeout
    queryCtx, cancel := context.WithTimeout(ctx, d.queryTimeout)
    defer cancel()

    rows, err := d.db.QueryContext(queryCtx, query)
    if err != nil {
        return nil, fmt.Errorf("executing query: %w", err)
    }
    defer rows.Close()

    // Get column names
    columns, err := rows.Columns()
    if err != nil {
        return nil, fmt.Errorf("getting columns: %w", err)
    }

    // Read results
    var results []map[string]interface{}
    count := 0

    for rows.Next() && count < limit {
        values := make([]interface{}, len(columns))
        valuePtrs := make([]interface{}, len(columns))

        for i := range values {
            valuePtrs[i] = &values[i]
        }

        if err := rows.Scan(valuePtrs...); err != nil {
            return nil, fmt.Errorf("scanning row: %w", err)
        }

        row := make(map[string]interface{})
        for i, col := range columns {
            row[col] = values[i]
        }

        results = append(results, row)
        count++
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("row iteration error: %w", err)
    }

    // Format results as table
    tableContent := d.formatResultsAsTable(columns, results)

    return &ToolResult{
        Content: []map[string]interface{}{
            {
                "type": "text",
                "text": fmt.Sprintf("Query returned %d rows:\n\n%s", len(results), tableContent),
            },
        },
    }, nil
}

func (d *DatabaseMCPServer) validateQuery(query string) error {
    // Parse SQL to ensure it's a SELECT statement
    query = strings.TrimSpace(strings.ToUpper(query))

    if !strings.HasPrefix(query, "SELECT") {
        return fmt.Errorf("only SELECT queries are allowed")
    }

    // Check for forbidden keywords
    forbidden := []string{"DROP", "DELETE", "UPDATE", "INSERT", "ALTER", "CREATE", "TRUNCATE"}
    for _, keyword := range forbidden {
        if strings.Contains(query, keyword) {
            return fmt.Errorf("forbidden keyword '%s' in query", keyword)
        }
    }

    // Validate table access if table restrictions are in place
    if len(d.allowedTables) > 0 {
        // Simple table name extraction (production would use proper SQL parser)
        if !d.containsAllowedTables(query) {
            return fmt.Errorf("query accesses tables not in allowed list")
        }
    }

    return nil
}
```

---

## 🎯 What's Next

In the next advanced tutorial, we'll explore **Advanced AI Patterns** where we'll:
- Master multi-model orchestration and ensemble techniques
- Learn advanced prompt engineering and chain-of-thought patterns
- Understand model switching and adaptive AI workflows
- Explore cutting-edge AI integration patterns

---

## 📝 Expert-Level Exercises

1. **Custom MCP Server**: Build a complete MCP server for your preferred external service (Jira, Slack, etc.)
2. **Security Enhancement**: Implement advanced sandboxing with container isolation
3. **Protocol Extension**: Design and implement custom MCP protocol extensions
4. **Performance Optimization**: Optimize MCP communication for high-frequency tool usage
5. **Plugin Marketplace**: Design a plugin distribution and management system

---

**Previous**: [Advanced 01: Performance Analysis](01-performance-analysis.md)
**Next**: [Advanced 03: Advanced AI Patterns](03-advanced-ai-patterns.md)

---

*🔌 **Extension Insight**: The MCP system is the future of AI tool integration. Mastering this architecture positions you at the forefront of AI extensibility patterns.*
