# Implementation: Automatic Support Ticket Assignment

## 0. Stack

- **Runtime:** Node.js + TypeScript
- **Web framework:** Express
- **Storage:** PostgreSQL via Prisma (requires a local Postgres instance; connection string via `DATABASE_URL`) — chosen over SQLite to mirror production behaviour, per review
- **Validation:** zod on request bodies
- **Testing:** Vitest + supertest

---

## 1. Data model

```prisma
model Company {
  id                 String   @id
  name               String
  timezone           String   // IANA tz name, e.g. "America/New_York"
  defaultTicketLimit Int

  agents              Agent[]
  coverageRequirements CoverageRequirement[]
  tickets             Ticket[]
}

model Agent {
  id               String   @id
  companyId        String
  name             String
  timezone         String   // IANA tz name
  active           Boolean  @default(true)   // on/off switch (assumption 8)
  ticketLimit      Int?     // override; null = use company.defaultTicketLimit
  lastAssignedAt   DateTime? // tie-break cursor (assumption 12/13)

  company Company        @relation(fields: [companyId], references: [id])
  shifts  AvailabilityShift[]
  tickets Ticket[]
}

model AvailabilityShift {
  id           String @id @default(cuid())
  agentId      String
  weekday      Int    // 0=Sunday .. 6=Saturday, in the agent's own timezone
  startMinute  Int    // minutes since local midnight, 0-1439
  endMinute    Int    // minutes since local midnight, 0-1439
  // endMinute <= startMinute means the shift wraps past midnight into the next weekday

  agent Agent @relation(fields: [agentId], references: [id])
}

model CoverageRequirement {
  id           String @id @default(cuid())
  companyId    String
  weekday      Int    // in the company's own timezone
  startMinute  Int
  endMinute    Int    // same midnight-wrap convention as AvailabilityShift
  minAgents    Int

  company Company @relation(fields: [companyId], references: [id])
}

model Ticket {
  companyId    String
  id           String                   // the ticket_id from the ticketing system — unique WITHIN a company only, not globally
  assignedAgentId String?
  assignedAt   DateTime?
  resolvedAt   DateTime?                // null = still active/unresolved
  lastRawStatus String?                 // last status string received, for audit only

  company Company @relation(fields: [companyId], references: [id])
  agent   Agent?  @relation(fields: [assignedAgentId], references: [id])
  decision AssignmentDecision?

  @@id([companyId, id])
}

model AssignmentDecision {
  id              String   @id @default(cuid())
  companyId        String
  ticketId        String
  chosenAgentId   String?             // null = no eligible agent (yet)
  reason          String              // human-readable summary
  consideredAgents Json                // per-agent: eligible?, load ratio, exclusion reason
  decidedAt       DateTime @default(now())

  ticket Ticket @relation(fields: [companyId, ticketId], references: [companyId, id])

  @@unique([companyId, ticketId])
}
```

Notes:
- An agent's **active (unresolved) ticket count** is `count(Ticket where assignedAgentId = agent.id and resolvedAt is null)` — no separate counter to keep in sync, no time window (matches prd.md assumption 11).
- `AssignmentDecision` is what makes assignment **idempotent** (assumption 16) and **explainable** (scope item "Explanation") — the same row answers both the ticketing system's retry and the team lead's later lookup.
- `Ticket` and `AssignmentDecision` are separate tables so a "no eligible agent" outcome can still be recorded and looked up, even though no agent is assigned.
- Ticket identity is the composite `(companyId, id)` throughout, matching prd.md's "unique within a company" — not a bare global id. Two different companies can both have a ticket `T-123` without colliding.
- `AssignmentDecision.chosenAgentId` is the single source of truth for whether a decision is frozen: `null` means it's a retriable attempt (see §3), non-null means it's permanent.

---

## 2. API

All routes are scoped under a company. No auth (out of scope).

### Ticketing-system-facing

| Method & path | Purpose |
|---|---|
| `POST /api/companies/:companyId/tickets/:ticketId/assign` | Core assignment call. Idempotent — if a decision already exists for this ticket, returns it unchanged instead of recomputing. |
| `POST /api/companies/:companyId/tickets/:ticketId/status` | Body `{ status: string }`. Marks the ticket resolved if `status` matches a terminal status; otherwise a no-op. No-op also if the ticket is unknown (assumption 17) or already resolved (finality). |

