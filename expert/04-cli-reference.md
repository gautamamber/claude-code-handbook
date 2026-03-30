# Expert 04 — Complete CLI Reference

---

## Command Structure

```
  claude [options] [prompt]

  claude [subcommand] [subcommand-options]
```

---

## All Flags — Organized by Category

### Session Management

```
  -p, --print                  Headless/non-interactive mode
  -c, --continue               Resume most recent session
  -r, --resume <id|name>       Resume specific session by ID or name
  -n, --name <name>            Set display name for this session
      --fork-session           Start new session ID when resuming
      --session-id <uuid>      Use specific UUID (must be UUID format)
      --init                   Run SessionStart hooks, start interactive
      --init-only              Run SessionStart hooks, then exit
```

### Context and Working Directory

```
      --add-dir <paths>        Add additional working directories
                               (comma or space separated)
                               Example: --add-dir ../lib ../apps
      --setting-sources <list> Load only specified config sources
                               Values: user,project,local
      --settings <file|json>   Load custom settings file or inline JSON
      --bare                   Skip ALL auto-discovery:
                               no CLAUDE.md, hooks, skills, MCP, plugins
      --plugin-dir <path>      Load plugin from path (repeatable)
      --disable-slash-commands Disable all skills and slash commands
```

### Tools and Permissions

```
      --allowedTools <list>    Auto-approve these tools (no prompts)
                               Format: "Bash,Read,Edit"
                               Fine-grained: "Bash(npm *),Edit(*.ts)"
      --disallowedTools <list> Block these tools entirely
      --tools <list>           Restrict to ONLY these tools
      --permission-mode <mode> Start in specific mode:
                               default|acceptEdits|plan|auto|dontAsk|bypassPermissions
      --dangerously-skip-permissions
                               Skip ALL permission checks
      --allow-dangerously-skip-permissions
                               Enable bypass without activating it
                               (lets agent/skill activate it later)
```

### Model and Performance

```
      --model <name>           Use model: sonnet|opus|haiku
                               Or full ID: claude-sonnet-4-6
      --effort <level>         Thinking depth: low|medium|high|max
                               (Opus only, ignored for other models)
      --enable-auto-mode       Add auto mode to Shift+Tab cycle
      --fallback-model <model> Fallback when primary overloaded
                               (headless mode only)
```

### System Prompt

```
      --system-prompt <text>   REPLACE entire default system prompt
      --system-prompt-file <f> Load replacement from file
      --append-system-prompt <text>
                               APPEND to default system prompt
      --append-system-prompt-file <f>
                               Append file contents to system prompt
```

### Agents and Workflow

```
      --agent <name>           Run as specific agent from .claude/agents/
      --agents <json>          Define agents inline (session-only, JSON)
  -w, --worktree <name>        Start in git worktree
                               Creates: .claude/worktrees/<name>/
      --tmux                   Create tmux session for worktree
      --teammate-mode <mode>   How teammates display:
                               auto|in-process|tmux
```

### Output and Formatting

```
      --output-format <fmt>    text (default) | json | stream-json
      --json-schema <schema>   Enforce JSON output schema
                               (requires --output-format json)
      --input-format <fmt>     text (default) | stream-json
      --include-partial-messages
                               Include streaming tokens in stream-json
      --verbose                Show detailed turn-by-turn output
                               Also toggleable in session: Ctrl+O
```

### Cost and Limits

```
      --max-budget-usd <n>     Stop if cost exceeds $n
                               (headless mode only)
      --max-turns <n>          Limit agentic loop iterations
                               (headless mode only)
      --no-session-persistence Don't write session to disk
                               (headless mode only)
```

### Remote and Web

```
      --remote <description>   Create web session on claude.ai
      --teleport               Resume web session in terminal
      --remote-control <name>  Enable remote control from claude.ai
      --rc <name>              Short form of --remote-control
      --chrome                 Enable Chrome browser automation
      --no-chrome              Disable Chrome for this session
```

### MCP Configuration

```
      --mcp-config <file|json> Load MCP servers from file or inline JSON
      --strict-mcp-config      Use ONLY --mcp-config, ignore all others
      --channels <servers>     Listen to MCP channel notifications
      --dangerously-load-development-channels
                               Load unapproved channels (dev only)
      --permission-prompt-tool <mcp-tool>
                               Use MCP tool to handle permission prompts
```

### Debugging and Development

```
      --debug [categories]     Enable debug logging
                               Optional filter: "api,mcp,hooks"
                               Exclude: "!statsig,!file"
      --maintenance            Run maintenance hooks and exit
      --betas <headers>        Add beta API headers
                               Example: "interleaved-thinking"
      --ide                    Auto-connect to running IDE
      --from-pr <number|url>   Resume sessions from GitHub PR
  -v, --version                Print version and exit
  -h, --help                   Show help
```

---

## Subcommands

### Authentication

