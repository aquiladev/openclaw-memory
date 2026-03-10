# OpenClaw Memory Management: The Complete Guide

*Based on insights from an OpenClaw codebase maintainer with 2+ months of daily production use.*

---

## The Core Principle

**If it's not written to a file, it doesn't exist.**

Instructions given only in chat do not survive long sessions. When compaction fires, chat-only instructions vanish from the agent's context. This has caused real incidents — even experienced AI researchers have lost control of agents because safety rules given in conversation disappeared after compaction.

---

## The Three Things That Matter Most

Do just these three and you're ahead of 95% of OpenClaw users:

1. **Put durable rules in files, not in chat.** Your `MEMORY.md`, `AGENTS.md` — those survive compaction. Instructions typed in conversation do not.

2. **Verify the memory flush is enabled and has enough buffer to trigger.** OpenClaw has a built-in safety net that saves context before compaction, but most people never check if it's working or give it enough room to fire.

3. **Make retrieval mandatory.** Add a rule to `AGENTS.md`: "search memory before acting." Without it, the agent guesses instead of checking its notes.

---

## Understanding the Four Memory Layers

Most people think of memory as one thing. It's actually four different systems, and they fail in different ways. Knowing which layer broke is 90% of fixing it.

### Layer 1: Bootstrap Files (Most Durable)

Your workspace files: `SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md`, `TOOLS.md` (and optionally `IDENTITY.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`). They are loaded from disk at session start and survive compaction because they are reloaded from disk, not from conversation history.

### Layer 2: Session Transcript

Every conversation is saved as a JSONL file on disk. When you continue a session, this transcript is rebuilt into context. But when the context window fills up, the transcript gets compacted — a compact summary replaces the detailed history. The model can no longer see the original messages. Sessions reset daily at 4:00 AM local time.

### Layer 3: LLM Context Window

The fixed-size container where everything competes for space. System prompt, workspace files, conversation history, tool calls, tool results — all in one ~200,000 token bucket. When it fills, compaction fires.

### Layer 4: Retrieval Index

A searchable layer that sits beside your memory files. The agent can query it with the `memory_search` tool to find relevant context from past sessions. This only works if information was written to files first.

---

## The Three Failure Modes

When your agent forgets something, it's always one of these:

### Failure A: Never Stored
The instruction, preference, or decision only existed in conversation and was never written to a file. When compaction fires or a new session starts, it's gone. **This is the most common cause by far.**

*Diagnostic:* Agent forgot a preference → probably never written to `MEMORY.md`.

### Failure B: Compaction Changed Context
A long session hit the token limit, compaction summarized old messages, and the summary was lossy — it dropped important details, nuance, or specific constraints. The agent now operates from the summary, not your original words.

*Diagnostic:* Agent forgot the whole conversation thread → compaction or session reset.

### Failure C: Pruning Trimmed Tool Results
Tool outputs (file reads, browser results, API responses) were trimmed by session pruning to optimize caching.

*Diagnostic:* Agent forgot what a tool returned → likely pruning.

---

## Compaction vs. Pruning: Know the Difference

These are completely different systems that most guides mix up.

| | **Compaction** | **Pruning** |
|---|---|---|
| **What it does** | Summarizes entire conversation history | Trims old tool results in-memory |
| **Triggered by** | Context window filling up | Cache TTL (time-to-live, default 5 min) |
| **Affects** | Everything: user messages, assistant messages, tool calls | Only tool result messages |
| **On-disk history** | Replaced by summary | Untouched |
| **Impact** | **Dangerous** — changes what the model sees | **Friendly** — reduces bloat without destroying conversation context |

Pruning is your friend. Compaction is the one that hurts.

---

## What Happens When Compaction Fires

### The Good Path: Maintenance Compaction

1. Context nears the limit
2. Pre-compaction memory flush fires first
3. Agent automatically saves important context to disk
4. Compaction summarizes old history
5. Agent continues with summary + recent messages (~last 20K tokens stay intact)

### The Bad Path: Overflow Recovery

1. Context gets too big, API rejects the request
2. We're already past the threshold — **memory flush doesn't happen**
3. OpenClaw enters damage control — compresses everything at once
4. No saving important stuff to disk first
5. **Maximum context loss**

The entire point of configuration is to stay on the good path and avoid the bad one.

### What Survives Compaction

**Lost:**
- Instructions only given in chat
- Preferences or decisions mentioned mid-session
- Older images
- All tool results
- Exact wording and nuance of earlier messages (summaries are lossy)

