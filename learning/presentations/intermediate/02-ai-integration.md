# Intermediate 02: AI Integration Deep Dive

**Duration**: 40-45 minutes
**Level**: 🔧 Intermediate
**Goal**: Master the AI provider system, agent orchestration, and prompt engineering

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand the complete AI provider abstraction layer
- Master agent orchestration and tool management
- Learn advanced prompt engineering and context building
- Understand how multiple AI models are coordinated
- Be able to add new AI providers and implement custom tools

---

## 📋 Prerequisites

- ✅ Completed Beginner level tutorials
- ✅ Solid understanding of HTTP APIs and JSON
- ✅ Experience with AI/LLM concepts and APIs
- ✅ Familiarity with design patterns (Strategy, Factory, Observer)

---

## 🏗️ AI System Architecture

The AI integration in Crush is built with a sophisticated multi-layer architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    User Interface Layer                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TUI Components (Chat, Input, Status)                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │ (user messages)
┌─────────────────────▼───────────────────────────────────────┐
│                   Agent Layer                               │
│  ┌─────────────────┬─────────────────┬─────────────────────┐ │
│  │ Coder Agent     │ Tool Manager    │ Context Builder     │ │
│  │ (orchestration) │ (capabilities)  │ (prompt assembly)   │ │
│  └─────────────────┴─────────────────┴─────────────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │ (structured requests)
┌─────────────────────▼───────────────────────────────────────┐
│                  Provider Layer                             │
│  ┌──────────────┬──────────────┬──────────────┬──────────┐  │
│  │ OpenAI       │ Anthropic    │ Groq         │ Gemini   │  │
│  │ Provider     │ Provider     │ Provider     │ Provider │  │
│  └──────────────┴──────────────┴──────────────┴──────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │ (HTTP requests)
┌─────────────────────▼───────────────────────────────────────┐
│                External AI APIs                             │
│  ┌──────────────┬──────────────┬──────────────┬──────────┐  │
│  │ OpenAI API   │ Claude API   │ Groq API     │ Gemini   │  │
│  └──────────────┴──────────────┴──────────────┴──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🤖 Provider Abstraction Layer

Let's examine how Crush abstracts different AI providers:

### Core Provider Interface

```bash
# Look at the provider interface definition
grep -A 20 "type.*Provider" internal/llm/provider/
```

**Provider Interface Design**:

```go
// Common interface for all AI providers
type Provider interface {
    // Core functionality
    Complete(ctx context.Context, req CompletionRequest) (*CompletionResponse, error)
    Stream(ctx context.Context, req CompletionRequest) (<-chan CompletionChunk, error)

    // Provider information
    Name() string
    Models() []Model
    MaxTokens(model string) int

    // Configuration
    Configure(config Config) error
    Validate() error
}

// Unified request format
type CompletionRequest struct {
    Model       string         `json:"model"`
    Messages    []Message      `json:"messages"`
    MaxTokens   int           `json:"max_tokens,omitempty"`
    Temperature float64       `json:"temperature,omitempty"`
    Tools       []Tool        `json:"tools,omitempty"`
    Stream      bool          `json:"stream,omitempty"`
}

// Unified response format
type CompletionResponse struct {
    ID      string           `json:"id"`
    Model   string           `json:"model"`
    Choices []Choice         `json:"choices"`
    Usage   Usage            `json:"usage"`
}
```

### Provider Implementation Examples

#### OpenAI Provider Implementation

```bash
# Examine OpenAI provider implementation
head -50 internal/llm/provider/openai.go
```

**Key Implementation Patterns**:

```go
type OpenAIProvider struct {
    client   *openai.Client
    config   *OpenAIConfig
    models   []Model
    baseURL  string
}

func (p *OpenAIProvider) Complete(ctx context.Context, req CompletionRequest) (*CompletionResponse, error) {
    // 1. Convert unified request to OpenAI format
    openaiReq := p.convertRequest(req)

    // 2. Add provider-specific settings
    openaiReq.Model = p.resolveModel(req.Model)
    openaiReq.MaxTokens = p.adjustMaxTokens(req.MaxTokens, req.Model)

    // 3. Make API call with retries and error handling
    resp, err := p.client.CreateChatCompletion(ctx, openaiReq)
    if err != nil {
        return nil, p.handleError(err)
    }

    // 4. Convert response back to unified format
    return p.convertResponse(resp), nil
}

// Provider-specific optimizations
func (p *OpenAIProvider) convertRequest(req CompletionRequest) openai.ChatCompletionRequest {
    openaiReq := openai.ChatCompletionRequest{
        Model:       req.Model,
        MaxTokens:   req.MaxTokens,
        Temperature: &req.Temperature,
    }

    // Convert messages with OpenAI-specific formatting
    for _, msg := range req.Messages {
        openaiMsg := openai.ChatCompletionMessage{
            Role:    msg.Role,
            Content: p.formatContent(msg.Content),
        }

        // Handle attachments for vision models
        if p.supportsVision(req.Model) && len(msg.Attachments) > 0 {
            openaiMsg.Content = p.formatMultimodalContent(msg)
        }

        openaiReq.Messages = append(openaiReq.Messages, openaiMsg)
    }

    return openaiReq
}
```

