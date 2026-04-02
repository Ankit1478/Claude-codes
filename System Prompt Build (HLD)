# Claude Code — System Prompt Build (HLD)
> High-Level Design Document

---

## 1. Architecture Overview

```
User Input
    │
    ▼
QueryEngine ──→ fetchSystemPromptParts() ──→ buildEffectiveSystemPrompt()
                       │                              │
                  ┌────┴────┐                         ▼
                  │ 3 parts │                   Priority Chain
                  └────┬────┘                         │
                       │                              ▼
                       └──────────► Final SystemPrompt[] ──→ Claude API
```

---

## 2. Three Data Sources

```
┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   getSystemPrompt() │  │ getUserContext()  │  │ getSystemContext()│
│   prompts.ts        │  │ context.ts        │  │ context.ts       │
├─────────────────────┤  ├──────────────────┤  ├──────────────────┤
│ • Identity          │  │ • CLAUDE.md      │  │ • Git branch     │
│ • Rules             │  │   (project       │  │ • Git status     │
│ • Tool guidance     │  │    instructions)  │  │ • Recent commits │
│ • Tone / style      │  │ • Current date   │  │                  │
│ • Memory            │  │                  │  │                  │
│ • Env info          │  │                  │  │                  │
│ • MCP instructions  │  │                  │  │                  │
└─────────────────────┘  └──────────────────┘  └──────────────────┘
        │                        │                       │
        └────────────────────────┼───────────────────────┘
                                 ▼
                          Sent to Claude API
```

---

## 3. System Prompt Structure

```
┌───────────────────────────────────────────────────┐
│            STATIC SECTIONS (cached)                │
│                                                    │
│  ┌──┐                                              │
│  │1 │ Intro    — "You are Claude Code..."          │
│  ├──┤                                              │
│  │2 │ System   — permissions, tool execution rules │
│  ├──┤                                              │
│  │3 │ Tasks    — code quality, security, no bloat  │
│  ├──┤                                              │
│  │4 │ Actions  — confirm destructive ops           │
│  ├──┤                                              │
│  │5 │ Tools    — "Use Read not cat, Edit not sed"  │
│  ├──┤                                              │
│  │6 │ Tone     — concise, no emojis, file:line     │
│  ├──┤                                              │
│  │7 │ Output   — "Go straight to the point"        │
│  └──┘                                              │
├─ ─ ─ ─ ─ ─ CACHE BOUNDARY ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ──┤
│          DYNAMIC SECTIONS (per-session)             │
│                                                    │
│  ┌──┐                                              │
│  │8 │ Session  — tool tips, skill hints            │
│  ├──┤                                              │
│  │9 │ Memory   — ~/.claude/memory/ files           │
│  ├──┤                                              │
│  │10│ Env      — CWD, platform, shell, model       │
│  ├──┤                                              │
│  │11│ Language — user language preference           │
│  ├──┤                                              │
│  │12│ MCP      — MCP server instructions           │
│  ├──┤                                              │
│  │13│ Scratch  — temp session directory             │
│  └──┘                                              │
└───────────────────────────────────────────────────┘
```

---

## 4. Priority Chain (which prompt wins)

```
Highest ┌──────────────────────┐
   ★    │ Override prompt      │  internal override
        ├──────────────────────┤
   ★    │ Coordinator prompt   │  multi-agent mode
        ├──────────────────────┤
   ★    │ Agent prompt         │  sub-agent definition
        ├──────────────────────┤
   ★    │ Custom prompt        │  --system-prompt flag
        ├──────────────────────┤
   ★    │ DEFAULT PROMPT       │  ← normal mode (sections 1-13)
        ├──────────────────────┤
   +    │ Append prompt        │  always added at end
Lowest  └──────────────────────┘
```

> First non-null wins. Append is always added regardless.

---

## 5. Special Modes

| Mode        | What changes                                  |
|-------------|-----------------------------------------------|
| Normal      | All 13 sections + context                     |
| Simple      | Only CWD + date (`env CLAUDE_CODE_SIMPLE`)    |
| Proactive   | Autonomous identity + memory + env            |
| Coordinator | Multi-agent coordinator prompt                |
| Agent       | Parent-defined prompt replaces default        |

---

## 6. Caching Strategy

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Sections 1-7   ──→  Global cache (cross-session)       │
│                       keyed by Blake2b hash              │
│                                                          │
│  Sections 8-13  ──→  Memoized (per-session)             │
│                       cleared on /clear or /compact      │
│                                                          │
│  MCP instruct   ──→  UNCACHED (can change mid-session)  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 7. Final API Payload

```
┌────────────────────────────────────────────────────┐
│  POST /v1/messages                                  │
│                                                    │
│  {                                                 │
│    system:   SystemPrompt[]    ← sections 1-13     │
│    context:  {                                     │
│      claudeMd:     "...",      ← CLAUDE.md         │
│      currentDate:  "2026-04-02",                   │
│      gitStatus:    "branch: main, ..."             │
│    },                                              │
│    messages: Message[],        ← conversation      │
│    tools:    ToolDef[],        ← 40+ tool schemas  │
│    model:    "claude-opus-4-6"                     │
│  }                                                 │
└────────────────────────────────────────────────────┘
```

---

## 8. Key Files

| File | Purpose |
|------|---------|
| `src/constants/prompts.ts` | Prompt text (sections 1–13) |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` |
| `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt()` |
| `src/constants/systemPromptSections.ts` | Cache / memoization logic |
| `src/context.ts` | `getUserContext()` + `getSystemContext()` |
| `src/QueryEngine.ts` | Orchestrates everything |
