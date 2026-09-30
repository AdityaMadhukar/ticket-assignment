# Implementation: Automatic Support Ticket Assignment

## 0. Stack

- **Runtime:** Node.js + TypeScript
- **Web framework:** Express
- **Storage:** SQLite via Prisma (single file DB, no server to run — matches "deployment and hosting is out of scope, service runs locally")
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
  id           String    @id            // the ticket_id from the ticketing system
  companyId    String
  assignedAgentId String?
  assignedAt   DateTime?
  resolvedAt   DateTime?                // null = still active/unresolved
  lastRawStatus String?                 // last status string received, for audit only

  company Company @relation(fields: [companyId], references: [id])
  agent   Agent?  @relation(fields: [assignedAgentId], references: [id])
  decision AssignmentDecision?

  @@unique([companyId, id])
}

model AssignmentDecision {
  id              String   @id @default(cuid())
  ticketId        String   @unique
  companyId       String
  chosenAgentId   String?             // null = no eligible agent
  reason          String              // human-readable summary
  consideredAgents Json                // per-agent: eligible?, load ratio, exclusion reason
  decidedAt       DateTime @default(now())

  ticket Ticket @relation(fields: [ticketId], references: [id])
}
```

Notes:
- An agent's **active (unresolved) ticket count** is `count(Ticket where assignedAgentId = agent.id and resolvedAt is null)` — no separate counter to keep in sync, no time window (matches prd.md assumption 11).
- `AssignmentDecision` is what makes assignment **idempotent** (assumption 16) and **explainable** (scope item "Explanation") — the same row answers both the ticketing system's retry and the team lead's later lookup.
- `Ticket` and `AssignmentDecision` are separate tables so a "no eligible agent" outcome can still be recorded and looked up, even though no agent is assigned.

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

1. Look up an existing `AssignmentDecision` for `(companyId, ticketId)`. If found, return it as-is (idempotent — assumption 16).
2. Load all **active** agents for the company.
3. Filter to agents **on shift right now**: convert the current instant to each agent's local weekday + minute-of-day (via the IANA tz database, which handles DST automatically), and check it against that agent's shifts, treating `endMinute <= startMinute` as wrapping into the next weekday.
4. Filter to agents **under limit**: `activeTicketCount(agent) < (agent.ticketLimit ?? company.defaultTicketLimit)`. A limit of `0` always excludes the agent.
5. If no agents remain: create the `Ticket` row (if not already present) and an `AssignmentDecision` with `chosenAgentId = null` and a reason listing why each agent was excluded. Return it.
6. Otherwise compute `ratio = activeTicketCount / effectiveLimit` per eligible agent; pick the lowest. Break ties by `lastAssignedAt` ascending (an agent never assigned before sorts first), then by `agentId` ascending as a final deterministic tiebreaker (assumption 13: same situation → same decision).
7. Create/update the `Ticket` row (`assignedAgentId`, `assignedAt`), bump the chosen agent's `lastAssignedAt`, and write the `AssignmentDecision`. Return it.

---

## 4. Coverage gap computation

Coverage requirements are defined in the **company's** timezone; agent shifts are defined in each **agent's own** timezone (prd.md assumptions 7 & 10). A weekly recurring local time doesn't convert to a fixed offset between two timezones year-round (DST transitions don't line up), so there's no single canonical "weekly grid" that's valid indefinitely.

**Design decision:** gaps are computed over a concrete date range (default: the next 7 calendar days), not an abstract recurring week:
1. Expand every active agent's weekly shifts into concrete UTC intervals for each date in the range (applying that date's actual DST offset in the agent's zone).
2. Expand the company's weekly coverage requirements into concrete UTC intervals the same way, in the company's zone.
3. For each requirement interval, count how many expanded agent-shift intervals overlap it at each point; any sub-interval where that count is below `minAgents` is reported as a gap, with its start/end in the company's local time.

This is a real design tradeoff worth flagging for review: it means "coverage gaps" is always a report over a specific window, not a single static weekly picture — the UI should present it as a date-ranged report (e.g., "gaps in the next 7 days") rather than a fixed weekly calendar.

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

- **Concurrent assign calls for the same new ticket.** A unique constraint on `Ticket.id` plus the transaction in §3 means only one request wins the insert; the other reads back the same persisted decision instead of racing to pick a different agent.
- **Shift or coverage window crossing midnight.** Handled uniformly by the `endMinute <= startMinute` wrap convention in both `AvailabilityShift` and `CoverageRequirement`.
- **DST transitions.** Never computed as a fixed offset — "is this agent on shift" always asks "what's this agent's local wall-clock time right now," so DST is handled by the tz database, not by application logic.
- **Agent with no configured shifts.** Always off-shift; not an error, just never eligible.
- **Ticket limit of 0.** Agent is always excluded before ratio math (avoids a 0/0 ratio).
- **Status update for a ticket that doesn't exist yet** (out-of-order delivery). No-op per assumption 17; not persisted as a pending fact, since assignment always precedes resolution in the intended flow.
- **Agent config changes after a decision was made.** Past `AssignmentDecision` rows are immutable snapshots; editing an agent's timezone/shifts/limit later never rewrites history.

---

## 7. Test plan

**Unit — eligibility & scoring**
- On-shift check: within a shift, outside all shifts, exactly at a boundary, a shift that wraps past midnight, a shift evaluated right across a DST transition.
- Limit check: under limit, at limit, over limit, limit of 0, agent-level override vs. company default.
- Ratio + tie-break: distinct ratios pick the lowest; equal ratios fall back to `lastAssignedAt`; fully-tied agents fall back to `agentId` (rerun the same inputs twice, assert identical output — assumption 13).
- Coverage gap expansion: a requirement fully covered, partially covered, uncovered, and one that wraps past midnight.

**Integration — API**
- Assign: happy path; no eligible agent (all three exclusion reasons represented); repeated call for the same ticket returns the identical stored decision.
- Status: unknown ticket → no-op response; non-terminal status → no change in agent load; terminal status → agent's active count drops by one; status update after resolution → no-op (finality).
- Explanation lookup returns exactly what `assign` returned.
- Fire two concurrent `assign` requests for the same new `ticket_id`; assert exactly one `AssignmentDecision` row is created and both responses match it.

**Scenario — mirrors PRD success criteria**
- A company where every agent is at their limit → new ticket gets "no eligible agent," reason lists everyone as at-limit.
- A coverage requirement with no agent shifts overlapping it → shows up as a full gap.
- Repeated ties across several tickets → assignments rotate rather than always landing on the same agent.