**Survives:**
- All workspace files (`SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md`, `TOOLS.md`)
- Daily memory logs (when accessed via search)
- Anything written to disk before compaction (the flush)
- Roughly the last ~20,000 tokens of recent messages

---

## The Three-Layer Defense System

No single mechanism is enough. You need all three working together.

### Layer 1: Pre-Compaction Memory Flush (Configuration)

This is the most important config change. The flush is built into OpenClaw — it triggers a silent agentic turn before compaction, reminding the model to write anything important to disk.

**Configuration (`~/.openclaw/openclaw.json`):**

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 40000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

**What each setting does:**

- **`reserveTokensFloor: 40000`** — Headroom. Reserved space for the memory flush turn and compaction summary without hitting overflow. The flush triggers at: `context_window - reserveTokensFloor - softThresholdTokens`. With a 200K context window: `200,000 - 40,000 - 4,000 = 156K tokens`. The default reserve of 20K is often insufficient; 40K is a practical baseline. If you rarely use big tools, you can go lower. If you read large files or web snapshots regularly, go higher. The exact number matters less than the principle: give the flush enough room to fire before overflow.

- **`memoryFlush.enabled: true`** — Should be on by default in recent versions, but verify in your config.

- **`softThresholdTokens: 4000`** — How far before the reserve floor the flush triggers. Default of 4,000 is fine for most setups.

- **`systemPrompt`** — The system-level instruction sent to the model during the flush turn.

- **`prompt`** — The user-level instruction sent to the model during the flush turn.

**Important:** The automated flush is a safety net, not a guarantee. The agent might not save everything important, and it can be bypassed by large single-turn token jumps — that's why we need the other two layers.

### Layer 2: Manual Memory Discipline

Experienced OpenClaw users complement the automatic flush with manual saves. It's a simple habit that catches what automation misses.

**When to save manually:**
- After finishing a large task before switching to a new one
- Before giving a new complex instruction
- When you've just made an important decision

**How:** Simply tell the agent: "Save this to MEMORY.md" or "Write today's key decisions to memory."

**The `/compact` command trick:**

Manual compaction on your terms is different from emergency compaction. Mid-session, you can run `/compact` to intentionally compress context and free up space. You can even guide it: `/compact and focus on decisions and open questions`.

But don't wait until you're near overflow — at that point, `/compact` might also fail, and your only option is a new session.

**Why both manual and automatic?**
- The automated flush is timing-based (fires at a token threshold)
- Manual saves are relevance-based (when you know something important just happened)
- Together they cover both cases

### Layer 3: File Architecture & Retrieval

#### Bootstrap Files (Always in Context)

| File | Purpose | Content |
|---|---|---|
| `SOUL.md` | Who the agent **is** | Tone, personality, broad boundaries |
| `AGENTS.md` | How the agent **operates** | Workflow rules, tool conventions, do/don't rules |
| `USER.md` | Who **you** are | Projects, clients, priorities, communication preferences, key people, technical environment |
| `MEMORY.md` | What's **true across every session** | Important decisions and why, learned preferences, rules from past mistakes |
| `TOOLS.md` | Tool-specific instructions | How to use specific tools, API conventions |
| `IDENTITY.md` | Extended identity | Additional identity configuration |
| `HEARTBEAT.md` | Periodic check-in rules | Heartbeat/cron behavior |
| `BOOTSTRAP.md` | Additional bootstrap config | Extra startup instructions |

**Simple rule:** Character → `SOUL.md`. Process → `AGENTS.md`.

**Pro tip:** Negative instructions are often the most valuable. When the agent does something annoying or wrong, add a rule so it doesn't happen again.

**Reality check:** These files are prompts, not real security. For actual protection, use system controls like tool permissions, workspace isolation, and allow lists.

#### The `memory/` Directory (Daily Logs)

Daily files (`memory/YYYY-MM-DD.md`) contain:
- Decisions made in conversations
- Active tasks and their status
- Output from pre-compaction flush (lands here automatically)

These aren't bootstrap-injected. The system usually reads today + yesterday automatically. Everything else is pulled in on-demand via `memory_search` or `memory_get`.

**Note:** Sub-agent sessions only inject `AGENTS.md` and `TOOLS.md`. Other bootstrap files are filtered out. If sub-agents lack your personality or preferences, that's why.

#### MEMORY.md Best Practices

- Keep it short — under 100 lines. It's a cheat sheet, not a journal.
- It's always loaded, so it's always in context — expensive in tokens and attention.
- Store: decisions, principles, project states, user preferences.
- Never store: API keys, tokens, secrets, or anything you wouldn't want in plain text.

#### The Memory Protocol (Add to AGENTS.md)

