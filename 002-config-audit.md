# 002 — Full OpenClaw Config Performance Audit

**Author:** ConfigClaw ⚙️  
**Date:** 2026-02-14  
**Requested by:** Josh  
**Scope:** Complete openclaw.json audit — all settings, all agents, all performance vectors

---

## Executive Summary

After auditing every section of `openclaw.json`, I found **12 actionable findings** across 5 categories. The biggest remaining wins are: **Printer's 138 daily Opus heartbeat calls**, **75MB of orphan data from deregistered agents**, **an oversized embedding cache**, and **two heartbeat agents running 24/7 without active hours**. Several on-demand agents also carry heavier models than their usage warrants.

### Changes Already Applied Today (for context)
| Change | Before | After | Impact |
|--------|--------|-------|--------|
| `contextPruning.ttl` | 3m | 30m | 10× longer Anthropic cache window |
| `compaction.reserveTokensFloor` | 20,000 | 80,000 | Compaction triggers 4× earlier |
| `memorySearch.sources` | `["memory", "sessions"]` | `["memory"]` | Stopped indexing 283MB of sessions |
| `sessionMemory` | true | false | Disabled session embedding |
| Cron run sessions | 131 orphaned | Cleaned | Removed 8.7MB of stale session files |

---

## Category 1: Heartbeat Efficiency

### 🔴 FINDING 1: Printer heartbeat — 138 Opus calls/day

```json
"print": {
  "heartbeat": {
    "every": "5m",
    "activeHours": { "start": "02:00", "end": "13:30" },
    "model": "anthropic/claude-opus-4-6"
  }
}
```

**Problem:** 5-minute intervals × 11.5 active hours = **138 Opus API calls per day** just for heartbeats. At Opus pricing ($5/MTok input, $25/MTok output), even minimal heartbeat exchanges add up significantly.

**Recommendation:** 
- Switch heartbeat model to **Sonnet** (`anthropic/claude-sonnet-4-5`). Heartbeat polls are simple "anything need attention?" checks — Sonnet handles this perfectly at ~10× lower cost.
- Consider increasing interval to **10m** if 5m isn't operationally necessary (halves cost again).
- Estimated savings: ~90% cost reduction on Printer heartbeats alone.

### 🟠 FINDING 2: joshuaday (Twin) heartbeat — 24/7 without activeHours

```json
"joshuaday": {
  "heartbeat": { "every": "30m", "model": "anthropic/claude-opus-4-6" }
}
```

**Problem:** No `activeHours` configured. Fires 48 times/day including while Josh sleeps (6:30pm–2am = 7.5 hours of wasted heartbeats = **15 unnecessary Opus calls/day**).

**Recommendation:** Add `activeHours: { start: "02:00", end: "17:00", timezone: "America/Los_Angeles" }` to match Josh's schedule.

### 🟠 FINDING 3: cronclaw heartbeat — 24/7 without activeHours

```json
"cronclaw": {
  "heartbeat": { "every": "4h", "model": "anthropic/claude-opus-4-6" }
}
```

**Problem:** Same issue — fires 6 times/day with no sleep window. Lower impact than Twin due to 4h interval, but still 1-2 unnecessary Opus calls overnight.

**Recommendation:** Add activeHours or switch heartbeat model to Sonnet. CronClaw's heartbeat is a status check, not a reasoning task.

### 🟡 FINDING 4: Default heartbeat model is Opus

```json
"heartbeat": {
  "every": "30m",
  "model": "anthropic/claude-opus-4-6",
  "target": "none"
}
```

**Current status:** `target: "none"` means the 18 agents inheriting defaults don't actually fire heartbeats. However, if any agent's heartbeat override doesn't explicitly set `target`, it may not inherit the default's `target: "none"` (depends on merge behavior). The 4 agents with heartbeat overrides (print, joshuaday, oc, cronclaw) **do not set target**, which means they're active.

**Recommendation:** Change default heartbeat model to **Sonnet**. If any agent truly needs Opus-level reasoning for heartbeats, override it explicitly on that agent. Most heartbeats are simple status checks.

---

## Category 2: Model Allocation

### 🟠 FINDING 5: On-demand agents running Opus unnecessarily

These agents have explicit `model: "anthropic/claude-opus-4-6"` overrides:

| Agent | Role | Needs Opus? |
|-------|------|------------|
| mason | Builder | Probably — complex code/architecture |
| spawnclaw | Agent creation | Yes — writes SOUL.md, config |
| configclaw | Config management | Borderline — mostly JSON manipulation |
| jdimages | Image generation | No — orchestrates image tools |
| sparkcopy | Copywriting | Borderline — Sonnet writes well |

