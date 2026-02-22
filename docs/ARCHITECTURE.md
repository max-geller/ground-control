# 🦞 Ground Control — Architecture

> How data flows from the API providers to your terminal.

This document describes how the Ground Control ecosystem fits together: how cost and usage data is collected, stored, and surfaced to both humans and agents.

---

## Overview

Ground Control follows a simple three-stage pipeline: **collect → store → act**.

```
                        ┌─────────────────┐
                        │  Anthropic API  │
                        └────────┬────────┘
                                 │
                        ┌────────┴────────┐
                        │   OpenAI API    │
                        └────────┬────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      OPENCLAW-RELAY     │
                    │   (Cloudflare Worker)   │
                    │                         │
                    │  • Polls usage APIs     │
                    │  • Normalizes data      │
                    │  • Forwards to host     │
                    └────────────┬────────────┘
                                 │
                          HTTPS POST
                         (JSON payload)
                                 │
              ┌──────────────────▼───────────────────┐
              │           HOST / VPS                 │
              │                                      │
              │  ┌────────────────────────────────┐  │
              │  │       OPENCLAW-TELEMETRY       │  │
              │  │                                │  │
              │  │  Collector ──▶ SQLite DB      │  │
              │  │                  │             │  │
              │  │           ┌─────┴──────┐       │  │
              │  │           │            │       │  │
              │  │        TUI View    CLI Query   │  │
              │  │        (human)     (agent)     │  │
              │  └────────────────────────────────┘  │
              │                                      │
              │  ┌────────────────────────────────┐  │
              │  │       OPENCLAW-DISPATCH        │  │
              │  │                                │  │
              │  │  Task Store ──▶ SQLite DB     │  │
              │  │                  │             │  │
              │  │           ┌─────┴──────┐       │  │
              │  │           │            │       │  │
              │  │        TUI View    CLI Commands│  │
              │  │        (human)     (agent)     │  │
              │  └────────────────────────────────┘  │
              │                                      │
              │  ┌────────────────────────────────┐  │
              │  │        OPENCLAW AGENTS         │  │
              │  │                                │  │
              │  │  • Read costs via CLI          │  │
              │  │  • Create/update tasks via CLI │  │
              │  │  • Suggest optimizations       │  │
              │  │  • Await human approval        │  │
              │  └────────────────────────────────┘  │
              └──────────────────────────────────────┘
```

---

## Component Details

### openclaw-relay

**What it is:** A Cloudflare Worker deployed on the edge.

**What it does:** Relay runs on a cron schedule (configurable, default 15 minutes) and pulls usage and billing data from the Anthropic and OpenAI APIs. It normalizes the data into a common format and forwards it to your host via HTTPS POST.

**Why Cloudflare Workers:** They're free (or near-free) on the Cloudflare free tier, they run globally on the edge, and they require zero server infrastructure. Relay doesn't need to live on your VPS — it runs independently and just pushes data to wherever your telemetry instance is listening.

**Data flow:**
```
Anthropic Usage API ──┐
                      ├──▶ Relay (normalize) ──▶ POST to host
OpenAI Usage API ─────┘
```

**Normalized payload (simplified):**
```json
{
  "provider": "anthropic",
  "model": "opus-4.6",
  "timestamp": "2026-02-21T14:00:00Z",
  "input_tokens": 12500,
  "output_tokens": 3200,
  "cache_write_tokens": 850,
  "cache_read_tokens": 200,
  "cost_usd": 0.47,
  "requests": 3
}
```

**Key design decisions:**
- Relay is stateless. It doesn't store anything — it just fetches and forwards.
- One Worker handles multiple providers. Provider-specific logic is isolated in adapter modules.
- The sync interval is configurable. 15 minutes is the default balance between freshness and API rate limits.

---

### openclaw-telemetry

**What it is:** A Go binary that runs on your host. Provides both a TUI (for humans) and a CLI (for agents).

**What it does:** Telemetry receives data from Relay, stores it in a local SQLite database, and provides cost monitoring, budget tracking, alerting, and optimization analysis.

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
CREATE TABLE usage_records (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    provider    TEXT NOT NULL,        -- 'anthropic', 'openai'
    model       TEXT NOT NULL,        -- 'opus-4.6', 'gpt-4o', etc.
    timestamp   TEXT NOT NULL,        -- ISO 8601
    input_tokens    INTEGER NOT NULL,
    output_tokens   INTEGER NOT NULL,
    cache_write_tokens INTEGER DEFAULT 0,
    cache_read_tokens  INTEGER DEFAULT 0,
    cost_usd    REAL NOT NULL,
    requests    INTEGER NOT NULL
);

