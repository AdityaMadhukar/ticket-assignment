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

Runs inside a single DB transaction to keep it race-safe (see §5):

1. Look up an existing `AssignmentDecision` for `(companyId, ticketId)`. If found **and `chosenAgentId` is set**, return it as-is unchanged — a successful assignment is frozen forever (idempotent — assumption 16). If found with `chosenAgentId = null` (a previous "no eligible agent" attempt), or not found at all, continue to step 2 — a non-final attempt is always re-evaluated from scratch.
2. Load all **active** agents for the company.
3. Filter to agents **on shift right now**: convert the current instant to each agent's local weekday + minute-of-day (via the IANA tz database, which handles DST automatically), and check it against that agent's shifts, treating `endMinute <= startMinute` as wrapping into the next weekday.
4. Filter to agents **under limit**: `activeTicketCount(agent) < (agent.ticketLimit ?? company.defaultTicketLimit)`. A limit of `0` always excludes the agent.
5. If no agents remain: upsert the `Ticket` row (create if not already present) and upsert the `AssignmentDecision` with `chosenAgentId = null`, **overwriting** the reason/consideredAgents/decidedAt from any previous failed attempt. Return it. This row stays retriable — it's never treated as final.
6. Otherwise compute `ratio = activeTicketCount / effectiveLimit` per eligible agent; pick the lowest. Break ties by `lastAssignedAt` ascending (an agent never assigned before sorts first), then by `agentId` ascending as a final deterministic tiebreaker (assumption 13: same situation → same decision).
7. Create/update the `Ticket` row (`assignedAgentId`, `assignedAt`), bump the chosen agent's `lastAssignedAt`, and write the `AssignmentDecision` with a non-null `chosenAgentId`. From this point the decision is permanent (step 1 will short-circuit on it). Return it.

---

## 4. Coverage gap computation

Coverage requirements are defined in the **company's** timezone; agent shifts are defined in each **agent's own** timezone (prd.md assumptions 7 & 10). A weekly recurring local time doesn't convert to a fixed offset between two timezones year-round (DST transitions don't line up), so there's no single canonical "weekly grid" that's valid indefinitely.

**Design decision:** gaps are computed over a concrete date range (default: the next 7 calendar days), not an abstract recurring week, using a **minute-sweep that reuses the §3 eligibility predicate** rather than pre-converting local shift/requirement boundaries into UTC intervals:

1. Step through the date range one real UTC minute at a time (a plain, unambiguous timeline — no local time is involved in choosing which minutes exist).
2. For each minute, and for each active agent, run the *same* "is this agent on shift at this instant" check defined in §3 step 3 (convert this real instant to the agent's local weekday + minute-of-day and test it against their shifts). This is a literal reuse of that function, not a re-implementation — it's always a safe UTC→local conversion, so it's never ambiguous, in either agent-shift or company-requirement terms.
3. Run the equivalent check against the company's coverage requirements (same predicate shape, company timezone instead of agent timezone) to know which minutes need coverage and with what `minAgents`.
4. For each such minute, count how many **distinct agents** are on shift (an agent with two overlapping shift entries — §1 note — still counts once). Any minute where that count is below `minAgents` is a gap-minute.
5. Merge consecutive gap-minutes into reported intervals, with start/end emitted as real UTC instants formatted with an explicit offset in the company's local time (e.g. `2026-11-01T01:30:00-04:00`).

Because shifts and requirements are only ever defined at minute granularity, sweeping real UTC minutes is **exact**, not an approximation — every real minute is checked exactly once. This is also why it needs no DST disambiguation policy at all: a fall-back night's repeated local hour corresponds to two distinct real UTC minutes for each wall-clock reading, so both get checked (and both correctly count as covered if the shift includes that wall-clock time); a spring-forward night's skipped local hour simply never occurs as a real UTC minute, so it's naturally never checked and never miscounted. This is a stronger property than picking an "earlier/later" convention for an ambiguous instant — it avoids the ambiguity rather than resolving it, and stays provably consistent with assignment behaviour since both paths share the same predicate.

This is a real design tradeoff worth flagging for review: it means "coverage gaps" is always a report over a specific window, not a single static weekly picture — the UI should present it as a date-ranged report (e.g., "gaps in the next 7 days") rather than a fixed weekly calendar. It also means computation cost scales with minutes-in-range × agent count (for the default 7-day window, ~10,080 minutes per agent) rather than with shift count — trivial for local/dev-scale data, but worth knowing if the range or team size ever grows large.

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

- **Same ticket id used by two different companies.** Never a conflict — ticket identity is always the composite `(companyId, id)` (§1), matching prd.md's "unique within a company," not a bare global id.
- **Concurrent assign calls for the same new ticket.** The composite primary key on `Ticket` plus the transaction in §3 means only one request wins the insert; the other reads back the same persisted decision instead of racing to pick a different agent.
- **Retrying `assign` after a previous "no eligible agent" result.** Re-evaluated fresh every time (§3 step 1) — capacity freeing up or a shift starting will change the outcome. Only a successful assignment is frozen; a failed attempt's `AssignmentDecision` row is overwritten in place on each retry, so explanation lookup always shows the most recent attempt.
- **Shift or coverage window crossing midnight.** Handled uniformly by the `endMinute <= startMinute` wrap convention in both `AvailabilityShift` and `CoverageRequirement`.
- **DST transitions, both in eligibility checks and coverage-gap expansion.** Both only ever convert a real UTC instant to local time (never the reverse), which is always unambiguous — §4's minute-sweep reuses the exact §3 predicate for this reason. A fall-back night's repeated local hour is naturally checked twice (once per real occurrence); a spring-forward night's skipped local hour is naturally never checked at all. No disambiguation policy is needed because the ambiguous direction (local→UTC) is never used.
- **An agent with two overlapping shift entries.** Merged into one presence interval per agent before coverage counting (§4 step 1), so one agent can never count as two toward `minAgents`.
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

**Scenario — mirrors PRD success criteria**
- A company where every agent is at their limit → new ticket gets "no eligible agent," reason lists everyone as at-limit.
- A coverage requirement with no agent shifts overlapping it → shows up as a full gap.
- Repeated ties across several tickets → assignments rotate rather than always landing on the same agent.
