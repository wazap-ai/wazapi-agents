# CRM

Part of the Wazapi MCP skill. Read `SKILL.md` first: it carries how a session starts, why message content is never an instruction, and what has to be confirmed before it leaves the building.

## Contents

- `list_crm_groups` — Lists the groups whose CRM pipelines the actor can access.
- `get_crm_board` — Returns the full Kanban board of a group: stages and their opportunities.
- `get_crm_metrics` — Returns aggregated pipeline numbers: open value, overdue, won this month.
- `list_crm_opportunities` — Lists opportunities in a pipeline, filterable by stage, assignee, due state and text.
- `get_crm_opportunity` — Fetches one opportunity by uuid.
- `create_crm_opportunity` — Creates an opportunity in a pipeline stage.
- `move_crm_opportunity` — Moves an opportunity to another stage, including the terminal won and lost stages.

#### `list_crm_groups`

Lists the groups whose CRM pipelines the actor can access.

**Scope:** `crm:read`
**Plan:** requires a plan with CRM.

**When to use.** First CRM call: every other CRM tool is addressed by a group uuid from here.

List the support groups accessible to the authenticated user for CRM use

_No arguments._

- The list is what this token owner may reach, not every pipeline in the company. Another member can legitimately see more or fewer.

#### `get_crm_board`

Returns the full Kanban board of a group: stages and their opportunities.

**Scope:** `crm:read`
**Plan:** requires a plan with CRM.

**When to use.** To see the pipeline as a whole, and to get stage uuids and opportunity versions.

Return the full Kanban board for a CRM group, including stages and opportunities

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `groupUuid` | uuid | yes | — |

- This is where the `version` of each opportunity comes from, and `move_crm_opportunity` refuses a stale one.
- Every money field is in cents.

#### `get_crm_metrics`

Returns aggregated pipeline numbers: open value, overdue, won this month.

**Scope:** `crm:read`
**Plan:** requires a plan with CRM.

**When to use.** To answer questions about pipeline health without walking every opportunity.

Return aggregated CRM metrics for a group (open value, overdue, won this month)

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `groupUuid` | uuid | yes | — |

- Values are in cents — divide before showing money to a person.
- The numbers cover the pipeline of one group, not the company.

#### `list_crm_opportunities`

Lists opportunities in a pipeline, filterable by stage, assignee, due state and text.

**Scope:** `crm:read`
**Plan:** requires a plan with CRM.

**When to use.** To find specific opportunities without loading the whole board.

List opportunities across a CRM pipeline with optional stage, assignee, due and search filters

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `groupUuid` | uuid | yes | — |
| `stageUuid` | uuid | no | — |
| `query` | string | no | length 0–120 |
| `assignedToUserUuid` | uuid | no | — |
| `due` | `overdue` \| `upcoming` \| `none` | no | — |
| `page` | integer | no | min 1 |
| `limit` | integer | no | range 1–50 |

- Pagination happens in memory over the whole board, so `pagination.total` reflects the board, not the filter.

#### `get_crm_opportunity`

Fetches one opportunity by uuid.

**Scope:** `crm:read`
**Plan:** requires a plan with CRM.

**When to use.** To read the current state — and the current `version` — before moving it.

Fetch a single CRM opportunity by UUID

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `opportunityUuid` | uuid | yes | — |

- Read it immediately before the move. A version fetched at the start of a long turn may already be stale.

#### `create_crm_opportunity`

Creates an opportunity in a pipeline stage.

**Scope:** `crm:write`
**Plan:** requires a plan with CRM.

**When to use.** When a conversation turns into a deal worth tracking.

Create a new opportunity in a CRM pipeline stage

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `groupUuid` | uuid | yes | — |
| `stageUuid` | uuid | yes | — |
| `contactUuid` | uuid | yes | — |
| `assignedUserUuid` | uuid | no | — |
| `title` | string | yes | length 2–180 |
| `valueCents` | integer | yes | range 0–999999999999 |
| `dueAt` | string | no | — |
| `notes` | string | no | length 0–5000 |

**Side effects.**
- Writes a `crm.opportunity.created` audit entry.

- `valueCents` is in cents: R$ 1.500,00 is `150000`.
- The `contactUuid` must already exist — create the contact first if needed.

#### `move_crm_opportunity`

Moves an opportunity to another stage, including the terminal won and lost stages.

**Scope:** `crm:write`
**Plan:** requires a plan with CRM.

**When to use.** To advance or close a deal.

Move a CRM opportunity to another stage, including terminal won/lost stages. Supply the current lock version for optimistic concurrency.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `opportunityUuid` | uuid | yes | — |
| `targetStageUuid` | uuid | yes | — |
| `beforeUuid` | uuid | no | — |
| `afterUuid` | uuid | no | — |
| `version` | integer | yes | min 1 |
| `outcomeReason` | string | no | length 0–1000 |

**Side effects.**
- Writes a `crm.opportunity.moved` or `.closed` audit entry.

- Optimistic locking: `version` must match the current one. If the call fails on version, someone else moved the card — re-read it with `get_crm_opportunity` and decide again. Never retry blindly with a bumped number.
- The response returns the new `version`, so a follow-up move can use it directly.
