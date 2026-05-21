# Room Allocation Agent by Apaleo

A room-assignment system for hotels running on the [Apaleo](https://apaleo.com) PMS. It reads tomorrow's (or today's) arrivals, asks an LLM to interpret each guest's free-text comment, then runs a Simulated Annealing optimiser to assign every reservation to a physical room while balancing guest requests, adjacency, upgrades, operational constraints, and inventory pressure.

Configuration is fully sheet-driven. A single Google Sheet controls feature on/off flags, priority ordering, the room-type hierarchy, loyalty tiers, property-specific phrase dictionaries, and the choice of property and day. The same workflow runs against multiple properties with no code changes.

---

## What it does

For every reservation arriving on the target day, the agent decides:

- Which physical room to assign (clean preferred, dirty acceptable, occupied as last resort)
- Whether an upgrade is appropriate (and to which category)
- How to honour adjacency requests (forced and optional)
- Whether the reservation is a continuation of a previous stay that must return to the same room (Anschlussbuchung / ASB)
- Whether the room should be force-rotated for legionella prevention

It then emails the front office a per-reservation breakdown with the room chosen, the score components that justified the choice, and a red banner for any reservation it couldn't fully satisfy.

---

## How it works

```
Triggers ──► Google Sheets (6 tabs) ──► BuildConfig ──► Apaleo (rooms + reservations + maintenance)
                                            │
                                            ▼
                                       AI Agent (LLM)
                                       interprets comments
                                            │
                                            ▼
                                Pre-assignment passes
                            (Legionella + ASB hard-binds)
                                            │
                                            ▼
                                   Simulated Annealing
                                  20,000 iterations,
                                exponential cooling
                                            │
                                            ▼
                              HTML report → Outlook email
                              Auto-assign rooms → Apaleo
```

### 1. Sheet-driven configuration
Six tabs feed a single `BuildConfig` node:
- **Setup** — feature flags (Y/N), priority ranks (1 = top), legionella threshold, optional-upgrade switch, property ID, day flag.
- **RoomRanks** — every room type's rank (1 = lowest category) and protected-suite flag.
- **SOPExamples** — positive/negative comment interpretation rules.
- **CommentGlossary** — property-specific phrase dictionary (e.g. `ASB`, `HF`, `LF`).
- **Memberships** — loyalty tiers, trigger phrases, the Upgrade action and mode (FORCED/OPTIONAL) each triggers.
- **Actions** — reference catalogue of every action the AI can emit.

`BuildConfig` reads all six tabs, applies the priority multiplier (`1 / priority^0.7`) to every baseline weight and cap, and emits a single config object consumed by every downstream node.

### 2. LLM comment interpretation
The AI agent receives the merged config and the day's reservations. For each, it produces three structured fields:
- `matchedRequests` — attributes the guest explicitly asked for
- `inferredMatchedRequests` — attributes the AI inferred from softer cues (must be a strong semantic link to a real attribute)
- `allowedActions` — structured operational signals (`EarlyCheckIn`, `ASB:<id>`, `FloorPreference: high|low`, `Upgrade 1..4`, `Adjacency:<reservationId>`)

The prompt is fully data-driven: glossary, SOP examples, memberships table, and the actions catalogue are all rendered from sheet data at runtime, so adding a new loyalty tier or a new property-specific phrase requires no prompt edits.

### 3. Pre-assignment passes (hard binds, run BEFORE the optimiser)
- **Legionella** — rooms vacant for at least `legionella_max_vacancy_days` (default 3) since their last checkout are bound to a feasible same-type reservation, ensuring water systems get flushed.
- **ASB (Anschlussbuchung)** — when a comment contains `ASB:<identifier>`, the identifier is resolved as a room number, a reservation ID, or a booking ID, in that order. A successful resolution hard-binds the reservation to that room. Failures are surfaced in the report as "Unmet ASBs" for manual review.

Pre-assigned reservations and their rooms exit the SA pool — the optimiser cannot move them.

### 4. Simulated Annealing optimiser
20,000 iterations, exponential cooling from temperature 200 down to 0.3. Each iteration proposes one of three moves:
- Swap (55%): exchange two reservations' rooms
- Reassign (30%): move one reservation to a different room (sampled from its top 5 attribute-matching candidates)
- Three-cycle (15%): rotate three reservations through each other's rooms

Better moves are always accepted; worse moves are accepted with a probability that decays as the temperature drops. The best assignment found across all iterations is kept.

### 5. Reporting
`Reporting_Prep` renders an HTML email with a summary bar (total score, assigned/unassigned/pre-assigned/unmet-ASB counts) and per-reservation cards showing what the AI matched, what the optimiser chose, and a full score breakdown.

---

## Scoring rulebook

Every soft-constraint weight is the baseline value multiplied by `1 / priority^0.7`, where priority is set on the Setup sheet. Disabled features (flag = N) contribute zero.

| Category | Baseline weight |
|---|---|
| Critical attribute satisfied / unsatisfied | +240 / −420, cap −1,200 |
| Preference attribute satisfied / unsatisfied | +90 / −140, cap −600 |
| Forced adjacency by distance | +320 / +210 / +80 / −280, cap ±1,600 |
| Optional adjacency by distance | +120 / +80 / +30 / −60, cap ±600 |
| Booking-group proximity | +140 / +110 / +70 / +30 / −40, cap ±1,600 |
| Upgrade quality (good / neutral / bad) | +80 / +10 / −120, cap ±200 |
| Room state (clean / inspect / dirty / tomorrow / occupied) | +80 / +20 / −120 / −260 / −350 |
| Floor preference (per step toward preferred direction) | ±15 |
| ECI dirty/occupied extra penalty | −400 |
| Legionella assignment bonus | +800 |
| All-forced-adjacency satisfied | +500 |
| Booking group fully clustered (distance ≤ 2) | +220 per booking |
| Unassigned reservation | −250,000 |
| Wrong room type (hard constraint) | −1,000,000 |

Upgrade quality is classified at runtime from live inventory:
- **Good** (free rooms in target category exceed demand by more than 3) → reward
- **Neutral** (surplus 0–3) → small reward
- **Bad** (target category oversold) → penalty

FORCED upgrades (e.g. Platinum, Diamond) bump quality one tier up (bad → neutral, neutral → good) and unlock access to protected suite ranks. OPTIONAL upgrades (e.g. Gold) get half weight and are blocked from suite ranks.

---

## Repository structure

```
.
├── README.md                                               This file
│
├── 00_n8n_workflow/
│   └── RoomAssignmentAgent_v1.json                         Full n8n workflow export (v1)
│
├── 01_Google_Sheets_Config/
│   └── RoomAllocationAgent_Setup_Productized-2.xlsx        Ready-to-use Setup spreadsheet template
│
└── 02_Proof/
    └── RoomAllocationAgent_Proof_against_greedy_solution.pdf  Optimality proof vs greedy baseline
```

---

## Setup

### Prerequisites
- An [Apaleo](https://apaleo.com) account with API access (Client Credentials grant)
- An [n8n](https://n8n.io) instance (cloud or self-hosted)
- A Google account for the Setup spreadsheet
- An [OpenRouter](https://openrouter.ai) API key (or compatible LLM provider)
- A Microsoft Outlook account for report delivery (or swap for any email node)

### One-time setup

1. **Apaleo** — register an integration in your Apaleo developer account. Note the client ID and secret. Make sure the integration has read access to reservations, bookings, units, and maintenance, plus write access for unit assignment and reservation patch.

2. **Google Sheets** — upload `01_Google_Sheets_Config/RoomAllocationAgent_Setup_Productized-2.xlsx` to Google Drive and open it as a Google Sheet. It already contains all six tabs (Setup, RoomRanks, SOPExamples, CommentGlossary, Memberships, Actions) with the expected column headers and example rows. Fill in your property's rows on `RoomRanks` (room type IDs, ranks, protected-suite flags) and adapt `CommentGlossary`/`SOPExamples` for your property's language.

3. **n8n** — import `00_n8n_workflow/RoomAssignmentAgent_v1.json`. You'll be prompted to assign credentials for:
   - Google Sheets (one credential, reused on all six setup nodes)
   - Apaleo OAuth2 (one credential)
   - OpenRouter (or your LLM provider of choice)
   - Microsoft Outlook (or substitute your email node)
   - HTTP Basic / Bearer for the Apaleo direct calls used for room assignment and reservation patches

4. **Repoint the Google Sheets nodes** to your copy of the Setup workbook. The six nodes are `Setup_Setup`, `Setup_RoomRanks`, `Setup_SOPExamples`, `Setup_Glossary`, `Setup_Memberships`, `Setup_Actions`.

5. **Configure the Setup tab**:
   - Set `property_id` to your Apaleo property code.
   - Set `assign_for_day` to `today` or `tomorrow`.
   - Set `legionella_max_vacancy_days` to your policy threshold (default 3).
   - Set `optional_upgrades_allowed` to `Y` or `N`.
   - Set each feature's `Enabled` flag and `Priority` rank. Priority 1 is most important; equal priorities mean equal weight.

6. **Test run** — trigger the workflow manually. Inspect `BuildConfig`'s output and check the `_debug.bucketCounts` field — every bucket should be non-zero. If any are zero, the corresponding sheet read didn't classify correctly.

### Adding a second property

1. Duplicate the Google Sheet.
2. Change `property_id` to the new property code.
3. Replace the rows in `RoomRanks` with the new property's room types.
4. Add property-specific phrases to `CommentGlossary` and `SOPExamples` as needed.
5. Duplicate the workflow in n8n and repoint its six Google Sheets nodes to the new sheet.

No code changes required.

---

## Optimality proof

`02_Proof/RoomAllocationAgent_Proof_against_greedy_solution.pdf` documents the formal comparison between the Simulated Annealing optimiser and a greedy room-assignment baseline, demonstrating that SA consistently produces superior assignments across a range of property sizes and arrival volumes.

---

## Known limitations

Things to be aware of when interpreting outputs or planning changes:

- **Upgrade quality is a snapshot.** The good/neutral/bad classification uses the inventory state at the moment the run starts. A "good upgrade" earlier in the day may become a "bad upgrade" later as bookings arrive. The optimiser does not look ahead. For more compliated upgrade logics we recommend a sperate agent. 
- **Adjacency relies on room numbering.** Two rooms are considered neighbours only if their numeric names differ by 1 on the same floor block. Named rooms (e.g. "Presidential Suite") have no neighbours in the graph, so any forced adjacency involving them scores at the distance 3+ penalty regardless of physical layout. A future improvement would be an explicit floor-map input.
- **Unassigned penalty dominates.** A reservation with a FORCED adjacency requirement may end up far from its required neighbour (−280 penalty) rather than unassigned (−250,000), and the optimiser will accept that trade. If adjacency is truly non-negotiable for some reservation pairs, they need their own hard constraint, not a soft gradient.
- **Global bonuses are tiebreakers.** With dozens of reservations each scoring in the hundreds, the +500 all-forced-adjacency bonus and the +220 per-booking cluster bonus rarely flip borderline decisions on their own. They function as tiebreakers rather than dominant signals.
- **Upsell action is Reserved for future revenue-aware logic.

---

## License
Apache 2.0

## Contributing

PRs welcome for: alternative scoring weights, additional pre-assignment passes (e.g. preventive maintenance scheduling), floor-map adjacency input, and look-ahead upgrade classification.