```markdown
## Memory Protocol

Before doing anything non-trivial, search memory first.

- Before answering questions about past work: search memory first
- Before starting any new task: check memory/today's date for active context
- When you learn something important: write it to the appropriate file immediately
- When corrected on a mistake: add the correction as a rule to MEMORY.md
- When a session is ending or context is large: summarize to memory/YYYY-MM-DD.md

## Retrieval Protocol

Before doing non-trivial work:
1. `memory_search` for the project/topic/user preference
2. `memory_get` the referenced file chunk if needed
3. Then proceed with the task
```

This shifts the agent from "I'll guess based on context" to "I'll check my notes before I act."

#### Group Chat Configuration (for Discord/Slack)

If the agent operates in group chats, add to `AGENTS.md`:

```markdown
## Group Chat Rules
- Only respond when: directly mentioned, asked a direct question, or you have genuinely useful info
- Do NOT respond to: side conversations, banter, logistics between others, greetings, link shares
- When in doubt -> respond with only: NO_REPLY
- NO_REPLY must be your ENTIRE message - nothing else
```

---

## Retrieval: Two Tracks

### Track A: Built-in Memory Search (Start Here)

- No extra installs needed
- Indexes `MEMORY.md` and everything in the `memory/` directory automatically
- Watches for file changes and rebuilds the index
- Local embedding model downloads automatically on first use
- Supports hybrid search (keyword + semantic/meaning-based matching)
- Runs locally on your computer, free after initial model download

**Hybrid search** means: keyword search finds exact words ("pricing"), while embeddings capture meaning ("we picked the $29 tier" matches a search for "pricing decision" even though the word "pricing" never appears).

**Track A+ variant:** Add extra paths to index additional Markdown files outside your workspace (project folders, notes directories, knowledge bases).

### Track B: QMD Backend (Advanced)

QMD (Query Markdown Documents) is an experimental memory backend that replaces the built-in indexer. Use it when:
- You need to search beyond your workspace
- You have thousands of files (e.g., Obsidian vault)
- You have many project docs, meeting notes, or past session transcripts

QMD returns small, relevant snippets instead of whole files, which keeps context smaller and helps with compaction.

**Note:** QMD is DM-only by default. It doesn't work in group chats. Check the scope in your config.

---

## Configuration Reference

### Context Pruning

Session pruning trims old tool results in-memory to delay compaction and improve caching. The on-disk transcript is untouched.

```json5
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "5m"
      }
    }
  }
}
```

