# Expert 11 — All Keyboard Shortcuts & Slash Commands

---

## Universal Shortcuts (Always Active)

```
  Ctrl+C         Cancel current operation / interrupt
  Ctrl+D         Exit Claude Code
  Ctrl+L         Clear terminal screen (keeps history)
  Ctrl+O         Toggle verbose transcript view
```

---

## Chat Input Shortcuts

```
  ─── SUBMIT ──────────────────────────────────────────────
  Enter          Submit message (single line)

  ─── MULTILINE INPUT ─────────────────────────────────────
  Shift+Enter    New line (requires: iTerm2, WezTerm, Ghostty, Kitty)
  Alt+Enter      New line (macOS with Option as Meta key)
  Ctrl+J         New line (works in ALL terminals)
  \ then Enter   Line continuation (works in ALL terminals)

  ─── OPEN IN EDITOR ──────────────────────────────────────
  Ctrl+G         Open prompt in $EDITOR (e.g., vim, nano)
  Ctrl+X Ctrl+E  Same as Ctrl+G

  ─── LINE EDITING ────────────────────────────────────────
  Ctrl+K         Delete from cursor to end of line (cuts)
  Ctrl+U         Delete from cursor to start of line (cuts)
  Ctrl+Y         Paste previously cut text
  Alt+Y          Cycle through paste history
  Ctrl+A         Move to start of line
  Ctrl+E         Move to end of line
  Alt+B          Move back one word
  Alt+F          Move forward one word
  Ctrl+W         Delete one word back

  ─── HISTORY ─────────────────────────────────────────────
  Up / Down      Navigate command history
  Ctrl+R         Reverse search history (type to filter)
```

---

## Mode and Configuration Shortcuts

```
  ─── PERMISSION MODE CYCLE ───────────────────────────────
  Shift+Tab      Cycle modes:
                 default → plan → acceptEdits → [auto] → bypassPermissions
                 (auto only appears if --enable-auto-mode set)

  ─── MODEL SELECTION ─────────────────────────────────────
  Meta+P         Open model picker
  (Alt+P on Linux/Windows, Option+P on macOS)

  ─── PERFORMANCE ─────────────────────────────────────────
  Meta+O         Toggle fast mode (faster streaming output)
  Meta+T         Toggle extended thinking on/off
                 (requires /terminal-setup to be run first)
```

---

## Task and Workflow Shortcuts

```
  ─── TASK MANAGEMENT ─────────────────────────────────────
  Ctrl+T         Toggle task list visibility

  ─── BACKGROUND EXECUTION ────────────────────────────────
  Ctrl+B         Background current task
                 (press twice to open in tmux pane)

  ─── KILL BACKGROUND AGENTS ──────────────────────────────
  Ctrl+X Ctrl+K  Kill all background agents
                 (press the sequence twice to confirm)

  ─── REWIND / CHECKPOINT ────────────────────────────────
  Esc Esc        Rewind to last checkpoint or summarize
```

---

## Image and Attachment

```
  ─── PASTE IMAGE ─────────────────────────────────────────
  Ctrl+V         Paste image from clipboard
  Alt+V          Paste image (Windows)
  Cmd+V          Paste image (iTerm2 on macOS)
```

---

## Vim Mode (enable with /vim)

Once enabled, your input box becomes a vim editor.

```
  ─── INSERT MODE ─────────────────────────────────────────
  i, I           Enter insert mode (start / beginning of line)
  a, A           Enter insert mode (after cursor / end of line)
  o, O           New line below/above, enter insert mode
  Escape         Return to NORMAL mode
  Ctrl+K, U, Y   Still work for delete/paste in insert mode

  ─── NORMAL MODE — NAVIGATION ────────────────────────────
  h, j, k, l     Move cursor (left, down, up, right)
  w, W           Move forward one word
  e, E           Move to end of word
  b, B           Move back one word
  0              Move to start of line
  $              Move to end of line
  ^              Move to first non-whitespace
  gg             Go to top
  G              Go to bottom

  ─── NORMAL MODE — EDITING ───────────────────────────────
  x              Delete character under cursor
  dd             Delete current line
  D              Delete from cursor to end of line
  cc             Change entire line
  C              Change from cursor to end of line
  cw             Change word
  r              Replace single character
  u              Undo
  Ctrl+R         Redo

  ─── NORMAL MODE — YANK/PASTE ────────────────────────────
  yy             Yank (copy) current line
  y$             Yank to end of line
  p              Paste after cursor
  P              Paste before cursor

  ─── NORMAL MODE — SEARCH ────────────────────────────────
  /pattern       Search forward
  n              Next match
  N              Previous match

  ─── MISC ────────────────────────────────────────────────
  .              Repeat last change
  :              Command mode (for vi commands like :q, :w)
```