```bash
claude auth login              # Interactive login (browser OAuth)
claude auth login --email E    # Login with specific email
claude auth login --sso        # SSO login
claude auth login --console    # Console/API key login
claude auth logout             # Sign out
claude auth status             # Check auth status
claude auth status --text      # Human-readable format
```

### Agents

```bash
claude agents                  # List all configured agents
claude agents list             # Same as above
```

### Auto Mode

```bash
claude auto-mode defaults      # Print built-in auto mode rules
claude auto-mode config        # Show effective auto mode config
```

### MCP Servers

```bash
claude mcp list                        # List all MCP servers
claude mcp get <name>                  # Show server details
claude mcp add <name> -- <cmd> [args]  # Add stdio server
claude mcp add --transport http <name> --url <url>
claude mcp add --transport sse <name> --url <url>
claude mcp remove <name>               # Remove server
claude mcp reset-project-choices       # Clear project approvals
```

### Plugins

```bash
claude plugin install <name>           # Install from marketplace
claude plugin install <name>@<market>  # Install from specific marketplace
claude plugin uninstall <name>         # Remove plugin
claude plugin enable <name>            # Enable disabled plugin
claude plugin disable <name>           # Disable without removing
claude plugin update <name>            # Update to latest
claude plugin list                     # List all plugins
```

### Other

```bash
claude remote-control          # Start remote control server
claude update                  # Upgrade to latest version
```

---

## All In-Session Slash Commands

### Context & Memory

```
/clear                         Start fresh session (history preserved for resume)
/compact [focus on <topic>]    Compress conversation history
/context                       Show context window breakdown + token counts
/cost                          Show token usage and estimated USD cost
/memory                        View and edit auto memory files
/claude.md                     Create or edit project CLAUDE.md
/status                        Session info: model, auth, permissions, session ID
```

### Configuration

```
/config                        Interactive settings menu
/model                         Switch model (sonnet/opus/haiku picker)
/effort                        Set effort level (low/medium/high/max)
/vim                           Toggle vim keybindings mode
/terminal-setup                Configure Shift+Enter multiline, Meta keys
/keybindings                   Open keybindings.json in editor
/theme                         Pick color theme
/statusline                    Configure status bar display
```

### Agents & Skills

```
/agents                        Manage agents (create, edit, delete)
/skills                        List available skills
/reload-plugins                Reload all plugins (after edits)
/hooks                         View active hooks configuration
/permissions                   View/edit permission settings
/auto-mode                     Review auto mode classifier rules
```

### Workflow

```
/checkpoint                    Create restore point (code + conversation)
/rewind                        Restore to checkpoint
/rename <name>                 Rename current session
/resume                        Interactive session picker
/session                       Show session ID and metadata
```

### Integrations

```
/mcp                           Manage MCP servers (list, auth, add, remove)
/plugin                        Manage plugins
/git                           Git workflow commands reference
/sandbox                       Toggle filesystem sandboxing
```

### Help & Diagnostics

```
/help [topic]                  Help (all commands, or specific topic)
/doctor                        Run diagnostics (auth, paths, keybindings)
/feedback                      Send feedback to Anthropic
/timeout <seconds>             Set MCP server connection timeout
/btw <question>                Ask side question without adding to context
```

---

## Key Flag Combinations

```bash
# Fastest possible call (minimal context, cheapest model)
claude -p "quick task" --bare --model haiku --max-turns 2

# Full CI mode (deterministic, no UI, with cost limit)
claude -p "task" \
  --bare \
  --model sonnet \
  --allowedTools "Read,Edit,Bash" \
  --max-turns 20 \
  --max-budget-usd 5.00 \
  --output-format json

# Plan before acting (safe mode)
claude --permission-mode plan "refactor auth module"

# Read-only exploration
claude --allowedTools "Read,Grep,Glob" "understand how auth works"

# Interactive with auto-approve edits but prompt for bash
claude --permission-mode acceptEdits

# Debug API calls
claude --debug api "why is this not working"

# Custom agent with worktree isolation
claude --agent security-auditor -w security-branch

# Extend existing session with new context
claude --resume <id> --add-dir ../shared-lib
```

---

## Exit Codes

```
  0    Success
  1    General error (bad arguments, config error)
  2    Tool blocked by hook (when using -p)
  3    Cost limit exceeded (--max-budget-usd)
  4    Turn limit exceeded (--max-turns)
  5    Authentication error
  130  Interrupted (Ctrl+C)
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────┐
│  CLI REFERENCE CHEATSHEET                                      │
├────────────────────────────────────────────────────────────────┤
│  Most used      : -p, -c, --model, --allowedTools              │
│  CI essentials  : --bare, --output-format json, --max-turns    │
│  Debug          : --debug, --verbose, Ctrl+O                   │
│  Safety         : --permission-mode plan, --max-budget-usd     │
│  Auth           : claude auth login/logout/status              │
│  MCP            : claude mcp add/remove/list                   │
│  Plugins        : claude plugin install/enable/disable         │
│  Session slash  : /clear /compact /context /cost /config       │
└────────────────────────────────────────────────────────────────┘
```
