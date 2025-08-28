# Intermediate 03: Terminal UI Framework

**Duration**: 35-40 minutes
**Level**: 🔧 Intermediate
**Goal**: Master the Bubble Tea UI system, component architecture, and state management

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand the Bubble Tea framework and its patterns
- Master component-based UI architecture
- Learn state management and event handling
- Understand how the chat interface is built
- Be able to create custom UI components

---

## 📋 Prerequisites

- ✅ Completed Beginner level tutorials
- ✅ Understanding of event-driven programming
- ✅ Familiarity with terminal capabilities and ANSI codes
- ✅ Experience with reactive UI patterns (helpful but not required)

---

## 🏗️ Bubble Tea Architecture in Crush

Crush uses [Bubble Tea](https://github.com/charmbracelet/bubbletea) v2, a powerful terminal UI framework based on The Elm Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                     Main TUI Program                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ App Model (Root State)                               │   │
│  │ - Current page                                       │   │
│  │ - Global state                                       │   │
│  │ - Component instances                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │ (contains)
┌─────────────────────▼───────────────────────────────────────┐
│                  Page Components                            │
│  ┌──────────────┬──────────────┬──────────────┬──────────┐  │
│  │ Chat Page    │ Session List │ Settings     │ Help     │  │
│  │ (main UI)    │ Page         │ Page         │ Page     │  │
│  └──────────────┴──────────────┴──────────────┴──────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │ (composed of)
┌─────────────────────▼───────────────────────────────────────┐
│                  UI Components                              │
│  ┌──────────────┬──────────────┬──────────────┬──────────┐  │
│  │ Message List │ Input Box    │ Status Bar   │ Spinner  │  │
│  │ Viewport     │ Component    │ Component    │ Component│  │
│  └──────────────┴──────────────┴──────────────┴──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 📱 Core Bubble Tea Patterns

### The Model-Update-View Pattern

Let's examine how Crush implements the core Bubble Tea pattern:

```bash
# Look at the main TUI model
head -50 internal/tui/tui.go
```

**Core TUI Structure**:

```go
// Main TUI model that holds all state
type Model struct {
    // Core application services
    app *app.App

    // UI state
    width, height int
    currentPage   page.Page
    pages         map[string]page.Page

    // Global UI state
    statusBar   statusbar.Model
    errorBanner error

    // Styling
    styles *styles.Styles

    // Component state
    focused     string
    initialized bool
}

// The Update function handles all messages/events
func (m *Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    var cmds []tea.Cmd

    // Handle global messages first
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height

        // Propagate resize to all pages
        for _, page := range m.pages {
            _, pageCmd := page.Update(msg)
            if pageCmd != nil {
                cmds = append(cmds, pageCmd)
            }
        }

    case tea.KeyMsg:
        // Global keybindings
        switch msg.String() {
        case "ctrl+c", "q":
            return m, tea.Quit
        case "ctrl+n":
            return m, m.switchToPage("new-session")
        case "ctrl+l":
            return m, m.switchToPage("session-list")
        }

        // Let current page handle the key
        if m.currentPage != nil {
            _, pageCmd := m.currentPage.Update(msg)
            if pageCmd != nil {
                cmds = append(cmds, pageCmd)
            }
        }

    case ErrorMsg:
        m.errorBanner = msg.Error
        cmds = append(cmds, m.clearErrorAfter(3*time.Second))

    case ClearErrorMsg:
        m.errorBanner = nil

    default:
        // Let current page handle other messages
        if m.currentPage != nil {
            _, pageCmd := m.currentPage.Update(msg)
            if pageCmd != nil {
                cmds = append(cmds, pageCmd)
            }
        }
    }

    return m, tea.Batch(cmds...)
}

// The View function renders the current state
func (m *Model) View() string {
    if !m.initialized {
        return "Initializing..."
    }

    var content string

    // Render current page
    if m.currentPage != nil {
        content = m.currentPage.View()
    } else {
        content = "No page loaded"
    }

    // Add status bar
    statusBar := m.statusBar.View()

    // Add error banner if present
    if m.errorBanner != nil {
        errorView := m.styles.ErrorBanner.Render(m.errorBanner.Error())
        content = errorView + "\n" + content
    }

    // Combine everything with proper sizing
    contentHeight := m.height - lipgloss.Height(statusBar)
    if m.errorBanner != nil {
        contentHeight-- // Account for error banner
    }

    contentView := lipgloss.NewStyle().
        Height(contentHeight).
        Render(content)

    return lipgloss.JoinVertical(lipgloss.Left, contentView, statusBar)
}
```

---

## 🧩 Component Architecture

### Reusable Component Design

Crush follows a component-based architecture where each UI element is self-contained:

```bash
# Examine the component structure
tree internal/tui/components/ -L 2
```

**Component Interface Pattern**:

```go
// Common interface for all UI components
type Component interface {
    // Bubble Tea integration
    Update(msg tea.Msg) (Component, tea.Cmd)
    View() string

    // Component lifecycle
    Init() tea.Cmd
    Focus() tea.Cmd
    Blur() tea.Cmd

    // State management
    SetSize(width, height int)
    Focused() bool
}

// Base component implementation
type BaseComponent struct {
    width, height int
    focused       bool
    styles        *styles.Styles
}

func (c *BaseComponent) SetSize(width, height int) {
    c.width = width
    c.height = height
}

func (c *BaseComponent) Focused() bool {
    return c.focused
}

func (c *BaseComponent) Focus() tea.Cmd {
    c.focused = true
    return nil
}

func (c *BaseComponent) Blur() tea.Cmd {
    c.focused = false
    return nil
}
```

### Chat Interface Components

Let's examine the main chat interface components:

#### 1. **Message List Component**

```bash
# Look at message list implementation
head -100 internal/tui/components/messagelist.go
```

```go
type MessageList struct {
    BaseComponent

    // Data
    messages []message.Message

    // UI state
    viewport     viewport.Model
    scrollOffset int

    // Rendering
    renderer *MessageRenderer

    // Auto-scroll behavior
    autoScroll    bool
    lastMsgCount  int
}

func (ml *MessageList) Update(msg tea.Msg) (Component, tea.Cmd) {
    var cmd tea.Cmd
    var cmds []tea.Cmd

    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        ml.SetSize(msg.Width, msg.Height)
        ml.viewport.Width = ml.width
        ml.viewport.Height = ml.height

    case tea.KeyMsg:
        if !ml.focused {
            return ml, nil
        }

        switch msg.String() {
        case "up", "k":
            ml.viewport.LineUp(1)
            ml.autoScroll = false

        case "down", "j":
            ml.viewport.LineDown(1)
            if ml.viewport.AtBottom() {
                ml.autoScroll = true
            }

        case "pgup":
            ml.viewport.HalfViewUp()
            ml.autoScroll = false

        case "pgdown":
            ml.viewport.HalfViewDown()
            if ml.viewport.AtBottom() {
                ml.autoScroll = true
            }

        case "home":
            ml.viewport.GotoTop()
            ml.autoScroll = false

        case "end":
            ml.viewport.GotoBottom()
            ml.autoScroll = true
        }

    case NewMessageMsg:
        ml.addMessage(msg.Message)

        // Auto-scroll to new messages
        if ml.autoScroll {
            cmds = append(cmds, ml.scrollToBottom())
        }

    case MessageUpdatedMsg:
        ml.updateMessage(msg.MessageID, msg.Message)
    }

    // Update viewport
    ml.viewport, cmd = ml.viewport.Update(msg)
    if cmd != nil {
        cmds = append(cmds, cmd)
    }

    return ml, tea.Batch(cmds...)
}

func (ml *MessageList) View() string {
    // Render all messages
    content := ml.renderMessages()

    // Set viewport content
    ml.viewport.SetContent(content)

    // Apply focus styling
    style := ml.styles.MessageList.Copy()
    if ml.focused {
        style = style.BorderStyle(lipgloss.DoubleBorder()).
               BorderForeground(ml.styles.FocusColor)
    }

    return style.Render(ml.viewport.View())
}

func (ml *MessageList) renderMessages() string {
    if len(ml.messages) == 0 {
        return ml.styles.EmptyState.Render("No messages yet. Start a conversation!")
    }

    var rendered []string

    for i, msg := range ml.messages {
        // Render individual message
        msgView := ml.renderer.RenderMessage(msg, i == len(ml.messages)-1)
        rendered = append(rendered, msgView)

        // Add separator between messages
        if i < len(ml.messages)-1 {
            rendered = append(rendered, ml.styles.MessageSeparator.Render(""))
        }
    }

    return strings.Join(rendered, "\n")
}
```

#### 2. **Input Component**

```go
type InputComponent struct {
    BaseComponent

    // Input handling
    textInput  textinput.Model
    multiLine  bool

    // History
    history    []string
    historyIdx int

    // State
    placeholder string
    submitKey   string
}

func (ic *InputComponent) Update(msg tea.Msg) (Component, tea.Cmd) {
    var cmd tea.Cmd
    var cmds []tea.Cmd

    switch msg := msg.(type) {
    case tea.KeyMsg:
        if !ic.focused {
            return ic, nil
        }

        switch msg.String() {
        case "enter":
            if ic.multiLine && !msg.Alt {
                // Insert newline in multi-line mode
                ic.textInput.SetValue(ic.textInput.Value() + "\n")
            } else {
                // Submit input
                value := strings.TrimSpace(ic.textInput.Value())
                if value != "" {
                    ic.addToHistory(value)
                    ic.textInput.SetValue("")
                    ic.historyIdx = len(ic.history)

                    return ic, func() tea.Msg {
                        return SubmitInputMsg{Content: value}
                    }
                }
            }

        case "up":
            if len(ic.history) > 0 && ic.historyIdx > 0 {
                ic.historyIdx--
                ic.textInput.SetValue(ic.history[ic.historyIdx])
            }

        case "down":
            if ic.historyIdx < len(ic.history)-1 {
                ic.historyIdx++
                ic.textInput.SetValue(ic.history[ic.historyIdx])
            } else if ic.historyIdx == len(ic.history)-1 {
                ic.historyIdx = len(ic.history)
                ic.textInput.SetValue("")
            }

        case "ctrl+u":
            // Clear current input
            ic.textInput.SetValue("")

        case "tab":
            // Auto-completion trigger
            return ic, func() tea.Msg {
                return AutoCompleteMsg{
                    CurrentText: ic.textInput.Value(),
                    CursorPos:   ic.textInput.Position(),
                }
            }
        }

    case AutoCompleteResultMsg:
        // Handle auto-completion results
        if msg.Suggestion != "" {
            current := ic.textInput.Value()
            newValue := current[:msg.StartPos] + msg.Suggestion + current[msg.EndPos:]
            ic.textInput.SetValue(newValue)
            ic.textInput.SetCursor(msg.StartPos + len(msg.Suggestion))
        }
    }

    // Update text input
    ic.textInput, cmd = ic.textInput.Update(msg)
    if cmd != nil {
        cmds = append(cmds, cmd)
    }

    return ic, tea.Batch(cmds...)
}

func (ic *InputComponent) View() string {
    // Input styling based on focus
    inputStyle := ic.styles.Input.Copy()
    if ic.focused {
        inputStyle = inputStyle.BorderStyle(lipgloss.RoundedBorder()).
                    BorderForeground(ic.styles.FocusColor)
    }

    // Multi-line indicator
    prompt := "❯ "
    if ic.multiLine {
        prompt = "│ "
    }

    inputView := prompt + ic.textInput.View()

    // Add placeholder if empty
    if ic.textInput.Value() == "" && ic.placeholder != "" {
        placeholderStyle := ic.styles.Placeholder
        inputView = prompt + placeholderStyle.Render(ic.placeholder)
    }

    return inputStyle.Render(inputView)
}
```

---

## 🎨 Styling and Theming System

### Centralized Style Management

```bash
# Examine the styling system
head -50 internal/tui/styles/styles.go
```

**Style Architecture**:

```go
type Styles struct {
    // Base colors
    Primary    lipgloss.AdaptiveColor
    Secondary  lipgloss.AdaptiveColor
    Accent     lipgloss.AdaptiveColor
    FocusColor lipgloss.AdaptiveColor
    ErrorColor lipgloss.AdaptiveColor

    // Component styles
    MessageList     lipgloss.Style
    MessageUser     lipgloss.Style
    MessageAssist   lipgloss.Style
    MessageSystem   lipgloss.Style

    Input           lipgloss.Style
    InputFocused    lipgloss.Style

    StatusBar       lipgloss.Style
    ErrorBanner     lipgloss.Style

    // Syntax highlighting
    CodeBlock       lipgloss.Style
    InlineCode      lipgloss.Style

    // Layout
    Sidebar         lipgloss.Style
    MainContent     lipgloss.Style
}

func NewStyles() *Styles {
    // Define adaptive colors that work in light and dark terminals
    primary := lipgloss.AdaptiveColor{
        Light: "#1f2937", // Dark gray for light terminals
        Dark:  "#f9fafb", // Light gray for dark terminals
    }

    accent := lipgloss.AdaptiveColor{
        Light: "#3b82f6", // Blue
        Dark:  "#60a5fa", // Lighter blue
    }

    return &Styles{
        Primary:    primary,
        Accent:     accent,
        FocusColor: accent,

        // Message styles with role-based coloring
        MessageUser: lipgloss.NewStyle().
            Foreground(primary).
            PaddingLeft(2).
            MarginBottom(1),

        MessageAssist: lipgloss.NewStyle().
            Foreground(accent).
            PaddingLeft(2).
            MarginBottom(1).
            Italic(true),

        MessageSystem: lipgloss.NewStyle().
            Foreground(lipgloss.AdaptiveColor{Light: "#6b7280", Dark: "#9ca3af"}).
            PaddingLeft(2).
            MarginBottom(1).
            Faint(true),

        // Input styling
        Input: lipgloss.NewStyle().
            Border(lipgloss.RoundedBorder()).
            BorderForeground(lipgloss.AdaptiveColor{Light: "#d1d5db", Dark: "#374151"}).
            Padding(0, 1),

        InputFocused: lipgloss.NewStyle().
            Border(lipgloss.DoubleBorder()).
            BorderForeground(accent).
            Padding(0, 1),

        // Code highlighting
        CodeBlock: lipgloss.NewStyle().
            Background(lipgloss.AdaptiveColor{Light: "#f3f4f6", Dark: "#1f2937"}).
            Foreground(lipgloss.AdaptiveColor{Light: "#1f2937", Dark: "#f9fafb"}).
            Padding(1, 2).
            MarginTop(1).
            MarginBottom(1),
    }
}
```

### Dynamic Theming

```go
// Theme switching capability
type ThemeManager struct {
    themes map[string]*Styles
    current string
}

func (tm *ThemeManager) SetTheme(name string) error {
    if theme, exists := tm.themes[name]; exists {
        tm.current = name
        // Broadcast theme change to all components
        return tm.broadcastThemeChange(theme)
    }
    return fmt.Errorf("theme %s not found", name)
}

// Responsive styling based on terminal capabilities
func (s *Styles) AdaptToTerminal() {
    profile := colorprofile.Detect()

    switch profile {
    case colorprofile.TrueColor:
        // Use full RGB colors
        s.Primary = lipgloss.Color("#1f2937")
        s.Accent = lipgloss.Color("#3b82f6")

    case colorprofile.ANSI256:
        // Use 256-color palette
        s.Primary = lipgloss.Color("240")
        s.Accent = lipgloss.Color("12")

    case colorprofile.ANSI:
        // Use basic 16 colors
        s.Primary = lipgloss.Color("8")
        s.Accent = lipgloss.Color("4")

    default:
        // Monochrome fallback
        s.Primary = lipgloss.NoColor{}
        s.Accent = lipgloss.NoColor{}
    }
}
```

---

## ⚡ Event System and State Management

### Custom Message Types

```go
// Custom messages for application-specific events
type (
    // AI interaction messages
    NewMessageMsg struct {
        Message message.Message
    }

    MessageUpdatedMsg struct {
        MessageID string
        Message   message.Message
    }

    StreamChunkMsg struct {
        SessionID string
        Content   string
        Done      bool
    }

    // UI interaction messages
    SubmitInputMsg struct {
        Content string
    }

    SwitchPageMsg struct {
        Page string
    }

    ShowErrorMsg struct {
        Error error
    }

    ClearErrorMsg struct{}

    // System messages
    LSPDiagnosticsMsg struct {
        File        string
        Diagnostics []lsp.Diagnostic
    }

    SessionCreatedMsg struct {
        Session session.Session
    }
)
```

### Command Pattern for Side Effects

```go
// Commands for handling side effects
func SubmitMessageCmd(sessionID, content string) tea.Cmd {
    return func() tea.Msg {
        // This runs in a goroutine
        response, err := aiService.ProcessMessage(context.Background(), sessionID, content)
        if err != nil {
            return ShowErrorMsg{Error: err}
        }

        return NewMessageMsg{Message: response}
    }
}

func StreamMessageCmd(sessionID, content string) tea.Cmd {
    return func() tea.Msg {
        stream, err := aiService.StreamMessage(context.Background(), sessionID, content)
        if err != nil {
            return ShowErrorMsg{Error: err}
        }

        // Return a command that produces multiple messages
        return tea.Sequence(
            func() tea.Msg {
                for chunk := range stream {
                    // Each chunk becomes a message
                    return StreamChunkMsg{
                        SessionID: sessionID,
                        Content:   chunk.Content,
                        Done:      chunk.Done,
                    }
                }
                return nil
            },
        )
    }
}

// Batch commands for complex operations
func CreateSessionAndSwitchCmd(name, provider, model string) tea.Cmd {
    return tea.Sequence(
        func() tea.Msg {
            session, err := sessionService.Create(context.Background(), name, provider, model)
            if err != nil {
                return ShowErrorMsg{Error: err}
            }
            return SessionCreatedMsg{Session: session}
        },
        func() tea.Msg {
            return SwitchPageMsg{Page: "chat"}
        },
    )
}
```

---

## 🚀 Advanced UI Patterns

### Virtualized Lists for Performance

```go
// Efficient rendering for large message lists
type VirtualizedMessageList struct {
    BaseComponent

    messages     []message.Message
    viewport     viewport.Model

    // Virtualization
    itemHeight   int
    visibleStart int
    visibleEnd   int

    // Caching
    renderedCache map[int]string
    cacheSize     int
}

func (vml *VirtualizedMessageList) calculateVisibleRange() {
    if vml.itemHeight == 0 {
        return
    }

    topOffset := vml.viewport.YOffset
    bottomOffset := topOffset + vml.viewport.Height

    vml.visibleStart = max(0, topOffset/vml.itemHeight-1) // Buffer above
    vml.visibleEnd = min(len(vml.messages), bottomOffset/vml.itemHeight+1) // Buffer below
}

func (vml *VirtualizedMessageList) renderVisible() string {
    vml.calculateVisibleRange()

    var rendered []string

    // Add spacer for items above visible range
    if vml.visibleStart > 0 {
        spacerHeight := vml.visibleStart * vml.itemHeight
        spacer := strings.Repeat("\n", spacerHeight)
        rendered = append(rendered, spacer)
    }

    // Render visible items
    for i := vml.visibleStart; i < vml.visibleEnd; i++ {
        if i >= len(vml.messages) {
            break
        }

        // Check cache first
        if cached, exists := vml.renderedCache[i]; exists {
            rendered = append(rendered, cached)
        } else {
            // Render and cache
            msgView := vml.renderMessage(vml.messages[i])
            vml.cacheMessage(i, msgView)
            rendered = append(rendered, msgView)
        }
    }

    // Add spacer for items below visible range
    remaining := len(vml.messages) - vml.visibleEnd
    if remaining > 0 {
        spacerHeight := remaining * vml.itemHeight
        spacer := strings.Repeat("\n", spacerHeight)
        rendered = append(rendered, spacer)
    }

    return strings.Join(rendered, "\n")
}
```

### Progressive Enhancement

```go
// Feature detection and progressive enhancement
type FeatureDetector struct {
    supportsColor     bool
    supportsMouse     bool
    supportsClipboard bool
    terminalSize      tea.WindowSizeMsg
}

func (fd *FeatureDetector) Detect() {
    // Color support
    profile := colorprofile.Detect()
    fd.supportsColor = profile != colorprofile.Ascii

    // Mouse support
    fd.supportsMouse = os.Getenv("TERM_PROGRAM") != "Apple_Terminal"

    // Clipboard support (platform-specific)
    fd.supportsClipboard = fd.detectClipboard()
}

func (m *Model) adaptToCapabilities(detector *FeatureDetector) {
    if !detector.supportsColor {
        // Use monochrome theme
        m.styles = NewMonochromeStyles()
    }

    if detector.supportsMouse {
        // Enable mouse interactions
        m.enableMouseMode()
    }

    if detector.supportsClipboard {
        // Add copy functionality
        m.enableCopyFeatures()
    }

    // Adapt layout to terminal size
    if detector.terminalSize.Width < 80 {
        m.useCompactLayout()
    }
}
```

---

## 🎯 What's Next

In the next intermediate tutorial, we'll explore **LSP Integration** where we'll:
- Understand the Language Server Protocol implementation
- Learn about client management and communication
- See how code context is gathered and used
- Master debugging and troubleshooting LSP issues

---

## 📝 Hands-On Exercises

1. **Custom Component**: Create a new UI component (e.g., a file browser or settings panel)
2. **Theme Development**: Design and implement a custom color theme
3. **Animation**: Add smooth animations for state transitions
4. **Accessibility**: Implement keyboard navigation improvements
5. **Performance**: Optimize rendering for very long conversations

---

**Previous**: [Intermediate 02: AI Integration Deep Dive](02-ai-integration.md)
**Next**: [Intermediate 04: LSP Integration](04-lsp-integration.md)

---

*🎨 **UI Insight**: Terminal UIs are making a comeback! The patterns you learn here apply to modern reactive UI frameworks - it's all about state management and unidirectional data flow.*
