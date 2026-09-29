# What We Did — log-report-perf (Plain English)

---

## The Goal

Build a TB3 task where an AI agent must fix a **slow database script** — not just find a bug in logic, but understand why a query performs badly and fix it properly.

The challenge: make the task hard enough that even the best AI models fail it, while keeping it fair (the correct solution is always reachable).

---

## What the Agent Sees

The agent gets a Docker container with:

- A Python script `/app/report.py` that generates a daily incident report
- A PostgreSQL database with **30 million rows** in a table called `log_events`
- The script takes over **60 seconds to run**
- The instruction: find what's slow, fix it, write an explanation

No hints. No pointers to specific files or functions. Just the terminal and the code.

---

## The Three Problems We Planted

We stacked three independent performance problems. The agent must fix **all three** — the verifier requires total runtime under 10 seconds, and each unfixed problem contributes enough latency to keep the total above the threshold.

---

### Problem 1 — The Bug Hiding in an Imported File

Most agents open `report.py` first. This is what they see:

```python
def fetch_main_report(conn):
    sql = get_main_report_sql()   # imported from query_builder
    with conn.cursor(...) as cur:
        cur.execute(sql)
        return cur.fetchall()
```

Clean. No obvious problem. The bug is not here.

The bug is in `query_builder.py`, which the agent has to discover and open separately:

```sql
WHERE DATE(le.timestamp) = CURRENT_DATE
```

**Why this is slow:**

`DATE(le.timestamp)` wraps the column inside a function. PostgreSQL has an index on the `timestamp` column, but the index stores raw timestamp values — not pre-computed date values. The database cannot use the index to answer this question. It has to check every single one of the 30 million rows one by one.

This is called a **non-sargable predicate** — a filter that defeats index use.

**The fix:**

```sql
WHERE le.timestamp >= CURRENT_DATE
  AND le.timestamp <  CURRENT_DATE + INTERVAL '1 day'
```

Now the column is compared directly to two boundary values. The index works perfectly and the database reads only ~10,000 rows instead of 30,000,000.

---

### Problem 2 — The Index That Looks Like It Works (But Doesn't)

When an agent investigates the database schema they see this:

```
Indexes:
  "idx_log_events_timestamp_critical" btree (timestamp) WHERE service_id IN (...)
```

There's already an index on `timestamp`. Most agents stop here and think the indexing problem is solved.

But this is a **partial index** — it only covers rows where the service belongs to the "critical" tier. That's about 25% of the table.

PostgreSQL can only use a partial index if the query's filter implies the index's condition. A general query that asks for all services today cannot use an index that only covers critical services. The database still does a full sequential scan.

**The trap:** The index looks like it covers timestamps. It doesn't cover the query.

**The fix:** Drop the misleading partial index, create a proper one that covers everything, and update the planner's statistics:

```sql
DROP INDEX idx_log_events_timestamp_critical;
CREATE INDEX idx_log_events_timestamp ON log_events (timestamp);
ANALYZE log_events;
```

The `ANALYZE` step matters because the database was seeded without running statistics — the planner has no idea how many rows exist and defaults to conservative (slow) choices.

---

### Problem 3 — A Second Slow Function Nobody Profiles

Even after fixing the main query, `report.py` has a second function: `fetch_service_summary`. It computes how many log events each of the 100 services had today.

Here is how it's written:

```sql
SELECT s.id, s.name, s.tier,
    (
        SELECT COUNT(*)
        FROM   log_events le2
        WHERE  le2.service_id = s.id
          AND  le2.timestamp >= CURRENT_DATE
          AND  le2.timestamp <  CURRENT_DATE + INTERVAL '1 day'
    ) AS today_count
FROM services s
```

This is a **correlated subquery**. For each of the 100 services, the database runs a separate COUNT query against `log_events`. That's 100 separate queries, each doing its own range scan. Even with the correct index, 100 × 0.1 seconds = 10 seconds on its own.

Agents that fix the main query see the script get much faster, assume they're done, and don't profile this function separately.

**The trap:** The total elapsed time drops from 65 s to ~12 s after the first two fixes — which feels like a success but is still above the 10-second threshold.

**The fix:** Replace the correlated subquery with a single aggregated join:

```sql
SELECT s.id, s.name, s.tier,
    COALESCE(agg.today_count, 0) AS today_count
FROM services s
LEFT JOIN (
    SELECT service_id, COUNT(*) AS today_count
    FROM   log_events
    WHERE  timestamp >= CURRENT_DATE
      AND  timestamp <  CURRENT_DATE + INTERVAL '1 day'
    GROUP BY service_id
) agg ON agg.service_id = s.id
```