`assign` response shape (also used by the explanation-lookup UI):
```json
{
  "ticketId": "T-123",
  "companyId": "acme",
  "assignedAgentId": "agent_7",
  "reason": "Lowest active/limit ratio among 3 eligible agents (0.40).",
  "consideredAgents": [
    { "agentId": "agent_7", "eligible": true, "activeTickets": 2, "limit": 5, "ratio": 0.40 },
    { "agentId": "agent_2", "eligible": false, "excludedBecause": "off-shift" },
    { "agentId": "agent_9", "eligible": false, "excludedBecause": "at-limit" }
  ],
  "decidedAt": "2026-09-30T10:15:00Z"
}
```

### Team-lead-facing (backs the UI)

| Method & path | Purpose |
|---|---|
| `GET /api/companies/:companyId/agents` | List agents with their timezone, active flag, ticket limit, and shifts. |
| `PUT /api/companies/:companyId/agents/:agentId` | Update timezone, active on/off switch, ticket limit override. |
| `PUT /api/companies/:companyId/agents/:agentId/shifts` | Replace the agent's full weekly shift list. |
| `GET /api/companies/:companyId/coverage-requirements` | List required coverage windows. |
| `PUT /api/companies/:companyId/coverage-requirements` | Replace the full weekly coverage requirement list. |
| `GET /api/companies/:companyId/coverage-gaps?from=&to=` | Computed report of under-staffed windows in the given date range (defaults to the next 7 days). |
| `GET /api/companies/:companyId/settings` | Get the company default ticket limit. |
| `PUT /api/companies/:companyId/settings` | Update the company default ticket limit. |
| `GET /api/companies/:companyId/tickets/:ticketId/explanation` | Read-only lookup of a past assignment decision (same shape as `assign`'s response). |

Agent, company and ticket **creation** is intentionally absent — prd.md scopes that out ("assumed to exist"); these are seeded directly into the DB.

---

## 3. Assignment algorithm

The read in step 1 is a plain, unguarded read — under concurrent calls for the same ticket, it is **not** what makes this race-safe. Two concurrent calls can both read "no decision yet" and both proceed to compute eligibility; that duplicated computation is allowed to happen. What must never happen is a losing computation's result clobbering a winning one — that's enforced entirely by the **write** in steps 5 and 7 being a single, conditionally-guarded upsert, not by locking or by the read:

1. Look up an existing `AssignmentDecision` for `(companyId, ticketId)`. If found **and `chosenAgentId` is set**, return it as-is unchanged — a successful assignment is frozen forever (idempotent — assumption 16). If found with `chosenAgentId = null` (a previous "no eligible agent" attempt), or not found at all, continue to step 2 — a non-final attempt is always re-evaluated from scratch.
2. Load all **active** agents for the company.
3. Filter to agents **under limit**: `activeTicketCount(agent) < (agent.ticketLimit ?? company.defaultTicketLimit)`. A limit of `0` always excludes the agent. Checked first since it's a cheap comparison against already-loaded counts, before paying for the timezone math in step 4.
4. Filter the remainder to agents **on shift right now**: convert the current instant to each agent's local weekday + minute-of-day (via the IANA tz database, which handles DST automatically), and check it against that agent's shifts, treating `endMinute <= startMinute` as wrapping into the next weekday.
5. If no agents remain: ensure the `Ticket` row exists (`INSERT ... ON CONFLICT (companyId, id) DO NOTHING` — this path never touches `assignedAgentId`/`assignedAt`), then write the attempt as `INSERT INTO AssignmentDecision (...) VALUES (..., chosenAgentId = null, ...) ON CONFLICT (companyId, ticketId) DO UPDATE SET reason = EXCLUDED.reason, consideredAgents = EXCLUDED.consideredAgents, decidedAt = EXCLUDED.decidedAt **WHERE AssignmentDecision.chosenAgentId IS NULL**`. The `WHERE` guard is the whole safety mechanism: it makes the update a no-op whenever the existing row already has a real agent on it. **If the write affected 0 rows, this computation lost the race** — discard it, re-read the row, and return the (successful) decision that's actually stored, so this caller doesn't report failure when the ticket was, in fact, assigned. Don't fall through to any agent/ticket side effects in that case.
6. Otherwise compute `ratio = activeTicketCount / effectiveLimit` per eligible agent; pick the lowest. Break ties by `lastAssignedAt` ascending (an agent never assigned before sorts first), then by `agentId` ascending as a final deterministic tiebreaker (assumption 13: same situation → same decision).
7. Attempt the **same guarded upsert** as step 5, but with the chosen `chosenAgentId` (still `ON CONFLICT (companyId, ticketId) DO UPDATE ... WHERE AssignmentDecision.chosenAgentId IS NULL`). Only if this write actually affects a row — meaning this computation is the one that legitimately wins — do the side effects: set `Ticket.assignedAgentId`/`assignedAt` and bump the chosen agent's `lastAssignedAt`, all inside the same transaction as the guarded write. **If the write affected 0 rows** (a concurrent call's success already committed first), discard this result entirely — no ticket/agent side effects — re-read, and return the already-committed decision instead. From the moment a guarded write actually lands with a non-null `chosenAgentId`, the decision is permanent (step 1 will short-circuit on it for every future call).

