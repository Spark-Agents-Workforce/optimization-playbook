# 002 — Full OpenClaw Config Performance Audit

**Author:** ConfigClaw ⚙️  
**Date:** 2026-02-14  
**Requested by:** Josh  
**Scope:** Complete openclaw.json audit — all settings, all agents, all performance vectors  
**Optimization goal:** Max intelligence at max speed. Opus everywhere. No model downgrades.

> **Correction (v2):** Original audit was framed around cost savings and model downgrades. That was wrong. Josh's directive: the smartest models must run at peak efficiency. Speed comes from giving Opus less to chew on per turn — not from replacing it.

---

## Executive Summary

After auditing every section of `openclaw.json`, I found **9 actionable speed optimizations** that preserve full Opus intelligence. The biggest wins: **reduce context window to cut API payload size**, **right-size the embedding cache to reduce memory pressure**, and **eliminate unnecessary session bloat from overnight heartbeats and orphan data**.

### Changes Already Applied Today
| Change | Before | After | Speed Impact |
|--------|--------|-------|-------------|
| `contextPruning.ttl` | 3m | 30m | 10× longer Anthropic cache window — massively less reprocessing |
| `compaction.reserveTokensFloor` | 20,000 | 80,000 | Compaction triggers 4× earlier — context stays tighter |
| `memorySearch.sources` | `["memory", "sessions"]` | `["memory"]` | Stopped querying 709MB of session embeddings |
| `sessionMemory` | true | false | Eliminated session embedding overhead |
| Cron run sessions | 131 orphaned | Cleaned | Removed 8.7MB of stale index fodder |

---

## Speed Optimizations (Preserving Full Opus Intelligence)

### 🔴 P1: Reduce `contextTokens` from 500K to 250K

```json
"contextTokens": 500000  // → 250000
```

**The problem:** Every API call sends the accumulated context to Anthropic. At 400K+ tokens (afternoon after heavy use), that's a massive payload. Even with prompt caching, the sheer volume impacts time-to-first-token, serialization overhead, and cache-miss penalty.

**The fix:** 250K context with 80K reserve floor means compaction triggers at 170K tokens. Compaction summarizes what's pruned, so intelligence is preserved — Opus works from a tighter, better-distilled context. Faster turns, same quality answers.

**Why this doesn't sacrifice intelligence:** The compaction system with memoryFlush preserves important context before summarizing. Opus working on a well-compacted 170K context produces answers just as good (often better — less noise) than Opus wading through 420K of raw accumulated history.

### 🔴 P1: Reduce `memorySearch.cache.maxEntries` from 100K to 25K

```json
"cache": { "maxEntries": 100000 }  // → 25000
```

**The problem:** `text-embedding-3-large` produces 3072-dimension vectors. At float32, each embedding is ~12KB. A 100K-entry cache could consume **1.2GB of process memory**. Even partially filled, this creates memory pressure that slows the entire gateway — SQLite queries, JSON processing, everything.

**The fix:** With sessionMemory disabled and sources limited to `["memory"]`, the embedding volume is dramatically lower. 25K entries is more than sufficient. Frees hundreds of MB of memory for the gateway process.

**Intelligence impact:** Zero. The cache is a lookup accelerator. Reducing its ceiling doesn't change search quality — only prevents unbounded memory growth.

---

### 🟠 P2: Add `activeHours` to joshuaday (Twin) heartbeat

```json
// Current:
"heartbeat": { "every": "30m", "model": "anthropic/claude-opus-4-6" }
// Recommended:
"heartbeat": { "every": "30m", "model": "anthropic/claude-opus-4-6",
  "activeHours": { "start": "02:00", "end": "17:00", "timezone": "America/Los_Angeles" } }
```

**The problem:** 15 Opus heartbeat calls fire while Josh sleeps (5:30pm–2am). Each one adds context to Twin's session that serves no purpose. When Josh wakes up, that bloated session is slower to process. Additionally, overnight heartbeats compete for API rate limits.

**Intelligence impact:** Zero. Heartbeats during sleep hours produce no actionable intelligence.

### 🟠 P2: Add `activeHours` to cronclaw heartbeat

Same reasoning. Lower volume (4h interval) but same pattern — overnight heartbeats add dead weight.

```json
"heartbeat": { "every": "4h", "model": "anthropic/claude-opus-4-6",
  "activeHours": { "start": "02:00", "end": "17:00", "timezone": "America/Los_Angeles" } }
```

### 🟠 P2: Clean 75MB orphan SQLite databases

Six embedding databases for agents no longer in the fleet:

| Agent | Size | Status |
|-------|------|--------|
| genghisclawn.sqlite | 22.3MB | Unknown — never in current roster |
| god.sqlite | 18.8MB | Unknown — never in current roster |
| royale.sqlite | 14.4MB | Removed 2026-02-14 |
| architect.sqlite | 12.9MB | Removed 2026-02-14 |
| tryclaw.sqlite | 6.5MB | Unknown |
| watchpost.sqlite | 0.1MB | Unknown |

**Speed impact:** 75MB of dead data on disk. Any tooling scanning the memory directory wastes I/O on these. Cleaning them reduces disk I/O noise.

Also: orphan agent directories (`agents/architect`: 5.1MB, `agents/royale`: 1.0MB) and orphan workspaces.

---

### 🟡 P3: Printer heartbeat interval — 5m → 10m

```json
"heartbeat": { "every": "5m" }  // → "10m"
```

**The problem:** Not about saving money — about **API concurrency contention.** 138 Opus calls/day from Printer alone means it's hitting the Anthropic API every 5 minutes. During Josh's active hours, this competes for rate limits and concurrent request slots with Josh's actual interactive work.

**The fix:** 10-minute intervals halve the API contention. Printer still catches issues within a reasonable window. Each heartbeat is still full Opus intelligence.

**Intelligence impact:** Zero per-heartbeat. Detection latency increases from 5m to 10m max.

### 🟡 P3: Workspace cleanup (1.4GB accumulated)

| Workspace | Size |
|-----------|------|
| workspace (default) | 693MB |
| workspace-uxplorer | 501MB |
| workspace-jdimages | 83MB |
| workspace-joshuaday | 70MB |

**Speed impact:** Disk I/O and potential memory-mapped file pressure from large directory trees. Not urgent but worth periodic maintenance.

---

## Findings Evaluated and Kept at Current Settings

These were considered and determined to be **correct as-is** under the max-intelligence-max-speed lens:

| Setting | Current | Verdict |
|---------|---------|---------|
| Default model | Opus 4.6 | ✅ Correct — max intelligence |
| All heartbeat models | Opus 4.6 | ✅ Correct — intelligence matters even on status checks |
| jdimages model | Opus 4.6 | ✅ Keep — orchestration quality matters |
| sparkcopy model | Opus 4.6 | ✅ Keep — writing quality matters |
| candidateMultiplier | 6 | ✅ Keep — better recall quality on memory search |
| cacheRetention: "long" | On Opus models | ✅ Correct — maximizes Anthropic server cache hits |
| hybrid search weights | 0.7 vector / 0.3 text | ✅ Good balance |
| thinkingDefault: "high" | Fleet-wide | ✅ Correct — max reasoning depth |
| blockStreamingDefault: "on" | Fleet-wide | ✅ Appropriate for multi-agent routing |
| Opus 4.5 in models list | Available | ✅ Harmless — zero cost when unused, available as fallback |

---

## The Speed Formula

> **Opus intelligence is non-negotiable. Speed comes from giving Opus less to chew on per turn.**

1. **Smaller context** (contextTokens: 250K) → less data per API call → faster responses
2. **Longer cache TTL** (30m, applied today) → more Anthropic cache hits → less reprocessing
3. **Earlier compaction** (80K floor, applied today) → context stays tighter → turns stay fast
4. **Less memory pressure** (cache: 25K, sessionMemory: off) → faster gateway operations
5. **Less contention** (activeHours, interval tuning) → API slots available for interactive work
6. **Less disk noise** (orphan cleanup) → faster I/O operations

---

## Priority Action Matrix

| Priority | Action | Effort | Speed Impact | Intelligence Impact | Needs Approval? |
|----------|--------|--------|-------------|-------------------|----------------|
| **P1** | contextTokens → 250K | 1 min | 🔴 High | None (compaction preserves) | Josh confirm |
| **P1** | Embedding cache → 25K | 1 min | 🔴 High | None | Self-execute |
| **P2** | joshuaday activeHours | 1 min | 🟠 Medium | None | Self-execute |
| **P2** | cronclaw activeHours | 1 min | 🟡 Low | None | Self-execute |
| **P2** | Orphan DB cleanup | 2 min | 🟡 Low | None | Josh confirm |
| **P3** | Printer interval → 10m | 1 min | 🟡 Low | None | Josh confirm |
| **P3** | Workspace cleanup | 10 min | 🟡 Low | None | Josh confirm |

---

*Generated by ConfigClaw ⚙️ — v2, corrected for max-intelligence-max-speed optimization lens. 2026-02-14.*
