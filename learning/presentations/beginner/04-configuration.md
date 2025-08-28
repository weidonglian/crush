# Tutorial 4: Configuration System

**Duration**: 20-25 minutes
**Goal**: Learn how Crush handles configuration and settings

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Understand the hierarchical configuration system
- Know how different config sources are merged
- Learn about JSON schema validation
- See how environment variables work
- Understand provider-specific settings

---

## 🔧 Configuration Philosophy

Crush follows the **"Convention over Configuration"** principle with smart defaults, but allows extensive customization when needed. The system is designed to:

1. **Work out of the box** - minimal required configuration
2. **Be flexible** - support project-specific and global settings
3. **Be secure** - handle API keys safely
4. **Be discoverable** - JSON schema provides autocomplete

---

## 📂 Configuration Sources & Priority

Configuration is loaded from multiple sources in this priority order:

```
1. Project-local files (.crush.json, crush.json)    [HIGHEST PRIORITY]
2. Global user config (~/.config/crush/crush.json)
3. Environment variables
4. Default values                                    [LOWEST PRIORITY]
```

### Let's Explore Each Source

#### 1. Project-Local Configuration

```bash
# Look for project config files in the crush repo
ls -la | grep -E "\.(crush\.json|crush\.json)"
```

These files live in your project root:
- `.crush.json` (preferred, hidden file)
- `crush.json` (alternative)

**Example project config**:
```json
{
  "$schema": "https://charm.land/crush.json",
  "lsp": {
    "go": {
      "command": "gopls",
      "env": {
        "GOTOOLCHAIN": "go1.25.0"
      }
    }
  },
  "providers": {
    "default": "anthropic"
  }
}
```

#### 2. Global User Configuration

```bash
# Check if you have a global config
ls -la ~/.config/crush/ 2>/dev/null || echo "No global config yet"
```

Located at: `~/.config/crush/crush.json`

**Example global config**:
```json
{
  "providers": {
    "anthropic": {
      "model": "claude-3-5-sonnet-20241022"
    },
    "openai": {
      "model": "gpt-4"
    }
  },
  "permissions": {
    "skipRequests": false,
    "allowedTools": ["file_read", "file_write"]
  }
}
```

#### 3. Environment Variables

Common environment variables:

```bash
# AI Provider API Keys
export ANTHROPIC_API_KEY="your-key-here"
export OPENAI_API_KEY="your-key-here"
export GROQ_API_KEY="your-key-here"

# Azure OpenAI
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_API_KEY="your-key"
export AZURE_OPENAI_API_VERSION="2024-02-01"

# AWS Bedrock
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="us-west-2"
```

---

## 🏗️ Configuration Loading Process

Let's examine how configuration is loaded:

```bash
# Look at the main config loading function
grep -A 20 "func Load" internal/config/load.go
```

**The Loading Process**:

```go
func Load(workingDir, dataDir string) (*Config, error) {
    // Step 1: Start with defaults
    cfg := NewConfig()

    // Step 2: Load global user config
    globalPath := filepath.Join(os.UserConfigDir(), "crush", "crush.json")
    if fileExists(globalPath) {
        if err := mergeFromFile(cfg, globalPath); err != nil {
            return nil, fmt.Errorf("loading global config: %w", err)
        }
    }

    // Step 3: Load project config (check both .crush.json and crush.json)
    projectConfigs := []string{
        filepath.Join(workingDir, ".crush.json"),
        filepath.Join(workingDir, "crush.json"),
    }

    for _, path := range projectConfigs {
        if fileExists(path) {
            if err := mergeFromFile(cfg, path); err != nil {
                return nil, fmt.Errorf("loading project config %s: %w", path, err)
            }
            break // Only load the first one found
        }
    }

    // Step 4: Apply environment variables
    applyEnvironmentVariables(cfg)

    // Step 5: Resolve paths and validate
    return resolve(cfg, workingDir, dataDir)
}
```

---

## 🔄 Configuration Merging

The merging strategy is **deep merge** - nested objects are merged recursively:

```bash
# Look at the merge logic
head -30 internal/config/merge.go
```

**Example of Deep Merging**:

Global config:
```json
{
  "providers": {
    "anthropic": {"model": "claude-3-sonnet"},
    "openai": {"model": "gpt-3.5-turbo"}
  },
  "lsp": {
    "go": {"command": "gopls"}
  }
}
```

Project config:
```json
{
  "providers": {
    "anthropic": {"model": "claude-3-5-sonnet"}
  },
  "lsp": {
    "typescript": {"command": "typescript-language-server"}
  }
}
```

**Final merged result**:
```json
{
  "providers": {
    "anthropic": {"model": "claude-3-5-sonnet"},  // Project override
    "openai": {"model": "gpt-3.5-turbo"}          // From global
  },
  "lsp": {
    "go": {"command": "gopls"},                    // From global
    "typescript": {"command": "typescript-language-server"}  // From project
  }
}
```

---

## 📋 Configuration Schema

Crush uses JSON Schema for validation and IDE autocomplete:

```bash
# Look at the schema file
head -20 schema.json
```

**Schema Benefits**:
1. **Validation**: Catches configuration errors early
2. **Autocomplete**: IDEs can provide suggestions
3. **Documentation**: Self-documenting configuration
4. **Type Safety**: Ensures correct value types

**Using the Schema in VS Code**:
```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    // VS Code will autocomplete available options here!
  }
}
```

---

## 🤖 Provider Configuration

Let's look at how different AI providers are configured:

```bash
# Examine provider configuration structure
grep -A 10 -B 5 "type.*Provider" internal/config/config.go
```

**Provider Configuration Structure**:

```go
type ProvidersConfig struct {
    Default    string                      `json:"default,omitempty"`
    Anthropic  *provider.AnthropicConfig   `json:"anthropic,omitempty"`
    OpenAI     *provider.OpenAIConfig      `json:"openai,omitempty"`
    Groq       *provider.GroqConfig        `json:"groq,omitempty"`
    Gemini     *provider.GeminiConfig      `json:"gemini,omitempty"`
    // ... more providers
}
```

**Example Provider Configurations**:

```json
{
  "providers": {
    "default": "anthropic",
    "anthropic": {
      "model": "claude-3-5-sonnet-20241022",
      "maxTokens": 4096,
      "temperature": 0.7
    },
    "openai": {
      "model": "gpt-4",
      "maxTokens": 4096,
      "temperature": 0.7,
      "baseURL": "https://api.openai.com/v1"
    },
    "groq": {
      "model": "llama-3.1-70b-versatile",
      "maxTokens": 2048
    }
  }
}
```

---

## 🔧 LSP Configuration

Language Server Protocol configuration is very flexible:

```json
{
  "lsp": {
    "go": {
      "command": "gopls",
      "args": ["-v"],
      "env": {
        "GOTOOLCHAIN": "go1.25.0",
        "GOPROXY": "direct"
      },
      "initializationOptions": {
        "usePlaceholders": true,
        "completeUnimported": true
      }
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "rootMarkers": ["package.json", "tsconfig.json"]
    },
    "python": {
      "command": "pylsp"
    },
    "rust": {
      "command": "rust-analyzer"
    }
  }
}
```

**LSP Configuration Options**:
- `command`: The LSP server executable
- `args`: Command-line arguments
- `env`: Environment variables for the LSP process
- `rootMarkers`: Files that indicate project root
- `initializationOptions`: LSP-specific settings

---

## 🛡️ Security and API Keys

### Environment Variable Handling

```bash
# Look at how environment variables are processed
grep -A 15 "applyEnvironmentVariables" internal/config/load.go
```

**Secure Practices**:

1. **API keys in environment variables** (never in config files)
2. **No default API keys** in the codebase
3. **Validation without exposure** (check if key exists without logging it)

### Example Environment Setup

```bash
# Create a .env file for development
cat > .env << 'EOF'
ANTHROPIC_API_KEY=your-anthropic-key-here
OPENAI_API_KEY=your-openai-key-here
GROQ_API_KEY=your-groq-key-here
EOF

# The app automatically loads .env files (thanks to godotenv/autoload)
```

---

## 🎯 Configuration Resolution

The final step is resolving relative paths and validating the config:

```bash
# Look at the resolve function
grep -A 20 "func resolve" internal/config/resolve.go
```

**Resolution Process**:

```go
func resolve(cfg *Config, workingDir, dataDir string) (*Config, error) {
    // 1. Set working directory
    if cfg.workingDir == "" {
        cfg.workingDir = workingDir
    }

    // 2. Set data directory (for database, logs, etc.)
    if dataDir != "" {
        cfg.dataDir = dataDir
    } else {
        cfg.dataDir = defaultDataDir()
    }

    // 3. Resolve LSP command paths
    for name, lspCfg := range cfg.LSP {
        if lspCfg.Command != "" {
            // Try to find the command in PATH
            if path, err := exec.LookPath(lspCfg.Command); err == nil {
                lspCfg.Command = path
            }
        }
    }

    // 4. Validate configuration
    if err := validate(cfg); err != nil {
        return nil, err
    }

    return cfg, nil
}
```

---

## 🔍 Configuration Debugging

You can inspect the loaded configuration:

```bash
# Use the schema command to see current config
# (This would show the merged configuration if crush was installed)
crush schema --show-config  # Hypothetical command
```

**Common Configuration Issues**:

1. **LSP not found**: Command not in PATH
2. **API key missing**: Environment variable not set
3. **Invalid JSON**: Syntax errors in config files
4. **Schema violations**: Invalid configuration values

---

## 🤔 Configuration Best Practices

### 1. **Use Project-Specific Config for Team Settings**
```json
{
  "lsp": {
    "go": {"command": "gopls"},
    "typescript": {"command": "typescript-language-server"}
  }
}
```

### 2. **Use Global Config for Personal Preferences**
```json
{
  "providers": {
    "default": "anthropic",
    "anthropic": {"temperature": 0.8}
  }
}
```

### 3. **Use Environment Variables for Secrets**
```bash
export ANTHROPIC_API_KEY="your-key"
# Never put API keys in config files!
```

### 4. **Commit Project Config, Not Personal Config**
```bash
# .gitignore
.env
.crush.json.local

# Do commit:
# .crush.json (team settings)
```

---

## 🎯 What's Next

In the next tutorial, we'll explore **Database and Persistence** where we'll:
- Understand the SQLite schema
- Learn about the migration system
- See how queries are generated with sqlc
- Explore the different data models

---

## 📝 Try This Yourself

Before moving to the next tutorial:

1. **Create a test config**: Make a `.crush.json` file with some LSP settings
2. **Explore the schema**: Look at `schema.json` to see all available options
3. **Check environment variables**: See what provider keys are detected
4. **Trace config loading**: Follow the code from `Load()` through `resolve()`

**Experiment**: Try creating both global and project configs and see how they merge!

---

**Previous**: [Tutorial 3: Entry Points and Bootstrap](03-entry-points.md)
**Next**: [Tutorial 5: Database and Persistence](05-database.md)

---

*💡 **Tip**: Configuration systems are often overlooked, but they're crucial for making applications flexible and user-friendly. Crush's system is a great example of thoughtful design!*
