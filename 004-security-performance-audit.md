# 004 — Security × Performance Audit

**Agent:** SecureClaw 🛡️  
**Date:** February 14, 2026  
**Fleet:** 22 agents, 155 sessions, macOS 26.2 (arm64), 36 GB RAM  
**OpenClaw:** 2026.2.12 (2026.2.13 available)

---

## Executive Summary

Security and performance are often treated as competing concerns. In practice, most security weaknesses in this fleet **also** waste resources — over-permissioned agents carry unnecessary context, unsandboxed execution means no resource isolation, and accumulated session data creates both a larger attack surface and slower I/O.

This audit found **14 findings** at the intersection of security and performance, organized by impact.

---

## 🔴 Critical: Tighten These First

### 1. No Sandbox Configured — Security Gap That Also Prevents Resource Isolation

**Config:** `agents.defaults.sandbox` is empty/unconfigured.

**Security impact:** Every agent runs with full host access. A compromised or hallucinating agent can read/write any file, execute any command, and access all network resources. Skills (which run unsandboxed with agent-level privileges) inherit this exposure.

**Performance impact:** Without sandboxing, there's no resource isolation between agents. A runaway `exec` in one agent can consume CPU/memory that starves others. Docker sandboxing provides cgroup-based resource limits.

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
This sandboxes all agents except `main` (Atlas). It's the minimum viable security posture for a 22-agent fleet. Reduces blast radius AND provides resource isolation.

**Estimated impact:** 🔒 Major security improvement. ⚡ Prevents resource starvation from runaway agents.

---

### 2. API Keys Stored Inline in Config — Unnecessary Exposure + Config Bloat

**Finding:** `env.vars` in `openclaw.json` contains 4 plaintext API keys:
- `OPENAI_API_KEY` (185 chars)
- `OPENROUTER_API_KEY` (70 chars)
- `MINIMAX_API_KEY` (93 chars)
- `X_BEARER_TOKEN` (108 chars)

Additionally, `models.providers.minimax.apiKey` duplicates the MiniMax key inline in the provider config (in addition to the `env.vars` entry).

**Security impact:** Every config read, config.get API call, backup, or debug dump exposes all secrets. The `gateway config.get` API returns the full `raw` config including plaintext keys to any authenticated caller. The minimax key is stored in two places.

**Performance impact:** Config is loaded into memory and passed around internally. Larger configs mean more memory per config parse. More importantly: the duplicated minimax key means two code paths resolve the same credential, adding unnecessary complexity.

**Recommendation:**
1. Move all API keys to environment variables via LaunchAgent plist or a `.env` file
2. Remove the duplicate `models.providers.minimax.apiKey` — it should resolve from `MINIMAX_API_KEY` env var automatically
3. Reference keys by env var name in config rather than embedding values

**Estimated impact:** 🔒 Eliminates secret exposure via config reads. ⚡ Marginal config size reduction.

---

## 🟡 Warning: Significant Savings Available

### 3. All Heartbeats Run on Opus ($5/$25 per Mtok) — Massive Cost Waste

**Config:** Default heartbeat model is `anthropic/claude-opus-4-6`. All 4 active heartbeat agents use it.

**The math:**

| Agent | Interval | Daily Beats | Est. Tokens/Beat | Daily Token Cost |
|---|---|---|---|---|
| Twin (joshuaday) | 30m | 48 | ~5k (input+output) | ~240k tokens |
| Printer (print) | 5m | 138 (weekdays) | ~5k | ~690k tokens |
| SecureClaw (oc) | 4h | 4 | ~20k (deep scan) | ~80k tokens |
| CronClaw (cronclaw) | 4h | 4 | ~10k | ~40k tokens |

**Total:** ~1M+ tokens/day on heartbeats alone. At Opus rates ($5 input / $25 output per Mtok), that's roughly **$15-25/day just on heartbeats**.

**Security relevance:** Watchdog heartbeats (twin-watchdog, printer-watchdog) are simple health checks. They don't need Opus-level reasoning. Haiku ($0.80/$4 per Mtok) handles watchdog logic fine — CronClaw's cron jobs already use Haiku for exactly this reason.