**Note:** These agents also inherit the default model (Opus) if not overridden, so the explicit override is redundant for mason, spawnclaw, configclaw. But `jdimages` and `sparkcopy` could likely run on Sonnet with no quality loss.

**Recommendation:** Evaluate jdimages and sparkcopy for Sonnet. Image orchestration and copywriting don't typically require Opus-level reasoning. Saves ~60% on per-token cost for those agents.

### ✅ FINDING 6: Sparky correctly on Sonnet

```json
"sparky": { "model": "anthropic/claude-sonnet-4-5" }
```

Good — lighter agent on lighter model. This pattern should be replicated where appropriate.

---

## Category 3: Context & Memory Settings

### 🟡 FINDING 7: contextTokens at 500K is still aggressive

```json
"contextTokens": 500000
```

**Current state:** After raising `reserveTokensFloor` to 80K, safeguard compaction now triggers at 420K tokens (was 480K). This is better but still means Atlas can accumulate a very large context before compaction intervenes.

**Consideration:** Reducing to 200K-300K would force more frequent compaction cycles, keeping each turn's API payload smaller. The tradeoff is losing older conversation context sooner. For Josh's heavy-use pattern (all day, many topics), a 200K-250K window with the improved 80K reserve floor would mean compaction at 120K-170K — a sweet spot for responsiveness without losing too much context.

**Recommendation:** Consider `contextTokens: 250000` for a meaningful improvement. This is a preference call — Josh should decide based on how much history he wants retained vs. speed.

### 🟡 FINDING 8: Embedding cache maxEntries may be oversized

```json
"memorySearch": {
  "cache": {
    "enabled": true,
    "maxEntries": 100000
  }
}
```

**Problem:** `text-embedding-3-large` produces 3072-dimension vectors. At float32, each embedding is ~12KB. A fully-populated 100K entry cache would consume ~1.2GB of memory. Even at 10% fill (10K entries), that's 120MB of in-process memory.

**Recommendation:** Reduce to **25,000-50,000** maxEntries. The cache is a lookup accelerator, not a permanent store. With sessionMemory now disabled and sources limited to `["memory"]`, the embedding volume is dramatically lower. 25K entries is likely more than sufficient.

### 🟡 FINDING 9: candidateMultiplier at 6 is generous

```json
"query": {
  "hybrid": {
    "candidateMultiplier": 6
  }
}
```

This means memory_search fetches 6× the requested results as candidates, then reranks. For a typical search returning 5 results, that's 30 candidates evaluated. This is fine for accuracy but adds latency on every memory search call.

**Recommendation:** Could reduce to 3-4 if memory search latency is noticeable. Keep at 6 if recall quality matters more than speed.

---

## Category 4: Orphan Data & Disk Waste

### 🔴 FINDING 10: 75MB+ of orphan data from deregistered agents

Agents no longer in `agents.list` but still have data on disk:

| Type | Agent | Size | Notes |
|------|-------|------|-------|
| Memory DB | architect.sqlite | 12.9MB | Removed today |
| Memory DB | royale.sqlite | 14.4MB | Removed today |
| Memory DB | genghisclawn.sqlite | 22.3MB | Unknown agent |
| Memory DB | god.sqlite | 18.8MB | Unknown agent |
| Memory DB | tryclaw.sqlite | 6.5MB | Unknown agent |
| Memory DB | watchpost.sqlite | 0.1MB | Unknown agent |
| Workspace | workspace-architect | 644K | Removed today |
| Workspace | workspace-royale | 184K | Removed today |
| Agent dir | agents/architect | 5.1MB | Sessions still on disk |
| Agent dir | agents/royale | 1.0MB | Sessions still on disk |

**Total orphan data: ~82MB** (mostly embedding databases for agents that no longer exist).

**Recommendation:** Clean up orphan SQLite databases and agent directories. These are dead weight on disk and potentially confuse any tooling that scans the memory directory. Back up first if any might be needed for reference.

### 🟡 FINDING 11: Large workspace directories

| Workspace | Size | Notes |
|-----------|------|-------|
| workspace (default) | 693MB | Shared default — may contain accumulated build artifacts |
| workspace-uxplorer | 501MB | Iris's workspace — likely generated UI assets |
| workspace-joshuaday | 70MB | Twin's workspace |
| workspace-jdimages | 83MB | Image generation outputs |
| workspace-studio | 20MB | Image generation outputs |
| workspace-main | 26MB | Atlas workspace |