---

## Terminal Setup for Better Shortcuts

Run once to configure your terminal:

```bash
/terminal-setup
```

This configures:
- Shift+Enter for multiline (if terminal supports it)
- Meta/Option key mapping (for Alt+ shortcuts)
- Extended thinking toggle (Meta+T)
- Proper escape sequence handling

---

## All Slash Commands Reference

### Context & Conversation

```
/clear                    Start new session (old session still resumable)
/compact                  Compress conversation (auto-selects what to keep)
/compact focus on X       Compress but preserve context about X
/context                  Show full context window breakdown
/cost                     Show token usage + USD cost for this session
/memory                   View and edit memory files (MEMORY.md + topic files)
/claude.md                Create or edit project CLAUDE.md
/status                   Session ID, model, auth type, permission mode
/btw <question>           Ask a side question without adding it to context
```

### Configuration

```
/config                   Interactive settings menu (model, thinking, etc.)
/model                    Quick model switcher
/effort                   Set thinking effort: low|medium|high|max
/vim                      Toggle vim keybindings mode
/terminal-setup           Configure terminal for multiline, Meta keys
/keybindings              Open ~/.claude/keybindings.json for editing
/theme                    Choose color theme
/statusline               Configure the status bar
/permissions              View/modify current permission settings
/auto-mode                Show auto mode classifier rules
/sandbox                  Toggle filesystem sandboxing
/timeout <sec>            Set MCP server timeout (seconds)
```

### Agents, Skills, Plugins

```
/agents                   Manage subagents (list, create, edit, delete)
/skills                   List all available skills with descriptions
/reload-plugins           Reload plugins after making changes
/hooks                    View active hooks configuration
/mcp                      Manage MCP servers (list, auth, add, remove)
/plugin                   Manage plugins (install, enable, disable, update)
```

### Session Management

```
/rename <name>            Rename current session (for /resume picker)
/resume                   Interactive picker to resume a past session
/session                  Display current session ID and metadata
/checkpoint               Create restore point (saves code + conversation state)
/rewind                   Restore to a checkpoint
```

### Git & Development

```
/git                      Show common git commands reference
```

### Diagnostics & Help

```
/doctor                   Run diagnostics (auth, paths, version, hooks, MCP)
/help                     Show all commands and available skills
/help <topic>             Context-specific help
/feedback                 Open feedback form for Anthropic
```

---

## Custom Keybindings

You can remap any action in `~/.claude/keybindings.json`:

```json
[
  {
    "key": "ctrl+enter",
    "action": "submit"
  },
  {
    "key": "ctrl+shift+c",
    "action": "clear"
  },
  {
    "key": "alt+r",
    "action": "compact"
  },
  {
    "key": "f5",
    "action": "rerun-last"
  }
]
```

Or use the `/keybindings` command to open the file directly in your editor.

---

## Quick Reference Card

```
  ┌─────────────────────────────────────────────────────────────┐
  │  MOST USED SHORTCUTS                                        │
  ├────────────────────────────────┬────────────────────────────┤
  │  Submit                        │  Enter                     │
  │  Multiline (universal)         │  Ctrl+J                    │
  │  Cancel/interrupt              │  Ctrl+C                    │
  │  Exit                          │  Ctrl+D                    │
  │  Verbose mode toggle           │  Ctrl+O                    │
  │  Cycle permission modes        │  Shift+Tab                 │
  │  Background task               │  Ctrl+B                    │
  │  Task list                     │  Ctrl+T                    │
  │  Toggle thinking               │  Meta+T                    │
  │  Model picker                  │  Meta+P                    │
  ├────────────────────────────────┴────────────────────────────┤
  │  MOST USED SLASH COMMANDS                                   │
  ├────────────────────────────────┬────────────────────────────┤
  │  New session                   │  /clear                    │
  │  Save context                  │  /compact                  │
  │  Context usage                 │  /context                  │
  │  Cost                          │  /cost                     │
  │  Memory                        │  /memory                   │
  │  Settings                      │  /config                   │
  │  Diagnostics                   │  /doctor                   │
  │  Help                          │  /help                     │
  └────────────────────────────────┴────────────────────────────┘
```
