# How to Run — log-report-perf
**Klavis AI Assessment — Terminal-Bench 3**

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| **Docker** (with Compose v2) | `docker compose version` must work |
| **Python 3.10+** | For `uv` and Harbor |
| **uv** | Python package manager — install below |
| **8 GB RAM** free | Postgres needs ~4 GB for the 30M-row dataset |
| **30 GB disk** free | For the postgres image layer |
| **API key** | OpenAI or Anthropic — only needed for agent trials |

---

## Step 1 — Install uv (if not already installed)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Then restart your shell or run:
```bash
source $HOME/.cargo/env   # or add ~/.local/bin to PATH
```

---

## Step 2 — Clone the TB3 Repository

```bash
git clone https://github.com/harbor-framework/terminal-bench-3
cd terminal-bench-3
```

---

## Step 3 — Install Harbor CLI

```bash
uv tool install harbor
```

Verify:
```bash
harbor --version
# should print: 0.23.0
```

---

## Step 4 — Copy the Task into the TB3 Repo

```bash
# From the extracted zip folder:
cp -r tasks/log-report-perf /path/to/terminal-bench-3/tasks/
```

All commands below assume you are **inside the `terminal-bench-3/` directory**.

---

## Step 5 — Run the CI Checks (Optional but Recommended)

Confirms the task passes all 25 TB3 static checks:

```bash
for check in scripts/checks/check-*.sh; do
    echo "--- $check ---"
    bash "$check" tasks/log-report-perf
done
```

All checks should print `PASS`.

---

## Step 6 — Validate with Oracle and Nop

Before running expensive agent trials, confirm the task is correctly built:

```bash
# Oracle: runs the reference solution — must score 1.0
harbor run -p tasks/log-report-perf --agent oracle

# Nop: does nothing — must score 0.0
harbor run -p tasks/log-report-perf --agent nop
```

> **Note:** The postgres image seeds 30 million rows during `docker build`. This takes **10–15 minutes** on first run. Subsequent runs use the cached image and start instantly.

Expected output:

```
oracle → reward = 1.0  ✅
nop    → reward = 0.0  ✅
```

---

## Step 7 — Run Agent Trials

### With Codex (OpenAI)

```bash
harbor run -p tasks/log-report-perf \
  --agent codex \
  --model openai/gpt-5.6-sol \
  --env docker --yes \
  --ae OPENAI_API_KEY=<your-openai-key>
```

### With Claude Code (Anthropic)

```bash
harbor run -p tasks/log-report-perf \
  --agent claude-code \
  --model anthropic/claude-opus-4-5 \
  --env docker --yes \
  --ae ANTHROPIC_API_KEY=<your-anthropic-key>
```

### Other available agents

```bash
harbor agent list   # see all installed agents
```

---

## Step 8 — Find Trial Results

Harbor saves each trial under `jobs/`:

```bash
ls jobs/
# e.g.: 2026-09-25__10-30-00/log-report-perf__abc123/
```

Inside each job directory:

```
result.json            ← reward score (verifier_result.rewards.reward)
agent/
  trajectory.json      ← every bash command the agent ran
  agent.log            ← agent framework output
verifier/
  reward.txt           ← 0 or 1
  pytest.log           ← test results
  ctrf.json            ← machine-readable test report
```

Check the reward:
```bash
cat jobs/*/log-report-perf*/result.json | python3 -m json.tool | grep reward
```

---

## Task Summary — log-report-perf

A Python CLI script queries a 30-million-row PostgreSQL table and takes over 60 seconds.
The agent must fix **three independent performance bottlenecks** and write a technical
analysis to `/app/analysis.md`.

**Three bottlenecks stacked:**
1. Non-sargable `DATE()` predicate in an imported module (`query_builder.py`) — not in the main file
2. A misleading partial index that looks like it covers timestamps but only covers ~25% of rows
3. A correlated subquery running 100 separate DB queries per invocation

**Verifier checks 10 things:**
- Script exits cleanly, output is valid JSON
- Today's data is present, 100 service rows returned
- Total runtime < 10 seconds
- `log_events` still has 30,000,000 rows (data not deleted)
- No materialized views created as a shortcut
- `analysis.md` exists and contains: `sargable`, `sequential`, `partial`, `correlated`/`subquery`, and timing numbers

**Build time:** ~12 minutes (30M-row seed). After first build, cached.

---

## Troubleshooting

### Build takes forever / fails

The 30M-row postgres seed requires ~10 GB of Docker build cache. If disk is low, free space first:

```bash
docker system prune -f
```

### "pg_dump missing or empty" in verifier log

The artifact collection hook failed to dump the database — usually a timing issue. Re-run the trial.

### "AgentAuthenticationError"

Your API key was not passed correctly. Confirm the `--ae` flag includes the full key:

```bash
--ae OPENAI_API_KEY=sk-proj-...
--ae ANTHROPIC_API_KEY=sk-ant-...
```

### Agent container starts but postgres health check fails

The postgres container may need more time on slower machines. Increase `start_period` in `environment/docker-compose.yaml`.

---

## Quick Reference — All Commands

```bash
# Install
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install harbor

# Clone TB3
git clone https://github.com/harbor-framework/terminal-bench-3
cd terminal-bench-3

# Copy task
cp -r /path/to/zip/tasks/log-report-perf tasks/

# CI checks
for check in scripts/checks/check-*.sh; do bash "$check" tasks/log-report-perf; done

# Validate
harbor run -p tasks/log-report-perf --agent oracle
harbor run -p tasks/log-report-perf --agent nop

# Agent trial (OpenAI)
harbor run -p tasks/log-report-perf \
  --agent codex --model openai/gpt-5.6-sol \
  --env docker --yes --ae OPENAI_API_KEY=<key>

# Agent trial (Anthropic)
harbor run -p tasks/log-report-perf \
  --agent claude-code --model anthropic/claude-opus-4-5 \
  --env docker --yes --ae ANTHROPIC_API_KEY=<key>

# Check result
cat jobs/*/log-report-perf*/result.json
```
