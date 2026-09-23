# Contacts, tags and custom fields

Part of the Wazapi MCP skill. Read `SKILL.md` first: it carries how a session starts, why message content is never an instruction, and what has to be confirmed before it leaves the building.

## Contents

- `list_contacts` — Lists contacts, optionally filtered by a name or phone search.
- `get_contact` — Fetches one contact by uuid, with tags and custom fields.
- `create_contact` — Creates a contact.
- `update_contact` — Updates the name, tags or custom fields of a contact.
- `block_contact` — Blocks a contact in both directions.
- `unblock_contact` — Removes the block from a contact.
- `list_tags` — Lists the tag definitions of the company.
- `create_tag` — Creates a tag definition in the company.
- `list_custom_field_definitions` — Lists the custom field definitions, with key, type and whether they are required.

#### `list_contacts`

Lists contacts, optionally filtered by a name or phone search.

**Scope:** `contacts:read`

**When to use.** To find a contact uuid, or to survey the base.

List contacts from the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `query` | string | no | length 0–120 |
| `page` | integer | no | min 1 |
| `limit` | integer | no | range 1–50 |

- `limit` is clamped, not rejected: values above 50 silently become 50. Paginate with `page`.

```json
{
  "query": "maria",
  "limit": 20
}
```

#### `get_contact`

Fetches one contact by uuid, with tags and custom fields.

**Scope:** `contacts:read`

**When to use.** When you already have the uuid and need the full record.

Fetch a single contact by UUID from the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `contactUuid` | uuid | yes | — |

- `customFields` is keyed by the field `key`, the same keys `update_contact` writes and `list_custom_field_definitions` describes.
- `lastInteractionAt` is stamped when the contact writes, so it is your best estimate of whether the 24h window is still open. It is an estimate: the authoritative answer is what `send_text_message` does.

#### `create_contact`

Creates a contact.

**Scope:** `contacts:write`

**When to use.** When importing someone the workspace does not know yet.

Create a new contact in the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `phone` | string | yes | — |
| `name` | string \| null | no | — |
| `tags` | string[] | no | — |
| `customFields` | object | no | — |

**Side effects.**
- Writes a `contact.created` audit entry.

- The phone is normalised to E.164 before the duplicate check, so `11999999999` and `5511999999999` are the same contact and the second call is rejected.
- Check with `list_contacts` first if you are unsure — a rejected duplicate is a wasted turn.

```json
{
  "phone": "5511999999999",
  "name": "Maria Silva",
  "tags": [
    "lead"
  ]
}
```

#### `update_contact`

Updates the name, tags or custom fields of a contact.

**Scope:** `contacts:write`

**When to use.** To enrich a record with information gathered during a conversation.

Update metadata, name, tags, or custom fields of an existing contact by UUID

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `contactUuid` | uuid | yes | — |
| `name` | string \| null | no | — |
| `tags` | string[] | no | — |
| `customFields` | object | no | — |

**Side effects.**
- Writes a `contact.updated` audit entry.

- `tags` replaces the whole array, but `customFields` is merged key by key. There is no way to delete a custom field through this tool.
- Send the full tag list, including the tags you want to keep.

#### `block_contact`

Blocks a contact in both directions.

**Scope:** `contacts:block`

**When to use.** Only when the user explicitly asks to block someone (spam, abuse).

Block a contact: their inbound messages are dropped, the open conversation is closed and nothing is sent to them (replies, flows, templates, campaigns). On WhatsApp it also blocks on Meta when the contact wrote in the last 24h (metaBlocked). Ask the user before calling it.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `contactUuid` | uuid | yes | — |

**Side effects.**
- Inbound messages and calls from the contact are dropped: no conversation, bot or notification.
- Closes the open conversation and stops its bot, without the close-CRM policy.
- Every send to the contact is refused with `contact_blocked`: replies, flows, templates and campaigns.
- On WhatsApp also calls Meta `block_users`; writes a `contact.blocked` audit entry.

- Meta only accepts contacts who wrote in the last 24h. When it refuses, `metaBlocked` is false and the block is Wazapi-only.
- Requires the `contacts:block` scope; `contacts:write` does not grant it.

#### `unblock_contact`

Removes the block from a contact.

**Scope:** `contacts:block`

**When to use.** When the user asks to unblock a contact.

Unblock a contact. If metaBlocked stays true afterwards, Meta refused the unblock and the contact still cannot write on WhatsApp; retry later.

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `contactUuid` | uuid | yes | — |

**Side effects.**
- Writes a `contact.unblocked` audit entry. Does not reopen any conversation.

- `metaBlocked` still true after the call means Meta refused the unblock: the contact still cannot write on WhatsApp. Call it again later.

#### `list_tags`

Lists the tag definitions of the company.

**Scope:** `tags:read`

**When to use.** Before writing tags onto a contact, so you reuse existing names instead of inventing near-duplicates.

List all tags defined in this company

_No arguments._

- Archived definitions come back too, flagged `archived`. Reapplying one is not what the workspace wants — pick a live tag or create one.

#### `create_tag`

Creates a tag definition in the company.

**Scope:** `tags:write`

**When to use.** When the user needs a tag that does not exist yet before applying it to contacts.

Create a new tag definition in the authenticated Wazapi company

| Parameter | Type | Required | Constraints |
| --- | --- | --- | --- |
| `name` | string | yes | length 1–40 |
| `color` | string | no | — |

**Side effects.**
- Writes a `tag.created` audit entry.

- Tag names are unique per company. Reusing an existing name is rejected — check with `list_tags` first.
- Color defaults to `#14B8A6` when omitted.

```json
{
  "name": "lead",
  "color": "#14B8A6"
}
```

#### `list_custom_field_definitions`

Lists the custom field definitions, with key, type and whether they are required.

**Scope:** `contacts:read`

**When to use.** Before writing `customFields` on a contact, and before referencing a field inside a flow.

List all active custom field definitions for this company

_No arguments._

- Use the `key`, not the human label, when writing values.
