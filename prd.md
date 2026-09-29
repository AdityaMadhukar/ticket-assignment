# PRD: Automatic Support Ticket Assignment

**Status:** Draft for review · **Stage:** 1 of 3 · **Author:** Aditya Madhukar

---

## 1. Problem

Support teams have agents working different hours, days and timezones: some cover mornings, some afternoons, some weekends. Today a team lead watches the incoming queue and hand-assigns each new ticket to whoever they believe is working. This breaks down as the team grows:

- **Tickets go stale.** Anything that arrives while the lead is offline sits unassigned for hours.
- **The lead is a bottleneck.** Triage takes up the lead's day instead of their own work.
- **Work is uneven.** Some agents get buried while others sit idle, because nobody can track everyone's load in their head.

Customers need new tickets to be assigned automatically, as soon as they arrive, to someone who:

- is on that company's team,
- is actually working at that moment, and
- isn't already overloaded.

The work should also be spread fairly across the team.

Three customer expectations define success:

1. **Coverage is visible.** A team lead can see when the team's availability does not cover the times that need coverage.
2. **No overloading.** Agents stop receiving new work once they already have too much active work.
3. **Decisions are explainable.** A team lead can understand why a ticket went to a particular person, or why it went to nobody.

## 2. Target users

| User | Role | What they need |
|---|---|---|
| **Team lead / support ops admin** | Primary user of the UI | Set and maintain when each agent works and in which timezone, spot gaps in coverage, set workload limits, and understand past assignment decisions. |
| **Ticketing system** | Caller of the API | For each new ticket, get back one agent to assign, or a clear "no eligible agent", quickly and consistently. |
| **Support agent** | Indirect beneficiary | Get work only while on shift, and never be piled on past a reasonable limit. Agents don't use the product directly. |

## 3. Scope

### In scope
- **Availability management (UI).** A team lead defines each agent's timezone and recurring weekly working hours for their company, and keeps them up to date.
- **Coverage visibility (UI).** A team lead defines the hours their company needs covered and can see where the team's schedule leaves those hours unstaffed or under-staffed.
- **Workload limits.** A team lead can set how much active work an agent may hold before they stop receiving new tickets.
- **Assignment (API).** Given a `company_id` and `ticket_id`, return who on that company's team should get the ticket, or indicate that no eligible agent is available.
- **Explanation.** Each assignment result says why that agent was chosen, and why others were not. A team lead can look this up later.

### Out of scope
- Login, roles and permissions: whoever uses the UI may do everything.
- Billing and account management.
- Creating or managing companies, agents and tickets. These are assumed to exist.
- Mobile apps.
- Holiday calendars and one-off overrides (PTO, sick days, shift swaps).
- Third-party on-call integrations (PagerDuty, Opsgenie, etc.).
- Skills-, language- or priority-based routing.
- Re-assigning tickets that are already assigned, and queueing tickets for later assignment.
- Deployment and hosting. The service runs locally.

## 4. Assumptions

**Data we are given**
1. Companies and agents already exist with stable ids. We seed sample data and do not build flows to create them.
2. Each agent belongs to exactly one company's team.
3. A `ticket_id` is an opaque identifier from the ticketing system, unique within a company. We receive no ticket content, priority or status.
4. The ticketing system calls the API once when a ticket arrives. The assignment is decided for "now", the moment of the call.
5. Any agent on a company's team can handle any of that company's tickets.

**What "available" means**

6. An agent's availability repeats weekly. It is a set of working hours per weekday, expressed in the agent's own local time and timezone. Local working hours stay the same across daylight-saving changes.
7. An agent can have several shifts in a day, and a shift can run past midnight.
8. An agent can be switched off entirely (e.g. left the team or on long leave). This is a standing on/off switch, not a dated override.
9. A company needs coverage for specific weekly hours in its own timezone (possibly 24/7), with a minimum number of agents on shift for each period.

**What "too much active work" means**

10. We do not learn when tickets are resolved. So an agent's active work is approximated as *the tickets we assigned to them within a recent time window* (default 8 hours, set per company). This is a deliberate simplification: a ticket that is still genuinely open after the window falls out of the count, so that agent can receive new tickets while it remains unresolved. We accept this blind spot rather than track resolution.
11. Each agent has a limit on active tickets. There is a company-wide default, which can be changed for individual agents. At the limit, they receive no new tickets.

**What "fair" means**

12. Fair means balancing current workload relative to each agent's limit, not ticket count over all time. Concretely, each eligible agent's load is `active tickets ÷ limit`; the ticket goes to whoever has the lowest ratio. When ratios are tied, the work rotates so the same person isn't always picked first.
13. The same situation always produces the same decision, so outcomes can be explained and reproduced.

**Behaviour at the edges**

14. If no one on the team is eligible, the API says so and explains why. The ticketing system leaves the ticket unassigned and may try again later.
15. Asking again about a ticket that was already assigned returns the original assignment rather than picking someone new.