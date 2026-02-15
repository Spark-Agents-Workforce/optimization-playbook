# 003 — Fleet Health Audit: Performance Deep Dive

**Date:** 2026-02-14 16:33 PST  
**Auditor:** DoctorClaw 🩺  
**Fleet Size:** 22 agents, ~100 active sessions, ~4.4M tokens in memory  
**Trigger:** Progressive afternoon slowdown, increasing compaction frequency

---

## Executive Summary

The fleet has **five compounding performance problems**, not one:

1. **`contextTokens` is still 500K** — compaction never triggers proactively (API limit is 200K)
2. **42% of session tokens are zombie sessions** that will never be used again
3. **19 of 22 agents run on Opus** — including heartbeats and simple monitoring tasks
4. **All 4 heartbeat agents use Opus** — Printer at 5-minute intervals on Opus is the most expensive agent in the fleet
5. **298 stale transcript files (211MB) accumulating** under the main agent with no cleanup

Estimated waste: **~60-70% of daily token spend** goes to sessions, models, and cron jobs that don't need it.

---

## 1. Session Health

### 1.1 Token Distribution

| Category | Sessions | Total Tokens | % of Fleet |
|----------|----------|-------------|------------|
| Agent main sessions | 17 | 1,319,565 | 30% |
| Cron sessions | 24 | 1,260,734 | 28% |
| Stale zombies (ephemeral/openresponses) | 8 | 1,867,731 | **42%** |
| **TOTAL** | **49** | **4,448,030** | **100%** |

**42% of all tokens in the session store are zombie sessions** — stale ephemeral and openresponses sessions from days ago that will never receive another message. They consume memory in the session registry and slow down session lookup operations.

### 1.2 Sessions Over 100K Tokens (Danger Zone)

With the 200K API limit, sessions above 100K are at 50%+ capacity and approaching compaction/failure territory:

| Tokens | % of 200K | Session | Status |
|--------|-----------|---------|--------|
| 363,597 | **182%** 🔴 | openresponses:6fd36216 | Zombie — already past API limit |
| 245,994 | **123%** 🔴 | ephemeral:57a9f8f0 | Zombie |
| 227,091 | **114%** 🔴 | openresponses:0e82c726 | Zombie |
| 226,506 | **113%** 🔴 | openresponses:87448add | Zombie |
| 218,643 | **109%** 🔴 | ephemeral:bf9f8391 | Zombie |
| 216,838 | **108%** 🔴 | openresponses:ce3b0933 | Zombie |
| 216,777 | **108%** 🔴 | openresponses:818f366f | Zombie |
| 184,269 | **92%** 🔴 | agent:studio:main | Active — will hit wall soon |
| 159,412 | **80%** 🟡 | agent:joshuaday:main | Active heartbeat |
| 155,714 | **78%** 🟡 | cron:Atlas Mobile Status Check | Stale cron |
| 152,285 | **76%** 🟡 | openresponses:ea4e2f56 | Zombie |
| 142,245 | **71%** 🟡 | agent:sensei:main | Stale — last active 2 days ago |
| 134,619 | **67%** 🟡 | cron:agent-monitor | Stale cron |
| 134,526 | **67%** 🟡 | agent:pixel:main | Active task |
| 120,704 | **60%** 🟡 | cron:uxplorer-09-integration | Stale cron |
| 117,471 | **59%** 🟡 | agent:cro:main | Stale — last active Feb 12 |
| 100,002 | **50%** 🟡 | agent:configclaw:main | Active today |

**17 sessions are over 100K tokens. 7 are already past the 200K API limit** (zombies that crashed and were never cleaned up).

### 1.3 Critical Finding: `contextTokens` Still at 500K

```
agents.defaults.contextTokens: 500000  ← STILL NOT FIXED
claude-opus-4-6 API limit:     200000
```

**This is the #1 performance problem.** Compaction triggers relative to `contextTokens`. At 500K, OpenClaw doesn't consider compacting until sessions are far past what the API will accept. Sessions grow unbounded from morning (~20K) to afternoon (~150K+) until they hit the hard API rejection at 200K.

