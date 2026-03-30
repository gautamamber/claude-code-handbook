# Claude Code Expert Curriculum — Index

> This folder covers everything needed to go from user → expert.
> Read in order, or jump to the topic you need.

---

## Learning Path

```
  BEGINNER                    INTERMEDIATE                  EXPERT
  ─────────                   ────────────                  ──────
  01-claude-md.md             deep/01-agentic-loop.md       expert/01-custom-agents.md
  02-hooks.md                 deep/02-tool-protocol.md      expert/02-multi-agent.md
  03-settings.md              deep/03-subagents.md          expert/03-headless-ci.md
  04-permissions.md           deep/04-extended-thinking.md  expert/04-cli-reference.md
  05-mcp.md                   deep/05-session-state.md      expert/05-debugging.md
  06-memory.md                                              expert/06-cost-optimization.md
  07-skills.md                                              expert/07-security.md
  08-worktrees.md                                           expert/08-team-governance.md
  09-context-window.md                                      expert/09-mcp-server-dev.md
  10-file-paths.md                                          expert/10-plugins.md
                                                            expert/11-shortcuts-commands.md
                                                            expert/12-authentication.md
```

---

## What Makes an Expert

```
  USER          knows features, uses them when told
  POWER USER    configures hooks, MCP, custom skills
  EXPERT        understands internals, designs systems,
                optimizes cost, secures deployments,
                builds custom tooling on top of Claude Code
```

---

## Expert Mental Model

```
  Claude Code = agentic loop + state manager + tool executor + hook dispatcher

  Your job as expert:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. Shape the loop      → CLAUDE.md, agents, skills          │
  │  2. Enforce rules       → hooks (exit 2 = hard stop)         │
  │  3. Extend capabilities → MCP servers, plugins               │
  │  4. Control costs       → model selection, context mgmt      │
  │  5. Secure deployments  → permissions, sandbox, governance   │
  │  6. Automate everything → headless mode, CI/CD               │
  └──────────────────────────────────────────────────────────────┘
```

---

## Files in This Folder

| File | Topic | Key Skill You Gain |
|------|-------|--------------------|
| `01-custom-agents.md` | Custom agent definitions | Design specialized AI workers |
| `02-multi-agent.md` | Multi-agent orchestration | Coordinate parallel AI systems |
| `03-headless-ci.md` | CI/CD and automation | Run Claude in pipelines |
| `04-cli-reference.md` | All CLI flags | Master every option |
| `05-debugging.md` | Debugging & diagnostics | Fix any problem fast |
| `06-cost-optimization.md` | Token & cost management | Reduce costs 10x |
| `07-security.md` | Security hardening | Deploy safely |
| `08-team-governance.md` | Team setup & policies | Scale across teams |
| `09-mcp-server-dev.md` | Build your own MCP servers | Create integrations |
| `10-plugins.md` | Plugin system | Package & distribute tooling |
| `11-shortcuts-commands.md` | All shortcuts & slash commands | Maximum productivity |
| `12-authentication.md` | Auth flows & credentials | Handle all auth scenarios |
