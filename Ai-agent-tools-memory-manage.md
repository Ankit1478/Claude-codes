# 🗄️ Caching WITHOUT API 

> **Same input → same output → why compute again?**
> Store the answer. Return the stored answer next time. That's ALL caching is.

---

## 4 Methods

### 1. Variable
Store the prompt in a variable at app start. Reuse the same variable every call. Never rebuild the prompt text.

- ✅ Costs nothing, zero overhead
- ❌ Dies when app restarts

### 2. Dictionary (In-Memory)
`Input → hash → check dictionary → found? return saved answer.`
Not found → call LLM → save answer in dictionary.

- ✅ Fast
- ❌ Dies when app restarts

### 3. File Cache
`Input → hash → check if file exists → found? read file.`
Not found → call LLM → save answer as a file on disk.

- ✅ Survives restart
- ❌ Slower than memory

### 4. Redis
`Input → hash → check Redis → found? return saved answer.`
Not found → call LLM → save in Redis with expiry time.

- ✅ Survives restart
- ✅ Fast
- ✅ Works across multiple servers

---

## How Each Works

When the **same question** comes in:

| Method | What it says | What it saves |
|---|---|---|
| **Variable** | "I have the prompt ready" | Rebuild time |
| **Dictionary** | "I answered this before" | LLM call |
| **File Cache** | "Answer is in `/cache/abc123.json`" | LLM call |
| **Redis** | "Answer is in Redis key `abc123`" | LLM call |

---

## When to Use

| Method | Use When |
|---|---|
| **Variable** | Always — costs nothing, basic optimization |
| **Dictionary** | Small app, few users, OK to lose cache on restart |
| **File Cache** | Side project, single server, need persistence |
| **Redis** | Production, multiple servers, millions of users |
