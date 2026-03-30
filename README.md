<div align="center">

<img src="https://img.shields.io/badge/Claude_Code-Handbook-6B21A8?style=for-the-badge&logo=anthropic&logoColor=white" alt="Claude Code Handbook"/>

# 📖 Claude Code Handbook

### A technically deep, structured reference for Claude Code — covering internals most guides never touch.

[![Stars](https://img.shields.io/github/stars/YOUR_USERNAME/claude-code-handbook?style=social)](https://github.com/YOUR_USERNAME/claude-code-handbook)
[![Forks](https://img.shields.io/github/forks/YOUR_USERNAME/claude-code-handbook?style=social)](https://github.com/YOUR_USERNAME/claude-code-handbook)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/YOUR_USERNAME/claude-code-handbook/pulls)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

<br/>

[🗂️ What's Inside](#-whats-inside) • [⚡ Quick Wins](#-quick-wins) • [📋 Coverage](#-coverage)

</div>

---

## Why This Exists

Official Claude Code documentation explains what features exist.
It rarely explains how they work internally or why they behave the way they do.

This handbook covers the details that matter in practice:

- How CLAUDE.md is actually loaded (it's a user message, not a system prompt)
- How hooks enforce rules deterministically (exit codes, not instructions)
- How the agentic loop works at the API level (tool_use/tool_result protocol)
- How the context window fills up and what compaction actually removes
- How subagents are isolated from the parent session
- How permission evaluation works step by step
- And a lot more that most users never discover

**26 files. 3 tiers. Every file has ASCII diagrams and a cheatsheet.**

---

## 📊 What's Inside

```
claude-code-handbook/
│
├── 📁 beginners/     10 files — Core components with diagrams
├── 📁 deep/           6 files — Internal mechanics and API protocol
└── 📁 expert/         9 files — Advanced patterns, CI/CD, and MCP development
```

<details>
<summary><b>📁 beginners/ — Core Components (click to expand)</b></summary>

| File | Covers |
|------|--------|
| [`01-claude-md.md`](beginners/01-claude-md.md) | Loading hierarchy, lazy loading, imports, path-scoped rules, size limits |
| [`02-hooks.md`](beginners/02-hooks.md) | All 20+ hook events, 4 hook types, exit code behavior, real examples |
| [`03-settings.md`](beginners/03-settings.md) | 4-scope hierarchy, primitive vs array merge, full settings schema |
| [`04-permissions.md`](beginners/04-permissions.md) | 6 permission modes, exact evaluation order, auto mode classifier internals |
| [`05-mcp.md`](beginners/05-mcp.md) | Deferred tool loading, all server types, tool naming, hook matchers |
| [`06-memory.md`](beginners/06-memory.md) | Two-tier loading model, 4 memory types, what to save vs skip |
| [`07-skills.md`](beginners/07-skills.md) | Full frontmatter spec, shell preprocessing, argument substitution, context fork |
| [`08-worktrees.md`](beginners/08-worktrees.md) | Git worktree lifecycle, parallel sessions, shared memory across worktrees |
| [`09-context-window.md`](beginners/09-context-window.md) | Context composition order, compaction steps, what survives vs gets removed |
| [`10-file-paths.md`](beginners/10-file-paths.md) | Every important path, commit vs gitignore guide, managed policy locations |

</details>

<details>
<summary><b>📁 deep/ — Internal Mechanics (click to expand)</b></summary>

| File | Covers |
|------|--------|
| [`01-agentic-loop.md`](deep/01-agentic-loop.md) | Full API call cycle, stop_reason values, streaming SSE format, error injection |
| [`02-tool-protocol.md`](deep/02-tool-protocol.md) | tool_use/tool_result wire format, parallel calls, all built-in tool schemas |
| [`03-subagents.md`](deep/03-subagents.md) | Context isolation, what the parent sends vs receives, transcript storage |
| [`04-extended-thinking.md`](deep/04-extended-thinking.md) | Thinking blocks, budget_tokens, context cost, encrypted thinking |
| [`05-session-state.md`](deep/05-session-state.md) | Stateless API, Bash shell resets, what persists on disk, /compact mechanics |

</details>

<details>
<summary><b>📁 expert/ — Advanced Patterns (click to expand)</b></summary>

| File | Covers |
|------|--------|
| [`01-custom-agents.md`](expert/01-custom-agents.md) | Full agent frontmatter spec, agents vs skills vs subagents, design patterns |
| [`02-multi-agent.md`](expert/02-multi-agent.md) | Sequential, parallel, supervisor+workers, agent teams, cost tradeoffs |
| [`03-headless-ci.md`](expert/03-headless-ci.md) | `-p` flag, output formats, piping, bare mode, GitHub Actions examples |
| [`04-cli-reference.md`](expert/04-cli-reference.md) | Every CLI flag, all subcommands, all in-session slash commands |
| [`05-debugging.md`](expert/05-debugging.md) | /doctor, --debug categories, transcript inspection, common failure patterns |
| [`06-cost-optimization.md`](expert/06-cost-optimization.md) | Model selection strategy, context management, 10x cost reduction checklist |
| [`09-mcp-server-dev.md`](expert/09-mcp-server-dev.md) | JSON-RPC protocol, minimal Node.js + Python server examples |
| [`11-shortcuts-commands.md`](expert/11-shortcuts-commands.md) | All keyboard shortcuts, all slash commands, vim mode reference |
| [`12-authentication.md`](expert/12-authentication.md) | Auth precedence order, OAuth flow, apiKeyHelper, AWS/GCP/Azure providers |

</details>

---

## ⚡ Quick Wins

Five things that will immediately change how you use Claude Code:

**1. CLAUDE.md is a user message, not a system prompt.**
Instructions are persuasive guidance — not enforced rules. Vague instructions fail silently.

**2. Hooks with `exit 2` are the only hard enforcement.**
Claude cannot bypass a hook that exits with code 2. Everything else is probabilistic.

**3. The Anthropic API is stateless. Every call re-sends the full conversation.**
Large file reads aren't free — they cost tokens on every subsequent API call in a session.

**4. Each Bash call runs in a completely fresh shell.**
`cd /tmp` in one call has zero effect on the next call. Always chain commands with `&&`.

**5. Subagents are fully isolated from the parent session.**
They receive only your explicit prompt — not your conversation history, not prior tool results.

---

## 📋 Coverage

This handbook covers:

**Components** — CLAUDE.md, Hooks, Settings, Permissions, MCP, Memory, Skills, Worktrees, Context Window

**Internals** — Agentic loop, Tool protocol (tool_use/tool_result), Subagent architecture, Extended thinking, Session state management

**Expert patterns** — Custom agents, Multi-agent orchestration, Headless/CI mode, Cost optimization, Security, MCP server development, Authentication

**Reference** — Full CLI flag reference, all slash commands, all keyboard shortcuts, all file paths


---

## 📄 License

MIT — free to use and share.

---

<div align="center">

**If this handbook helped you, please ⭐ star the repo.**

[⬆ Back to top](#-claude-code-handbook)

</div>
# claude-code-handbook