#### Anthropic Provider Implementation

```go
type AnthropicProvider struct {
    client *anthropic.Client
    config *AnthropicConfig
}

func (p *AnthropicProvider) Complete(ctx context.Context, req CompletionRequest) (*CompletionResponse, error) {
    // Anthropic has different message format (system vs user/assistant)
    systemMsg, messages := p.extractSystemMessage(req.Messages)

    anthropicReq := anthropic.MessageRequest{
        Model:     p.resolveModel(req.Model),
        MaxTokens: req.MaxTokens,
        Messages:  p.convertMessages(messages),
        System:    systemMsg, // Anthropic-specific system message handling
    }

    resp, err := p.client.CreateMessage(ctx, anthropicReq)
    if err != nil {
        return nil, p.handleAnthropicError(err)
    }

    return p.convertAnthropicResponse(resp), nil
}

// Anthropic-specific message handling
func (p *AnthropicProvider) extractSystemMessage(messages []Message) (string, []Message) {
    var systemContent string
    var userMessages []Message

    for _, msg := range messages {
        if msg.Role == "system" {
            systemContent += msg.Content + "\n"
        } else {
            userMessages = append(userMessages, msg)
        }
    }

    return strings.TrimSpace(systemContent), userMessages
}
```

---

## 🎭 Agent System Deep Dive

The agent system orchestrates AI interactions and tool usage:

### Agent Architecture

```bash
# Examine the agent implementation
head -100 internal/llm/agent/agent.go
```

**Agent Responsibilities**:

```go
type CoderAgent struct {
    // Provider management
    providers map[string]provider.Provider
    current   provider.Provider

    // Tool system
    tools     map[string]Tool
    toolMgr   *ToolManager

    // Context management
    contextBuilder *ContextBuilder
    sessionMgr     session.Service

    // Configuration
    config *AgentConfig
}

// Main interaction loop
func (a *CoderAgent) ProcessMessage(ctx context.Context, input UserInput) (*Response, error) {
    // 1. Build context from current session
    context, err := a.contextBuilder.Build(ctx, input.SessionID)
    if err != nil {
        return nil, fmt.Errorf("building context: %w", err)
    }

    // 2. Determine if tools are needed
    toolsNeeded := a.analyzeToolRequirements(input.Content)

    // 3. Prepare the prompt
    prompt := a.buildPrompt(context, input, toolsNeeded)

    // 4. Make AI request
    response, err := a.current.Complete(ctx, provider.CompletionRequest{
        Model:    a.config.Model,
        Messages: prompt.Messages,
        Tools:    toolsNeeded,
    })
    if err != nil {
        return nil, fmt.Errorf("AI completion: %w", err)
    }

    // 5. Execute any tool calls
    if len(response.ToolCalls) > 0 {
        return a.executeToolsAndFollow(ctx, response, input.SessionID)
    }

    // 6. Format and return response
    return a.formatResponse(response), nil
}
```

### Tool Management System

```bash
# Look at tool management
head -50 internal/llm/agent/mcp-tools.go
```

**Tool System Architecture**:

```go
// Tool interface for extensibility
type Tool interface {
    Name() string
    Description() string
    Parameters() map[string]interface{}
    Execute(ctx context.Context, args map[string]interface{}) (*ToolResult, error)
    RequiresPermission() bool
}

// Built-in tools
type FileTool struct {
    basePath string
    allowed  []string
}

func (t *FileTool) Execute(ctx context.Context, args map[string]interface{}) (*ToolResult, error) {
    action := args["action"].(string)
    path := args["path"].(string)

    // Security check
    if !t.isPathAllowed(path) {
        return nil, fmt.Errorf("path not allowed: %s", path)
    }

    switch action {
    case "read":
        content, err := os.ReadFile(path)
        if err != nil {
            return nil, err
        }
        return &ToolResult{
            Type:    "text",
            Content: string(content),
        }, nil

    case "write":
        content := args["content"].(string)
        err := os.WriteFile(path, []byte(content), 0644)
        return &ToolResult{
            Type:    "success",
            Content: fmt.Sprintf("File %s written successfully", path),
        }, err

    default:
        return nil, fmt.Errorf("unknown action: %s", action)
    }
}

// MCP (Model Context Protocol) integration
type MCPToolManager struct {
    servers map[string]*mcp.Server
    tools   map[string]Tool
}

func (m *MCPToolManager) LoadMCPServer(config MCPServerConfig) error {
    server, err := mcp.NewServer(config.Command, config.Args...)
    if err != nil {
        return err
    }

    // Discover available tools from the server
    tools, err := server.ListTools(context.Background())
    if err != nil {
        return err
    }

    // Register tools with the agent
    for _, tool := range tools {
        m.tools[tool.Name] = &MCPTool{
            server: server,
            spec:   tool,
        }
    }

    m.servers[config.Name] = server
    return nil
}
```

---

## 📝 Prompt Engineering System

The prompt system is sophisticated and modular:

### Prompt Template System

```bash
# Examine prompt templates
ls internal/llm/prompt/
cat internal/llm/prompt/coder.go | head -50
```

**Prompt Architecture**:

```go
type PromptBuilder struct {
    templates map[string]*template.Template
    context   *ContextGatherer
    config    *PromptConfig
}

// Main system prompt for coding tasks
const CoderSystemPrompt = `You are Crush, an expert AI coding assistant.

## Your Capabilities
- Analyze and understand code in multiple programming languages
- Write, debug, and optimize code
- Explain complex technical concepts clearly
- Use tools to read, write, and analyze files
- Execute shell commands when needed (with user permission)

## Context Available
{{if .LSPContext}}
### Language Server Analysis
{{.LSPContext}}
{{end}}

{{if .FileContext}}
### Relevant Files
{{range .FileContext}}
**{{.Path}}** ({{.Language}})
'''{{.Language}}
{{.Content}}
'''
{{end}}
{{end}}

{{if .ErrorContext}}
### Recent Errors
{{.ErrorContext}}
{{end}}

## Guidelines
- Be precise and accurate in your code suggestions
- Explain your reasoning when making significant changes
- Ask for clarification if the request is ambiguous
- Always consider security and best practices
- Use the available tools effectively`

// Dynamic context building
func (p *PromptBuilder) BuildContext(ctx context.Context, sessionID string) (*Context, error) {
    context := &Context{
        SessionID: sessionID,
    }

    // 1. Get LSP context if available
    if lspContext, err := p.gatherLSPContext(ctx, sessionID); err == nil {
        context.LSPContext = lspContext
    }

    // 2. Get relevant files from session history
    files, err := p.gatherRelevantFiles(ctx, sessionID)
    if err == nil {
        context.FileContext = files
    }

    // 3. Get recent errors
    if errors, err := p.gatherRecentErrors(ctx, sessionID); err == nil {
        context.ErrorContext = errors
    }

    // 4. Get conversation history with smart truncation
    history, err := p.gatherConversationHistory(ctx, sessionID)
    if err == nil {
        context.ConversationHistory = p.truncateHistory(history)
    }

    return context, nil
}
```

### Context Gathering System

