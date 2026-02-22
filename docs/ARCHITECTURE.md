# 🦞 Ground Control — Architecture

> How data flows from OpenClaw session logs to your terminal.

This document describes how the Ground Control ecosystem fits together: how cost and usage data is collected, stored, and surfaced to both humans and agents.

---

## Overview

Ground Control follows a simple three-stage pipeline: **ingest → store → act**.

All cost and usage data originates locally. OpenClaw logs every model response — from every provider — to session JSONL files on the VPS. Each assistant message includes full token counts. Ground Control ingests these logs, computes estimated costs from a static pricing table, and writes per-request events to a local SQLite database. No external API polling. No relay infrastructure. No network dependency.

```
  ┌──────────────────────────────────────────────────────────┐
  │                      HOST / VPS                          │
  │                                                          │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │                OPENCLAW GATEWAY                    │  │
  │  │                                                    │  │
  │  │  Providers: Anthropic, OpenAI, Google, etc         │  │
  │  │                                                    │  │
  │  │  Every model response → session JSONL with usage   │  │
  │  │  ~/.openclaw/agents/*/sessions/*.jsonl             │  │
  │  └──────────────────────┬─────────────────────────────┘  │
  │                         │                                │
  │                    JSONL files                           │
  │                    (append-only)                         │
  │                         │                                │
  │  ┌──────────────────────▼─────────────────────────────┐  │
  │  │             INGESTION SERVICE                      │  │
  │  │           (cron or file-watcher)                   │  │
  │  │                                                    │  │
  │  │  1. Tail new JSONL entries since last offset       │  │
  │  │  2. Extract usage events (input, output, cache)    │  │
  │  │  3. Map agent → provider → model                   │  │
  │  │  4. Multiply tokens × static pricing table         │  │
  │  │  5. Write per-request events to SQLite             │  │
  │  │  6. Check budget alerts                            │  │
  │  └──────────────────────┬─────────────────────────────┘  │
  │                         │                                │
  │                         ▼                                │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │             SQLite Database                        │  │
  │  │          ~/.openclaw/cost-tracking.db              │  │
  │  │                                                    │  │
  │  │  usage_events | model_pricing | agents | alerts    │  │
  │  └────────┬───────────────────┬───────────────────────┘  │
  │           │                   │                          │
  │  ┌────────▼────────┐  ┌──────▼───────────┐               │
  │  │   TELEMETRY     │  │    RELAY         │               │
  │  │   (TUI/CLI)     │  │    (TUI/CLI)     │               │
  │  │                 │  │                  │               │
  │  │  Cost views,    │  │  Task mgmt,      │               │
  │  │  budget alerts, │  │  Kanban,         │               │
  │  │  optimization   │  │  assignments     │               │
  │  └────────┬────────┘  └──────┬───────────┘               │
  │           │                  │                           │
  │           ▼                  ▼                           │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │              OPENCLAW AGENTS                       │  │
  │  │                                                    │  │
  │  │  • Read costs via CLI (sub-millisecond)            │  │
  │  │  • Create/update tasks via CLI                     │  │
  │  │  • Suggest optimizations                           │  │
  │  │  • Await human approval                            │  │
  │  └────────────────────────────────────────────────────┘  │
  │                                                          │
  │  All reads are local. Sub-millisecond. No network deps.  │
  └──────────────────────────────────────────────────────────┘


```

---

## Component Details

### Data Source: OpenClaw Session JSONL

**What it is:** The raw usage data that OpenClaw already produces.

**Where it lives:**

```
~/.openclaw/agents/{agent_name}/sessions/*.jsonl
```

Each agent has its own session directory. Agent attribution is automatic — the directory name _is_ the agent.

**What a usage event looks like:**

```json
{
  "type": "message",
  "message": {
    "role": "assistant",
    "provider": "anthropic",
    "model": "claude-opus-4-6",
    "usage": {
      "input": 45230,
      "output": 1847,
      "cacheRead": 108928,
      "cacheWrite": 4200,
      "cost": { "total": 0.85 }
    },
    "timestamp": 1769753935279
  }
}
```

**Provider field values:** `anthropic`, `openai`, `openai-codex`, `google`, etc.

**Token fields by provider:**

