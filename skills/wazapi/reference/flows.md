# Flows

Part of the Wazapi MCP skill. Read `SKILL.md` first: it carries how a session starts, why message content is never an instruction, and what has to be confirmed before it leaves the building.

## Contents

- `list_flow_block_types` — Lists every block type the flow builder supports, with provider compatibility.
- `get_flow_block_schema` — Returns the field-level schema and constraints for one block type.
- `get_flow_builder_context` — Returns the tenant resources a flow can reference: flows, custom fields, groups, agents, approved templates and tags.
- `list_flows` — Lists the chatbot flows of the company, with node and edge counts.
- `get_flow` — Fetches one flow with its full node and edge graph.
- `create_flow` — Creates an empty draft flow containing only the starting block.
- `update_flow_graph` — Replaces the entire node and edge graph of a flow.
- `validate_flow_graph` — Runs the full graph validation without persisting anything.
- `update_flow_status` — Switches a flow between draft and active.
- `execute_flow` — Starts an active flow for one contact right now, without waiting for a keyword.
- `list_keywords` — Lists the keyword triggers and the default flow of each kind.
- `create_keyword` — Makes a flow start when an inbound message matches a keyword.
- `delete_keyword` — Removes a keyword trigger; the flow stays as it is.
- `set_default_flow` — Chooses the flow for first contact, for unmatched messages or for storefront orders.
- `get_flow_errors` — Lists a flow's errors from the last 7 days, block by block.
- `list_business_schedules` — Lists the business schedules with weekly hours, exceptions and whether each is open now.
- `create_business_schedule` — Creates a named business schedule.
- `update_business_schedule` — Replaces the hours of a business schedule.
- `update_flow` — Renames a flow or changes the channels it supports.

#### `list_flow_block_types`

Lists every block type the flow builder supports, with provider compatibility.

**Scope:** `flows:read`

**When to use.** Step 1 of building or editing a flow. Never guess a block type — the list is authoritative and includes which providers each block supports.

List the block types supported by the Wazapi flow builder, including provider compatibility and category

_No arguments._

- `providerSupport` matters: a flow whose `supportedProviders` includes instagram cannot use a block marked unsupported there, and validation will reject the whole graph.

#### `get_flow_block_schema`

Returns the field-level schema and constraints for one block type.

**Scope:** `flows:read`

**When to use.** Step 2 of building a flow. Call it for every block type you intend to use, before writing any node `data`.

Return the detailed schema, field requirements, and constraints for one Wazapi flow block type

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `blockType` | `starting_block` \| `send_text` \| `send_template` \| `send_sms` \| `send_buttons` \| `collect_input` \| `condition` \| `action` \| `delay` \| `go_to_flow` \| `http_request` \| `send_list` \| `send_media` \| `go_to_node` \| `assign_agent` \| `end_flow` \| `note` \| `random_branch` \| `split_test` \| `send_reaction` \| `send_location` \| `notify_webhook` \| `track_event` \| `create_order` \| `cart` \| `discount` \| `store_link` \| `checkout` \| `send_email` \| `openai_assistant` \| `wait_for_event` \| `business_hours` \| `ai_agent` | yes | — |

- Use cart for a shared draft without reserving stock. Checkout mode cart takes cartUuid and waits for a link, order or payment; the customer chooses payment in the storefront. Existing order mode remains unchanged.
- The `data` object of a node is validated field by field against this schema. Guessing field names is the most common cause of a rejected graph.

```json
{
  "blockType": "collect_input"
}
```

#### `get_flow_builder_context`

Returns the tenant resources a flow can reference: flows, custom fields, groups, agents, approved templates and tags.

**Scope:** `flows:read`

**When to use.** Step 3, before writing a graph that references anything by id or uuid. Blocks like `go_to_flow`, `assign_agent` and `send_template` point at real records, and the server rejects cross-tenant references.

Return the current tenant resources and constraints needed to design valid Wazapi flows before creating them

_No arguments._

