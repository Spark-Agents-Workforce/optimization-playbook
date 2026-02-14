# 001: Session Memory & Embedding DB Performance

**Date:** 2026-02-14  
**Fleet size:** 22 agents  
**Severity:** High — primary cause of afternoon performance degradation

## The Problem

OpenClaw blazing fast in the morning, noticeably slower by afternoon. More frequent compactions, longer response times, degraded usability after hours of heavy use.

## Root Cause

Multiple compounding issues, all tied to context and memory accumulation throughout the day.

### 1. Embedding Database Bloat (Primary)

With `memorySearch.experimental.sessionMemory: true`, OpenClaw indexes session transcripts into a SQLite embedding database. In our fleet, this produced a **711MB SQLite file** (`main.sqlite`).

**Breakdown:**

| Component | Size | % of Total |
|-----------|------|-----------|
| embedding_cache | 240 MB | 33.8% |
| chunks.embedding (sessions) | 191 MB | 26.9% |
| chunks_vec (vector index) | ~67 MB | 9.4% |
| chunks_fts + overhead | ~171 MB | 24.0% |
| chunks.embedding (memory files) | 12 MB | 1.7% |
| chunks.text (all) | 10 MB | 1.4% |
| Indexes/journal | ~20 MB | 2.8% |

**Key finding:** 94% of the database (5,137 of 5,462 chunks) came from session transcripts. Only 6% (325 chunks) were from actual memory files — the useful, curated knowledge.

**Why it bloats:**
- Embeddings stored as JSON text (3.2x larger than binary representation)
- Every embedding stored twice (chunks table + embedding_cache)
- Raw tool outputs, JSON blobs, and API responses all get indexed — mostly noise
- A single 88MB session transcript generated 2,419 chunks / 90MB of embeddings

**Why morning is fast:** DB hasn't grown much from today's activity. SQLite page cache is warm.  
**Why afternoon is slow:** Hours of conversation add thousands of new embeddings. Cache pressure increases. Query latency rises.

### 2. Context Window Accumulation

Default config allowed 500K tokens before compaction with only 20K reserve floor:

```json
{
  "contextTokens": 500000,
  "compaction": {
    "mode": "safeguard",
    "reserveTokensFloor": 20000
  }
}
```

By afternoon, each API call sends hundreds of thousands of tokens. Even with prompt caching, this means longer time-to-first-token and heavier compaction operations when they finally trigger.

### 3. Cache TTL Too Short

```json
{
  "contextPruning": {
    "mode": "cache-ttl",
    "ttl": "3m",
    "keepLastAssistants": 3
  }
}
```

A 3-minute TTL means most context in a long session is un-cached and fully reprocessed by the API on every turn. Morning conversations are short enough that most content stays cached. By afternoon, the 3-minute window covers a tiny fraction of the massive context.

### 4. Session Accumulation

132 orphaned cron run sessions were feeding the embedding index. Heartbeat agents firing 24/7 (no active hours configured) added to session file count and embedding pressure.

## The Fixes

### Applied

| Change | Before | After | Impact |
|--------|--------|-------|--------|
| `memorySearch.experimental.sessionMemory` | `true` | `false` | DB queries drop from 5,462 vectors to 325 (16x faster) |
| `memorySearch.sources` | `["memory", "sessions"]` | `["memory"]` | Only curated knowledge searched |
| `contextPruning.ttl` | `3m` | `30m` | More context stays Anthropic-cached between turns |
| `compaction.reserveTokensFloor` | `20000` | `80000` | Compaction triggers earlier, keeps turns snappy |
| Stale cron sessions | 132 orphaned | Cleaned | Stops feeding the embedding index |
| Heartbeat `activeHours` | 24/7 | `02:00-17:00` | Stops firing during sleep hours |

### Considered but not applied

| Change | Rationale |
|--------|-----------|
| Reduce `contextTokens` 500K → 200K | Forces earlier compaction but reduces available context |
| VACUUM main.sqlite | Ran it — no space recovered. DB is dense, not fragmented |
| Delete main.sqlite entirely | Nuclear option — rebuilds from scratch |
| `text-embedding-3-small` instead of `large` | Halves embedding storage (1536 vs 3072 dims) |

## Key Insight

> Session memory is a crutch that papers over agents not updating their memory files.

The memory system (MEMORY.md + daily logs) should be the canonical knowledge store. If something matters, agents should write it down — not rely on searching 88MB transcripts full of tool outputs and JSON blobs. 

Disabling session memory and relying on curated memory files gave us a **16x reduction in search vectors** with no meaningful loss of recall. The important stuff was already in memory files.

## Config Snippet

```json
{
  "agents": {
    "defaults": {
      "contextTokens": 500000,
      "compaction": {
        "mode": "safeguard",
        "reserveTokensFloor": 80000
      },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "30m",
        "keepLastAssistants": 3
      },
      "memorySearch": {
        "sources": ["memory"],
        "experimental": {
          "sessionMemory": false
        }
      }
    }
  }
}
```

## Monitoring

After applying these changes, watch for:
- Faster `memory_search` response times
- Less frequent compactions
- Consistent performance morning through afternoon
- Whether disabling session memory causes any recall gaps (re-enable if needed — it's a config flip, no data is lost)
