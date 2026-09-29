# klavis — TB3 Task Submission: `log-report-perf`

A Terminal-Bench 3 (TB3) task submission for the Klavis AI Assessment.

---

## Overview

`log-report-perf` is a database performance task where an AI agent must diagnose and fix a Python CLI script that takes over 60 seconds to query a 30-million-row PostgreSQL table — and write a technical explanation of every fix.

The task is designed to be hard: three independent performance bottlenecks are stacked so that fixing any two out of three still fails the 10-second runtime threshold. All 6 frontier model trials (3 Codex, 3 Claude Code) scored 0.

---

## What the Agent Must Do

1. Find and fix **three performance bottlenecks** in the codebase
2. Bring total script runtime from ~65 seconds to under **10 seconds**
3. Write `/app/analysis.md` explaining each fix with specific technical vocabulary

---

## The Three Bottlenecks

| # | Problem | Why Agents Miss It |
|---|---------|-------------------|
| 1 | Non-sargable `DATE()` predicate in `query_builder.py` | Bug is not in the main file |
| 2 | Misleading partial index that looks like it covers timestamps | Index exists but only covers ~25% of rows |
| 3 | Correlated subquery running 100 separate DB queries | Agents stop profiling after fixing the main query |

---

## Repository Structure

```
.
├── README.md
├── HOW_TO_RUN.md          ← Full setup and run instructions
├── docs/
│   └── My_Implementation.md   ← Plain-English design explanation
└── tasks/
    └── log-report-perf/       ← TB3 task directory
```

---

## Quick Start

See [HOW_TO_RUN.md](HOW_TO_RUN.md) for full setup instructions.

**Requirements:** Docker (Compose v2), Python 3.10+, `uv`, 8 GB RAM, 30 GB disk.

```bash
# Install Harbor CLI
uv tool install harbor

# Clone TB3 and copy the task
git clone https://github.com/harbor-framework/terminal-bench-3
cp -r tasks/log-report-perf terminal-bench-3/tasks/

# Validate (oracle must score 1.0, nop must score 0.0)
harbor run -p tasks/log-report-perf --agent oracle
harbor run -p tasks/log-report-perf --agent nop
```

---

## Verifier

The verifier runs in an isolated Docker container and checks 10 things:

- Script exits cleanly, output is valid JSON
- Today's data is present, 100 service rows returned
- Total runtime **< 10 seconds**
- `log_events` still has 30,000,000 rows (data not deleted)
- No materialized views created as a shortcut
- `analysis.md` contains: `sargable`, `sequential`, `partial`, `correlated`/`subquery`, and timing numbers

---

## Trial Results

| Agent | Trials | Score |
|-------|--------|-------|
| Codex (gpt-5.6-sol) | 3 | 0 / 3 |
| Claude Code (claude-opus-4-5) | 3 | 0 / 3 |

**Pass rate: 0 / 6** — by design. The oracle scores 1.0, confirming the task is solvable.

---

## Design Notes

See [docs/My_Implementation.md](docs/My_Implementation.md) for a full plain-English walkthrough of the design decisions, traps, and calibration strategy.