- Every uuid you put in a node must come from here (or from another list tool). An invented uuid fails validation.
- `availableResources.businessSchedules` is what the `business_hours` block needs: a schedule uuid from another company is rejected, and that block accepts no outgoing handle other than `inside` and `outside`.
- `availableResources.aiAgents` is what the `ai_agent` block needs. Schedules are created in the dashboard. Use `create_ai_agent` to configure an inactive AI agent, then activate it in the dashboard. Knowledge sources can be added with `create_knowledge_source`.

#### `list_flows`

Lists the chatbot flows of the company, with node and edge counts.

**Scope:** `flows:read`

**When to use.** To find a flow by name or to survey what already exists before creating a new one.

List chatbot flows from the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `query` | string | no | length 0–120 |
| `status` | `draft` \| `active` | no | — |
| `page` | integer | no | min 1 |
| `limit` | integer | no | range 1–50 |

- Only an `active` flow answers real contacts and only an active one can be started with `execute_flow`.
- A flow whose node count is 1 is an empty shell — it has the starting block and nothing else.

```json
{
  "query": "boas-vindas",
  "status": "active",
  "limit": 20
}
```

#### `get_flow`

Fetches one flow with its full node and edge graph.

**Scope:** `flows:read`

**When to use.** Always immediately before `update_flow_graph`. The update is a full replace, so you need the current graph to modify it without deleting the rest.

Fetch a single chatbot flow by UUID from the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |

- The returned `graph.nodes` / `graph.edges` are exactly the shape `update_flow_graph` expects back.

#### `create_flow`

Creates an empty draft flow containing only the starting block.

**Scope:** `flows:write`

**When to use.** When the user wants a new flow. It provisions the shell; the actual content goes in through `update_flow_graph`.

Create a new draft chatbot flow in the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | string | yes | length 1–120 |
| `supportedProviders` | `whatsapp` \| `instagram` \| `messenger`[] | no | 1–3 items |

**Side effects.**
- Writes a `flow.created` audit entry.

- The flow starts as `draft` and does not run until `update_flow_status` activates it.
- `supportedProviders` is fixed at creation and constrains which blocks the graph may use.

```json
{
  "name": "Boas-vindas",
  "supportedProviders": [
    "whatsapp"
  ]
}
```

#### `update_flow_graph`

Replaces the entire node and edge graph of a flow.

**Scope:** `flows:write`

**When to use.** Last step of building or editing a flow, after `validate_flow_graph` returns ok.

Replace the full node and edge graph of an existing Wazapi flow after validating block schemas, provider compatibility, and tenant references

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |
| `nodes` | object[] | yes | 1–300 items |
| `edges` | object[] | yes | 0–800 items |
| `nodes[].key` | string | yes | length 1–120 |
| `nodes[].type` | `starting_block` \| `send_text` \| `send_template` \| `send_sms` \| `send_buttons` \| `collect_input` \| `condition` \| `action` \| `delay` \| `go_to_flow` \| `http_request` \| `send_list` \| `send_media` \| `go_to_node` \| `assign_agent` \| `end_flow` \| `note` \| `random_branch` \| `split_test` \| `send_reaction` \| `send_location` \| `notify_webhook` \| `track_event` \| `create_order` \| `cart` \| `discount` \| `store_link` \| `checkout` \| `send_email` \| `openai_assistant` \| `wait_for_event` \| `business_hours` \| `ai_agent` | yes | — |
| `nodes[].position` | object | yes | — |
| `nodes[].data` | object | no | — |
| `edges[].source` | string | yes | length 1–120 |
| `edges[].sourceHandle` | string \| null | no | — |
| `edges[].target` | string | yes | length 1–120 |
| `edges[].targetHandle` | string \| null | no | — |

**Side effects.**
- Replaces the whole graph. Nodes and edges absent from your payload are deleted.
- Writes a `flow.graph.saved` audit entry.

- This is not a patch. Call `get_flow` first and send the full graph back with your changes applied, or you will silently destroy the rest of the flow.
- Limits: 1 to 300 nodes, at most 800 edges.
- Cart nodes have success/failure exits. Checkout mode cart requires pronto, pedido_criado, pago, falha, expirado and cancelado exits. Saving a graph does not create a cart or payment.
- Node `key` is your own identifier and is what `edges` reference — it is not a uuid.