```go
// LSP-enhanced context gathering
func (p *PromptBuilder) gatherLSPContext(ctx context.Context, sessionID string) (*LSPContext, error) {
    session, err := p.sessionSvc.Get(ctx, sessionID)
    if err != nil {
        return nil, err
    }

    // Get current working files
    workingFiles := p.getWorkingFiles(ctx, sessionID)

    lspContext := &LSPContext{}

    for _, file := range workingFiles {
        language := p.detectLanguage(file.Path)
        client := p.lspManager.GetClient(language)
        if client == nil {
            continue
        }

        // Get diagnostics (errors, warnings)
        diagnostics, err := client.GetDiagnostics(ctx, file.Path)
        if err == nil {
            lspContext.Diagnostics = append(lspContext.Diagnostics, diagnostics...)
        }

        // Get symbols (functions, classes, etc.)
        symbols, err := client.GetDocumentSymbols(ctx, file.Path)
        if err == nil {
            lspContext.Symbols = append(lspContext.Symbols, symbols...)
        }

        // Get references for current cursor position
        if file.CursorPosition != nil {
            refs, err := client.GetReferences(ctx, file.Path, *file.CursorPosition)
            if err == nil {
                lspContext.References = append(lspContext.References, refs...)
            }
        }
    }

    return lspContext, nil
}

// Smart history truncation to fit context window
func (p *PromptBuilder) truncateHistory(history []Message) []Message {
    maxTokens := p.config.MaxHistoryTokens
    currentTokens := 0

    // Keep recent messages and important messages
    var truncated []Message

    // Always include the last few messages
    recentCount := min(len(history), 5)
    for i := len(history) - recentCount; i < len(history); i++ {
        msg := history[i]
        tokens := p.estimateTokens(msg.Content)
        if currentTokens+tokens <= maxTokens {
            truncated = append(truncated, msg)
            currentTokens += tokens
        }
    }

    // Add important messages from earlier in the conversation
    for i := len(history) - recentCount - 1; i >= 0; i-- {
        msg := history[i]
        if p.isImportantMessage(msg) {
            tokens := p.estimateTokens(msg.Content)
            if currentTokens+tokens <= maxTokens {
                truncated = append([]Message{msg}, truncated...)
                currentTokens += tokens
            }
        }
    }

    return truncated
}

func (p *PromptBuilder) isImportantMessage(msg Message) bool {
    // Messages with code blocks
    if strings.Contains(msg.Content, "```") {
        return true
    }

    // Error messages
    if strings.Contains(strings.ToLower(msg.Content), "error") {
        return true
    }

    // Tool usage
    if len(msg.ToolCalls) > 0 {
        return true
    }

    // Long messages are likely important
    if len(msg.Content) > 500 {
        return true
    }

    return false
}
```

---

## 🔄 Provider Switching and Model Management

### Dynamic Provider Switching

```go
type ProviderManager struct {
    providers map[string]provider.Provider
    current   string
    fallbacks []string
    config    *ProviderConfig
}

func (pm *ProviderManager) SwitchProvider(name, model string) error {
    provider, exists := pm.providers[name]
    if !exists {
        return fmt.Errorf("provider %s not found", name)
    }

    // Validate model availability
    if !provider.SupportsModel(model) {
        return fmt.Errorf("provider %s does not support model %s", name, model)
    }

    // Test provider connectivity
    if err := provider.Validate(); err != nil {
        return fmt.Errorf("provider validation failed: %w", err)
    }

    pm.current = name
    return nil
}

// Automatic failover
func (pm *ProviderManager) CompleteWithFailover(ctx context.Context, req CompletionRequest) (*CompletionResponse, error) {
    var lastErr error

    // Try current provider first
    if resp, err := pm.providers[pm.current].Complete(ctx, req); err == nil {
        return resp, nil
    } else {
        lastErr = err
        slog.Warn("Primary provider failed", "provider", pm.current, "error", err)
    }

    // Try fallback providers
    for _, fallback := range pm.fallbacks {
        provider := pm.providers[fallback]
        if provider == nil {
            continue
        }

        // Adjust request for fallback provider
        adjustedReq := pm.adjustRequestForProvider(req, fallback)

        if resp, err := provider.Complete(ctx, adjustedReq); err == nil {
            slog.Info("Failover successful", "provider", fallback)
            return resp, nil
        } else {
            slog.Warn("Fallback provider failed", "provider", fallback, "error", err)
            lastErr = err
        }
    }

    return nil, fmt.Errorf("all providers failed, last error: %w", lastErr)
}
```

### Model Capability Management

```go
type ModelCapabilities struct {
    MaxTokens     int      `json:"max_tokens"`
    SupportsTools bool     `json:"supports_tools"`
    SupportsVision bool    `json:"supports_vision"`
    CostPer1K     float64  `json:"cost_per_1k"`
    Languages     []string `json:"languages"`
}