This gives the exact merge behavior needed under a race: a success is only ever installed by the write that gets there first, a later failing computation can never overwrite it (guard blocks it), and a later *successful* computation can still legitimately overwrite an earlier *failed* attempt (guard passes, since the stored `chosenAgentId` was null) — which is the retriability §3 step 1 depends on. Two concurrent calls for a brand-new ticket may both pay the cost of computing eligibility, but only one of their writes ever lands, and both callers converge on returning that same single decision.

---

## 4. Coverage gap computation

**Why this can't just be a weekly schedule.** Say the company is based in Phoenix (Arizona doesn't observe DST) and requires coverage 9am–5pm Phoenix time. An agent lives in New York (which does observe DST) and works a fixed 9am–5pm New York shift. In summer, New York is 3 hours ahead of Phoenix, so that shift actually covers 12pm–8pm Phoenix time. In winter, it's only 2 hours ahead, covering 11am–7pm Phoenix time instead. The same recurring local shift lines up with different Phoenix hours depending on the time of year, because the two cities don't change their clocks the same way. So there's no single weekly pattern that stays correct year-round — the coverage report has to be computed for a specific date range (default: the next 7 days), not a fixed recurring week.

**How it's computed:**

1. Work out which real minutes in the range actually need coverage: for each real minute, convert it to the company's own local time and check it against the company's weekly coverage requirements. This only ever converts a real, known instant into local time — never the reverse — so it's unambiguous. Each matching minute is tagged with the `minAgents` it requires. (A requirement covering the full week still only needs its own local weekday/time boundaries checked this way, not the whole calendar re-derived some other way.)
2. For those minutes only — not every minute in the range — check how many agents are on shift, reusing the exact same "is this agent on shift right now" check from §3 step 4. Same safe direction: real instant → agent's local time, never the reverse.
3. Count distinct agents on shift at each required minute. An agent with two overlapping shift entries still counts once — this is a yes/no check per agent, not an interval count, so double-counting one agent can't happen. Any required minute where the count is below `minAgents` is a gap.
4. Merge consecutive gap-minutes into reported ranges, with each boundary shown as a real timestamp with its UTC offset, so a time near a DST change is never ambiguous to read.

Because shifts and requirements are only ever defined to the minute, checking real minutes one at a time is exact, not an approximation — and only ever converting real-instant-to-local-time (never local-to-real-instant) is what makes DST a non-issue here: a repeated local hour (clocks falling back) is two different real minutes, both checked; a skipped local hour (clocks springing forward) simply never occurs as a real minute, so there's nothing to miss. Restricting the check to minutes that actually need coverage (step 1) also keeps the cost proportional to how much coverage is required, not the length of the whole date range.

---

## 5. Main UI flow

1. **Agents & availability.** List of agents; per agent, edit timezone, active on/off toggle, ticket limit override, and weekly shift blocks (add/remove per weekday, including overnight shifts).
2. **Coverage.** Define weekly required-coverage windows + minimum agent count; view the computed gap report for the next 7 days, highlighting under-staffed ranges.
3. **Workload settings.** Edit the company-wide default ticket limit.
4. **Explanation lookup.** Enter a ticket id, see who it was assigned to (or why nobody was eligible) and the per-agent reasoning breakdown.

No login/roles — anyone with the URL can do all of the above (prd.md, out of scope).

---

## 6. Edge cases

Beyond prd.md §4 "Behaviour at the edges" (15–17), implementation-level cases:

- **Concurrent assign calls for the same new ticket.** Both may independently compute eligibility (the step-1 read isn't guarded), but only one write ever lands — the guarded upsert in §3 steps 5/7 makes the loser discard its result and return whatever actually got committed, instead of racing to persist two different outcomes.
- **A concurrent success and a concurrent "no eligible agent" for the same ticket.** The success can never be clobbered by the failure, regardless of which one's computation happens to finish first — the guarded upsert (`WHERE chosenAgentId IS NULL`) only lets the failure's write land if nothing has succeeded yet. If the failure's write happens to commit first, a later success can still legitimately overwrite it.
- **Retrying `assign` after a previous "no eligible agent" result.** Re-evaluated fresh every time (§3 step 1) — capacity freeing up or a shift starting will change the outcome. Only a successful assignment is frozen; a failed attempt's `AssignmentDecision` row is overwritten in place on each retry, so explanation lookup always shows the most recent attempt.
- **DST transitions, both in eligibility checks and coverage-gap computation.** Both only ever convert a real instant to local time (never the reverse), which is always unambiguous — §4 reuses the exact §3 on-shift check for this reason. A fall-back night's repeated local hour is naturally checked twice (once per real occurrence); a spring-forward night's skipped local hour is naturally never checked at all. No disambiguation policy is needed because the ambiguous direction (local→real-instant) is never used.
- **An agent with two overlapping shift entries.** Counted once toward `minAgents`, never twice — coverage counting is a yes/no check per agent per minute (§4 step 3), not an interval count, so overlapping entries can't inflate the count.
- **Agent with no configured shifts.** Always off-shift; not an error, just never eligible.
- **Ticket limit of 0.** Agent is always excluded before ratio math (avoids a 0/0 ratio).
- **Status update for a ticket that doesn't exist yet** (out-of-order delivery). No-op per assumption 17; not persisted as a pending fact, since assignment always precedes resolution in the intended flow.
- **Agent config changes after a decision was made.** Past **successful** `AssignmentDecision` rows are immutable snapshots; editing an agent's timezone/shifts/limit later never rewrites a frozen decision. (An unresolved *failed* attempt is, by design, re-evaluated against current config on its next retry.)

---

## 7. Test plan

**Unit — eligibility & scoring**
- On-shift check: within a shift, outside all shifts, exactly at a boundary, a shift that wraps past midnight, a shift evaluated right across a DST transition.
- Limit check: under limit, at limit, over limit, limit of 0, agent-level override vs. company default.
- Ratio + tie-break: distinct ratios pick the lowest; equal ratios fall back to `lastAssignedAt`; fully-tied agents fall back to `agentId` (rerun the same inputs twice, assert identical output — assumption 13).
- Coverage gap expansion: a requirement fully covered, partially covered, uncovered, and one that wraps past midnight.
- Coverage double-counting: one agent with two overlapping shift entries against a `minAgents: 2` requirement still reports a gap (must not be satisfied by one person).
- Coverage DST expansion: a shift covering a fall-back night's repeated local hour is reported as covered during **both** real UTC occurrences of that hour, not just one; a shift referencing a spring-forward night's skipped local hour contributes no coverage for that (nonexistent) window and doesn't error or produce a negative-length interval.

**Integration — API**
- Assign: happy path; no eligible agent (all three exclusion reasons represented); repeated call for the same ticket returns the identical stored decision.
- Assign retry after failure: ticket gets "no eligible agent," then an agent's capacity frees up (or a shift starts) before the next `assign` call for the same ticket — the retry now succeeds, not another frozen "no eligible agent."
- Same `ticket_id` used by two different companies: each company's `assign`/`status` calls are independent and never collide.
- Status: unknown ticket → no-op response; non-terminal status → no change in agent load; terminal status → agent's active count drops by one; status update after resolution → no-op (finality).
- Explanation lookup returns exactly what `assign` returned.
- Fire two concurrent `assign` requests for the same new `ticket_id`; assert exactly one `AssignmentDecision` row is created and both responses match it.
- Force one concurrent computation to succeed and the other to fail for the same new `ticket_id` (e.g. stub eligibility so one call sees the agent as in-limit and the other sees it as at-limit): assert the stored decision is always the successful one regardless of which call's write reaches the DB first, and that both callers' responses reflect that success, not a null result.

**Scenario — mirrors PRD success criteria**
- A company where every agent is at their limit → new ticket gets "no eligible agent," reason lists everyone as at-limit.
- A coverage requirement with no agent shifts overlapping it → shows up as a full gap.
- Repeated ties across several tickets → assignments rotate rather than always landing on the same agent.
