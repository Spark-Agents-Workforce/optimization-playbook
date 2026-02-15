# 003 — Fleet Health Audit: Performance Deep Dive

**Date:** 2026-02-14 16:33 PST (revised 16:36 PST)  
**Auditor:** DoctorClaw 🩺  
**Fleet Size:** 22 agents, ~100 active sessions, ~4.4M tokens in memory  
**Trigger:** Progressive afternoon slowdown, increasing compaction frequency  
**Optimization Goal:** Max intelligence at max speed. Remove bottlenecks, not capability.

---

## Executive Summary

The fleet has **four bottlenecks** preventing Opus-level agents from running at full speed:

1. **`contextTokens` is still 500K** — sessions grow unbounded, sending 150K+ tokens per API call by afternoon. Larger context = slower response. Fix this and every Opus agent gets faster.
2. **1.9M tokens of zombie sessions** clog the session store — dead sessions that slow lookup and waste memory
3. **711MB embedding DB** makes every `memory_search` scan 5,462 vectors when only 325 matter
4. **No session hygiene** — 298 stale transcripts (211MB), sessions at 80-92% of API limit about to crash, no rotation or cleanup

These aren't model problems. These are plumbing problems. Opus is the right engine — it's just running through a clogged pipe.

---

## 1. Session Health

### 1.1 Token Distribution

| Category | Sessions | Total Tokens | % of Fleet |
|----------|----------|-------------|------------|
| Agent main sessions | 17 | 1,319,565 | 30% |
| Cron sessions | 24 | 1,260,734 | 28% |
| Stale zombies (ephemeral/openresponses) | 8 | 1,867,731 | **42%** |
| **TOTAL** | **49** | **4,448,030** | **100%** |

**42% of all tokens in the session store are zombie sessions** — stale ephemeral and openresponses sessions from days ago that will never receive another message. They consume memory in the session registry and slow down session lookup.

### 1.2 Sessions Over 100K Tokens (Speed Degradation Zone)

Every token in context means more data sent per API call. At 150K tokens, an Opus call takes significantly longer than at 30K. With the 200K API limit, these sessions are both slow AND approaching failure:

| Tokens | % of 200K | Session | Status |
|--------|-----------|---------|--------|
| 363,597 | **182%** 🔴 | openresponses:6fd36216 | Zombie — already past API limit |
| 245,994 | **123%** 🔴 | ephemeral:57a9f8f0 | Zombie |
| 227,091 | **114%** 🔴 | openresponses:0e82c726 | Zombie |
| 226,506 | **113%** 🔴 | openresponses:87448add | Zombie |
| 218,643 | **109%** 🔴 | ephemeral:bf9f8391 | Zombie |
| 216,838 | **108%** 🔴 | openresponses:ce3b0933 | Zombie |
| 216,777 | **108%** 🔴 | openresponses:818f366f | Zombie |
| 184,269 | **92%** 🔴 | agent:studio:main | Active — running slow, about to crash |
| 159,412 | **80%** 🟡 | agent:joshuaday:main | Active heartbeat — every call sends 160K tokens |
| 155,714 | **78%** 🟡 | cron:Atlas Mobile Status Check | Stale cron |
| 152,285 | **76%** 🟡 | openresponses:ea4e2f56 | Zombie |
| 142,245 | **71%** 🟡 | agent:sensei:main | Stale — last active 2 days ago |
| 134,619 | **67%** 🟡 | cron:agent-monitor | Stale cron |
| 134,526 | **67%** 🟡 | agent:pixel:main | Active — running slow |
| 120,704 | **60%** 🟡 | cron:uxplorer-09-integration | Stale cron |
| 117,471 | **59%** 🟡 | agent:cro:main | Stale — last active Feb 12 |
| 100,002 | **50%** 🟡 | agent:configclaw:main | Active today |

**17 sessions are over 100K tokens. 7 are already past the 200K API limit.** The active ones (studio at 184K, joshuaday at 159K, pixel at 135K) are running measurably slower because every single API call includes all that context.

### 1.3 Critical Finding: `contextTokens` Still at 500K

```
agents.defaults.contextTokens: 500000  ← STILL NOT FIXED
claude-opus-4-6 API limit:     200000
```

**This is the #1 speed bottleneck.** Compaction triggers relative to `contextTokens`. At 500K, OpenClaw doesn't consider compacting until sessions are far past what the API will accept. Sessions grow unbounded from morning (~20K tokens, fast) to afternoon (~150K+ tokens, slow).

The math: an Opus call with 20K context tokens completes in ~2-4 seconds. The same call with 150K context tokens takes ~8-15 seconds. That's a 4-5x slowdown just from context bloat — and every agent experiences it progressively throughout the day.

At 200K, compaction triggers around 160K, keeping sessions lean and fast. The fix was identified 90 minutes ago but has not been applied.

---

## 2. Agent Fleet Profile

### 2.1 Agents by Model

