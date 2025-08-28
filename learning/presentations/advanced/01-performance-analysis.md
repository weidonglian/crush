# Advanced 01: Performance Analysis & Optimization

**Duration**: 45-50 minutes
**Level**: 🚀 Advanced
**Goal**: Master performance analysis, optimization strategies, and scalability patterns

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Master Go profiling tools and techniques for Crush
- Identify and resolve performance bottlenecks across all system layers
- Understand memory management and garbage collection impact
- Design scalability improvements and optimization strategies
- Implement advanced monitoring and observability patterns

---

## 📋 Prerequisites

- ✅ Completed Intermediate level tutorials
- ✅ Expert-level Go programming with performance optimization experience
- ✅ Deep understanding of computer systems, memory management, and concurrency
- ✅ Experience with profiling tools (pprof, benchmarking, tracing)
- ✅ Knowledge of database performance tuning and system architecture

---

## 🔬 Performance Analysis Foundation

### Built-in Profiling Infrastructure

Crush includes sophisticated profiling capabilities:

```go
// main.go - Production-ready profiling
if os.Getenv("CRUSH_PROFILE") != "" {
    go func() {
        slog.Info("Serving pprof at localhost:6060")
        if httpErr := http.ListenAndServe("localhost:6060", nil); httpErr != nil {
            slog.Error("Failed to pprof listen", "error", httpErr)
        }
    }()
}
```

**Advanced Profiling Setup**:

```bash
# Enable profiling with custom port
export CRUSH_PROFILE=1
export CRUSH_PROFILE_PORT=6061

# Start crush with profiling
crush &

# Collect CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Collect memory profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Collect goroutine analysis
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Collect mutex contention
go tool pprof http://localhost:6060/debug/pprof/mutex

# Collect allocation profile
go tool pprof http://localhost:6060/debug/pprof/allocs
```

---

## 🔍 System-Wide Performance Characteristics

### Application Startup Analysis

**Startup Performance Bottlenecks**:

1. **Configuration Loading**: File I/O and JSON parsing
2. **Database Connection**: SQLite file access and migrations
3. **LSP Client Initialization**: Process spawning and communication setup
4. **AI Provider Authentication**: Network requests and token validation
5. **TUI Initialization**: Terminal capability detection and rendering setup

**Measurement Strategy**:

```go
// Add timing instrumentation
func instrumentedStartup() {
    start := time.Now()
    defer func() {
        slog.Info("Total startup time", "duration", time.Since(start))
    }()

    // Config loading
    configStart := time.Now()
    cfg, err := config.Load(workingDir, dataDir)
    slog.Info("Config loaded", "duration", time.Since(configStart))

    // Database setup
    dbStart := time.Now()
    conn, err := db.Connect(ctx, dbPath)
    slog.Info("Database connected", "duration", time.Since(dbStart))

    // LSP initialization
    lspStart := time.Now()
    app.initLSPClients(ctx)
    slog.Info("LSP clients initialized", "duration", time.Since(lspStart))
}
```

### Runtime Performance Patterns

**Hot Paths Analysis**:

```bash
# Profile a typical user session
go tool pprof -top http://localhost:6060/debug/pprof/profile?seconds=60
```

**Expected Top Functions**:
1. **JSON marshaling/unmarshaling** (AI provider communication)
2. **SQLite query execution** (message persistence)
3. **Terminal rendering** (TUI updates)
4. **LSP communication** (language server protocol)
5. **String processing** (prompt building, content parsing)

---

## 🚀 Memory Management Deep Dive

### Memory Allocation Patterns

**High-Allocation Components**:

```go
// AI Response Processing - High allocation area
func processAIResponse(response *openai.ChatCompletion) (*message.Message, error) {
    // String concatenation in loops - potential optimization
    var content strings.Builder  // ✅ Better than string concatenation
    for _, choice := range response.Choices {
        content.WriteString(choice.Message.Content)
    }

    // JSON marshaling - memory intensive
    attachments := make([]message.Attachment, 0, len(files))  // ✅ Pre-allocate
    for _, file := range files {
        data, err := os.ReadFile(file.Path)  // ❌ Loads entire file into memory
        if err != nil {
            return nil, err
        }
        attachments = append(attachments, message.Attachment{
            Name: file.Name,
            Data: data,  // Consider streaming for large files
        })
    }

    return &message.Message{
        Content:     content.String(),
        Attachments: attachments,
    }, nil
}
```

**Memory Optimization Strategies**:

#### 1. **Object Pooling for Frequent Allocations**

