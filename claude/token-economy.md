# Token Economy — efficient tool use, context discipline

Every token you read costs money and dilutes context. A coding agent that reads 10x more than necessary is not only expensive — it produces worse reasoning, because the relevant signal is buried in noise.

This file covers how to spend tokens like money: targeted reads, search before scanning, parallel work, background work, and protecting your main conversation context from bloat.

For verification discipline (which sometimes *requires* spending tokens), see `claude/EPISTEMICS.md`. The principle there is "verify, don't guess." The principle here is "verify *efficiently*."

---

## 1. The cost of every token

Two costs accumulate for every token you read into your context:

1. **Direct cost:** prompt tokens are billed per request.
2. **Context cost:** the more your context fills, the less room there is for the work itself. Models reason worse with cluttered context. Important signal gets buried.

A 2000-line file read in full is ~3000 tokens. If 50 lines were enough, you spent 60x more than needed — and gave the model 60x more noise to filter.

**RULE:** Default to the smallest read that answers your question. Expand only when needed.

---

## 2. The decision tree: which tool, when

Before you reach for `Read` on a file, ask:

```
Do you know the EXACT file you need?
├── YES → Read with limit/offset if file is large
└── NO → Do you know what to search for (symbol, string, pattern)?
    ├── YES → Grep (with output_mode "files_with_matches" or "count" first)
    └── NO → Glob (for path patterns) OR Explore subagent (for surveys)
```

### Read

- Use when: you know the file and want its contents.
- For files >500 lines: use `limit` + `offset` to read only the section you need.
- For files <500 lines: read in full is fine.
- **Never speculatively read a directory listing** — use Glob or `ls` via Bash.

### Grep

- Use when: you're looking for symbols, strings, or patterns and don't know which file.
- **First pass:** `output_mode: "files_with_matches"` — lists which files match. Cheap.
- **Second pass:** `output_mode: "content"` only on the files you actually need to read, with a small `-C` context window.
- For frequency counting: `output_mode: "count"`.

### Glob

- Use when: you need to enumerate files by pattern (e.g., `src/**/*.test.ts`).
- Much cheaper than `find` for path-based queries.

### Explore subagent (or equivalent)

- Use when: the search is broad enough that it'd take 3+ targeted queries.
- The subagent reads on your behalf and reports a summary — keeps your main context clean.
- Best for "where is X defined / which files reference Y" across an unfamiliar codebase.

### Bash

- Use when: you genuinely need shell capability (running tests, builds, git commands).
- **Never** use `cat`, `head`, `tail`, `sed`, `awk`, `echo` when a dedicated tool exists.
- Truncate large outputs: `git log --oneline -20` not `git log`. `head -100` if a command spews.

---

## 3. Read with `limit` and `offset`

For files >500 lines, never read in full unless you genuinely need the whole thing.

### Pattern

```
Read(file_path: "src/big-service.ts", offset: 0, limit: 50)
```

If you need section 200-260:

```
Read(file_path: "src/big-service.ts", offset: 200, limit: 60)
```

### When you genuinely need the full file

- Refactoring it
- Translating it to another language
- Generating documentation that covers everything

Most other times: read what you need, then expand if the surrounding context matters.

---

## 4. Don't re-read just-edited files

After `Edit` or `Write` succeeds, the harness tracks the new file state for you. Re-reading the file you just modified is a common token-waster.