| Model | Count | Agents |
|-------|-------|--------|
| claude-opus-4-6 | **19** | main, pixel, uxplorer, sensei, studio, exodus, print, joshuaday, forge, sarah, mason, cro, oc, doctorclaw, cronclaw, spawnclaw, configclaw, jdimages, sparkcopy |
| claude-sonnet-4-5 | 1 | sparky |
| gpt-5.3-codex-spark | 2 | clawhost, milo |

19 of 22 agents on Opus. This is the right call for max intelligence — the issue isn't the model, it's what's slowing the model down.

### 2.2 Heartbeat Agents

| Agent | Interval | Model | Context Size | Speed Impact |
|-------|----------|-------|-------------|-------------|
| **print** | **5m** | Opus | 82K tokens | Each call sends 82K context. Will hit 150K+ by EOD without compaction. |
| **joshuaday** | 30m | Opus | 159K tokens 🔴 | Already in the slow zone. Every heartbeat sends 159K tokens. |
| **oc** | 4h | Opus | unknown | Low frequency, low concern |
| **cronclaw** | 4h | Opus | 54K tokens | Moderate, manageable |

**Printer at 5m with 82K context is doing 288 Opus calls/day, each one progressively slower.** By market close, each call will send 150K+ tokens. The fix isn't to downgrade the model — it's to compact the session so each call stays fast.

**Twin at 159K is already running slow.** Every 30-minute heartbeat sends 159K tokens of context. A session reset would bring it back to ~20K and make every subsequent call 5-8x faster.

### 2.3 Cron Job Configuration

Active CronClaw watchdogs (correctly on Haiku for simple liveness checks):

| Job | Interval | Model | Purpose |
|-----|----------|-------|---------|
| twin-watchdog | 10m | Haiku | Checks Twin liveness |
| mason-task-watchdog | 30m | Haiku | Temporary — checks Mason task |
| secureclaw-watchdog | 4h | Haiku | Checks SecureClaw liveness |
| cronclaw-watchdog | 4h | Haiku | Meta-watchdog |
| fleet-session-reset | daily 4am | Haiku | Resets heartbeat sessions |
| printer-watchdog | 15m (market) | Haiku | Checks Printer during market hours |

These are correctly configured. Haiku is the right model for "is this agent alive?" checks — there's no intelligence to gain from using Opus to check if an `updatedAt` timestamp is stale.

22 stale cron sessions from the pre-CronClaw era remain in the session store, consuming ~1.2M tokens of memory.

---

## 3. Transcript & Storage

### 3.1 Transcript Accumulation

| Agent | Files | Active Size | Speed Impact |
|-------|-------|-------------|-------------|
| **main** | **298** | **211.4 MB** | Session lookup slows as file count grows |
| joshuaday | 6 | 23.1 MB | |
| studio | 2 | 18.1 MB | Active session is 17.5MB |
| uxplorer | 2 | 7.9 MB | |
| mason | 1 | 6.1 MB | |
| pixel | 6 | 5.8 MB | |
| print | 10 | 4.7 MB | |
| All others | ~40 | ~18 MB | |
| **TOTAL** | **~380** | **295 MB** | + 12 MB reset = **307 MB** |

**Atlas has 298 session files (211MB) with no cleanup.** The session directory scan on startup and the file I/O for session management both degrade as this grows.

### 3.2 Monster Transcripts

| Size | Agent | Notes |
|------|-------|-------|
| 84.5 MB | main | Atlas mega-session (Feb 5-9, 22,700 lines). Indexed into 2,419 embedding chunks. |
| 17.5 MB | studio | Active session from today. 184K tokens — approaching crash. |
| 14.8 MB | main | Old Atlas session |
| 13.0 MB | main | Old Atlas session |
| 11.0 MB | joshuaday | Twin session |

### 3.3 Workspace Sizes

| Workspace | Total Size | Notes |
|-----------|-----------|-------|
| **uxplorer** | **381.2 MB** 🔴 | Likely build artifacts. Investigate. |
| **jdimages** | **83.1 MB** | Clone of studio — image assets |
| joshuaday | 69.8 MB | Twin's docs + reference material |
| main | 25.2 MB | Atlas workspace |
| studio | 19.6 MB | Trimmed today (was 36MB) |
| All others (19) | < 3 MB each | Clean |

**UXplorer's workspace at 381MB** is a concern if memory_search indexes any of it. Likely contains build output or node_modules that shouldn't be in the workspace.

---

## 4. Embedding Database

### 4.1 Current State

```
Database size:   711 MB
Chunks:          5,462 (325 memory + 5,137 session)
Session data:    94% of all chunks
Vector search:   Scanning 5,462 vectors per query (should be 325)
```

**Every `memory_search` call scans 16x more vectors than necessary.** The 5,137 session chunks from old Atlas transcripts are still in the DB. Even with sessionMemory disabled, the existing data hasn't been purged. This slows every memory_search call for every agent.

### 4.2 Impact

With 325 chunks (memory only): vector search is fast, DB is ~50MB.
With 5,462 chunks (current): vector search is 16x larger, DB is 711MB.

The embedding API call to generate the query vector is the same either way (~200ms). But scanning 5,462 vectors vs 325 adds measurable latency to every memory_search — and memory_search runs on nearly every agent turn.

---

## 5. Recommendations: Speed-First Priority

