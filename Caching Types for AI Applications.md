# Caching Types for AI Applications — Cheatsheet

## All Caching Types

### 1. Prompt Caching
Send same system prompt → API remembers it → 90% off next time.

### 2. Response Caching (Redis/Memcached)
Same question asked before → return saved answer → no API call.

### 3. Semantic Caching
Similar (not exact) question asked → return similar answer.
Uses embeddings to match "fix bug" ≈ "resolve the issue".

### 4. KV Cache (Key-Value)
LLM internally caches attention computations for previous tokens.
Built into the model. You don't control it.

### 5. Context Caching (Google Gemini)
Upload large context once → reuse across multiple API calls.
Pay once for 500-page doc, query it many times.

### 6. Session Caching
Store conversation history in Redis/DB → restore on reconnect.
User closes browser, comes back, conversation continues.

### 7. Tool/Schema Caching
Cache tool definitions in memory → don't recompute every call.

### 8. File State Caching (LRU)
Cache file contents in memory → don't re-read from disk.
100 files max, evict oldest when full.

### 9. Embedding Caching
Cache vector embeddings → don't re-embed same text.
Text → vector is expensive. Do it once, store in DB.

### 10. CDN/Edge Caching
Cache AI responses at CDN edge → serve from nearest server.
For public/common queries (FAQ bots, docs).