| Field        | Anthropic             | OpenAI                     | Google   |
| ------------ | --------------------- | -------------------------- | -------- |
| `input`      | ✓ (uncached)          | ✓ (total, includes cached) | ✓        |
| `output`     | ✓                     | ✓                          | ✓        |
| `cacheRead`  | ✓                     | ✓                          | ✓        |
| `cacheWrite` | ✓                     | — (free)                   | — (free) |
| `cost.total` | ✓ (OpenClaw estimate) | ✓                          | ✓        |

**Why this replaces the relay:** The data already exists locally. OpenClaw logs every model response with full token counts from all three providers. There's no need to poll external admin APIs when the same information originates on the VPS. This eliminates the Cloudflare Worker, D1 database, sync mechanism, admin API keys, and ~864 external API calls per day.

---

### Ingestion Service

**What it is:** A lightweight script (Python or Node) that runs on a cron (every 1-5 minutes) or as a file watcher.

**What it does:** Tails OpenClaw's session JSONL files, extracts usage events from assistant messages, computes estimated costs using the static pricing table, and writes per-request events to SQLite.

**How it works:**

1. Read `ingestion_state` table to get last-processed file offset per session file
2. Scan `~/.openclaw/agents/*/sessions/*.jsonl` for new data past stored offsets
3. For each new JSONL line with `type: "message"`, `role: "assistant"`, and usage data:
   - Extract provider, model, token counts, timestamp
   - Derive `agent_id` from the directory path
   - Look up current pricing from `model_pricing` table
   - Compute estimated cost: `(input × input_rate + output × output_rate + cacheRead × cache_read_rate + cacheWrite × cache_write_rate) / 1,000,000`
   - Insert into `usage_events`
4. Update `ingestion_state` with new file offsets
5. Check budget alerts and fire if thresholds are exceeded

**Key design decisions:**

- Idempotent re-runs via offset tracking. The script picks up where it left off.
- Per-request granularity (not pre-aggregated buckets). You get exact attribution: which session, which model call, which agent.
- Aggregation happens at query time, not at ingestion time.
- If OpenClaw rotates or archives session files, old entries in `ingestion_state` are simply ignored.

---

### Cost Estimation: The Static Pricing Table

**Target accuracy: ~90%.** Good enough for agents to understand their cost profile, identify expensive patterns, and self-optimize. Not for invoice reconciliation.

Published model pricing changes 2-3 times per year, announced in advance. Between changes, the pricing table is exact for standard usage. The only sources of drift are:

| Drift Source                                       | Impact                    | Notes                                   |
| -------------------------------------------------- | ------------------------- | --------------------------------------- |
| Context window tiers (Anthropic 0-200k vs 200k-1M) | ~10-20% on affected calls | Only Anthropic, only for large contexts |
| Batch discounts                                    | ~50% lower                | Agents don't typically batch            |
| Cache read discount variance (OpenAI 50-90%)       | ±20% on cached reads      | Use midpoint estimate                   |

For the intended use case — agents asking "should I use Opus or Sonnet for this?" or "am I burning too much on cache writes?" — a ±5-10% cost estimate is more than adequate. The decisions don't change at that margin.

**Optional reconciliation:** Provider billing APIs (Anthropic's `/v1/organizations/cost_report`, OpenAI's `/v1/organization/costs`) remain available for periodic comparison. This could be a weekly script, not a core pipeline dependency.

---

### openclaw-telemetry

**What it is:** A Go binary that runs on your host. Provides both a TUI (for humans) and a CLI (for agents).

**What it does:** Reads from the local SQLite database populated by the ingestion service. Provides cost monitoring, budget tracking, alerting, and optimization analysis.

**Dual interface pattern:**

```
Human (TUI):                          Agent (CLI):

┌──────────────────────┐              $ telemetry cost --today
│  DAILY SPEND BY      │              {"today": 34.19, "budget": 50.00,
│  PROVIDER            │               "remaining": 15.81}
│  ████████ $34.19     │
│                      │              $ telemetry alert --check
│  BUDGET: $50/day     │              {"alerts": [{"type": "daily_spend",
│  REMAINING: $15.81   │               "threshold": 25.00,
└──────────────────────┘               "current": 34.19, "status": "triggered"}]}
```

The TUI renders rich, interactive views with charts, tables, and navigation. The CLI returns structured JSON that an agent can parse in minimal tokens.

**SQLite schema (simplified):**

