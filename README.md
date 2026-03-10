# OpenClaw Memory Management Guide

A comprehensive guide to understanding, configuring, and optimizing memory in OpenClaw agents — so your agent stops forgetting everything.

Based on insights from an OpenClaw codebase maintainer with 2+ months of daily production use. Source: [Why your OpenClaw agent forgets everything (and how to fix it)](https://www.youtube.com/watch?v=oN__gKJnPls)

## What's Inside

### [The Guide](guide.md)

A complete reference covering:

- **The four memory layers** — bootstrap files, session transcript, LLM context window, and retrieval index
- **Three failure modes** — why your agent forgets and how to diagnose each one
- **Compaction vs. pruning** — the difference most guides get wrong
- **Configuration** — recommended `openclaw.json` settings with `reserveTokensFloor`, memory flush, and hybrid search
- **File architecture** — what goes in `SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md`, and daily logs
- **The memory protocol** — a rule for `AGENTS.md` that makes the agent search before acting
- **Retrieval tracks** — Track A (built-in search), Track A+ (extra paths), Track B (QMD for large vaults)
- **Diagnostics** — using `/context list`, `/status`, `/compact`, and `/verbose`
- **Memory hygiene** — daily, weekly, and git backup practices

### [Claude Code Skill](skills/)

A ready-to-use skill for Claude Code that helps you manage OpenClaw memory:

```
skills/
├── SKILL.md                        # Main skill instructions
└── references/
    └── config-examples.md          # Copy-paste config templates
```

**To install:** Copy the `skills/` directory into your Claude Code skills location (e.g., `~/.claude/skills/openclaw-memory/` or your project's `.claude/skills/` directory).

The skill triggers whenever you mention memory issues, compaction, context loss, bootstrap files, or any OpenClaw memory-related topic. It guides Claude through diagnostics, configuration, and best practices.

## Quick Start

If you read nothing else, do these three things:

1. **Put durable rules in files, not chat.** Write important instructions to `MEMORY.md` and `AGENTS.md`. Chat instructions don't survive compaction.

2. **Enable and tune the memory flush.** Add this to `~/.openclaw/openclaw.json`:
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

3. **Make retrieval mandatory.** Add to your `AGENTS.md`:
   ```markdown
   ## Memory Protocol
   Before doing anything non-trivial, search memory first.
   ```

## License

MIT