One pass over `log_events` instead of 100. Runtime for this function drops from ~10 s to < 0.1 s.

---

## The Documentation Requirement

Fixing the code is not enough. The agent must also write `/app/analysis.md` — a technical explanation of what it found and how it fixed it.

The verifier checks the file for specific words:

| Required word | Tests that the agent understood |
|--------------|-------------------------------|
| `sargable` or `non-sargable` | The DATE() predicate issue |
| `sequential` or `seq scan` | Reading EXPLAIN ANALYZE output |
| `partial` | The partial index limitation |
| `correlated` or `subquery` | The N+1 query pattern |
| A number followed by `s`, `ms`, etc. | Actual timing measurements |

A vague writeup saying "I added an index and the query got faster" fails these checks.

---

## How We Built It

### The Database

30 million rows is big enough that the slow path (sequential scan) takes 60–120 seconds, and the fast path (index scan) takes under 0.3 seconds. The difference is dramatic and unambiguous.

The data was seeded during the Docker image build — not at container startup. Seeding 30M rows takes about 12 minutes. If we did this at startup, Harbor's timeout would kill it. By baking it into the image, the container starts instantly.

We also intentionally skipped `ANALYZE` after seeding and disabled PostgreSQL's autovacuum on the table. This leaves the planner with stale (zero-row) statistics, making it even more likely to choose sequential scans.

### The Partial Index Trap

We pre-created a misleading index that looks like it solves the performance problem. This is the key to why agents fail:

1. Agent fixes the DATE() predicate in query_builder.py ✓
2. Agent runs `\d log_events` to check indexes
3. Agent sees an index on `timestamp` and concludes indexing is fine ✗
4. Agent does not read the `WHERE` clause on the index
5. `EXPLAIN ANALYZE` still shows Seq Scan — agent is confused
6. Agent gives up or loops without making progress

### The File Structure Trap

The most important bug is not in the file agents open first. `report.py` is clean and readable. The broken SQL is in `query_builder.py` which is imported. Agents must trace the function call across the import boundary.

### The Threshold Calibration

The 10-second threshold was chosen so that:
- Fixing Bottleneck 1 only: still ~10–15 s (correlated subquery)
- Fixing Bottleneck 2 only: still ~60 s (DATE() still defeats the full index)
- Fixing Bottleneck 3 only: still ~60 s (main query still sequential scan)
- Fixing Bottleneck 1 + 2: still ~10–12 s (correlated subquery remains)
- Fixing all three: ~0.2–0.5 s

Every partial combination still fails. You must fix all three.

---

## How the Verifier Works

The verifier runs in a completely separate Docker container that the agent never touches.

1. It restores a `pg_dump` of the agent's modified database — this captures any indexes the agent created or dropped
2. It runs `report.py` from scratch and measures wall-clock time
3. It checks the JSON output for correctness (right fields, right row counts, data not deleted)
4. It reads `/app/analysis.md` and checks for the required vocabulary

The verifier writes `reward.txt` with either `1` (all 10 tests pass) or `0` (anything failed).

---

## What Happened in Agent Trials

We ran 6 trials — 3 Codex, 3 Claude Code. All 6 scored 0.

**Codex:** Fixed the DATE() predicate and created a new index, but did not drop the partial index. Saw the new index in `\d log_events` and concluded indexing was done. Never found the correlated subquery. Runtime stayed above 30 s.

**Claude Code:** Correctly fixed the DATE() predicate, identified the partial index as a problem, dropped it, created the unconditional index. Did not time `fetch_service_summary` separately. Runtime dropped to ~12 s — close, but still above the 10 s threshold by about 2 seconds. Reward = 0.

**Pass rate: 0 / 6**

This is the outcome we designed for. The task is hard enough to stop frontier models, but the oracle (running the reference solution) scores 1.0 — proving the task is solvable.

---

## Summary

We built a database performance task with three hidden problems, each designed to fool a different diagnostic approach:

- **Problem 1** fools agents that only read the main file
- **Problem 2** fools agents that check for indexes but don't read partial index conditions
- **Problem 3** fools agents that profile only the main function and call it done

The 10-second performance threshold was calibrated so that fixing any two out of three still fails. The documentation requirement catches agents that fix the code but don't understand what they fixed.

Result: 0 out of 6 frontier model trials passed.