```go
// Pool for JSON encoding buffers
var jsonBufferPool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func optimizedJSONMarshal(v interface{}) ([]byte, error) {
    buf := jsonBufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        jsonBufferPool.Put(buf)
    }()

    encoder := json.NewEncoder(buf)
    if err := encoder.Encode(v); err != nil {
        return nil, err
    }

    // Copy to avoid referencing pooled buffer
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result, nil
}
```

#### 2. **Streaming for Large Content**

```go
// Stream large file attachments instead of loading into memory
func streamFileAttachment(w io.Writer, filePath string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return err
    }
    defer file.Close()

    // Use buffered copy to control memory usage
    _, err = io.CopyBuffer(w, file, make([]byte, 64*1024))  // 64KB buffer
    return err
}
```

#### 3. **Context-Aware Memory Management**

```go
// Implement memory pressure awareness
func (s *MessageService) processLargeContext(ctx context.Context, messages []Message) error {
    // Monitor memory usage
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    if m.Alloc > maxMemoryThreshold {
        // Trigger garbage collection
        runtime.GC()

        // Use chunked processing
        return s.processMessagesInChunks(ctx, messages, chunkSize)
    }

    return s.processMessagesNormally(ctx, messages)
}
```

---

## ⚡ Database Performance Optimization

### Query Performance Analysis

**SQLite Performance Monitoring**:

```go
// Custom database driver with query timing
type instrumentedConn struct {
    *sql.DB
}

func (c *instrumentedConn) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
    start := time.Now()
    result, err := c.DB.ExecContext(ctx, query, args...)
    duration := time.Since(start)

    if duration > slowQueryThreshold {
        slog.Warn("Slow query detected",
            "query", query,
            "duration", duration,
            "args", args)
    }

    return result, err
}
```

**Index Optimization Strategies**:

```sql
-- Analyze query patterns
EXPLAIN QUERY PLAN
SELECT m.* FROM messages m
JOIN sessions s ON m.session_id = s.id
WHERE s.updated_at > datetime('now', '-1 day')
ORDER BY m.created_at DESC
LIMIT 50;

-- Add composite indexes for common query patterns
CREATE INDEX idx_messages_session_created
ON messages(session_id, created_at DESC);

CREATE INDEX idx_sessions_updated_provider
ON sessions(updated_at DESC, provider);

-- Partial indexes for specific conditions
CREATE INDEX idx_messages_user_recent
ON messages(session_id, created_at DESC)
WHERE role = 'user';
```

**Connection Pool Optimization**:

```go
// Optimized connection configuration
func configureDB(conn *sql.DB) {
    // Connection pool settings
    conn.SetMaxOpenConns(25)                // Limit concurrent connections
    conn.SetMaxIdleConns(5)                 // Keep some connections warm
    conn.SetConnMaxLifetime(5 * time.Minute) // Rotate connections

    // SQLite-specific optimizations
    conn.Exec("PRAGMA journal_mode = WAL")     // Better concurrency
    conn.Exec("PRAGMA synchronous = NORMAL")   // Balanced durability/performance
    conn.Exec("PRAGMA cache_size = -64000")    // 64MB cache
    conn.Exec("PRAGMA temp_store = MEMORY")    // Temp tables in memory
    conn.Exec("PRAGMA mmap_size = 268435456")  // 256MB memory mapping
}
```

---

## 🔀 Concurrency Performance Optimization

### Goroutine Management Analysis

**Goroutine Leak Detection**:

```go
// Monitor goroutine growth
func monitorGoroutines() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            count := runtime.NumGoroutine()
            if count > goroutineThreshold {
                slog.Warn("High goroutine count detected", "count", count)

                // Collect goroutine profile
                buf := &bytes.Buffer{}
                pprof.Lookup("goroutine").WriteTo(buf, 1)
                // Analyze or log the profile
            }
        }
    }
}
```

**Event System Optimization**:

```go
// Optimized event processing with backpressure
func (a *App) optimizedEventLoop() {
    eventBuffer := make([]tea.Msg, 0, 100)
    ticker := time.NewTicker(16 * time.Millisecond)  // ~60fps processing

    for {
        select {
        case event := <-a.events:
            eventBuffer = append(eventBuffer, event)

            // Batch process when buffer is full or on ticker
            if len(eventBuffer) >= cap(eventBuffer) {
                a.processBatchedEvents(eventBuffer)
                eventBuffer = eventBuffer[:0]  // Reset slice
            }

        case <-ticker.C:
            if len(eventBuffer) > 0 {
                a.processBatchedEvents(eventBuffer)
                eventBuffer = eventBuffer[:0]
            }

        case <-a.ctx.Done():
            return
        }
    }
}
```

### LSP Client Performance