func (pm *ProviderManager) SelectOptimalModel(req CompletionRequest) (string, string, error) {
    // Analyze requirements
    needsTools := len(req.Tools) > 0
    needsVision := pm.hasImageAttachments(req.Messages)
    tokenCount := pm.estimateTokens(req)

    var bestProvider, bestModel string
    var bestScore float64

    for providerName, provider := range pm.providers {
        for _, model := range provider.Models() {
            caps := pm.getModelCapabilities(providerName, model.Name)

            // Check hard requirements
            if needsTools && !caps.SupportsTools {
                continue
            }
            if needsVision && !caps.SupportsVision {
                continue
            }
            if tokenCount > caps.MaxTokens {
                continue
            }

            // Calculate score (lower is better for cost-effectiveness)
            score := caps.CostPer1K

            // Bonus for preferred providers
            if providerName == pm.config.PreferredProvider {
                score *= 0.9
            }

            // Bonus for models with more capabilities
            if caps.SupportsTools {
                score *= 0.95
            }

            if bestProvider == "" || score < bestScore {
                bestProvider = providerName
                bestModel = model.Name
                bestScore = score
            }
        }
    }

    if bestProvider == "" {
        return "", "", fmt.Errorf("no suitable model found for requirements")
    }

    return bestProvider, bestModel, nil
}
```

---

## 🎯 Advanced AI Integration Patterns

### Streaming Response Handling

```go
func (a *CoderAgent) StreamResponse(ctx context.Context, req CompletionRequest) (<-chan ResponseChunk, error) {
    provider := a.providers[a.currentProvider]

    // Start streaming from provider
    providerStream, err := provider.Stream(ctx, req)
    if err != nil {
        return nil, err
    }

    // Create output channel with buffering
    output := make(chan ResponseChunk, 10)

    go func() {
        defer close(output)

        var accumulated strings.Builder
        var toolCalls []ToolCall

        for chunk := range providerStream {
            // Process chunk based on type
            switch chunk.Type {
            case "content":
                accumulated.WriteString(chunk.Content)

                // Send incremental update
                select {
                case output <- ResponseChunk{
                    Type:    "content",
                    Content: chunk.Content,
                    Delta:   true,
                }:
                case <-ctx.Done():
                    return
                }

            case "tool_call":
                toolCalls = append(toolCalls, chunk.ToolCall)

            case "done":
                // Process any tool calls
                if len(toolCalls) > 0 {
                    a.executeToolCallsStreaming(ctx, toolCalls, output)
                }

                // Send final response
                select {
                case output <- ResponseChunk{
                    Type:     "done",
                    Content:  accumulated.String(),
                    ToolCalls: toolCalls,
                }:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()

    return output, nil
}
```

### Advanced Error Handling and Retries

```go
type RetryConfig struct {
    MaxRetries   int           `json:"max_retries"`
    BaseDelay    time.Duration `json:"base_delay"`
    MaxDelay     time.Duration `json:"max_delay"`
    BackoffMult  float64       `json:"backoff_multiplier"`
    RetryableErrors []string   `json:"retryable_errors"`
}

func (p *OpenAIProvider) CompleteWithRetry(ctx context.Context, req CompletionRequest) (*CompletionResponse, error) {
    var lastErr error

    for attempt := 0; attempt <= p.retryConfig.MaxRetries; attempt++ {
        if attempt > 0 {
            // Exponential backoff with jitter
            delay := time.Duration(float64(p.retryConfig.BaseDelay) *
                     math.Pow(p.retryConfig.BackoffMult, float64(attempt-1)))
            if delay > p.retryConfig.MaxDelay {
                delay = p.retryConfig.MaxDelay
            }

            // Add jitter (±25%)
            jitter := time.Duration(rand.Float64() * float64(delay) * 0.5)
            delay = delay + jitter - time.Duration(float64(delay)*0.25)

            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }

        resp, err := p.Complete(ctx, req)
        if err == nil {
            return resp, nil
        }

        lastErr = err

        // Check if error is retryable
        if !p.isRetryableError(err) {
            break
        }

        slog.Warn("Request failed, retrying",
            "attempt", attempt+1,
            "error", err,
            "provider", p.Name())
    }

    return nil, fmt.Errorf("request failed after %d attempts: %w",
        p.retryConfig.MaxRetries+1, lastErr)
}

func (p *OpenAIProvider) isRetryableError(err error) bool {
    // Rate limiting
    if strings.Contains(err.Error(), "rate_limit_exceeded") {
        return true
    }

    // Temporary server errors
    if strings.Contains(err.Error(), "internal_server_error") {
        return true
    }

    // Network timeouts
    if strings.Contains(err.Error(), "timeout") {
        return true
    }

    return false
}
```

---

## 🎯 What's Next

In the next intermediate tutorial, we'll explore **Terminal UI Framework** where we'll:
- Master the Bubble Tea component system
- Understand state management and event handling
- Learn about custom component development
- See how the chat interface is built and optimized

---

## 📝 Hands-On Exercises

1. **Custom Provider**: Implement a new AI provider for a service like Cohere or Together AI
2. **Tool Development**: Create a custom tool that integrates with your development workflow
3. **Prompt Optimization**: Modify the prompt templates to improve responses for your use case
4. **Context Enhancement**: Add new types of context gathering (git history, project structure, etc.)

---

**Previous**: [Intermediate 01: Database Architecture](01-database-architecture.md)
**Next**: [Intermediate 03: Terminal UI Framework](03-terminal-ui.md)

---

*🔧 **Pro Tip**: The AI integration layer is where the magic happens. Understanding this system deeply will help you customize Crush for any AI-powered workflow you can imagine.*