### Track A Config (Built-in)

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 40000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      },
      "memorySearch": {
        "enabled": true,
        "provider": "local",
        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf"
        },
        "query": {
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3
          }
        },
        "cache": {
          "enabled": true
        }
      },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "5m"
      }
    }
  }
}
```

### Track A+ Config (Extra Paths)

Same as Track A with additional indexed paths:

```json5
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "local",
        "extraPaths": [
          "~/Documents/Obsidian/ProjectNotes/**/*.md",
          "~/Documents/specs/**/*.md"
        ]
      }
    }
  }
}
```

### Track B Config (QMD)

Same compaction and pruning config as Track A, plus:

```json5
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "searchMode": "search",
      "includeDefaultMemory": true,
      "sessions": {
        "enabled": true
      },
      "paths": [
        { "name": "obsidian", "path": "~/Documents/Obsidian", "pattern": "**/*.md" },
        { "name": "docs", "path": "~/Documents/project-docs", "pattern": "**/*.md" }
      ]
    }
  }
}
```

---

## Bootstrap File Limits

- **Per-file limit:** 20,000 characters (default)
- **Combined limit:** 150,000 characters across all bootstrap files (~50K tokens)
- These are character counts, not tokens
- Files exceeding the limit are truncated using a 70/20/10 split: 70% head, 20% tail, 10% truncation marker
- If a file is not in context, it has zero effect on the agent

### Diagnosing with `/context list`

Run `/context list` in your OpenClaw session. Check:
1. Is `MEMORY.md` actually loading? If missing or not listed, it's not in context.
2. Is anything showing truncated? If raw chars ≠ injected chars, part of the file is invisible.
3. If files are truncated, adjust the per-file or combined limits in config.

Use `/context detail <file>` for deep analysis of a specific file's injection.

---

## Cost Implications

Every message includes the entire system prompt and conversation history. Prompt caching means you pay ~90% less for repeated tokens. But compaction invalidates the cache — the next request pays full price to re-cache everything.

**Every unnecessary compaction is both a reliability problem and a cost problem.**

Two things break the cache:
1. **Compaction** — rewrites conversation history
2. **Changing prompt inputs** — if you constantly rewrite `MEMORY.md` or inject dynamic status blocks, cache no longer matches

Keep workspace files stable and `MEMORY.md` small to maximize cache hits.

---

## Memory Hygiene Over Time

### Daily
- Append to daily log (happens automatically via flush)

### Weekly
- Promote durable rules and important decisions from daily logs into `MEMORY.md`
- Remove anything that's no longer current or important from `MEMORY.md`
- Set up a weekly cron job to automate this analysis

### Git Backup
- Run `git init` in your workspace directory
- Set up auto-commit by daily cron or heartbeat
- Get full history with diffs — roll back as far as you want
- **Never commit:** credentials folder or `openclaw.json` (contains API keys/tokens)

---

## Troubleshooting

Common issues and fixes:

- **Preference not remembered:** Check `MEMORY.md` loading via `/context list`. Verify preference is written to file. If missing, add to `MEMORY.md` now.
- **`memory_search` disabled:** Confirm memory files exist. Check if local embedding model downloaded successfully. Run `/context list` to verify loaded files.
- **Tool results forgotten:** This indicates session pruning (temporary), not compaction. Write important tool outputs to memory. Re-run tool if critical.
- **Compaction too late:** Use `/compact` proactively before overflow. Raise `reserveTokensFloor` to trigger earlier. If stuck in overflow deadlock, use `/new` or CLI recovery.
- **Flush didn't run:** Can be bypassed by large single-turn token jumps. Raise `reserveTokensFloor` buffer. Use manual save points as backup.
- **Agent forgets tools after long sessions:** Known issue with long Discord sessions. Use `/new` to reset; proper memory files allow continuation.
- **Everything forgotten overnight:** Sessions reset daily (default 4:00 AM local time). Only bootstrap files and searchable memory carry over. This is expected behavior.

**Version note:** Ensure v2026.2.23 or later for compaction bug fixes.

---

## Essential Slash Commands

| Command | Purpose |
|---|---|
| `/context list` | Check what's loaded and whether files are truncated |
| `/context detail <file>` | Deep analysis of specific file injection |
| `/compact [focus]` | Manual compaction with optional focus guidance |
| `/status` | Verify model, context window size, thinking level |
| `/new` or `/reset` | Start fresh session with clean context (use when switching tasks) |
| `/verbose` | Debug memory search results |

---

## Defense-in-Depth Summary

| Layer | Function | Enablement |
|---|---|---|
| Workspace files | Identity + instructions immune to compaction | Structure SOUL.md, AGENTS.md, USER.md, MEMORY.md |
| Pre-compaction flush | Automatic safety net before compression | Verify `memoryFlush.enabled: true` + tune `reserveTokensFloor` |
| Manual memory saves | Relevance-based preservation of important decisions | Habit: "save this to memory" before task switches |
| Strategic `/compact` | Clear decks before new important instructions | `/compact` before, not after, new context |
| Session pruning | Trim tool bloat to delay compaction + save caching | `contextPruning.mode: "cache-ttl"` |
| Hybrid search | Find memories even when wording differs | `query.hybrid.enabled: true` in `memorySearch` |
| Extra paths (Track A+) | Index external docs without switching backends | `memorySearch.extraPaths` for small doc sets |
| QMD (Track B) | Search entire knowledge base | `memory.backend: "qmd"` |
| Git backup | Full history, diffs, rollback for memory files | `git init` in workspace, auto-commit cron |
| Memory hygiene | Prevent bootstrap bloat and context waste | Weekly: distill daily logs into MEMORY.md |

---

## The Full System at a Glance

1. **Workspace files** → compaction-immune instructions
2. **Pre-compaction flush** → automatic safety net
3. **Manual saves** → relevance-based preservation
4. **Strategic `/compact`** → clearing the decks on your terms
5. **Session pruning** → delays compaction, saves on caching
6. **Hybrid search** → finds things even when wording differs
7. **Extra paths / QMD** → searching beyond the workspace
8. **Git backup** → rollback if something goes wrong
9. **Weekly memory hygiene** → prevents bloat

---

## Five Things to Remember

1. **Files are memory.** If it's not on disk, it doesn't exist.
2. **Verify and tune the flush.** Set `reserveTokensFloor` to 40K.
3. **Compact proactively.** `/compact` is your friend mid-session.
4. **Search before acting.** Put this rule in `AGENTS.md`.
5. **Pruning is your friend.** It trims tool bloat and helps caching. Compaction is the one that hurts.

---

*Now go save this to a file. You know what happens if you don't.*