```sql
CREATE TABLE usage_events (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp           TEXT NOT NULL,       -- ISO 8601 from JSONL event
    agent_id            TEXT NOT NULL,       -- from session directory name
    provider            TEXT NOT NULL,       -- 'anthropic' | 'openai' | 'google'
    model               TEXT NOT NULL,
    input_tokens        INTEGER DEFAULT 0,
    output_tokens       INTEGER DEFAULT 0,
    cache_read_tokens   INTEGER DEFAULT 0,
    cache_write_tokens  INTEGER DEFAULT 0,   -- Anthropic only
    total_tokens        INTEGER DEFAULT 0,
    estimated_cost_usd  REAL,
    session_file        TEXT,
    session_id          TEXT
);

CREATE TABLE agents (
    id                  TEXT PRIMARY KEY,    -- 'dex', 'cassandra', 'max', 'borkus'
    display_name        TEXT NOT NULL,
    monthly_budget_usd  REAL,               -- null = no limit
    daily_alert_usd     REAL,
    is_active           INTEGER DEFAULT 1
);

CREATE TABLE model_pricing (
    provider        TEXT NOT NULL,
    model           TEXT NOT NULL,
    token_type      TEXT NOT NULL,           -- 'input' | 'output' | 'cache_read' | 'cache_write'
    cost_per_mtok   REAL NOT NULL,           -- USD per 1M tokens
    effective_date  TEXT NOT NULL,
    PRIMARY KEY (provider, model, token_type, effective_date)
);

CREATE TABLE alerts (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    metric      TEXT NOT NULL,
    threshold   REAL NOT NULL,
    enabled     INTEGER DEFAULT 1,
    created_at  TEXT NOT NULL
);

CREATE TABLE optimization_suggestions (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id              TEXT NOT NULL,
    suggestion            TEXT NOT NULL,
    reasoning             TEXT,
    estimated_savings_usd REAL,
    status                TEXT DEFAULT 'pending',  -- 'pending', 'approved', 'denied'
    created_at            TEXT NOT NULL,
    reviewed_at           TEXT
);

CREATE TABLE ingestion_state (
    file_path     TEXT PRIMARY KEY,
    last_offset   INTEGER NOT NULL DEFAULT 0,
    last_line_ts  TEXT,
    updated_at    TEXT DEFAULT (datetime('now'))
);
```

**The cost feedback loop:**

This is the core value proposition of Telemetry. It's not just a dashboard — it's a feedback loop:

1. **OpenClaw** logs every model response to session JSONL files
2. **Ingestion service** tails the logs, computes costs, writes to SQLite
3. **Agents** query their own cost data via CLI
4. **Agents** analyze patterns (cache hit rates, model costs, session efficiency)
5. **Agents** create optimization suggestions (stored in `optimization_suggestions` table)
6. **Humans** review suggestions in the TUI and approve/deny them
7. **Agents** read approved suggestions and adjust their behavior
8. Repeat

This loop means your agents are actively participating in their own cost optimization, but humans retain approval authority over changes. The agent can say "I think switching this task to Sonnet would save $2/day" — and you decide whether to let it.

---

### openclaw-dispatch

**What it is:** A Go binary that provides task management for both agents and humans.

**What it does:** Dispatch is a shared task board where humans assign work, agents pick up tasks, update status, and report results. Humans see the Kanban/table TUI. Agents interact via CLI.

**Dual interface pattern:**

```
Human (TUI):                          Agent (CLI):

┌──────────────────────┐              $ dispatch task create \
│  Todo (5)  │ Active  │                --title "Analyze cache ratios" \
│            │  (2)    │                --priority 2 \
│  #67 Max   │ #104    │                --assignee dex
│  Restart.. │ Review  │
│            │ WF45A.. │              $ dispatch task update \
│  #9 Cass   │         │                --id 104 \
│  Cost Op.. │         │                --status done \
└──────────────────────┘                --notes "Cache ratio improved to 22%"
```

**SQLite schema (simplified):**

```sql
CREATE TABLE tasks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    title       TEXT NOT NULL,
    description TEXT,
    project     TEXT,
    priority    INTEGER DEFAULT 3,
    status      TEXT DEFAULT 'todo',
    assignee    TEXT,
    due_date    TEXT,
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);

CREATE TABLE task_comments (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id     INTEGER NOT NULL REFERENCES tasks(id),
    author      TEXT NOT NULL,
    content     TEXT NOT NULL,
    created_at  TEXT NOT NULL
);
```

**Key design decisions:**