The fix was identified 90 minutes ago but has not been applied.

---

## 2. Model Efficiency

### 2.1 Agents by Model

| Model | Count | Agents |
|-------|-------|--------|
| claude-opus-4-6 | **19** | main, pixel, uxplorer, sensei, studio, exodus, print, joshuaday, forge, sarah, mason, cro, oc, doctorclaw, cronclaw, spawnclaw, configclaw, jdimages, sparkcopy |
| claude-sonnet-4-5 | 1 | sparky |
| gpt-5.3-codex-spark | 2 | clawhost, milo |

**19 of 22 agents (86%) run on Opus.** This is the most expensive model in the fleet. Many of these agents don't need Opus-level reasoning.

### 2.2 Heartbeat Model Waste

| Agent | Interval | Model | Cost Impact |
|-------|----------|-------|-------------|
| **print** | **5m** | **Opus** 🔴 | ~288 Opus calls/day. Most expensive agent in the fleet. |
| **joshuaday** | **30m** | **Opus** 🔴 | ~48 Opus calls/day for Twitter monitoring |
| **oc** | **4h** | **Opus** 🔴 | ~6 Opus calls/day for security checks |
| **cronclaw** | **4h** | **Opus** 🔴 | ~6 Opus calls/day for fleet monitoring |

**Printer at 5-minute Opus heartbeats is burning ~288 Opus calls per day.** This is almost certainly the single most expensive line item in the fleet. A heartbeat check ("scan for trades, check positions") doesn't require Opus-level reasoning — Sonnet handles this pattern identically.

### 2.3 Model Downgrade Candidates

Agents that could run on Sonnet with no quality loss:

| Agent | Current | Recommended | Rationale |
|-------|---------|-------------|-----------|
| print (heartbeat) | Opus | **Sonnet** | Heartbeat checks are scan-and-report. Sonnet excels here. |
| joshuaday (heartbeat) | Opus | **Sonnet** | Timeline scanning, engagement decisions. Sonnet is sufficient. |
| oc (heartbeat) | Opus | **Sonnet** | Security scans are checklist operations. |
| cronclaw (heartbeat) | Opus | **Sonnet** | Fleet monitoring is status checks. |
| mason | Opus | **Sonnet** | Task execution, coding. Sonnet handles this well. |
| sarah | Opus | **Sonnet** | PR/communications. No complex reasoning needed. |
| configclaw | Opus | **Sonnet** | Config management. Deterministic operations. |
| sparkcopy | Opus | Keep Opus | Creative writing benefits from Opus reasoning. |
| spawnclaw | Opus | Keep Opus | Agent design requires deep reasoning. |

**Switching heartbeat models alone would reduce the most expensive recurring costs by ~80%.**

### 2.4 Cron Job Model Waste

**All 24 tracked cron sessions show `claude-opus-4-6` as the model**, even the ones configured with Haiku. This is because the cron session inherits the agent's default model, not the cron's `payload.model`.

Active CronClaw watchdogs are correctly configured with `anthropic/claude-haiku-4-5` in their payload — but the 13 stale `main` agent crons (agent-monitor, Atlas Mobile Status Check, uxplorer builders, etc.) all ran on Opus at the agent default.

---

## 3. Transcript & Storage

### 3.1 Transcript Accumulation

| Agent | Files | Active Size | Notes |
|-------|-------|-------------|-------|
| **main** | **298** | **211.4 MB** | + 12.2 MB reset files. Massive accumulation. |
| joshuaday | 6 | 23.1 MB | Twin's timeline scanning generates volume |
| studio | 2 | 18.1 MB | Current active session is 17.5 MB |
| uxplorer | 2 | 7.9 MB | |
| mason | 1 | 6.1 MB | Single long-running task |
| pixel | 6 | 5.8 MB | |
| print | 10 | 4.7 MB | Market hours accumulation |
| doctorclaw | 6 | 4.1 MB | |
| cronclaw | 29 | 3.3 MB | Many small cron sessions |
| All others | ~20 | ~10 MB | |
| **TOTAL** | **~380** | **295 MB** | + 12 MB reset = **307 MB** |