CREATE TABLE budgets (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    period      TEXT NOT NULL,        -- 'daily', 'weekly', 'monthly'
    amount_usd  REAL NOT NULL,
    provider    TEXT,                 -- NULL = all providers
    created_at  TEXT NOT NULL
);

CREATE TABLE alerts (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    metric      TEXT NOT NULL,        -- 'daily_spend', 'hourly_spend', 'cache_write_ratio'
    threshold   REAL NOT NULL,
    enabled     INTEGER DEFAULT 1,
    created_at  TEXT NOT NULL
);

CREATE TABLE optimization_suggestions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id    TEXT NOT NULL,
    suggestion  TEXT NOT NULL,
    reasoning   TEXT,
    estimated_savings_usd REAL,
    status      TEXT DEFAULT 'pending', -- 'pending', 'approved', 'denied'
    created_at  TEXT NOT NULL,
    reviewed_at TEXT
);
```

**The cost feedback loop:**

This is the core value proposition of Telemetry. It's not just a dashboard — it's a feedback loop:

1. **Relay** collects raw usage data from APIs
2. **Telemetry** stores and aggregates it
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
    priority    INTEGER DEFAULT 3,    -- 1 (highest) to 4 (lowest)
    status      TEXT DEFAULT 'todo',  -- 'todo', 'in_progress', 'blocked', 'done'
    assignee    TEXT,                 -- 'max', 'cassandra', 'dex', etc.
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

Here's a concrete comparison:

| Operation | Google Sheets Approach | Ground Control CLI |
|---|---|---|
| Read today's cost | ~2,000-4,000 tokens (API call, parse HTML/JSON response, navigate sheet structure) | ~50-150 tokens (`telemetry cost --today` → small JSON response) |
| Create a task | ~1,500-3,000 tokens (format API request, handle auth, parse response) | ~80-200 tokens (`dispatch task create --title "..." --priority 2`) |
| Check budget status | ~2,000-3,500 tokens (query sheet, aggregate, calculate) | ~40-100 tokens (`telemetry budget --check` → single JSON object) |
| List active tasks | ~2,500-4,000 tokens (query sheet, filter, format) | ~100-400 tokens (`dispatch task list --status in_progress` → JSON array) |

Over hundreds of agent interactions per day, this adds up to real dollar savings. The architecture pays for itself.

---

## Data Flow Summary

```
1. COLLECT    Relay polls Anthropic/OpenAI APIs on a schedule
                         │
2. TRANSPORT  Relay POSTs normalized JSON to your host
                         │
3. STORE      Telemetry collector writes to SQLite
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

## Deployment Model

The typical Ground Control deployment looks like this:

```
┌─────────────────────┐          ┌──────────────────────┐
│  Cloudflare Edge    │          │  Your VPS / Host     │
│                     │          │                      │
│  openclaw-relay     │──HTTPS──▶│  openclaw-telemetry  │
│  (Worker + Cron)    │          │  openclaw-dispatch   │
│                     │          │  OpenClaw agents     │
│  Free tier works    │          │  SQLite databases    │
└─────────────────────┘          └──────────────────────┘
```

**Minimum requirements for the host:**
- Linux (any distro), macOS, or Windows
- ~50MB disk for binaries
- ~10MB RAM per tool (Go is efficient)
- SQLite 3 (usually pre-installed)
- Inbound HTTPS for Relay data (or run Relay locally)

This runs comfortably on a $5/month KVM VPS. It was designed to.

---

## Future Architecture Considerations

**Cross-tool integration:** Telemetry and Dispatch currently operate independently with separate SQLite databases. A future integration point would allow telemetry data to inform dispatch decisions — for example, automatically creating a high-priority task when a budget alert fires, or pausing low-priority agent tasks when spend exceeds a threshold.

**Plugin system:** The current architecture is intentionally simple. As the ecosystem grows, a lightweight plugin interface could allow community-built extensions (new providers, custom alert handlers, alternative storage backends) without bloating the core tools.

**Multi-host support:** The current model assumes a single host. For users running agents across multiple machines, a future version of Relay could aggregate data from multiple hosts into a central Telemetry instance.

---

*This document is a living reference. As the architecture evolves, so will this doc. If something is unclear or outdated, please [open an issue](https://github.com/max-geller/ground-control/issues).*