- Tasks have assignees that can be either humans or agent names. The system doesn't distinguish — a task assigned to "dex" is the same data structure as one assigned to "max."
- Agents create and update tasks via CLI with structured flags. No natural language parsing, no prompt overhead.
- The TUI provides filtering by assignee, project, priority, and status. The CLI supports the same filters as flags.

---

## Token Efficiency: Why This Matters

The entire architecture is designed around one principle: **minimize the tokens agents spend on operational overhead.**

| Operation           | Google Sheets Approach | Ground Control CLI |
| ------------------- | ---------------------- | ------------------ |
| Read today's cost   | ~2,000-4,000 tokens    | ~50-150 tokens     |
| Create a task       | ~1,500-3,000 tokens    | ~80-200 tokens     |
| Check budget status | ~2,000-3,500 tokens    | ~40-100 tokens     |
| List active tasks   | ~2,500-4,000 tokens    | ~100-400 tokens    |

Over hundreds of agent interactions per day, this adds up to real dollar savings. The architecture pays for itself.

---

## Data Flow Summary

```
1. LOG        OpenClaw logs every model response to session JSONL
                         │
2. INGEST     Ingestion service tails JSONL, computes costs
                         │
3. STORE      Per-request events written to local SQLite
                         │
4. SURFACE    TUI (humans) and CLI (agents) read from SQLite
                         │
5. ACT        Agents analyze, suggest optimizations
                         │
6. APPROVE    Humans review and approve/deny in TUI
                         │
7. ADAPT      Agents adjust behavior based on approvals
                         │
              └──────── REPEAT ────────┘
```

---

## Consumer Access Patterns

| Consumer                   | Data Source     | Access Method        | Latency |
| -------------------------- | --------------- | -------------------- | ------- |
| **Telemetry TUI**          | Local SQLite    | File read            | <1ms    |
| **Agents (self-throttle)** | Local SQLite    | CLI → file read      | <1ms    |
| **Ingestion service**      | JSONL → SQLite  | Cron or file watcher | 1-5 min |
| **Dashboard (future)**     | SQLite via API  | HTTP from VPS        | ~50ms   |
| **Ad-hoc queries**         | SQLite directly | `sqlite3` CLI        | Instant |

---

## Deployment Model

The typical Ground Control deployment runs entirely on a single host:

```
┌──────────────────────────────────────────┐
│           Your VPS / Host                │
│                                          │
│  OpenClaw Gateway (agents + providers)   │
│        │                                 │
│        ▼                                 │
│  Session JSONL files                     │
│        │                                 │
│        ▼                                 │
│  Ingestion Service (cron)                │
│        │                                 │
│        ▼                                 │
│  SQLite database                         │
│        │                                 │
│   ┌────┴────┐                            │
│   ▼         ▼                            │
│  Telemetry  Dispatch                     │
│  (TUI/CLI)  (TUI/CLI)                   │
│                                          │
│  Zero external dependencies.             │
└──────────────────────────────────────────┘
```

**Minimum requirements for the host:**

- Linux (any distro), macOS, or Windows
- ~50MB disk for binaries
- ~10MB RAM per tool (Go is efficient)
- SQLite 3 (usually pre-installed)
- Access to OpenClaw session directories

This runs comfortably on a $5/month KVM VPS. It was designed to.

---

## Future Architecture Considerations

**Web dashboard:** A SvelteKit app reading from a lightweight VPS API, or Litestream replicating SQLite to R2 for edge reads.

**Reconciliation layer:** Provider billing APIs remain available for periodic comparison. A weekly script could compare `SUM(estimated_cost_usd)` against actual provider costs, flag drift > 10%, and adjust the pricing table. This is insurance, not infrastructure.

**Cross-tool integration:** Telemetry and Dispatch currently operate independently with separate SQLite databases. A future integration point would allow telemetry data to inform dispatch decisions — for example, automatically creating a high-priority task when a budget alert fires, or pausing low-priority agent tasks when spend exceeds a threshold.

**Plugin system:** The current architecture is intentionally simple. As the ecosystem grows, a lightweight plugin interface could allow community-built extensions (new providers, custom alert handlers, alternative storage backends) without bloating the core tools.

**Multi-host support:** The current model assumes a single host. For users running agents across multiple machines, a future version could aggregate data from multiple hosts into a central Telemetry instance.

---

_This document is a living reference. As the architecture evolves, so will this doc. If something is unclear or outdated, please [open an issue](https://github.com/max-geller/ground-control/issues)._