**Efficient LSP Communication**:

```go
// Connection pooling for LSP clients
type LSPClientPool struct {
    clients map[string]*lsp.Client
    mu      sync.RWMutex

    // Metrics
    requestCounts map[string]int64
    responseTimes map[string]time.Duration
}

func (p *LSPClientPool) GetClient(language string) (*lsp.Client, error) {
    p.mu.RLock()
    client, exists := p.clients[language]
    p.mu.RUnlock()

    if exists && client.IsHealthy() {
        return client, nil
    }

    // Create new client with optimization
    return p.createOptimizedClient(language)
}

func (p *LSPClientPool) createOptimizedClient(language string) (*lsp.Client, error) {
    config := &lsp.ClientConfig{
        BufferSize:    64 * 1024,          // Large buffer for responses
        Timeout:       5 * time.Second,     // Reasonable timeout
        MaxConcurrent: 10,                  // Limit concurrent requests

        // Custom transport with compression
        Transport: &lsp.CompressedTransport{
            CompressionLevel: 6,  // Balance compression vs CPU
        },
    }

    return lsp.NewClient(language, config)
}
```

---

## 📊 Terminal UI Performance

### Rendering Optimization

**Efficient TUI Updates**:

```go
// Differential rendering to minimize terminal writes
type OptimizedRenderer struct {
    lastFrame    []byte
    dirtyRegions []Region
    frameBuffer  *bytes.Buffer
}

func (r *OptimizedRenderer) Render(model tea.Model) {
    // Build new frame
    r.frameBuffer.Reset()
    view := model.View()
    r.frameBuffer.WriteString(view)

    currentFrame := r.frameBuffer.Bytes()

    // Calculate diff from last frame
    diffs := r.calculateDiffs(r.lastFrame, currentFrame)

    // Only render changed regions
    for _, diff := range diffs {
        r.renderRegion(diff)
    }

    // Update frame buffer
    copy(r.lastFrame, currentFrame)
}

// Use ANSI escape sequences for efficient cursor movement
func (r *OptimizedRenderer) renderRegion(region DiffRegion) {
    // Move cursor to region start
    fmt.Printf("\x1b[%d;%dH", region.StartRow, region.StartCol)

    // Write only changed content
    os.Stdout.Write(region.Content)
}
```

**Input Processing Optimization**:

```go
// Debounced input processing
type DebouncedInput struct {
    lastInput time.Time
    buffer    []rune
    callback  func(string)
    delay     time.Duration
}

func (d *DebouncedInput) ProcessKey(r rune) {
    d.buffer = append(d.buffer, r)
    d.lastInput = time.Now()

    // Debounce rapid keystrokes
    time.AfterFunc(d.delay, func() {
        if time.Since(d.lastInput) >= d.delay {
            input := string(d.buffer)
            d.buffer = d.buffer[:0]  // Reset buffer
            d.callback(input)
        }
    })
}
```

---

## 🎯 AI Provider Performance

### Request Optimization

**Connection Pooling and Reuse**:

```go
// Optimized HTTP client for AI providers
func createOptimizedHTTPClient() *http.Client {
    transport := &http.Transport{
        // Connection pooling
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,

        // Timeouts
        DialTimeout:         10 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
        ResponseHeaderTimeout: 30 * time.Second,

        // Keep-alive
        DisableKeepAlives: false,

        // Compression
        DisableCompression: false,
    }

    return &http.Client{
        Transport: transport,
        Timeout:   60 * time.Second,
    }
}
```

**Request Batching and Caching**:

```go
// Intelligent request batching
type RequestBatcher struct {
    pending   []*AIRequest
    timer     *time.Timer
    batchSize int
    timeout   time.Duration
}

func (b *RequestBatcher) AddRequest(req *AIRequest) {
    b.pending = append(b.pending, req)

    // Batch when full or after timeout
    if len(b.pending) >= b.batchSize {
        b.processBatch()
    } else if b.timer == nil {
        b.timer = time.AfterFunc(b.timeout, b.processBatch)
    }
}

// Response caching with LRU eviction
type ResponseCache struct {
    cache *lru.Cache
    ttl   time.Duration
}

func (c *ResponseCache) Get(key string) (*AIResponse, bool) {
    if item, found := c.cache.Get(key); found {
        cached := item.(*CachedResponse)
        if time.Since(cached.Timestamp) < c.ttl {
            return cached.Response, true
        }
        c.cache.Remove(key)  // Expired
    }
    return nil, false
}
```

---

## 📈 Performance Monitoring & Observability

### Custom Metrics Collection