**RULE:** After a successful `Edit`/`Write`, do not Read the file unless something specific is unclear (e.g., you want to verify a surrounding block you didn't see).

If the edit failed, the tool tells you. If it succeeded, trust it.

---

## 5. Parallel tool calls

When you need multiple pieces of independent information, request them in **one message with parallel tool calls**, not serially.

### Good

One message containing:
- `Read(README.md)`
- `Read(package.json)`
- `Bash(git log --oneline -10)`

These all run in parallel. Three operations, one round trip.

### Bad

Message 1: Read README. → wait → Message 2: Read package.json. → wait → Message 3: Run git log.

Three round trips. Three times the latency. Same total token cost, but slower and more conversational noise.

**RULE:** Independent operations always go in parallel. Sequential is only for operations where the result of one informs the next.

---

## 6. Background long-running commands

For commands that take more than ~10 seconds (full builds, large test suites, slow installs), use the background flag (e.g., `run_in_background: true`).

### Benefits

- Your main conversation continues immediately.
- The harness notifies you when the command completes.
- You can read partial output via the appropriate tool while it runs.

### When NOT to background

- Short commands (lint a single file, run a quick test) — overhead exceeds benefit.
- Commands whose output you need before you can proceed.

---

## 7. Cache warmth

If your agent runtime supports prompt caching (Anthropic's API does, with a 5-minute TTL), structure your work to stay within the cache window.

### Patterns

- **Don't sleep 300+ seconds.** That's the worst-of-both: you pay the cache miss without amortizing it. Either drop to <270s (stay in cache) or commit to 20+ minutes (one cache miss buys a much longer wait).
- **Don't poll in tight loops.** A 30s poll burns the cache every cycle for nothing changing most of the time. Poll less often, or use a notification mechanism.

### Practically

- Background-running tests with notification on completion (not polling).
- For long waits, do something useful in the meantime instead of sleeping.
- If you're waiting on a build that takes 8 minutes, sleep ~270s twice and pay one cache miss, not 8 times by polling every minute.

---

## 8. Compact responses

The tokens you generate also cost. Brevity is engineering discipline.

### Rules

- **One sentence per update.** Most progress updates are one line.
- **No restating what the user said.** They know what they asked.
- **No restating what tool calls just did.** The tool results are visible.
- **End-of-turn summary: one or two sentences.** What changed. What's next. Stop.
- **No "Let me..." preambles.** State what you're doing in the act of doing it (in code or via tools), not in prose.

### Headers and structure

- Use them when they aid scanning. Don't use them for two-paragraph responses.
- A bulleted list of 3 items doesn't need a header. A list of 15 items in three categories does.

### What to skip entirely

- Explaining the obvious
- Narrating internal deliberation
- "I think the best approach would be..." → state the approach
- Closing pleasantries ("Hope this helps!")

---

## 9. TodoWrite for state, not narration

If your agent has a todo system, use it for **state you'll refer to** — not for thinking out loud.

### Use a todo for

- Multi-step tasks (3+ distinct steps)
- Long-running work that may span context compactions
- User-visible progress

### Don't use a todo for

- "Read file" → just read it
- Single-step operations
- Anything you'll finish in the current response

A 12-item todo list for a 12-step task is communication. A 12-item todo list for a 3-step task is theater.

---

## 10. Subagents for survey work

Subagents (separate agent invocations) have their own context windows. Use them to:

### Protect your main context

When you need broad exploration ("find every place that handles authentication"), a subagent reads, summarizes, and returns a report — without polluting your main conversation with the file contents.

### Parallelize independent investigations

Launch two subagents in one message — one to audit security, one to audit performance — they run in parallel. You get two reports.

### When NOT to use a subagent

- The target is already known (use `Read` directly).
- The task is single-step (overhead exceeds benefit).
- You need control over each step (a subagent is autonomous within its task).

### Subagent prompting

Brief the subagent as if it just walked into the room. It does not see your conversation. State the goal, the relevant context, and what shape of report you want. Specify the response length cap.

---

## 11. Bash output truncation

A `git log` with no flags can output thousands of lines. A `grep -r` across a repo can output tens of thousands. Most of that is noise.

### Patterns

- `git log --oneline -20` — last 20 commits, one line each.
- `git log -p -- path/to/file | head -200` — last few patches for one file, capped.
- `ls -la dir/ | head -50` — first 50 entries.
- `grep -r 'pattern' src/ | head -50` — first 50 matches.
- `find . -name '*.ts' | wc -l` — count, don't list, when you only need the number.

### Set limits explicitly

If a Bash command has a max-output flag, use it. If not, pipe to `head` or `tail`.

If you genuinely need all of a large output, redirect it to a file and read sections:

```
git log -p > /tmp/log.txt
# then Read with offset/limit
```

---

## 12. Context window awareness

Your context window has a finite size. As the conversation progresses, the context fills with: system prompts, prior turns, tool results, file contents you've read.

When the context approaches the limit, the harness typically compacts older turns into summaries. This is automatic but lossy.

### What survives compaction

- Recent turns
- Files currently in your context
- Active tool state

### What gets summarized

- Old conversation
- File contents read long ago

### Implications

- **Read once, retain.** Don't re-read the same file at every step "to refresh" — it's still in context.
- **Be explicit about important state.** If something was decided 50 turns ago and matters now, restate it rather than relying on it being in context.
- **Take notes.** For state that must survive compaction, use a persistent mechanism (todo list, project memory, scratchpad file).

---

## 13. The "is this read worth its tokens" question

Before any `Read` call, ask: **what specific question am I answering?**

### Examples

| Question | Good answer | Bad answer |
|----|----|----|
| "What does `parseConfig` do?" | Read the file with `offset`/`limit` around the function. | Read the whole file. |
| "Is there a test for X?" | Grep for X in `*.test.*` files. | Read every test file. |
| "What's the project structure?" | Glob for `src/**/*` and read directory names. | Read every file. |
| "What did the user mean by 'the search bug'?" | Re-read their last message. | Read the search module. |

If you can't articulate the specific question, the read is speculative. Speculative reads are the biggest source of token waste.

---

## 14. Anti-patterns

### Reading the whole repo

`Read` on every file in `src/`. Even with a fast model, this is a token avalanche. Always start with Grep/Glob and read only what matches.

### Re-running expensive commands

Running `npm test` to "see what's happening" after every small change. Run it once at the end of a logical unit of work, not after every line.

### "Just in case" context

Reading a file because it might be relevant. Either you have evidence it's relevant (a Grep hit, a clear reference) or you don't. Reading on a hunch is gambling with tokens.

### Verbose chain-of-thought in chat

Writing out a paragraph of reasoning before each tool call. The model has its own internal reasoning; chat output is for user-visible decisions, not internal deliberation.

### Quoting back file contents

After reading a file, repeating its contents in your response. The user can see the file. The diff (if you edit it) is what they need. Quoting wastes generation tokens and adds nothing.

### Multiple sequential Reads of the same large file

If you need bits from a large file, plan the reads. One read with the right offset/limit is better than five reads of overlapping ranges.

---

## 15. The mental model

Think of tokens like an API rate limit you're paying for, because that's exactly what they are.

- Every read is a request. Make it count.
- Every word you generate is a request. Be concise.
- Every "let me check" without a target is wasted spend.
- The model's reasoning quality drops as context fills. Lean context = sharper reasoning.

**The discipline:** spend tokens only when you've articulated what you're spending them on. Apply this to reads, to writes, to chat output, to tool calls. The result is faster, cheaper, and — counter-intuitively — better-reasoned work.