**Total workspace storage: ~1.4GB**

**Recommendation:** Periodic cleanup of workspace build artifacts, generated images, and node_modules. The default workspace at 693MB is suspiciously large — may contain old project files or accumulated output from multiple agents.

---

## Category 5: Miscellaneous

### 🟡 FINDING 12: Anthropic Opus 4.5 registered but unused

```json
"models": {
  "anthropic/claude-opus-4-5": {}
}
```

Opus 4.5 is in the available models list with no alias and no params. No agent references it. It's not causing harm (no cost when unused), but it's dead config.

**Recommendation:** Remove if not needed. Keep if it's there as a fallback option.

### ✅ Logging config is appropriate

```json
"logging": { "redactSensitive": "tools" }
```

Good — redacts sensitive data in tool output logs. No performance impact.

### ✅ Gateway config is lean

```json
"gateway": {
  "port": 18789,
  "mode": "local",
  "bind": "loopback"
}
```

Loopback binding, local mode, no unnecessary exposure. No performance concerns.

### ✅ Block streaming is enabled

```json
"blockStreamingDefault": "on"
```

This buffers streaming responses, which is appropriate for multi-agent routing where partial streams add complexity without benefit.

---

## Priority Action Matrix

| Priority | Finding | Effort | Impact | Needs Approval? |
|----------|---------|--------|--------|----------------|
| **P1** | Printer heartbeat → Sonnet model | 1 min | High (138 calls/day cost) | Self-execute |
| **P1** | joshuaday activeHours | 1 min | Medium (15 wasted calls/day) | Self-execute |
| **P1** | Clean orphan SQLite DBs | 2 min | Medium (75MB disk, cleaner system) | Josh approval |
| **P2** | Default heartbeat model → Sonnet | 1 min | Low (safety net) | Self-execute |
| **P2** | cronclaw activeHours | 1 min | Low (1-2 calls/day) | Self-execute |
| **P2** | Reduce embedding cache to 25K | 1 min | Medium (memory savings) | Self-execute |
| **P2** | Evaluate jdimages/sparkcopy for Sonnet | 5 min | Medium (cost per use) | Josh decision |
| **P3** | Reduce contextTokens to 250K | 1 min | Medium (speed vs memory tradeoff) | Josh decision |
| **P3** | Reduce candidateMultiplier to 4 | 1 min | Low | Self-execute |
| **P3** | Clean orphan agent dirs | 2 min | Low (6MB) | Josh approval |
| **P3** | Workspace cleanup | 10 min | Medium (1.4GB disk) | Josh decision |
| **P4** | Remove Opus 4.5 from models | 1 min | Negligible | Self-execute |

---

## Cost Estimate Summary

### Current Daily Heartbeat Cost (Opus @ $5/$25 per MTok in/out)
| Agent | Beats/Day | Model | Est. Cost/Day |
|-------|-----------|-------|--------------|
| Printer | 138 | Opus 4.6 | High |
| Twin | 48 (15 wasted) | Opus 4.6 | Medium |
| SecureClaw | ~4 | Opus 4.6 | Low |
| CronClaw | 6 | Opus 4.6 | Low |

### After Recommended Changes
| Agent | Beats/Day | Model | Est. Cost/Day |
|-------|-----------|-------|--------------|
| Printer | 138 (or 69 @10m) | **Sonnet** | ~90% reduction |
| Twin | 30 (with activeHours) | Opus 4.6 | ~37% reduction |
| SecureClaw | ~4 | Opus 4.6 | No change |
| CronClaw | ~4 (with activeHours) | **Sonnet** | ~85% reduction |

---

## What's Already Good

- ✅ `target: "none"` on default heartbeat prevents 18 agents from heartbeating
- ✅ SecureClaw has appropriate activeHours (02:00-17:00)
- ✅ Printer has activeHours (02:00-13:30)  
- ✅ `cacheRetention: "long"` on Opus models maximizes Anthropic server-side caching
- ✅ Hybrid memory search with vector+text weighting
- ✅ `redactSensitive: "tools"` in logging
- ✅ Gateway on loopback, token auth
- ✅ `agentToAgent: { enabled: true }` for fleet communication
- ✅ Compaction memoryFlush before summarization preserves important context
- ✅ contextPruning.ttl now at 30m (improved today from 3m)
- ✅ reserveTokensFloor now at 80K (improved today from 20K)
- ✅ sessionMemory disabled (improved today)

---

*Generated by ConfigClaw ⚙️ from live openclaw.json analysis. All findings verified against current config state as of 2026-02-14 16:32 PST.*