```go
// Comprehensive metrics system
type MetricsCollector struct {
    // Request metrics
    requestCounts    *prometheus.CounterVec
    requestDurations *prometheus.HistogramVec

    // Database metrics
    queryDurations   *prometheus.HistogramVec
    connectionPool   prometheus.Gauge

    // Memory metrics
    heapSize         prometheus.Gauge
    goroutineCount   prometheus.Gauge

    // Custom business metrics
    sessionCount     prometheus.Gauge
    messageRate      *prometheus.CounterVec
}

func (m *MetricsCollector) RecordRequestDuration(provider, model string, duration time.Duration) {
    m.requestDurations.WithLabelValues(provider, model).Observe(duration.Seconds())
    m.requestCounts.WithLabelValues(provider, model).Inc()
}

// Runtime metrics collection
func (m *MetricsCollector) collectRuntimeMetrics() {
    go func() {
        ticker := time.NewTicker(10 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                var stats runtime.MemStats
                runtime.ReadMemStats(&stats)

                m.heapSize.Set(float64(stats.HeapAlloc))
                m.goroutineCount.Set(float64(runtime.NumGoroutine()))

            case <-m.ctx.Done():
                return
            }
        }
    }()
}
```

### Performance Testing Framework

```go
// Benchmark suite for critical paths
func BenchmarkMessageProcessing(b *testing.B) {
    app := setupTestApp()
    messages := generateTestMessages(1000)

    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        for _, msg := range messages {
            app.ProcessMessage(context.Background(), msg)
        }
    }
}

// Load testing with realistic scenarios
func TestConcurrentSessions(t *testing.T) {
    const numSessions = 100
    const messagesPerSession = 50

    app := setupTestApp()
    var wg sync.WaitGroup

    start := time.Now()

    for i := 0; i < numSessions; i++ {
        wg.Add(1)
        go func(sessionID int) {
            defer wg.Done()

            session := createTestSession(sessionID)
            for j := 0; j < messagesPerSession; j++ {
                msg := createTestMessage(sessionID, j)
                app.ProcessMessage(context.Background(), msg)
            }
        }(i)
    }

    wg.Wait()
    duration := time.Since(start)

    t.Logf("Processed %d sessions with %d messages each in %v",
        numSessions, messagesPerSession, duration)

    // Assert performance requirements
    maxDuration := 30 * time.Second
    if duration > maxDuration {
        t.Errorf("Performance test exceeded maximum duration: %v > %v",
            duration, maxDuration)
    }
}
```

---

## 🎯 Scalability Design Patterns

### Horizontal Scaling Preparation

```go
// Design for distributed deployment
type DistributedConfig struct {
    NodeID          string
    RedisURL        string  // Shared state
    DatabaseShards  []string // Database partitioning
    LoadBalancerURL string
}

// Session affinity for stateful components
type SessionRouter struct {
    nodes map[string]*Node
    hash  consistent.Hash
}

func (r *SessionRouter) RouteSession(sessionID string) *Node {
    nodeKey := r.hash.Get(sessionID)
    return r.nodes[nodeKey]
}
```

### Resource Management

```go
// Adaptive resource management
type ResourceManager struct {
    cpuQuota    float64
    memoryLimit int64

    currentCPU    float64
    currentMemory int64
}

func (r *ResourceManager) ShouldThrottle() bool {
    return r.currentCPU > r.cpuQuota*0.8 ||
           r.currentMemory > r.memoryLimit*0.9
}

func (r *ResourceManager) AdaptiveSleep() {
    if r.ShouldThrottle() {
        // Exponential backoff under load
        time.Sleep(time.Duration(r.currentCPU/10) * time.Millisecond)
    }
}
```

---

## 🎯 What's Next

In the next advanced tutorial, we'll explore **Extension Architecture** where we'll:
- Design and implement MCP (Model Context Protocol) extensions
- Build custom tools and integrations
- Create plugin systems and APIs
- Understand security boundaries and sandboxing

---

## 📝 Expert-Level Exercises

1. **Performance Audit**: Conduct a complete performance audit of a running Crush instance
2. **Custom Profiler**: Implement domain-specific profiling for AI request patterns
3. **Load Testing**: Design and implement realistic load testing scenarios
4. **Memory Optimization**: Identify and fix memory leaks in long-running sessions
5. **Scalability Plan**: Design a plan for handling 10x current load

---

**Previous**: [Intermediate Level Tutorials](../intermediate/)
**Next**: [Advanced 02: Extension Architecture](02-extension-architecture.md)

---

*🚀 **Expert Insight**: Performance optimization is an art that combines deep technical knowledge with real-world constraints. The patterns shown here are used in production systems handling millions of requests.*