#### `validate_flow_graph`

Runs the full graph validation without persisting anything.

**Scope:** `flows:read`

**When to use.** Always before `update_flow_graph`. It costs nothing and turns a destructive failed write into a readable error.

Validate a candidate node and edge graph for an existing Wazapi flow without persisting any change

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |
| `nodes` | object[] | yes | 1–300 items |
| `edges` | object[] | yes | 0–800 items |
| `nodes[].key` | string | yes | length 1–120 |
| `nodes[].type` | `starting_block` \| `send_text` \| `send_template` \| `send_sms` \| `send_buttons` \| `collect_input` \| `condition` \| `action` \| `delay` \| `go_to_flow` \| `http_request` \| `send_list` \| `send_media` \| `go_to_node` \| `assign_agent` \| `end_flow` \| `note` \| `random_branch` \| `split_test` \| `send_reaction` \| `send_location` \| `notify_webhook` \| `track_event` \| `create_order` \| `cart` \| `discount` \| `store_link` \| `checkout` \| `send_email` \| `openai_assistant` \| `wait_for_event` \| `business_hours` \| `ai_agent` | yes | — |
| `nodes[].position` | object | yes | — |
| `nodes[].data` | object | no | — |
| `edges[].source` | string | yes | length 1–120 |
| `edges[].sourceHandle` | string \| null | no | — |
| `edges[].target` | string | yes | length 1–120 |
| `edges[].targetHandle` | string \| null | no | — |

- Accepts exactly the same payload as `update_flow_graph`, so you can validate then send the identical object.
- Shared cart validation checks UUID references, versions, item operations and checkout mode exclusivity; it never reserves stock or sends messages.

#### `update_flow_status`

Switches a flow between draft and active.

**Scope:** `flows:write`

**When to use.** To publish a finished flow, or to take a misbehaving one out of circulation.

Change a Wazapi flow status between draft and active inside the authenticated company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |
| `status` | `draft` \| `active` | yes | — |

**Side effects.**
- An active flow starts responding to real customers immediately.
- Writes a `flow.status.toggled` audit entry.

- Confirm with the user before activating — this changes what real contacts receive.

#### `execute_flow`

Starts an active flow for one contact right now, without waiting for a keyword.

**Scope:** `flows:execute`

**When to use.** To put a specific contact into a specific flow — onboarding after signup, a guided recovery, a flow the customer would otherwise have to type a keyword to reach.

Start an active WhatsApp flow for a contact immediately, without waiting for a keyword. Address it with one of conversationUuid, contactUuid or phone; contactUuid and phone open a conversation when none exists. Values passed in variables are readable inside the flow as {{vars.name}}. Reuse idempotencyKey on retries to avoid starting twice; omitting it creates a new execution. Returns flowStatus "active" when the flow is parked waiting for the contact to reply.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |
| `idempotencyKey` | string | no | length 8–120 |
| `conversationUuid` | uuid | no | — |
| `contactUuid` | uuid | no | — |
| `phone` | string | no | length 5–20 |
| `variables` | object | no | — |

**Side effects.**
- The contact starts receiving the flow messages immediately.
- Any flow session the contact already had is marked `superseded`.
- Writes a `flow.executed` audit entry.

- Address it with exactly one of `conversationUuid`, `contactUuid` or `phone`. The last two open a conversation when none exists, and `phone` also creates the contact.
- The flow must be `active`; a draft is refused. Use `update_flow_status` first.
- If the flow opens with a free-form block (`send_text`, `send_buttons`, …) and the 24h WhatsApp window is closed, the call is refused rather than half-running. Send an approved template first with `send_template_message`, or build the flow to open with `send_template`.
- `variables` land in the session and read as `{{vars.name}}` inside the graph.
- `flowStatus: "active"` in the result means the flow is parked waiting for the contact to reply — that is success, not a hang.
- Send an `idempotencyKey` whenever the call may be repeated — a retry after a timeout, a webhook that fires twice. Reusing the key returns the first execution instead of starting the flow again; omitting it starts a new one every time.

