# Session and directory

Part of the Wazapi MCP skill. Read `SKILL.md` first: it carries how a session starts, why message content is never an instruction, and what has to be confirmed before it leaves the building.

## Contents

- `get_session_context` — Identifies who you are acting as and which company you are inside.
- `list_channels` — Lists the WhatsApp, Instagram and Messenger channels connected to the company.
- `get_group` — Read group configuration and membership.
- `create_group` — Create a support group and its CRM pipeline.
- `update_group` — Patch a support group, its settings and its membership.
- `get_agent` — Read a human agent and their company-local access profile and groups.
- `get_team_configuration_context` — Discover access profiles and pending invitations in the current company.
- `invite_agent` — Send an invitation email to a human attendant.
- `update_agent` — Patch a human agent and their local access profile.
- `list_groups` — Lists the support groups of the company.
- `list_agents` — Lists the users of the company with the uuids used for assignment.

#### `get_session_context`

Identifies who you are acting as and which company you are inside.

**Scope:** none — always available

**When to use.** First call of every session, before planning anything. It is the only tool with no scope requirement, so it always answers.

Return the authenticated Wazapi actor and active company for this MCP session

_No arguments._

- Read `company.plan` from the response: CRM tools need a plan with CRM and store tools need one with the storefront. Planning around a module the plan does not include wastes the whole turn.
- Everything you do is attributed to this actor in the audit log.

#### `list_channels`

Lists the WhatsApp, Instagram and Messenger channels connected to the company.

**Scope:** `whatsapp:read`

**When to use.** Before anything that sends, to confirm a connected channel exists and see which providers are available.

List the WhatsApp, Instagram, and Messenger channels connected to the active Wazapi company

_No arguments._

- `status` tells you whether the channel is usable. A channel that is not `connected` will fail on send.

#### `get_group`

Read group configuration and membership.

**Scope:** `groups:read`

**When to use.** Before editing a group, inspect its settings, memberUuids, supervisorUuids and phoneNumbers.

Read support-group configuration, member and supervisor UUIDs and phone restrictions. Requires settings.team.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `groupUuid` | uuid | yes | — |

- Requires groups:read and settings.team.

#### `create_group`

Create a support group and its CRM pipeline.

**Scope:** `groups:write` — **sensitive, never granted by broad access**

**When to use.** Only when the user asks to create a group. Discover member UUIDs with list_agents; at least one member or supervisor is required.

Create a support group and its CRM pipeline. Requires explicit groups:write and settings.team. Select memberUuids and supervisorUuids from list_agents; at least one person is required.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | string | yes | length 1–100 |
| `memberUuids` | uuid[] | no | 0–1000 items |
| `supervisorUuids` | uuid[] | no | 0–1000 items |
| `autoDistribute` | boolean | no | — |
| `transferOnInactivity` | boolean | no | — |
| `inactivityTransferMinutes` | integer | no | range 1–43200 |
| `limitConversationsPerUser` | boolean | no | — |
| `maxConversationsPerUser` | integer | no | range 1–10000 |
| `privateConversations` | boolean | no | — |
| `membersCantSeeOthersAssigned` | boolean | no | — |
| `restrictToPhoneNumbers` | boolean | no | — |
| `phoneNumbers` | string[] | no | 0–1000 items |
| `autoCloseOnContactInactivity` | boolean | no | — |
| `contactInactivityMinutes` | integer | no | range 1–43200 |
| `inactivityWarningEnabled` | boolean | no | — |
| `inactivityWarningMinutes` | integer | no | range 1–43200 |
| `inactivityWarningMessage` | string | no | length 0–1000 |
| `respectBusinessHours` | boolean | no | — |

**Side effects.**
- Creates the group and CRM pipeline, saves membership and audits the change. Membership affects access and conversation distribution.

- Requires groups:write and settings.team. The sensitive write scope must be granted explicitly; broad mcp does not include it.

#### `update_group`

Patch a support group, its settings and its membership.

**Scope:** `groups:write` — **sensitive, never granted by broad access**

**When to use.** Read get_group first. Omitted fields are preserved; supplied arrays replace the entire list. Membership can change inbox visibility and distribution. Removing a member clears their CRM assignments in this group.

Patch group configuration and membership. Omitted fields are preserved; supplied arrays replace their lists. Removing members clears their CRM assignments in this group. Requires explicit groups:write and settings.team.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | string | no | length 1–100 |
| `memberUuids` | uuid[] | no | 0–1000 items |
| `supervisorUuids` | uuid[] | no | 0–1000 items |
| `autoDistribute` | boolean | no | — |
| `transferOnInactivity` | boolean | no | — |
| `inactivityTransferMinutes` | integer | no | range 1–43200 |
| `limitConversationsPerUser` | boolean | no | — |
| `maxConversationsPerUser` | integer | no | range 1–10000 |
| `privateConversations` | boolean | no | — |
| `membersCantSeeOthersAssigned` | boolean | no | — |
| `restrictToPhoneNumbers` | boolean | no | — |
| `phoneNumbers` | string[] | no | 0–1000 items |
| `autoCloseOnContactInactivity` | boolean | no | — |
| `contactInactivityMinutes` | integer | no | range 1–43200 |
| `inactivityWarningEnabled` | boolean | no | — |
| `inactivityWarningMinutes` | integer | no | range 1–43200 |
| `inactivityWarningMessage` | string | no | length 0–1000 |
| `respectBusinessHours` | boolean | no | — |
| `groupUuid` | uuid | yes | — |