**The main (Atlas) agent has 298 session files consuming 211MB.** This is 72% of all transcript storage. No session cleanup, TTL, or rotation is configured.

### 3.2 Top Monster Transcripts

| Size | Agent | Session |
|------|-------|---------|
| 84.5 MB | main | ba09ed85 — Atlas mega-session (Feb 5-9, 22,700 lines) |
| 17.5 MB | studio | eabc2823 — Active session (today) |
| 14.8 MB | main | 4c967b4d |
| 13.0 MB | main | ea3b043a |
| 11.0 MB | joshuaday | Twin session |

The 84.5MB Atlas session is a single JSONL file with 22,700 lines. This was indexed into the embedding DB, generating 2,419 chunks and 90MB of embeddings.

### 3.3 Workspace Sizes

| Workspace | Total Size | MD Files | Notes |
|-----------|-----------|----------|-------|
| **uxplorer** | **381.2 MB** 🔴 | 5,634 KB | Massive — likely build artifacts |
| **jdimages** | **83.1 MB** 🟡 | 30 KB | Clone of studio — image assets |
| joshuaday | 69.8 MB | 2,332 KB | Twin's docs + reference material |
| main | 25.2 MB | 94 KB | Atlas workspace |
| studio | 19.6 MB | 21 KB | Trimmed today (was 36MB) |
| doctorclaw | 2.8 MB | 143 KB | Includes backups from today |
| All others (18) | < 0.5 MB each | — | Clean |

**UXplorer's workspace is 381MB** — likely contains build output, node_modules, or large assets. This doesn't affect bootstrap performance (bootstrapMaxChars is per-file), but it impacts disk I/O during memory_search if sessions are indexed from that workspace.

---

## 4. Embedding Database

### 4.1 Current State

```
Database size:  711 MB
Chunks:         5,462 (325 memory + 5,137 session)
Session data:   94% of chunks, 94% of embedding storage
Status:         sessionMemory disable may not be applied yet
```

The embedding DB was analyzed in detail in [001-session-memory-performance.md]. Key point: **even with sessionMemory disabled, the existing 5,137 session chunks remain in the DB** until a rebuild or prune. Every `memory_search` still scans all 5,462 vectors until those orphaned session chunks are removed.

### 4.2 Recommended Actions

1. **Rebuild the DB** — delete `main.sqlite` and let it rebuild from memory files only (~50MB)
2. **Or** run a manual prune — `DELETE FROM chunks WHERE source='sessions'` + `DELETE FROM files WHERE source='sessions'` + VACUUM
3. Monitor that new sessions are NOT being indexed (confirm sessionMemory is off)

---

## 5. Cron Job Efficiency

### 5.1 Active Cron Jobs

| Job | Agent | Interval | Model | Purpose |
|-----|-------|----------|-------|---------|
| twin-watchdog | cronclaw | 10m | Haiku ✅ | Checks Twin liveness |
| mason-task-watchdog | cronclaw | 30m | Haiku ✅ | Temporary — checks Mason task |
| secureclaw-watchdog | cronclaw | 4h | Haiku ✅ | Checks SecureClaw liveness |
| cronclaw-watchdog | cronclaw | 4h | Haiku ✅ | Meta-watchdog (self-check) |
| fleet-session-reset | cronclaw | daily 4am | Haiku ✅ | Resets heartbeat sessions |
| printer-watchdog | cronclaw | 15m (market hours) | Haiku ✅ | Checks Printer during market hours |
| sessionMemory-checkin | main | one-shot | — | Reminder (auto-deletes) |

**Good news:** The active CronClaw-managed watchdogs are all correctly on Haiku. This was set up properly today.

### 5.2 Stale Cron Sessions (Atlas legacy)

These are orphaned cron sessions from before CronClaw existed. They ran under the `main` agent on Opus:

| Session | Tokens | Status |
|---------|--------|--------|
| Atlas Mobile Status Check | 155,714 | Stale |
| agent-monitor | 134,619 | Stale |
| Atlas Thinking Enforcer | 73,544 | Stale |
| uxplorer-09-integration | 120,704 | Stale |
| uxplorer-08-entity-system | 55,579 | Stale |
| uxplorer-07-context-panel | 52,845 | Stale |
| uxplorer-06-topbar | 54,357 | Stale |
| uxplorer-05-navrail | 37,384 | Stale |
| uxplorer-04-animations | 31,315 | Stale |
| uxplorer-03-design-tokens | 29,191 | Stale |
| uxplorer-10-morning-report | 39,099 | Stale |
| Forge Monitor (5min) | 24,905 | Stale |
| Forge Brand Iterator | 68,383 | Stale |
| Sensei Module Writer | 56,660 | Stale |
| printer-watchdog (main) | 44,802 | Stale |
| twin-watchdog (main) | 28,940 | Stale |
| RTNU Monitor | 60,401 | Stale |
| LUNA Monitor | 39,822 | Stale |
| watchpost-weekly-rtnu | 25,468 | Stale |
| watchpost-weekly-luna | 25,244 | Stale |
| Tweet Vault Resurface | 18,594 | Stale |
| Atlas Agent Monitor | 23,079 | Stale |

**22 stale cron sessions consuming ~1.2M tokens in the session store.** These cron jobs were either disabled or replaced by CronClaw but their sessions were never cleaned up. ConfigClaw reportedly cleaned 132 stale cron sessions earlier today, but 22 remain.

### 5.3 Temporary Watchdog Status

- **mason-task-watchdog:** Should be disabled when Mason's BasedRank task completes
- **twin-watchdog:** Running every 10 minutes. Could be reduced to 30m once Twin is stable.

---

## 6. Bootstrap Sizes

Per-file truncation at 20K default. No agent is at risk of per-file truncation, but total bootstrap size affects every API call.

| Agent | Bootstrap (est.) | % of 20K | Notes |
|-------|-----------------|----------|-------|
| sparkcopy | 19,152 | 95.7% 🟡 | Heavy SOUL.md (13.2K) but justified for creative task |
| joshuaday | ~18,000 | ~90% | Twin's large SOUL.md with voice rules |
| studio | ~17,000 | ~85% | Art generation + brand context |
| pixel | ~15,000 | ~75% | Landing page context |
| print | ~14,000 | ~70% | Trading rules + market knowledge |
| spawnclaw | ~16,800 | ~84% | Agent creation pipeline + KB on-demand |
| doctorclaw | ~12,000 | ~60% | Diagnostic protocols |
| Most others | < 10,000 | < 50% | ✅ Healthy |

No immediate bootstrap concerns. The heaviest agents are creative/specialized tasks where context weight pays for itself.

---

## 7. Recommendations (Priority Order)

### 🔴 P0: Fix `contextTokens` NOW

```
agents.defaults.contextTokens: 500000 → 200000
```

This is the root cause of afternoon slowdown. It was identified 90 minutes ago. It has not been applied. Every minute it remains at 500K, sessions grow unbounded toward the API wall.

**Impact:** Compaction triggers at ~160K instead of never. Sessions stay fast all day.

### 🔴 P1: Purge Zombie Sessions

Delete or archive:
- 7 openresponses sessions over 200K tokens (already past API limit — will crash if reused)
- All ephemeral sessions older than 24 hours
- 22 stale cron sessions from the `main` agent

**Impact:** Frees ~1.9M tokens from the session store. Faster session lookup.

### 🔴 P2: Switch Heartbeat Models to Sonnet

| Agent | From | To | Saves |
|-------|------|-----|-------|
| print | Opus 5m | **Sonnet 5m** | ~288 Opus calls/day → 288 Sonnet calls (~10x cheaper) |
| joshuaday | Opus 30m | **Sonnet 30m** | ~48 Opus calls/day → Sonnet |
| oc | Opus 4h | **Sonnet 4h** | ~6 Opus calls/day → Sonnet |
| cronclaw | Opus 4h | **Sonnet 4h** | ~6 Opus calls/day → Sonnet |

**Impact:** Single biggest cost reduction. Heartbeat checks are scan-and-report operations that produce identical results on Sonnet vs Opus.

### 🟡 P3: Rebuild Embedding Database