```json
{
  "flowUuid": "00000000-0000-0000-0000-000000000000",
  "phone": "5511999999999",
  "variables": {
    "nome": "Maria",
    "plano": "pro"
  }
}
```

#### `list_keywords`

Lists the keyword triggers and the default flow of each kind.

**Scope:** `flows:read`

**When to use.** Before creating a trigger, and to answer "why does this flow never start?": an active flow with no trigger and no default slot never runs on its own.

List the keyword triggers that start flows (keyword, match type, flow) and the default flows per kind (welcome, default_reply, store_order).

_No arguments._

#### `create_keyword`

Makes a flow start when an inbound message matches a keyword.

**Scope:** `flows:write`

**When to use.** Right after building and activating a flow that customers should reach by typing something ("menu", "boleto").

Make a flow start when an inbound message matches a keyword. Without a trigger or a default flow, an active flow never starts on its own.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `keyword` | string | yes | length 1–120 |
| `matchType` | `exact` \| `starts_with` \| `contains` | yes | — |
| `flowUuid` | uuid | yes | — |

**Side effects.**
- Goes live immediately for every inbound message of the company.

- Match is case-insensitive. `exact` is the safe default; `contains` catches words inside longer messages and can steal traffic from other flows.
- The same keyword with the same match type cannot exist twice.

```json
{
  "keyword": "boleto",
  "matchType": "exact",
  "flowUuid": "<flow uuid>"
}
```

#### `delete_keyword`

Removes a keyword trigger; the flow stays as it is.

**Scope:** `flows:write`

**When to use.** When a trigger points at the wrong flow or is stealing traffic. Confirm with the user first.

Remove a keyword trigger. The flow itself is not touched.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `keywordUuid` | uuid | yes | — |

#### `set_default_flow`

Chooses the flow for first contact, for unmatched messages or for storefront orders.

**Scope:** `flows:write`

**When to use.** When the user wants a flow to greet every new contact (`welcome`), answer anything no trigger caught (`default_reply`) or confirm store orders (`store_order`).

Choose the flow that runs for a kind of event: welcome (first contact), default_reply (no trigger matched) or store_order (storefront order). Pass flowUuid null to turn the slot off.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `kind` | `welcome` \| `default_reply` \| `store_order` | yes | — |
| `flowUuid` | uuid \| null | yes | — |

**Side effects.**
- Replaces the flow that held the slot before.

- `store_order` needs the store permission; the other kinds need keywords.

#### `get_flow_errors`

Lists a flow's errors from the last 7 days, block by block.

**Scope:** `flows:read`

**When to use.** When a customer or the user reports that the bot stopped or answered wrong. Read this before editing the graph.

List the last errors of a flow in the past 7 days (block, node key, message, when): the same list behind the warning icon on the flow card.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |

#### `list_business_schedules`

Lists the business schedules with weekly hours, exceptions and whether each is open now.

**Scope:** `flows:read`

**When to use.** Before building a business_hours block or answering "are we open on the holiday?".

List the named business schedules with their weekly hours, date exceptions and whether they are open right now.

_No arguments._

#### `create_business_schedule`

Creates a named business schedule.

**Scope:** `flows:write`

**When to use.** When a flow must branch on opening hours and the right schedule does not exist yet.

Create a named business schedule, used by the business_hours flow block and by group auto-close. The first schedule of the company becomes the default.

