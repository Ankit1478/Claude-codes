# 🧠 A1. Prompt Caching — Full Deep Dive

> **How Claude Code saves 86% on token costs using a 3-level caching system.**
> Study this to understand production-grade LLM cost optimization.

---

## 📑 Table of Contents

1. [What is a Token?](#1-what-is-a-token)
2. [Why Does It Cost Money?](#2-why-does-it-cost-money)
3. [The Problem](#3-the-problem)
4. [The Solution — Boundary Marker](#4-the-solution--boundary-marker)
5. [How the API Receives It](#5-how-the-api-receives-it)
6. [How cache\_control Is Sent to API](#6-how-cache_control-is-sent-to-api)
7. [getCacheControl — The Cache Config](#7-getcachecontrol--the-cache-config)
8. [Section Memoization — Dynamic Section Cache](#8-section-memoization--dynamic-section-cache)
9. [The Resolver — How Cache Lookup Works](#9-the-resolver--how-cache-lookup-works)
10. [Context Memoization — Compute Once Per Session](#10-context-memoization--compute-once-per-session)
11. [Cache Invalidation — When Caches Clear](#11-cache-invalidation--when-caches-clear)
12. [Tool Schema Caching (Bonus)](#12-tool-schema-caching-bonus)
13. [The Complete Flow — All Layers Together](#13-the-complete-flow--all-layers-together)
14. [Cost Savings Breakdown](#14-cost-savings-breakdown)
15. [The 4 Files That Make It Work](#15-the-4-files-that-make-it-work)
16. [AI Engineer Takeaways](#16-ai-engineer-takeaways)

---

## 1. What is a Token?

Before text reaches Claude, it's broken into **tokens**:

```
"Hello, how are you?" → ["Hello", ",", " how", " are", " you", "?"]
                         = 6 tokens
```

| Rule of Thumb | Value |
|---|---|
| 1 token | ≈ 4 characters |
| 1 token | ≈ 0.75 words |
| 100 words | ≈ 130 tokens |

> The Claude Code system prompt alone = **~15,000 tokens**

---

## 2. Why Does It Cost Money?

**Claude API pricing (Opus model):**

| Type | Cost per 1M tokens |
|---|---|
| Input tokens | $15.00 ← what you **send** |
| Output tokens | $75.00 ← what Claude **replies** |
| **Cached** input | **$1.50** ← **90% cheaper!** |

> 💡 **Key insight:** Normal input = $15/M · Cached input = $1.50/M = **90% discount**

---

## 3. The Problem

Every API call sends the **full** system prompt + **full** history:

```
Turn 1:  [system prompt 15K] + [your message 100]
Turn 2:  [system prompt 15K] + [turn 1 history 2K]  + [your message 100]
Turn 3:  [system prompt 15K] + [turn 1-2 history 5K] + [your message 100]
...
Turn 20: [system prompt 15K] + [history 50K] + [your message 100]
```

The system prompt is **identical every single time**.

```
15K tokens × 20 turns = 300K tokens = $4.50 WASTED
```

> Why send `"You are Claude Code..."` 20 times? **Cache it. Send it once. Pay once.**

---

## 4. The Solution — Boundary Marker

Claude Code splits its prompt into **two halves** using a marker string.

**`src/constants/prompts.ts` (line 560–576)**

```typescript
return [
  // ─── STATIC (same for ALL users, ALL sessions) ───
  getSimpleIntroSection(),          // "You are Claude Code..."
  getSimpleSystemSection(),         // permission rules
  getSimpleDoingTasksSection(),     // code quality rules
  getActionsSection(),              // safety rules
  getUsingYourToolsSection(),       // tool usage rules
  getSimpleToneAndStyleSection(),   // tone rules
  getOutputEfficiencySection(),     // "be concise"

  // ═══ THIS STRING SPLITS THE TWO HALVES ═══
  SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
  // = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

  // ─── DYNAMIC (different per session) ───
  ...resolvedDynamicSections,       // memory, env, MCP, etc.
]
```

| Section | Behaviour |
|---|---|
| **Above** boundary | Same text for everyone → **cache globally** |
| **Below** boundary | Different per user/session → **cannot cache globally** |

---

## 5. How the API Receives It

`splitSysPromptPrefix()` in `src/utils/api.ts` (line 321) takes the prompt array and splits it into blocks with cache instructions.

**`src/utils/api.ts` (line 362–404)**

```typescript
// Input:
["You are Claude Code...", "Rules...", ..., BOUNDARY, "Memory...", ...]

// Output:
[
  {
    text: "You are Claude Code...\n\nRules...\n\n...",
    cacheScope: 'global'   // ← cache for ALL users worldwide
  },
  {
    text: "Memory...\n\nEnv info...\n\n...",
    cacheScope: null        // ← no caching (personal data)
  }
]
```

**How it works in code:**

```typescript
const boundaryIndex = systemPrompt.findIndex(
  s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY
)

for (let i = 0; i < systemPrompt.length; i++) {
  const block = systemPrompt[i]

  if (i < boundaryIndex) {
    staticBlocks.push(block)   // before boundary → STATIC
  } else {
    dynamicBlocks.push(block)  // after boundary → DYNAMIC
  }
}

// Static  → cacheScope: 'global'
// Dynamic → cacheScope: null
```

---

## 6. How `cache_control` Is Sent to API

`buildSystemPromptBlocks()` in `src/services/api/claude.ts` (line 3213) converts blocks into the final API format.

**`src/services/api/claude.ts` (line 3213–3237)**

```typescript
function buildSystemPromptBlocks(systemPrompt, enablePromptCaching) {
  return splitSysPromptPrefix(systemPrompt).map(block => {
    return {
      type: 'text',
      text: block.text,

      // Add cache_control only if caching is enabled AND scope is set
      ...(enablePromptCaching && block.cacheScope !== null && {
        cache_control: getCacheControl({
          scope: block.cacheScope,  // 'global' or 'org'
        }),
      }),
    }
  })
}
```

**What gets sent to the API:**

```json
{
  "system": [
    {
      "type": "text",
      "text": "You are Claude Code...[all static sections]...",
      "cache_control": {
        "type": "ephemeral",
        "scope": "global",
        "ttl": "1h"
      }
    },
    {
      "type": "text",
      "text": "Memory...[all dynamic sections]..."
    }
  ]
}
```

---

## 7. `getCacheControl` — The Cache Config

**`src/services/api/claude.ts` (line 358–374)**

```typescript
function getCacheControl({ scope, querySource }) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

**Cache scopes:**

| Scope | Who shares the cache |
|---|---|
| `'global'` | ALL Claude Code users worldwide |
| `'org'` | Users in the same organization |
| `null` | No caching |

**TTL (Time To Live):**

| Setting | Duration |
|---|---|
| default | 5 minutes |
| `'1h'` | 1 hour (for longer sessions) |

---

## 8. Section Memoization — Dynamic Section Cache

Dynamic sections can't be cached by the API (they contain personal data), but they **can** be computed once and reused in-memory.

**`src/constants/systemPromptSections.ts`**

```typescript
// TYPE DEFINITION
type SystemPromptSection = {
  name: string           // unique identifier
  compute: () => string  // function that generates the text
  cacheBreak: boolean    // true = recompute every turn
}

// WAY 1: Memoized (computed ONCE, cached until /clear)
function systemPromptSection(name, compute) {
  return { name, compute, cacheBreak: false }
}

// WAY 2: Uncached (recomputed EVERY turn)
function DANGEROUS_uncachedSystemPromptSection(name, compute, reason) {
  return { name, compute, cacheBreak: true }
  // reason is REQUIRED ← forces developer to explain WHY
}
```

**`src/constants/prompts.ts` (line 491–555)**

```typescript
const dynamicSections = [
  // These compute ONCE and cache:
  systemPromptSection('session_guidance', () => getSessionGuidance()),
  systemPromptSection('memory',           () => loadMemoryPrompt()),
  systemPromptSection('env_info',         () => computeEnvInfo()),
  systemPromptSection('language',         () => getLanguageSection()),
  systemPromptSection('output_style',     () => getOutputStyle()),
  systemPromptSection('scratchpad',       () => getScratchpad()),

  // This recomputes EVERY turn (MCP servers can disconnect):
  DANGEROUS_uncachedSystemPromptSection(
    'mcp_instructions',
    () => getMcpInstructionsSection(),
    'MCP servers connect/disconnect between turns'  // ← reason explained
  ),
]
```

---

## 9. The Resolver — How Cache Lookup Works

**`src/constants/systemPromptSections.ts` (line 43–58)**

```typescript
async function resolveSystemPromptSections(sections) {
  const cache = getSystemPromptSectionCache()  // Map<string, string>

  return Promise.all(
    sections.map(async (section) => {

      // CHECK 1: Is it cached AND not cache-breaking?
      if (!section.cacheBreak && cache.has(section.name)) {
        return cache.get(section.name)  // ← INSTANT, no computation
      }

      // OTHERWISE: Compute the value
      const value = await section.compute()  // ← may do I/O, file reads, etc.

      // Save to cache for next turn
      setSystemPromptSectionCacheEntry(section.name, value)

      return value
    }),
  )
}
```

**What happens per turn:**

| Turn | `memory` | `env_info` | `mcp_instructions` |
|---|---|---|---|
| Turn 1 | cache MISS → compute → save | cache MISS → compute → save | `cacheBreak=true` → recompute |
| Turn 2 | cache HIT → instant ✅ | cache HIT → instant ✅ | `cacheBreak=true` → recompute |
| Turn 3–100 | cache HIT → instant ✅ | cache HIT → instant ✅ | `cacheBreak=true` → recompute |

---

## 10. Context Memoization — Compute Once Per Session

**`src/context.ts` (line 116–189)**

```typescript
// getUserContext: computed ONCE, result stored forever in session
const getUserContext = memoize(async () => {
  const claudeMd = getClaudeMds(await getMemoryFiles())
  return {
    claudeMd,                                   // project instructions
    currentDate: "Today's date is 2026-04-11",  // today's date
  }
})

// getSystemContext: computed ONCE, result stored forever in session
const getSystemContext = memoize(async () => {
  // Run 5 git commands IN PARALLEL:
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),                   // git branch --show-current
    getDefaultBranch(),            // main or master
    exec('git status --short'),    // modified files
    exec('git log --oneline -5'),  // last 5 commits
    exec('git config user.name'),  // your git name
  ])

  return {
    gitStatus: `Branch: ${branch}\nStatus: ${status}\nCommits: ${log}`
  }
})
```

**What happens:**

| Turn | Action | Time |
|---|---|---|
| Turn 1 | Read CLAUDE.md + run 5 git commands | ~250ms |
| Turn 2+ | Return cached result | ~0ms ✅ |

> 💡 **Savings:** 250ms × 99 turns = **~25 seconds saved** per session

---

## 11. Cache Invalidation — When Caches Clear

**`src/constants/systemPromptSections.ts` (line 65–68)**

```typescript
function clearSystemPromptSections() {
  clearSystemPromptSectionState()  // clears the Map
  clearBetaHeaderLatches()         // resets API flags
}
```

| Event | What gets cleared |
|---|---|
| Every turn | Nothing (use cache) ✅ |
| `/clear` command | Section cache cleared, recompute next turn |
| `/compact` command | Section cache cleared, recompute next turn |
| MCP change | Only `mcp_instructions` (`cacheBreak=true`) |
| New session | Everything fresh (new Map, new memoize) |
| App update | Global API cache miss (new hash) |

> **Rule:** Clear as **little** as possible. More cache hits = less cost = faster response.

---

## 12. Tool Schema Caching *(Bonus)*

Not just the prompt — **tool definitions** are also cached.

**`src/utils/api.ts` (line 119–208)**

```typescript
async function toolToAPISchema(tool) {
  // Cache key = tool name (or name:schema for dynamic tools)
  const cacheKey = tool.inputJSONSchema
    ? `${tool.name}:${JSON.stringify(tool.inputJSONSchema)}`
    : tool.name

  const cache = getToolSchemaCache()

  let base = cache.get(cacheKey)
  if (!base) {
    // Cache miss: compute schema
    base = {
      name: tool.name,
      description: await tool.prompt(),
      input_schema: zodToJsonSchema(tool.inputSchema),
    }
    cache.set(cacheKey, base)
  }

  return base  // cache hit: instant
}
```

> 💡 **Why:** 40+ tools × schema computation = expensive. Cache once, reuse every turn. Also prevents mid-session schema drift from feature flag flips.

---

## 13. The Complete Flow — All Layers Together

```
YOU TYPE: "fix the bug"
     │
     ▼
QueryEngine.submitMessage()
     │
     │ ── LAYER 1: Context Memoization ──
     │
     ├── getUserContext()
     │   Turn 1:  read CLAUDE.md + date    (~50ms)
     │   Turn 2+: return cached            (0ms) ✅
     │
     ├── getSystemContext()
     │   Turn 1:  5 git commands           (~200ms)
     │   Turn 2+: return cached            (0ms) ✅
     │
     │ ── LAYER 2: Section Memoization ──
     │
     ├── resolveSystemPromptSections()
     │   ├── 'memory'    → Turn 1: compute. Turn 2+: cached ✅
     │   ├── 'env_info'  → Turn 1: compute. Turn 2+: cached ✅
     │   ├── 'language'  → Turn 1: compute. Turn 2+: cached ✅
     │   └── 'mcp_instr' → EVERY turn: recompute (volatile) ❌
     │
     │ ── LAYER 3: API Prompt Caching ──
     │
     ├── buildSystemPromptBlocks()
     │   ├── Static sections  → cache_control: {scope: 'global'} ✅
     │   └── Dynamic sections → no cache_control               ❌
     │
     │ ── LAYER 4: Tool Schema Caching ──
     │
     ├── toolToAPISchema() × 40 tools
     │   Turn 1:  compute all 40    (slow)
     │   Turn 2+: return cached     (0ms) ✅
     │
     ▼
API CALL: system + messages + tools → Claude
```

---

## 14. Cost Savings Breakdown

**20-turn conversation (realistic example):**
- System prompt: 15K tokens
- Avg history: 30K tokens
- Avg tools: 5K tokens

**❌ Without Caching**

| Turn | Prompt | History | Tools | Cost |
|---|---|---|---|---|
| 1 | 15,000 | 0 | 5,000 | $0.30 |
| 2 | 15,000 | 3,000 | 5,000 | $0.35 |
| 5 | 15,000 | 15,000 | 5,000 | $0.53 |
| 10 | 15,000 | 35,000 | 5,000 | $0.83 |
| 20 | 15,000 | 60,000 | 5,000 | $1.20 |
| **TOTAL** | 300,000 | 600,000 | 100,000 | **$15.00** |

**✅ With Caching**

| Turn | Prompt | History | Tools | Cost |
|---|---|---|---|---|
| 1 (miss) | 15,000 | 0 | 5,000 | $0.30 |
| 2 (hit) | 1,500* | 3,000 | 500* | $0.07 |
| 5 (hit) | 1,500* | 15,000 | 500* | $0.25 |
| 10 (hit) | 1,500* | 35,000 | 500* | $0.55 |
| 20 (hit) | 1,500* | 60,000 | 500* | $0.93 |
| **TOTAL** | 43,500 | 600,000 | 14,500 | **$9.87** |

> *Cached tokens charged at $1.50/1M instead of $15/1M

**💰 Savings: $5.13 per conversation (34%)**
*(Prompt-only savings: 86%. Total is lower because history still grows and can't be cached this way.)*

---

## 15. The 4 Files That Make It Work

### `src/constants/prompts.ts`
Builds the prompt array with the boundary marker.

| Line | Purpose |
|---|---|
| 114 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` constant |
| 491 | Dynamic sections with `systemPromptSection()` |
| 513 | `DANGEROUS_uncachedSystemPromptSection()` |
| 560 | Final assembly: static + boundary + dynamic |

### `src/constants/systemPromptSections.ts`
Section cache layer (memoization).

| Line | Purpose |
|---|---|
| 20 | `systemPromptSection()` — create cached section |
| 32 | `DANGEROUS_uncachedSystemPromptSection()` — volatile |
| 43 | `resolveSystemPromptSections()` — cache lookup + compute |
| 65 | `clearSystemPromptSections()` — invalidation |

### `src/utils/api.ts`
Splits prompt at boundary, assigns cache scopes.

| Line | Purpose |
|---|---|
| 80 | `CacheScope` type: `'global' \| 'org'` |
| 321 | `splitSysPromptPrefix()` — boundary splitting |
| 394 | static → `cacheScope: 'global'` |
| 396 | dynamic → `cacheScope: null` |

### `src/services/api/claude.ts`
Builds final API payload with `cache_control`.

| Line | Purpose |
|---|---|
| 358 | `getCacheControl()` — `{type: 'ephemeral', scope, ttl}` |
| 3213 | `buildSystemPromptBlocks()` — final text blocks |
| 3229 | Attaches `cache_control` to static blocks |

---

## 16. AI Engineer Takeaways

### ✂️ Concept 1: Split your prompt at a boundary
- Static text (same for everyone) → **above** the boundary
- Dynamic text (per-user/session) → **below** the boundary
- The boundary is just a marker string

### 📦 Concept 2: Use 3 cache levels
| Level | Type | Savings |
|---|---|---|
| Level 1 | API cache (global/org scope, hash-based) | 90% cost savings |
| Level 2 | Session cache (in-memory Map) | Compute once per session |
| Level 3 | No cache | Only for truly volatile data — document why |

### 🔁 Concept 3: Memoize context computation
File reads, git commands, config loading → **do once, store result**.
Use `lodash.memoize()` or similar.

### 🔧 Concept 4: Cache tool schemas too
Tool definitions are also tokens. Cache them.

### 🧹 Concept 5: Invalidate lazily
- Don't clear caches on every turn
- Only clear on explicit user actions (`/clear`, `/compact`)
- Stale is OK for most things

### ⚠️ Concept 6: Name your cache-breakers
```typescript
DANGEROUS_uncachedSystemPromptSection(name, compute, REASON)
```
Force the developer to explain **why** caching can't be used. This prevents accidental cache-breaking.

### ⚡ Concept 7: Parallel computation
- `getUserContext()` and `getSystemContext()` run in parallel
- 5 git commands run in parallel
- Use `Promise.all()` everywhere

---

*Source: Claude Code internal architecture — `src/constants/prompts.ts`, `src/constants/systemPromptSections.ts`, `src/utils/api.ts`, `src/services/api/claude.ts`*