```bash
rm ~/.openclaw/memory/main.sqlite
# Rebuilds from memory files only on next memory_search
# New size: ~50MB instead of 711MB
```

Or surgical prune:
```sql
DELETE FROM chunks WHERE source='sessions';
DELETE FROM files WHERE source='sessions';
DELETE FROM embedding_cache; -- rebuild on demand
VACUUM;
```

**Impact:** 711MB → ~50MB. memory_search scans 325 vectors instead of 5,462 (16x faster).

### 🟡 P4: Transcript Cleanup Policy

The `main` agent has 298 session files (211MB) with no cleanup. Implement:
- **Archive sessions older than 7 days** to a compressed backup
- **Delete sessions older than 30 days** (important content should be in MEMORY.md by then)
- **Set sessionTTL if available** — auto-expire inactive sessions

### 🟡 P5: Downgrade Task Agents That Don't Need Opus

| Agent | Current | Recommended | Why |
|-------|---------|-------------|-----|
| mason | Opus | Sonnet | Coding task agent |
| sarah | Opus | Sonnet | Communications — no complex reasoning |
| configclaw | Opus | Sonnet | Config management is deterministic |
| exodus | Opus | Sonnet | Data migration tool |
| sensei | Opus | Sonnet | Tutorial writing |

Keep Opus for: main (orchestration), joshuaday tasks (creative voice), studio (art), pixel (design), spawnclaw (agent design), doctorclaw (diagnosis), cro (analysis), sparkcopy (creative writing), print tasks (trading decisions).

### 🟢 P6: UXplorer Workspace Cleanup

The `uxplorer` workspace is **381MB** — almost certainly contains build output or node_modules. This doesn't affect bootstrap but wastes disk and slows backup operations.

### 🟢 P7: Session Reset for Bloated Active Sessions

These active sessions should be reset (they'll rebuild context from compaction summary):

| Session | Tokens | Why |
|---------|--------|-----|
| studio:main | 184,269 | 92% of API limit |
| joshuaday:main | 159,412 | 80% of API limit |
| sensei:main | 142,245 | Hasn't been active in 2 days |
| pixel:main | 134,526 | 67% of API limit |
| cro:main | 117,471 | Hasn't been active since Feb 12 |

---

## 8. Cost Impact Estimate

Assuming Opus at ~$15/MTok input + $75/MTok output, Sonnet at ~$3/MTok + $15/MTok:

| Fix | Estimated Daily Savings |
|-----|------------------------|
| Heartbeat → Sonnet (P2) | **~80% reduction** on 348 heartbeat calls/day |
| Kill zombie sessions (P1) | One-time: frees 1.9M tokens of memory |
| Fix contextTokens (P0) | Prevents emergency compaction cycles (each costs a full context replay) |
| Task agent downgrades (P5) | **~50% reduction** on 5 agents |
| DB rebuild (P3) | Faster memory_search = faster every API call |

**Conservative estimate: P0 + P2 alone would reduce daily Opus token consumption by 40-60%.**

---

## 9. Fleet Health Scorecard

| Metric | Status | Value |
|--------|--------|-------|
| contextTokens config | 🔴 Critical | 500K (should be 200K) |
| Zombie sessions | 🔴 Critical | 42% of token store |
| Heartbeat models | 🔴 Critical | All 4 on Opus |
| Embedding DB size | 🟡 Warning | 711MB (should be ~50MB) |
| Transcript accumulation | 🟡 Warning | 307MB, no cleanup policy |
| Model assignments | 🟡 Warning | 19/22 on Opus |
| Cron session cleanup | 🟡 Warning | 22 stale sessions remaining |
| UXplorer workspace | 🟡 Warning | 381MB |
| Bootstrap sizes | ✅ Healthy | All within limits |
| Active cron config | ✅ Healthy | CronClaw watchdogs on Haiku |
| Agent file placement | ✅ Healthy | Fixed earlier today |
| Agent identities | ✅ Healthy | All correct |

**Overall fleet health: 🟡 Degraded — fixable with 3 config changes (P0, P1, P2).**

---

*Report generated by DoctorClaw 🩺 — 2026-02-14 16:33 PST*