| Parameter | Type | Required | Constraints | Description |
| --- | --- | --- | --- | --- |
| `name` | string | yes | length 1–120 | — |
| `timezone` | string | yes | length 1–64 | IANA time zone, e.g. America/Sao_Paulo |
| `weekly` | object | yes | — | Opening intervals per weekday (HH:mm); an empty list means closed that day |
| `exceptions` | object[] | no | 0–100 items | Dates that override the week (closed, or special hours). Omitted on update keeps the current ones. |
| `useCompanyHolidays` | boolean | no | — | — |
| `weekly.mon` | object[] | yes | 0–6 items | — |
| `weekly.tue` | object[] | yes | 0–6 items | — |
| `weekly.wed` | object[] | yes | 0–6 items | — |
| `weekly.thu` | object[] | yes | 0–6 items | — |
| `weekly.fri` | object[] | yes | 0–6 items | — |
| `weekly.sat` | object[] | yes | 0–6 items | — |
| `weekly.sun` | object[] | yes | 0–6 items | — |
| `exceptions[].id` | string | yes | length 1–40 | — |
| `exceptions[].label` | string | yes | length 1–120 | — |
| `exceptions[].start` | string | yes | — | — |
| `exceptions[].end` | string | yes | — | — |
| `exceptions[].recurring` | boolean | yes | — | — |
| `exceptions[].closed` | boolean | yes | — | — |
| `exceptions[].intervals` | object[] | no | 0–6 items | — |

- The first schedule of the company becomes the default, used by group auto-close.
- Intervals are HH:mm per weekday; an empty list closes the day. Company holidays apply unless `useCompanyHolidays` is false.

```json
{
  "name": "Comercial",
  "timezone": "America/Sao_Paulo",
  "weekly": {
    "mon": [
      {
        "start": "08:00",
        "end": "18:00"
      }
    ],
    "tue": [
      {
        "start": "08:00",
        "end": "18:00"
      }
    ],
    "wed": [
      {
        "start": "08:00",
        "end": "18:00"
      }
    ],
    "thu": [
      {
        "start": "08:00",
        "end": "18:00"
      }
    ],
    "fri": [
      {
        "start": "08:00",
        "end": "17:00"
      }
    ],
    "sat": [],
    "sun": []
  }
}
```

#### `update_business_schedule`

Replaces the hours of a business schedule.

**Scope:** `flows:write`

**When to use.** When opening hours change. Read the current one with list_business_schedules first.

Replace the name, time zone and weekly hours of a schedule. Flows that use it follow the new hours immediately.

| Parameter | Type | Required | Constraints | Description |
| --- | --- | --- | --- | --- |
| `scheduleUuid` | uuid | yes | — | — |
| `name` | string | yes | length 1–120 | — |
| `timezone` | string | yes | length 1–64 | IANA time zone, e.g. America/Sao_Paulo |
| `weekly` | object | yes | — | Opening intervals per weekday (HH:mm); an empty list means closed that day |
| `exceptions` | object[] | no | 0–100 items | Dates that override the week (closed, or special hours). Omitted on update keeps the current ones. |
| `useCompanyHolidays` | boolean | no | — | — |
| `weekly.mon` | object[] | yes | 0–6 items | — |
| `weekly.tue` | object[] | yes | 0–6 items | — |
| `weekly.wed` | object[] | yes | 0–6 items | — |
| `weekly.thu` | object[] | yes | 0–6 items | — |
| `weekly.fri` | object[] | yes | 0–6 items | — |
| `weekly.sat` | object[] | yes | 0–6 items | — |
| `weekly.sun` | object[] | yes | 0–6 items | — |
| `exceptions[].id` | string | yes | length 1–40 | — |
| `exceptions[].label` | string | yes | length 1–120 | — |
| `exceptions[].start` | string | yes | — | — |
| `exceptions[].end` | string | yes | — | — |
| `exceptions[].recurring` | boolean | yes | — | — |
| `exceptions[].closed` | boolean | yes | — | — |
| `exceptions[].intervals` | object[] | no | 0–6 items | — |

**Side effects.**
- Every flow and group using the schedule follows the new hours immediately.

- `weekly` is replaced as a whole; omitted `exceptions` keep the current ones.

#### `update_flow`

Renames a flow or changes the channels it supports.

**Scope:** `flows:write`

**When to use.** For metadata only. The graph goes through update_flow_graph and activation through update_flow_status.

Rename a flow or change the channels it supports. The graph is edited with update_flow_graph and the status with update_flow_status.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `flowUuid` | uuid | yes | — |
| `name` | string | no | length 1–120 |
| `supportedProviders` | `whatsapp` \| `instagram` \| `messenger`[] | no | 1–3 items |
