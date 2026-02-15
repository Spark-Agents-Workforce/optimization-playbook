# 004 — Security × Performance Audit: Max Intelligence at Max Speed

**Agent:** SecureClaw 🛡️  
**Date:** February 14, 2026  
**Fleet:** 22 agents, 155 sessions, macOS 26.2 (arm64), 36 GB RAM  
**OpenClaw:** 2026.2.12 (2026.2.13 available)  
**Directive:** Strip away overhead, not power. Quality is non-negotiable.

---

## Executive Summary

This audit focuses on the intersection of security and performance through one lens: **what's making the best models run slower than they should?** Every finding is either overhead that can be stripped without reducing capability, or a security tightening that also happens to speed things up.

The single biggest performance bottleneck right now is **memory pressure** — 35 GB of 36 GB RAM used, 698 MB free. The machine is running hot. Everything that touches disk or memory under these conditions is slower than it needs to be.

---

## 🔴 Critical: Speed Killers

### 1. Memory Pressure Is Severe — 698 MB Free of 36 GB

**Measured:** `PhysMem: 35G used (4119M wired, 7463M compressor), 698M unused`

The system is actively using 7.4 GB in compressor (macOS's in-memory swap). This means memory pages are being compressed/decompressed on every access pattern change. Every context switch, every API response landing, every file read is competing for physical memory.

**What's consuming it:**

| Process | RSS | Notes |
|---|---|---|
| Cloudflare workerd (3 procs) | 1,134 MB | agent-royale API |
| openclaw-gateway | 486 MB | 22 agents, 155 sessions, memory search index |
| Vite dev server (UXplorer) | 69 MB | Running since Thursday, UXplorer agent is disabled |
| esbuild (UXplorer) | 27 MB | Same — orphaned dev tooling |
| Next.js dev server | 79 MB | landing-page on port 3456 |
| Total Node.js footprint | **1,829 MB** | Just the agent/dev processes |

**Speed impact:** When the gateway needs to assemble a 500k-token context for an Opus call, it's fighting for memory against compressed pages. Response parsing, JSON serialization of large tool outputs, memory search index lookups — all slower under pressure.

**Security relevance:** A memory-starved gateway is more likely to crash or exhibit degraded behavior. The 7.4 GB compressor usage means sensitive data (session contents, API responses) is being compressed/decompressed in memory, increasing the window where it's in an intermediate state.

**Recommendations (no capability loss):**
1. Kill the orphaned UXplorer dev server: `kill 33354 33345 33360` — frees ~165 MB. Agent is disabled, server is doing nothing.
2. Evaluate whether all 3 workerd processes (1.1 GB) need to run concurrently
3. If the Next.js landing page dev server isn't actively being developed, stop it — 79 MB back

**Impact:** ⚡ Immediately faster gateway response assembly. 🔒 More stable gateway under load.

---

### 2. 711 MB SQLite Database Still on Disk After sessionMemory Disabled

**Finding:** `~/.openclaw/memory/main.sqlite` is 711 MB. sessionMemory was disabled today to fix afternoon performance degradation — but the database file is still there.

**Speed impact:** Even with sessionMemory disabled, this 711 MB file exists on disk. If any code path touches it (stat, open, check), it's a heavy I/O operation. More importantly, it's 711 MB that could be freed from the filesystem cache, giving the OS more room to cache files that actually matter (session transcripts, workspace files, the gateway binary itself).

**Additional stale databases (62 MB total):**
- `genghisclawn.sqlite` — 22 MB (deleted agent)
- `god.sqlite` — 19 MB (deleted agent)
- `tryclaw.sqlite` — 6.5 MB (deleted agent)
- `watchpost.sqlite` — 68 KB (deleted agent)

**Recommendation:**
```bash
# Archive then remove — recoverable if needed
mkdir -p ~/.openclaw/memory/archive
mv ~/.openclaw/memory/main.sqlite ~/.openclaw/memory/archive/
mv ~/.openclaw/memory/{genghisclawn,god,tryclaw,watchpost}.sqlite ~/.openclaw/memory/archive/
```

**Impact:** ⚡ 773 MB freed from disk, better filesystem cache utilization. 🔒 Removes stale conversation embeddings.

---

### 3. Prompt Cache Optimization — Stable System Prompts = Faster TTFT

**How Anthropic caching works:** The system prompt + injected workspace files are sent with every API call. If this prefix is identical across calls, Anthropic serves it from cache at 10x cheaper AND significantly faster (no re-processing). With `cacheRetention: "long"`, cached prefixes persist for extended periods.

**What breaks cache hits:**
- Any change to SOUL.md, AGENTS.md, TOOLS.md, USER.md, IDENTITY.md, MEMORY.md, or HEARTBEAT.md invalidates the cache for that agent
- MEMORY.md changes on every compaction/memory flush — this is expected and unavoidable
- But workspace file churn from other sources breaks the cache unnecessarily

**Current system prompt sizes (bytes injected per call):**

| Agent | Total Workspace Files | Notes |
|---|---|---|
| Twin (joshuaday) | 34,839 bytes | 14 KB HEARTBEAT.md alone |
| Spark Copy | 19,152 bytes | 13 KB SOUL.md |
| SecureClaw (oc) | 24,696 bytes | Heavy TOOLS.md + AGENTS.md |
| Printer (print) | 25,745 bytes | 7 KB each SOUL + HEARTBEAT |
| Mason | 21,438 bytes | |
| Main (Atlas) | 18,212 bytes | |

**Every byte in the system prompt is sent on every API call.** Larger prompts = slower time-to-first-token (TTFT), even with caching — because cache lookup time scales with prefix size.

**Recommendations:**
1. **Audit MEMORY.md volatility** — if memory flushes happen frequently, consider increasing `compaction.memoryFlush.softThresholdTokens` from 4000 to 8000. Fewer flushes = fewer cache invalidations = more cache hits.
2. **Twin's HEARTBEAT.md is 14 KB (262 lines)** — this is a detailed X engagement playbook. It's powerful, but every byte rides along on every API call, even non-heartbeat turns. Consider whether some of the strategy notes could live in MEMORY.md (which is expected to change) rather than HEARTBEAT.md (which breaks the prompt cache when it changes).
3. **Spark Copy's SOUL.md is 13 KB** — the largest SOUL.md in the fleet. If it's stable (doesn't change often), this is fine — it'll stay cached. If it's being iterated on, each edit breaks the cache.

**Impact:** ⚡ More prompt cache hits = faster TTFT + lower cost. No capability reduction — same prompts, just more stable caching.

---

## 🟡 Warning: Overhead to Strip

### 4. No Sandbox = No Resource Isolation Between Agents

**Config:** `agents.defaults.sandbox` is empty/unconfigured.

**Speed impact (the part people miss):** Without Docker sandboxing, all 22 agents share the same OS resources with zero isolation. A runaway `exec` from one agent (e.g., an agent that accidentally runs `find /` or downloads a large file) consumes CPU/memory/disk that starves every other agent. Sandboxing provides cgroup-based resource limits that prevent one agent from degrading the fleet.

**Security impact:** Every agent has full host access. A compromised or hallucinating agent can read/write any file, execute any command, access all network resources. Skills run unsandboxed with agent-level privileges.

**Recommendation:**
```json
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "non-main"
    }
  }
}
```
Sandboxes all agents except Atlas. Agents that need host access (Twin for `xapi`, Printer for `alpaca`) would need explicit elevated permissions — which is actually better because it documents exactly who needs what.

**Impact:** ⚡ Resource isolation prevents agent resource starvation. 🔒 Major security improvement.

---

### 5. 3.1 GB of Reclaimable Disk — Faster I/O When the SSD Has Room

**Breakdown:**

| Component | Size | Status |
|---|---|---|
| Browser profile | 1.8 GB | Chromium data — cookies, cache, ML models |
| Quarantine | 630 MB | Dead agent data incl. 185 MB git pack |
| UXplorer node_modules | 493 MB | `workspace-uxplorer/orchestrator-v3/` — agent is disabled |
| Session transcripts | 271 MB (main alone) | 298 files, largest 84 MB |
| **Total reclaimable** | **~3.1 GB** | |

**Speed impact:** SSDs perform better with free space (wear leveling, garbage collection). More importantly, the filesystem cache competes with these files. Every byte the OS caches from stale quarantine data or unused browser profiles is a byte it can't use to cache active session transcripts or workspace files.

**Security impact:** Old browser profiles contain cookies and cached authenticated sessions. Quarantine may contain old API keys or conversation histories. 298 session transcripts in main contain full conversation histories.

**Quick wins (no capability impact):**
```bash
# 1. Clean quarantine (630 MB)
rm -rf ~/.openclaw/quarantine/

# 2. Clean UXplorer node_modules (493 MB) — agent is disabled
rm -rf ~/.openclaw/workspace-uxplorer/orchestrator-v3/node_modules/

# 3. Archive old session transcripts
# (implement a retention policy — keep last 7 days, archive rest)
```

**Impact:** ⚡ Better filesystem cache utilization. 🔒 Removes stale sensitive data.

---

### 6. API Keys Inline in Config — Security Risk Without Speed Benefit

**Finding:** `env.vars` in `openclaw.json` contains 4 plaintext API keys (OPENAI_API_KEY, OPENROUTER_API_KEY, MINIMAX_API_KEY, X_BEARER_TOKEN). Additionally, `models.providers.minimax.apiKey` duplicates the MiniMax key.

**Speed impact:** Minimal directly, but the duplicate MiniMax key means two resolution paths for the same credential. The config is also larger than necessary, and every `config.get` API call transfers these secrets.

**Security impact:** Every config read, debug dump, or backup exposes all secrets. The `gateway config.get` API returns plaintext keys in the `raw` and `config` response fields to any authenticated caller. Secrets should live in environment variables, not config files.

**Recommendation:** Move secrets to LaunchAgent environment variables or a `.env` file. Remove the duplicate `minimax.apiKey` from provider config.

**Impact:** 🔒 Eliminates secret exposure via config reads. ⚡ Cleaner config resolution.

---

### 7. Workspace Directories at 755 — Tighten Without Cost

**Finding:** All 24 workspace directories are `755` (world-readable). Should be `700`.

```bash
chmod 700 ~/.openclaw/workspace-*
```

**Impact:** 🔒 Defense-in-depth. ⚡ Zero performance cost.

---

### 8. Logs Growing Without Rotation — 70 MB and Climbing

**Finding:** 51 MB in `/tmp/openclaw/` (2 days), 19 MB in `~/.openclaw/logs/`. At ~25 MB/day, this grows to 750 MB/month without rotation.

**Speed impact:** Large log files mean slower log writes (append to large file), and if any process reads logs (tail, grep for debugging), it gets slower over time.

**Recommendation:** Rotate logs weekly, keep 7 days, compress older:
```bash
find /tmp/openclaw -name '*.log' -mtime +7 -delete
```

**Impact:** ⚡ Faster log I/O. 🔒 Limits sensitive data persistence in logs.

---

## 🟢 Already Optimized — No Changes Needed

### 9. Prompt Cache Retention ✅
`cacheRetention: "long"` on Opus and OR-Opus. This is correct — maximizes Anthropic prompt cache hits, reducing both cost (10x cheaper reads) and latency (cached prefixes are faster to process).

### 10. Context Pruning ✅
`cache-ttl: 30m` with `keepLastAssistants: 3`. Good balance — prevents context bloat while keeping recent conversation. Aggressive enough to keep calls fast.

### 11. Compaction ✅
`safeguard` mode with 80k reserve floor and memory flush enabled. This prevents context overflow without losing information. The memory flush writes to disk before compacting, preserving knowledge.

### 12. Network Exposure ✅
Gateway bound to loopback. No unnecessary ports. Tailscale Serve for remote access. No Node.js processes listening on non-local interfaces. Clean.

### 13. Credential File Permissions ✅
All auth-profiles.json at 600. All credential files at 600. Credentials directory at 700. `logging.redactSensitive: "tools"` enabled.

### 14. Channel Security ✅
iMessage: `dmPolicy: "pairing"`, `groupPolicy: "allowlist"`. Slack: `groupPolicy: "allowlist"`. No open attack surface.

---

## The Speed Stack: What Makes Opus Respond Faster

Understanding the latency chain helps prioritize:

```
1. Gateway assembles context     ← Memory pressure slows this (finding #1)
2. System prompt sent to API     ← Larger prompt = slower; cache miss = much slower (finding #3)
3. Anthropic processes request   ← We can't control this, but cache hits skip re-processing
4. Response streams back         ← blockStreamingDefault: on holds until complete
5. Gateway processes response    ← Memory pressure slows this (finding #1)
6. Tool calls execute            ← No sandbox = no resource isolation (finding #4)
7. Loop back to step 1           ← Disk I/O for transcript writes (finding #5)
```

**The three highest-leverage changes:**
1. **Free memory** — Kill orphaned processes, archive stale databases. The gateway is fighting for every megabyte.
2. **Maximize prompt cache hits** — Keep workspace files stable. Every cache hit is a faster TTFT.
3. **Enable sandboxing** — Not for restriction, but for resource isolation. One bad `exec` shouldn't slow the whole fleet.

---

## Priority Action Plan (Speed-First)

| # | Action | Speed Impact | Security Impact | Effort |
|---|---|---|---|---|
| 1 | Kill orphaned UXplorer dev server + esbuild | ⚡⚡⚡ ~165 MB RAM freed | — | 1 min |
| 2 | Archive main.sqlite + stale DBs | ⚡⚡ 773 MB disk freed | 🔒 Stale data removed | 2 min |
| 3 | Clean quarantine (630 MB) | ⚡⚡ Disk + cache freed | 🔒 Old secrets removed | 1 min |
| 4 | Clean UXplorer node_modules (493 MB) | ⚡⚡ Disk freed | — | 1 min |
| 5 | Enable sandbox `non-main` | ⚡⚡ Resource isolation | 🔒🔒🔒 Major | Medium |
| 6 | Increase memoryFlush threshold (4k→8k) | ⚡ Fewer cache invalidations | — | Low |
| 7 | chmod 700 workspaces | — | 🔒 Defense-in-depth | 1 min |
| 8 | Move API keys to env vars | ⚡ Cleaner config | 🔒🔒 Secret hygiene | Medium |
| 9 | Set up log rotation | ⚡ Long-term I/O | 🔒 Data retention | Low |

**Items 1-4 can be done in 5 minutes and free ~165 MB RAM + 1.9 GB disk.** That's the biggest immediate speed win.

---

## Key Insight

**The fleet's power is maxed out — Opus everywhere, high thinking, 500k context, long cache retention. Good. The problem isn't the engine, it's the road.** Memory pressure, stale data competing for cache, orphaned processes eating RAM — these are the speed killers. Strip the overhead, keep the power.

---

*Generated by SecureClaw 🛡️ — OpenClaw Security & Performance Agent*  
*Fleet: Spark Agents Workforce*