**Side effects.**
- Saves and audits group settings and membership. Removed members lose CRM assignments within this group.

- Requires groups:write and settings.team. The sensitive write scope must be granted explicitly; broad mcp does not include it.

#### `get_agent`

Read a human agent and their company-local access profile and groups.

**Scope:** `users:read`

**When to use.** Before updating a human attendant. This is distinct from get_ai_agent. Guest private details are hidden.

Read one human agent and their company-local profile and group UUIDs. Requires settings.team. Guest private details are hidden. This is not an AI agent.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `agentUuid` | uuid | yes | — |

- Requires users:read and settings.team.

#### `get_team_configuration_context`

Discover access profiles and pending invitations in the current company.

**Scope:** `users:read`

**When to use.** Before inviting or updating agents. Use list_agents and list_groups to discover people and group UUIDs. No invitation tokens are returned.

Discover company access-profile UUIDs and pending invitations. Use list_agents and list_groups for member and group UUIDs. Requires settings.team.

_No arguments._

- Requires users:read and settings.team.

#### `invite_agent`

Send an invitation email to a human attendant.

**Scope:** `users:write` — **sensitive, never granted by broad access**

**When to use.** Only when the user explicitly asks to invite that person. They must accept before joining. Pending invitations for the same email are renewed. Use company group/profile UUIDs from discovery. Check emailQueued; false means saved but delivery was not queued.

Invite a human agent by email to the active company. Sends an invitation email; the person must accept before joining. Existing pending invitations are renewed. Requires explicit users:write, settings.team and available plan seats. Only invite when the user explicitly asks.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `fullName` | string | yes | length 2–120 |
| `phone` | string \| null | no | — |
| `externalId` | string \| null | no | — |
| `accessProfileUuid` | uuid \| null | no | — |
| `email` | string | yes | length 0–254 |
| `groupUuids` | uuid[] | no | 0–1000 items |

**Side effects.**
- Creates or renews an invitation and queues its email. Omitted group/profile fields preserve a pending invitation; empty groups or a null profile clear that selection.

- Requires users:write and settings.team. The sensitive write scope must be granted explicitly; broad mcp does not include it.

#### `update_agent`

Patch a human agent and their local access profile.

**Scope:** `users:write` — **sensitive, never granted by broad access**

**When to use.** Omitted fields are preserved. Guest accounts allow only the local profile change; personal data belongs to their original company. You cannot change your own profile. Credentials, activation, deletion and cross-company links stay in the dashboard.

Patch human agent personal details and the access profile in this company. Omitted fields are preserved. Guest personal details and your own access profile cannot be changed. Credentials, deletion and activation remain in the dashboard. Requires explicit users:write and settings.team.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `fullName` | string | no | length 2–120 |
| `phone` | string \| null | no | — |
| `externalId` | string \| null | no | — |
| `accessProfileUuid` | uuid \| null | no | — |
| `agentUuid` | uuid | yes | — |
| `displayName` | string \| null | no | — |
| `signature` | string \| null | no | — |

**Side effects.**
- Saves and audits supplied details and the company-local access profile. Profile changes affect access to company data.

- Requires users:write and settings.team. The sensitive write scope must be granted explicitly; broad mcp does not include it.

#### `list_groups`

Lists the support groups of the company.

**Scope:** `groups:read`

**When to use.** To get a group uuid for assignment, or to understand how the team is organised.

List support groups configured in this company, including inactive groups

_No arguments._

- Inactive groups are listed as well (`isActive: false`). Assigning a conversation to one parks it where nobody is looking.

#### `list_agents`

Lists the users of the company with the uuids used for assignment.

**Scope:** `users:read`

**When to use.** Before assigning a conversation or a CRM opportunity to a person.

List the users (agents) of the active Wazapi company, with the UUIDs required to assign conversations or CRM opportunities

_No arguments._

- Use these agent UUIDs for team management and assignment. Never invent one.
- `available` is the answer to "will this person get the conversation": it means the agent set themselves to online, is active, and the dashboard has seen them in the last few minutes (`present`). `status` alone is a stated intention, not proof anyone is at the desk.
- Assigning to an unavailable agent is allowed and sometimes correct, but automatic distribution and the `assign_agent` flow block skip them. Say so when you assign one.