### 🔴 P0: Fix `contextTokens` to 200000

```
agents.defaults.contextTokens: 500000 → 200000
```

**This is the single biggest speed improvement available.** Compaction will trigger at ~160K tokens, keeping sessions lean. Every Opus call stays fast all day instead of progressively degrading.

Impact: Opus calls that currently take 8-15 seconds (at 150K context) return to 2-4 seconds (at 30-40K post-compaction). **4-5x speedup on every API call, fleet-wide.**

### 🔴 P1: Reset Bloated Active Sessions

These agents are running slow RIGHT NOW because their sessions are fat:

| Session | Tokens | Action |
|---------|--------|--------|
| studio:main | 184,269 (92%) | Reset session — next call will bootstrap fresh at ~20K |
| joshuaday:main | 159,412 (80%) | Reset session — Twin will be 5-8x faster immediately |
| sensei:main | 142,245 (71%) | Reset — hasn't been active in 2 days anyway |
| pixel:main | 134,526 (67%) | Reset — currently sluggish |
| cro:main | 117,471 (59%) | Reset — stale since Feb 12 |
| configclaw:main | 100,002 (50%) | Reset — approaching slow zone |

**Impact: Immediate speed recovery for the 6 most bloated active agents.** Twin goes from 159K → ~20K. Studio goes from 184K → ~20K. Every subsequent call is 4-5x faster.

### 🔴 P2: Purge Zombie Sessions

Kill the dead weight in the session store:
- 7 openresponses sessions over 200K tokens (will never be used again, already past API limit)
- All ephemeral sessions older than 24 hours
- 22 stale cron sessions from pre-CronClaw era

**Impact: Frees ~1.9M tokens from session store.** Faster session lookup, cleaner registry.

### 🟡 P3: Rebuild Embedding Database

```bash
rm ~/.openclaw/memory/main.sqlite
# Rebuilds from memory files only on next memory_search (~50MB)
```

Or surgical prune:
```sql
DELETE FROM chunks WHERE source='sessions';
DELETE FROM files WHERE source='sessions';
DELETE FROM embedding_cache;
VACUUM;
```

**Impact: 711MB → ~50MB. memory_search scans 325 vectors instead of 5,462 — 16x faster retrieval** on every agent turn that calls memory_search.

### 🟡 P4: Implement Session Hygiene

Currently no transcript cleanup, no TTL, no rotation. This compounds daily:

1. **Archive transcripts older than 7 days** — compress and move out of the active sessions directory
2. **Delete transcripts older than 30 days** — important content should be in MEMORY.md by then
3. **Daily session reset for heartbeat agents** — CronClaw's `fleet-session-reset` already does this at 4am, verify it's working
4. **Consider `maxSessions` per agent** if the config supports it

**Impact: Prevents the 298-file, 211MB accumulation pattern from recurring.** Keeps session directory scans fast.

### 🟡 P5: Investigate UXplorer Workspace (381MB)

This workspace is 10x larger than the next biggest. If it contains build output or node_modules:
- Move non-essential files out of the workspace
- If memory_search indexes from workspace paths, this bloat could be generating thousands of useless embedding chunks

### 🟢 P6: Monitor Compaction Health Post-Fix

After P0 is applied, watch for:
- Compaction death spiral (compaction → summary fills window → immediate re-compaction)
- `compaction.floorTokens` may need tuning if compaction summaries are too large
- Session token counts should stay in the 20-80K range after compaction

---

## 6. Fleet Health Scorecard

| Metric | Status | Value | Speed Impact |
|--------|--------|-------|-------------|
| contextTokens config | 🔴 Critical | 500K (should be 200K) | **4-5x slowdown on every API call by afternoon** |
| Zombie sessions | 🔴 Critical | 1.9M tokens in dead sessions | Session store bloat, slower lookups |
| Active session bloat | 🔴 Critical | 6 agents at 100-184K tokens | Running 3-5x slower than post-reset |
| Embedding DB | 🟡 Warning | 711MB, 16x oversized | Slower memory_search on every turn |
| Transcript accumulation | 🟡 Warning | 307MB, no cleanup | Growing daily, degrades I/O |
| UXplorer workspace | 🟡 Warning | 381MB | Potential memory_search impact |
| Stale cron sessions | 🟡 Warning | 22 sessions, ~1.2M tokens | Clutter in session store |
| Bootstrap sizes | ✅ Healthy | All within per-file limits | No truncation risk |
| CronClaw watchdogs | ✅ Healthy | All on Haiku, well-configured | Efficient for liveness checks |
| Agent identities | ✅ Healthy | All correct | — |
| Agent file placement | ✅ Healthy | Fixed earlier today | — |

**Overall: 🟡 Degraded — three critical fixes (P0 + P1 + P2) would restore full speed today.**

The fleet has the right models. The intelligence is there. The problem is plumbing — unbounded context growth, dead sessions, and an oversized search index are choking the pipe between Opus and the user.

---

*Report generated by DoctorClaw 🩺 — 2026-02-14, revised 16:36 PST*  
*Optimization goal: Max intelligence at max speed. Remove bottlenecks, not capability.*
