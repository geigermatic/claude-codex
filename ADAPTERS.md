# Adapters — using claude-codex with other agents

This kit is tuned for **Claude Code**, but the principles are agent-agnostic. With minor adaptation, you can use the same discipline layer with Cursor, Aider, Cline, Continue, Codex, or any other coding agent.

The principles in `claude/*.md` and the templates in `templates/*.md` work as-is — they describe engineering practice, not Claude internals. The skills in `skills/*` are Claude-specific (they invoke Claude Code's skill system, TodoWrite, subagents). This file maps the Claude-specific constructs to their equivalents elsewhere.

---

## What ports easily

| Component | Portability | Notes |
|----|----|----|
| `CLAUDE.md` (entry point) | Easy | Rename the file; same content works |
| `claude/principles.md` | As-is | Pure principles; tool-agnostic |
| `claude/EPISTEMICS.md` | As-is | The verification rules apply to any LLM agent |
| `claude/testing.md` | As-is | Testing discipline is language-agnostic |
| `claude/security.md` | As-is | Security principles are agent-agnostic |
| `claude/git-workflow.md` | As-is | Git is git |
| `claude/documentation.md` | As-is | Doc standards are tool-agnostic |
| `claude/token-economy.md` | Mostly | Most rules apply; specific tool names (Read, Grep) need renaming for other agents |
| `claude/languages/*.md` | As-is | Language guides don't depend on the agent |
| `templates/*` | As-is | Templates are pure markdown |
| `skills/*` | Needs porting | Claude-specific tooling; see the table below |

---

## Agent-specific config files

Drop the discipline into your agent's preferred config location:

| Agent | Config file | How to load principles |
|----|----|----|
| **Claude Code** | `CLAUDE.md` in project root, or `~/.claude/CLAUDE.md` for global | `@path/to/principle.md` imports |
| **Cursor** | `.cursorrules` in project root | Concatenate principles into the file, or use `Always` in `Cursor Settings → Rules for AI` |
| **Aider** | `.aider.conf.yml` → `read: [...]` | List the principle files; Aider loads them into the system context |
| **Cline** | `.clinerules` in project root | Plain text; concatenate principles |
| **Continue** | `~/.continue/config.json` → `systemMessage` field | Concatenate into the system message |
| **OpenAI Codex CLI** | Custom instructions in CLI flags or env var | Plain text |
| **Any other** | Most agents have a system message hook | Concatenate principles in the system prompt |

### Recommended pattern for non-Claude agents

Since most agents lack a multi-file import system:

```bash
# Generate a single file with all principles concatenated:
cat claude/EPISTEMICS.md \
    claude/principles.md \
    claude/testing.md \
    claude/security.md \
    claude/token-economy.md \
    claude/git-workflow.md \
    claude/documentation.md \
    > CONCATENATED-PRINCIPLES.md

# Then point your agent's config at CONCATENATED-PRINCIPLES.md.
```

Add the language-specific files (`languages/typescript.md`, `languages/rust.md`) as appropriate.

---

## Tool / construct mapping

The `skills/` workflows assume Claude Code's specific tools. Here's how to translate the key concepts to other agents:

### TodoWrite (task tracking)

| Agent | Equivalent |
|----|----|
| Claude Code | `TodoWrite` tool |
| Cursor | Compose mode shows tasks in the sidebar; or maintain a checklist in chat |
| Aider | Maintain a list in your message; no built-in todo system |
| Cline | Built-in todo plugin (if installed) |
| Continue | No native equivalent; use chat-level checklist |

**Workaround for agents without native todos:** include a "Plan" section in your prompt and ask the agent to maintain it across turns.

### Subagents (parallel context-isolated work)

| Agent | Equivalent |
|----|----|
| Claude Code | `Agent` tool with `subagent_type` |
| Cursor | New Composer session for each independent task |
| Aider | `/ask` to start a fresh sub-conversation |
| Cline | New task |
| Continue | New session |

**Workaround for agents without subagents:** spawn parallel conversations manually, or accept that you'll keep all the context in one place (which costs tokens but works).

### Slash commands / skills

| Agent | Equivalent |
|----|----|
| Claude Code | Files in `~/.claude/skills/` or project `.claude/skills/`, invoked as `/skill-name` |
| Cursor | Custom commands via `@command-name`; configure in Cursor Settings |
| Aider | Custom commands in `.aider.conf.yml` |
| Cline | Custom prompts / tasks |
| Continue | Slash commands in `config.json` |

**To port a claude-codex skill** (e.g., `/prd-author`):

1. Copy the body of `skills/prd-author/SKILL.md`.
2. Strip the frontmatter (`name`, `description`).
3. Paste it into your agent's custom-command config.
4. Trigger with the agent's custom-command mechanism.

The protocol inside the skill (asking questions, generating output) works the same regardless of agent.

### Plan mode / Compose mode

| Agent | Equivalent |
|----|----|
| Claude Code | `EnterPlanMode` tool (the agent stops, presents a plan, waits for approval) |
| Cursor | Compose with explicit "plan first" prompt; toggle the plan/agent mode |
| Aider | `/architect` mode (separate model for planning, separate for coding) |
| Cline | Plan mode toggle |
| Continue | No built-in; ask explicitly "plan before implementing" |

**Use case:** before a non-trivial change, the agent presents the plan; the user approves; then the agent executes.

### File reading with offsets / limits

| Agent | Equivalent |
|----|----|
| Claude Code | `Read` with `offset` and `limit` parameters |
| Cursor | Manual: select the region you want the AI to see |
| Aider | `/add path/to/file` — adds the whole file (no slicing) |
| Cline | Reads files via the file-system tool; section limits vary |

**Workaround:** for large files, pre-slice via shell (`sed -n '100,200p' file.ts > slice.ts`), then point the agent at the slice.

### Background tasks

| Agent | Equivalent |
|----|----|
| Claude Code | `run_in_background` parameter on `Bash` and `Agent` tools |
| Cursor | No native; run in a separate terminal |
| Aider | Runs shell commands; no formal background |
| Cline | Has background task support |
| Continue | No native |

**Workaround:** run long commands in a separate terminal; report back when done.

---

## What you lose with non-Claude agents

Honest accounting:

- **Skill interactivity is reduced.** Claude Code skills can call tools, spawn agents, and emit structured output. Most other agents can't do this — skills become "longer system prompts."
- **Multi-file import is rare.** Claude's `@path` syntax is uncommon. You'll likely need to concatenate.
- **Background work is harder.** Other agents may not have an event-driven completion notification.
- **Token economy rules apply differently.** The specific tool names (Read with offset, Grep with output_mode) don't translate, but the *principles* (read targeted ranges, search before reading) do.

What you keep:

- **All principles** — verification, testing, naming, security, documentation, etc.
- **All templates** — PRD, ERRORS, doc-audit, SLICE-WORKFLOW.
- **The mental model** — that AI-assisted coding needs discipline, not enthusiasm.

The discipline is the load-bearing piece. Tools serve it.

---

## A starter for Cursor users

```
# .cursorrules

You are operating under engineering discipline rules. Read them as a senior engineer reads team conventions: non-optional, with judgment for edge cases.

# Core habits
1. Verify, don't guess. Before claiming X works, run X. Before changing a name, find every caller.
2. Test the behavior before you write the code. Failing test first, always for non-trivial work.
3. Stay in scope. Deliver what was asked. Mention things you'd fix; don't silently fix them.
4. Name things precisely. Functions are verbs. Booleans start with is/has/can/should.
5. Articulate the tradeoff. Every proposal includes the cost.
6. Never push without permission. Each push is its own ask.

# Then paste the contents of:
# - claude/EPISTEMICS.md
# - claude/principles.md
# - claude/testing.md
# - claude/security.md
# - claude/git-workflow.md
# - claude/documentation.md
# - claude/languages/typescript.md (if applicable)
# - claude/languages/rust.md (if applicable)
```

Save this as `.cursorrules` in your project root. Cursor auto-loads it.

---

## A starter for Aider users

```yaml
# .aider.conf.yml
read:
  - claude-codex/claude/EPISTEMICS.md
  - claude-codex/claude/principles.md
  - claude-codex/claude/testing.md
  - claude-codex/claude/security.md
  - claude-codex/claude/git-workflow.md
  - claude-codex/claude/documentation.md
```

Or concatenate into a single file and `read:` that.

---

## A starter for Continue users

```json
{
  "systemMessage": "You are operating under engineering discipline rules. {paste concatenated principles here}"
}
```

---

## A note on Claude API direct users

If you're using the Claude API directly (not Claude Code), you can use the same principles as the system message. The `claude-api` skill in Claude Code (if you have it) covers this — but the core idea is the same: load the principles in the system message; let the model operate under them.

For multi-turn API apps, consider:

- **Cache the principles in the prompt cache** (Anthropic's prompt caching). They don't change between turns; caching them costs 90% less.
- **Refresh the cache every 4 minutes** (Anthropic's TTL is 5 min) for active sessions.

---

## Contributing adapter improvements

If you use claude-codex with another agent and discover a better mapping, open a PR adding to this file. The goal is to make it easy for any developer to adopt the discipline regardless of which agent they're paying for.