**Recommendation:**
- Set `agents.defaults.heartbeat.model` to `anthropic/claude-haiku-4-5`
- Override to Opus only for agents that need deep reasoning in heartbeats (SecureClaw's security scans arguably benefit from Opus)
- Twin and Printer heartbeats should absolutely be Haiku

**Estimated impact:** 🔒 No security regression. ⚡ **~80-90% cost reduction on heartbeats** (~$12-22/day saved).

---

### 4. 500k Default Context Window — Most Agents Don't Need It

**Config:** `agents.defaults.contextTokens: 500000`

**Finding:** All 22 agents inherit a 500k context window. Most agents (Pixel, Studio, Sensei, Exodus, CRO, Forge, Sarah, etc.) are task-specific and rarely exceed 50-100k tokens per session.

**Security impact:** Larger context windows mean more data in memory per session. If a session is compromised (prompt injection), the attacker has access to a larger conversation history.

**Performance impact:** Larger contexts = slower API calls (more tokens to process), higher memory usage, more expensive cache writes ($6.25/Mtok for Opus cache writes), and longer compaction cycles.

**Recommendation:**
- Set `agents.defaults.contextTokens` to `200000` (200k)
- Override to 500k only for agents that genuinely need it (Atlas, Twin, Mason)
- Task-specific agents (Pixel, Studio, Forge) could work with 100k

**Estimated impact:** 🔒 Reduces data exposure per session. ⚡ Faster API response times, lower memory pressure, reduced cache costs.

---

### 5. Session Transcripts Accumulating Without Cleanup — 706 MB

**Finding:**

| Store | Transcripts | Size |
|---|---|---|
| main (Atlas) | 298 | 271 MB |
| uxplorer | 3 + archive | 125 MB |
| studio | 2 | 70 MB |
| pixel | 6 | 68 MB |
| joshuaday | 6 | 28 MB |
| **Total** | **382+** | **~600 MB** |

The largest single transcript is **84 MB** (`main/ba09ed85...`).

**Security impact:** Old transcripts contain full conversation histories including tool outputs, file contents, and potentially sensitive data. They persist indefinitely on disk with world-readable permissions (see finding #7).

**Performance impact:** 
- Disk I/O pressure from large files
- If sessionMemory is re-enabled, these would all need indexing (recall: the 711 MB SQLite DB that caused the afternoon performance degradation was built from indexing these transcripts)
- Compaction must read/write these files

**Recommendation:**
1. Archive transcripts older than 7 days to compressed storage
2. Delete transcripts older than 30 days (or move to cold storage)
3. The 84 MB Atlas transcript should be investigated — that's abnormally large

**Estimated impact:** 🔒 Reduces sensitive data persistence. ⚡ Frees ~500 MB disk, prevents future sessionMemory DB bloat.

---

### 6. Memory SQLite Databases — 983 MB Total, Growing

**Finding:**

| Database | Size | Agent Status |
|---|---|---|
| main.sqlite | 711 MB | Active |
| uxplorer.sqlite | 63 MB | Disabled |
| joshuaday.sqlite | 34 MB | Active |
| genghisclawn.sqlite | 22 MB | **Deleted agent** |
| doctorclaw.sqlite | 21 MB | Active |
| studio.sqlite | 21 MB | Disabled |
| god.sqlite | 19 MB | **Deleted agent** |
| milo.sqlite | 15 MB | Active (Codex) |
| royale.sqlite | 14 MB | Disabled |
| architect.sqlite | 13 MB | **Deleted agent** |

**Security impact:** Databases for deleted/retired agents (genghisclawn, god, architect, tryclaw, watchpost) persist with full embeddings of historical conversations. This is unnecessary data retention.

**Performance impact:** `sessionMemory` was disabled specifically because `main.sqlite` at 711 MB caused performance degradation. The other databases (132 MB for deleted agents alone) consume disk for no benefit.

**Recommendation:**
1. Delete SQLite databases for retired agents: `genghisclawn.sqlite`, `god.sqlite`, `tryclaw.sqlite`, `watchpost.sqlite` (~62 MB)
2. Consider `VACUUM` on `main.sqlite` if sessionMemory is re-enabled
3. For disabled agents with no planned re-use, archive their databases

**Estimated impact:** 🔒 Removes stale data. ⚡ Frees 60+ MB immediately, prevents confusion.

---

### 7. Workspace Directories at 755 — Should Be 700

**Finding:** All 24 workspace directories have permissions `755` (world-readable):
```
workspace-main: 755
workspace-joshuaday: 755
workspace-oc: 755
... (all 24)
```

Meanwhile, the parent `~/.openclaw/` is correctly `700`, and `openclaw.json` is correctly `600`.

**Security impact:** Any local user/process can read workspace contents. Workspaces contain SOUL.md, AGENTS.md, TOOLS.md, memory files, and potentially sensitive operational data. The parent directory's 700 permission mitigates this (other users can't traverse to the workspace), but defense-in-depth says the workspaces themselves should be restrictive.

**Performance impact:** None directly, but a compromised workspace file (e.g., via a writable TOOLS.md) could inject instructions that cause agents to perform expensive operations.

**Recommendation:**
```bash
chmod 700 ~/.openclaw/workspace-*
```

**Estimated impact:** 🔒 Defense-in-depth improvement. ⚡ No performance impact.

---

### 8. Browser Data — 1.8 GB

**Finding:** `~/.openclaw/browser/` consumes 1.8 GB. This is the Chromium profile used for browser automation.

**Security impact:** Browser profile contains cookies, local storage, cached pages, and potentially authenticated sessions from browser automation tasks.

**Performance impact:** 1.8 GB of disk for a feature that most agents don't use regularly. The Chromium model store alone has ML models (35 MB tflite files).

**Recommendation:**
1. Audit whether browser automation is actively needed
2. If used rarely, clear the browser profile periodically
3. Consider disabling `browser control: enabled` in security settings if not actively used

**Estimated impact:** 🔒 Reduces cookie/session exposure. ⚡ Frees up to 1.8 GB disk.

---

### 9. Quarantine Directory — 630 MB of Deleted Agent Data

**Finding:** `~/.openclaw/quarantine/` contains 630 MB, including data from retired agents (e.g., `ledger`). This includes a full copy of the OpenClaw source code in `.git/objects/pack/` (185 MB pack file).

**Security impact:** Quarantined data may contain old API keys, conversation histories, or agent configurations that should have been purged.

**Performance impact:** 630 MB of dead weight on disk.

**Recommendation:**
1. Audit quarantine contents for sensitive data
2. If nothing needs recovery, delete: `rm -rf ~/.openclaw/quarantine/`
3. Or compress for archival: `tar czf quarantine-backup.tar.gz ~/.openclaw/quarantine/ && rm -rf ~/.openclaw/quarantine/`

**Estimated impact:** 🔒 Removes stale sensitive data. ⚡ Frees 630 MB.

---

### 10. Subagent Thinking Level Set to "high" Globally

**Config:** `agents.defaults.subagents.thinking: "high"`

**Finding:** Every subagent spawn uses `thinking: "high"` by default. Thinking mode significantly increases token usage (and cost) per turn. Many subagent tasks are simple lookups or delegated writes that don't benefit from extended reasoning.

**Security impact:** Higher thinking levels produce more internal reasoning text, which could theoretically leak more context in error scenarios.

**Performance impact:** `thinking: "high"` can 2-5x the token count per response. With `subagents.maxConcurrent: 8`, that's up to 8 simultaneous high-thinking sessions competing for API rate limits.

**Recommendation:**
- Set `agents.defaults.subagents.thinking` to `"low"` or `"medium"`
- Let spawners override to `"high"` when the task requires it

**Estimated impact:** 🔒 Reduces reasoning data exposure. ⚡ Significant token/cost reduction for subagent tasks.

---

### 11. Logs Growing Unbounded — 51 MB in /tmp, 19 MB in .openclaw/logs

**Finding:**
- `/tmp/openclaw/`: 51 MB (2 days of logs: 27 MB for Feb 13, 22 MB for Feb 14 so far)
- `~/.openclaw/logs/`: 19 MB
- No log rotation policy detected

**Security impact:** Logs may contain partial request/response data, tool outputs, and error messages with sensitive context. `logging.redactSensitive: "tools"` is configured (good), but logs still grow unbounded.

**Performance impact:** At ~25 MB/day, logs will consume ~750 MB/month. Disk I/O for continuous logging adds overhead.

**Recommendation:**
1. Implement log rotation (keep 7 days, compress older)
2. Set up a cron job: `find /tmp/openclaw -name '*.log' -mtime +7 -delete`
3. The 19 MB in `~/.openclaw/logs/` may be redundant with `/tmp/openclaw/` — investigate and consolidate

**Estimated impact:** 🔒 Limits sensitive data in logs. ⚡ Prevents eventual disk pressure.

---

### 12. Agent-to-Agent Enabled Globally, No Restrictions

**Config:** `tools.agentToAgent.enabled: true` with no `allowFrom` restrictions.

**Finding:** Any agent can send messages to any other agent. Atlas (main) has `subagents.allowAgents` listing 21 agents, meaning it can spawn any agent as a subagent.

**Security impact:** A compromised agent could message other agents to exfiltrate data or trigger actions. There's no agent-to-agent access control.

**Performance impact:** Unrestricted agent-to-agent communication means any agent can trigger expensive operations on any other agent. A cascading spawn could overwhelm the fleet (mitigated somewhat by `maxConcurrent: 4`).

**Recommendation:**
- Restrict `subagents.allowAgents` per agent (most agents don't need to spawn others)
- Currently only `main`, `uxplorer`, and `joshuaday` have `allowAgents` defined — this is good, but the default allows messaging without spawning

**Estimated impact:** 🔒 Limits lateral movement. ⚡ Prevents cascade spawns.

---

## 🟢 Info: Good Practices Already in Place

### 13. Credential File Permissions — All Correct ✅

| Path | Expected | Actual |
|---|---|---|
| `~/.openclaw/` | 700 | 700 ✅ |
| `openclaw.json` | 600 | 600 ✅ |
| `credentials/` | 700 | 700 ✅ |
| All credential files | 600 | 600 ✅ |
| All auth-profiles.json | 600 | 600 ✅ |

No secrets found in workspace files (SOUL.md, TOOLS.md, AGENTS.md, USER.md, memory/).

### 14. Network Exposure — Minimal ✅

- Gateway bound to loopback only
- No Node.js processes listening on non-local interfaces
- Tailscale Serve provides remote access behind Tailscale auth
- No unnecessary ports open

---

## Disk Usage Summary

| Component | Size | Action |
|---|---|---|
| Browser data | 1.8 GB | Audit/clean |
| Memory SQLite | 983 MB | Delete retired agent DBs, VACUUM active |
| Agent sessions | 706 MB | Archive old transcripts |
| Quarantine | 630 MB | Delete or archive |
| Workspaces | 790 MB | Clean UXplorer node_modules (501 MB) |
| Logs | 70 MB | Set up rotation |
| Media | 93 MB | Audit |
| **Total** | **5.5 GB** | **~3 GB reclaimable** |

---

## Priority Action Plan

| # | Action | Security | Performance | Effort |
|---|---|---|---|---|
| 1 | Enable sandbox mode `non-main` | 🔴 High | ⚡ Medium | Medium |
| 2 | Switch heartbeat models to Haiku | — | ⚡ High ($) | Low |
| 3 | Reduce default context to 200k | 🟡 Medium | ⚡ Medium | Low |
| 4 | Move API keys to env vars | 🔴 High | ⚡ Low | Medium |
| 5 | Set workspace dirs to 700 | 🟡 Medium | — | Low |
| 6 | Delete retired agent SQLite DBs | 🟡 Medium | ⚡ Low | Low |
| 7 | Archive old session transcripts | 🟡 Medium | ⚡ Medium | Medium |
| 8 | Clean quarantine (630 MB) | 🟡 Medium | ⚡ Low | Low |
| 9 | Set up log rotation | 🟡 Medium | ⚡ Low | Low |
| 10 | Reduce subagent thinking default | 🟢 Low | ⚡ Medium ($) | Low |
| 11 | Audit/clean browser data (1.8 GB) | 🟡 Medium | ⚡ Low | Low |
| 12 | Remove minimax duplicate key | 🟢 Low | ⚡ Low | Low |

---

## Key Insight

**The single highest-impact change is switching heartbeat models from Opus to Haiku.** It requires zero security trade-offs and likely saves $15-25/day. The second highest is enabling sandboxing, which improves both security posture and resource isolation simultaneously.

The fleet has strong fundamentals (loopback binding, token auth, correct file permissions, redacted logging). The gaps are in resource governance — too much access, too much context, too much data retention, and too-expensive models for routine tasks.

---

*Generated by SecureClaw 🛡️ — OpenClaw Security & Performance Agent*  
*Fleet: Spark Agents Workforce*
