---
name: terlido-newsletter
description: How Terlido Newsletter works and how to set it up correctly through the Terlido MCP server - publications (brands), sending mailbox, subscribers and consent, lists, segments, imports, broadcasts (prepare, test, schedule), automations, webhooks, and diagnosing sends. Use whenever an agent works with Terlido Newsletter or any `newsletter_*` tool - "set up a newsletter", "import subscribers", "send/schedule a broadcast", "build a welcome sequence", "why did nobody get my newsletter" - even if the word "skill" is never used.
---

# Terlido Newsletter

Terlido Newsletter sends newsletters from the workspace's **own connected mailboxes** (no shared
platform sender). Subscribers live inside a **publication** (the UI calls it a **brand**). Contacts,
tags, custom fields and the Blocklist are shared with Terlido Outreach.

## Start every session

1. Call `whoami`. Check the scopes:
   - `newsletter:read` - reads only. Works on the Free plan.
   - `newsletter:write`, `newsletter:send`, `newsletter:automations` - need **Pro**. Checked on every call.
     A workspace that just started Newsletter is on a 14-day Pro trial; when it ends without payment
     it drops to Free and these scopes stop working.
   - No Newsletter tools at all: the token was issued before Newsletter scopes existed. Ask the user to
     disconnect the agent and reconnect with the Newsletter boxes ticked. There is no refresh path.
2. Call `newsletter_list_publications`, `newsletter_list_mailboxes` and `newsletter_get_usage` to see what exists.
3. There is **no tool to list or read broadcasts or automations**. Keep every id you create.

## Words

- **Publication / brand** - one newsletter. Has a default mailbox, from name, reply-to, postal
  address, time zone, double opt-in (default on), archive (default off), auto UTMs (default on).
- **Subscriber** - one per (publication, email). Status is per publication:
  `active`, `pending` (never confirmed), `unsubscribed`, `bounced`, `complained`.
  "Dormant" is a filter (no verified click in 90 days), not a status. Only `active` people get email.
- **List** - every publication has a default list "All subscribers" plus up to 50 named lists.
  Removing from a list is **not** an unsubscribe.
- **Segment** - saved rules `{match: "all"|"any", rules[1..20]}` on status, source, tag, custom field,
  subscribe date, last click, list, another segment, or another publication. Clicks only, never opens.
- **Broadcast** - one email to an audience. `draft` -> `scheduled` -> `sending` -> `sent`, or `cancelled`.
- **Automation** - triggered sequence. `draft` -> `active` <-> `paused`.

## Setup order for a first newsletter

Do these in order. Steps marked **web app** cannot be done over MCP - tell the user exactly what to click.

1. **Mailbox (web app, admin).** Connect a Microsoft or IMAP/SMTP mailbox in Mailboxes and tick
   **Newsletter** in its Access column. Confirm with `newsletter_list_mailboxes`: it must be listed
   with `connected: true`. The mailbox's domain should have SPF, DKIM and DMARC (Domains page).
2. **Publication.** `newsletter_create_publication {name}` (Free allows 1).
3. **Publication settings.** `newsletter_update_publication` with at least:
   `defaultMailboxId`, `postalAddress` (**required** - no send without it), `fromName`, `timezone` (IANA).
   Optional: `replyToEmail` (replies then skip Terlido's inbox), `archivePublic`, `doubleOptIn`,
   confirm/goodbye email text, `monthlySendLimit` (admin, Pro), and the email style:
   `accentColor` and `emailBackgroundColor` (`#rrggbb`, null = default) and `emailFont`
   (`sans`, `serif` or `system`). The logo is set in the app and is shared with the public page.
   The name cannot be changed over MCP.
4. **Lists** (optional). `newsletter_create_list {publicationId, name}`.
5. **Subscribers.** Only add people who opted in. Always pass a truthful `consentSource`:
   `signed_up_on_site`, `customers_opted_in` or `migrated_from_platform`.
   - Many: `newsletter_import_csv {publicationId, listId?, consentSource, csv, mapping?, tagName?}`
     (CSV text with a header, max 20,000 rows / 500,000 chars). Map exactly one column to `email`;
     `custom:<key>` sets a custom field. Then poll `newsletter_get_import {importId}`.
   - One: `newsletter_upsert_subscriber {publicationId, listId?, email, firstName?, lastName?, consentSource, customFields?}`
     (`customFields` is written only when this call creates the contact).
   - Both add people as **active right away**: no confirmation email, no welcome automation.
   - Existing subscribers are left unchanged. Suppressed addresses are skipped.
   - Check with `newsletter_count_subscribers` and `newsletter_list_subscribers` (its `counts` gives
     `all`, `dormant`, `byStatus` without paging).
   - Attributes for people who are already subscribers: `newsletter_set_attributes` (see "Attributes").
6. **Signup form (web app).** Subscribers -> Signup form gives embed code. Form signups are `pending`
   until they click the confirmation link.
7. **Segment** (optional). Test with `newsletter_preview_segment {publicationId, rules}` first. Its
   count includes every status, so add a `status is active` rule to count sendable people.
8. **Broadcast** - see the next section.

## Lists and segments

Lists are who someone signed up as; segments are rules. A broadcast goes to its lists (minus
excluded lists), narrowed by at most one segment, and only to `active` subscribers.

- `newsletter_list_lists {publicationId}` - lists with active member counts, default list first.
- `newsletter_create_list {publicationId, name, description?}` - max 50 per publication.
- `newsletter_update_list {listId, name?, description?}` - the default list can be renamed too.
- `newsletter_delete_list {listId}` - refused for the default list and for a list a signup form uses.
- `newsletter_add_to_list` / `newsletter_remove_from_list {listId, subscriptionIds[1..500]}` -
  `subscriptionIds` come from `newsletter_list_subscribers`. Adding skips people from another
  publication and people who are not `active` or `pending`. Removing is not an unsubscribe.
- `newsletter_list_subscribers` also takes `sort` (`subscriber`, `brand`, `status`, `lists`, `source`,
  `subscribed`, `lastClick`, `engagement`) and `dir` (`asc` / `desc`), default `subscribed` `desc`. The
  order covers everyone who matches, not just the page, so "the 50 most recently active" is one call:
  `{sort: "lastClick", dir: "desc", take: 50}`.
- `newsletter_list_segments {publicationId?}`.
- `newsletter_preview_segment {publicationId, rules}` - count before saving (includes every status).
- `newsletter_create_segment {publicationId, name, rules}`.
- `newsletter_update_segment {segmentId, publicationId, name, rules}` - send all fields.
- `newsletter_delete_segment {segmentId}` - refused while another segment uses it. A broadcast
  already prepared with it keeps its recipients.

Segment `rules` (1-20 rules, `match` is `all` or `any`):

```json
{ "match": "all", "rules": [
  { "field": "status", "op": "is", "value": "active" },
  { "field": "custom_field", "key": "plan", "op": "equals", "value": "paid" },
  { "field": "last_click", "op": "within_days", "days": 30 }
] }
```

Each rule has a `field` and an `op`:
- `status` / `source` - `is`, `is_not` + `value`
- `tag` - `has`, `has_not` + `tagId`
- `custom_field` - `key` + `equals`, `not_equals`, `contains` (+ `value`), `is_set`, `is_not_set`
- `subscribed` - `before`, `after` + `date` (YYYY-MM-DD), or `within_days`, `more_than_days` + `days`
- `last_click` - `within_days`, `not_within_days` + `days`, or `never`
- `segment` - `in`, `not_in` + `segmentId` (one level deep)
- `publication` - `subscribed`, `not_subscribed` + `publicationId`
- `list` - `in`, `not_in` + `listId`

`days` is 1-3650. Extra keys are rejected.

## Send a broadcast

1. `newsletter_list_templates` - 6 starters (`starter:weekly-digest`, `starter:announcement`, ...) plus workspace templates.
2. `newsletter_create_broadcast {publicationId, templateId?, listIds?, excludeListIds?}` -> `{broadcastId, mailboxId}`.
   No lists = whole brand. Each call makes a new draft, so don't retry blindly.
3. `newsletter_update_broadcast {broadcastId, subject, preheader?, bodyHtml, segmentId?, subjectB?, trackOpens?, trackClicks?, ...}`.
   Drafts only. If it returns `cleared: true`, any earlier prepare is void.
4. `newsletter_preview_broadcast {broadcastId}` - check subject, HTML and text.
5. `newsletter_send_test {broadcastId}` - goes only to the user who connected the agent. Recommended.
6. `newsletter_prepare_broadcast {broadcastId}` - fixes the recipient list and runs the checklist.
   Read `recipientCount`, `excludedCount`, `daysNeeded`, `todayRoom`, `checklist.ready`, `checklist.firstBlocker`.
   **Show the user the recipient count and `daysNeeded` and get a clear yes before scheduling.**
7. `newsletter_schedule_broadcast {broadcastId, preparedHash, operationId, sendAt?}`
   - `operationId`: a new caller-chosen id (16-64 chars, `[A-Za-z0-9._:-]`). After a timeout or
     network error, retry with the **same** `operationId` - it replays, it never sends twice.
   - `sendAt`: ISO time **with `Z` or an offset**, within one year. Leave it out to send now.
8. Watch with `newsletter_get_broadcast_report {broadcastId}` (`status`, `progress`, clicks, bounces).
9. Optional: `newsletter_set_broadcast_archive {broadcastId, published: true}` after the first email went out.
   The page is public only if the publication's `archivePublic` is on too.

Undo: `newsletter_unschedule_broadcast` (scheduled -> draft; next schedule needs a new `operationId`),
`newsletter_cancel_broadcast` (stops the rest; sent email stays sent), `newsletter_duplicate_broadcast` (copy as new draft).

### Checklist rows that block a send

`postal_address`, `mailbox` (connected + Newsletter access), `audience` (at least 1 recipient),
`unsubscribe`, `content` (subject + visible body), `send_limit` (only with a monthly limit),
`click_tracking` (only with a Subject B or a poll). `subject_length` and `test_send` are advice only.
Any edit after prepare makes schedule fail with `stale` - prepare again and use the new hash.

## Writing content

- `bodyHtml` is **HTML** (max 512 KB). No markdown. Use inline `style`. Allowed tags: `p br strong b em
  i u s a ul ol li blockquote h1 h2 h3 span div img hr table thead tbody tr td th`. `class`, `<style>`,
  scripts and `<html>/<head>/<body>` are stripped.
- A button the editor can edit is `<table rel="button"><tbody><tr><td rel="button"><a rel="button"
  href="…">Label</a></td></tr></tbody></table>`. Style it with extra `rel` words, the same on all three
  elements, in this order: `outline` or `text`, then `square` or `pill`, then `center` or `right`
  (e.g. `rel="button outline pill center"`). No word = filled, rounded, left. The colour is the brand accent.
- Merge tags: `{{firstName}}`, `{{lastName}}`, `{{email}}`, `{{company}}`, `{{title}}`,
  `{{customFields.key}}`, `{{company.key}}`. Fallback: `{{firstName | "there"}}` (not inside an HTML attribute).
  Subject and preheader are personalized too.
- `{a|b}` is **spintax** (picks one at random). Don't write literal braces with a pipe.
- Unsubscribe link, preferences link, postal address and List-Unsubscribe headers are added to every
  email automatically. Don't add your own footer. `{{unsubscribe}}` adds an extra inline link if wanted.
- Images: no upload tool. Use absolute `https://` URLs. Uploads and the logo are web app only.
- Subject A/B: set `subjectB` (needs click tracking). A test group gets A or B first, then the subject
  with more verified clicks wins for the rest.

## How sending works (set expectations)

- One broadcast = one mailbox. Automations send from the publication's default mailbox.
- The worker sends at most ~20 emails per mailbox per minute, spaced ~2.5 s apart.
- Newsletter is **low priority** in the mailbox's shared daily budget: it gets what is left after
  Outreach campaigns, warmup and transactional email. A big list from one small mailbox takes **days**.
  `daysNeeded` from prepare tells you how many. More mailboxes = faster.
- If the provider throttles (SMTP 421 / 4.7.x, Graph 429) the mailbox pauses 1-60 minutes, or 4 hours
  for a quota error, then resumes. Nothing is lost.
- People who became inactive or blocked after prepare are `skipped` at send time.

## Unsubscribes and the Blocklist

- A newsletter unsubscribe or complaint is a **subscriber status** for that one publication. Other
  publications keep sending. It **also stops Outreach campaign email** to that address.
- An agent **cannot** re-subscribe someone who unsubscribed, bounced or complained (`not_resubscribable`).
  Only the reader can, from the preferences page or a form.
- The workspace **Blocklist** (shared with Outreach) blocks every newsletter send, import, upsert and test.
  A hard bounce adds the address to it.
- `newsletter_unsubscribe_subscriber {publicationId, email}` unsubscribes.
  `newsletter_delete_subscriber {subscriptionId}` hard-deletes with consent history - **cannot be undone**; ask first.

## Attributes (custom fields)

Attributes are custom fields on the contact, such as `plan`, `company_size` or `trial_ends`. Contacts are
shared with Outreach, so a value set here shows there too. Use them in segments (`custom_field` rule),
in merge tags (`{{customFields.plan}}`), in automation conditions (`custom_field_equals`) and in the
`field.matches` automation trigger.

- **Set or clear on existing subscribers:**
  `newsletter_set_attributes {publicationId, emails: [...], customFields: {...}}`
  ```json
  { "publicationId": "...", "emails": ["ana@example.com", "li@example.com"],
    "customFields": { "plan": "paid", "seats": 5, "trial": false, "old_key": null } }
  ```
  - 1-500 addresses per call; call again for more.
  - Values are text, number, `true`/`false` or `null` (clears the field). Write dates as text `YYYY-MM-DD`.
  - Only the keys you send change; every other field stays.
  - A key the workspace has never used becomes a new custom field (listed in `createdKeys`, max 200 per
    workspace). Its type comes from the first value you send: number -> number, `true`/`false` ->
    boolean, `YYYY-MM-DD` -> date, anything else -> text. Later values must fit that type, so send the
    right kind the first time. Check the spelling of existing keys first, so you don't create `Plan`
    next to `plan`.
  - A bad key, or a value that doesn't fit the field's type, fails the whole call before anything is written.
  - Returns `requested`, `matched`, `changed`, `notFound` (not a subscriber of this publication),
    `invalid` (not an email address) and `createdKeys`. Safe to repeat.
  - An automation with a `field.matches` trigger picks the change up within a minute.
- **New subscribers:** `newsletter_upsert_subscriber {..., customFields}` writes them only when that call
  creates the contact. For a CSV, map a column to `custom:<key>`.

## Automations

1. `newsletter_create_automation {publicationId, trigger?}` - makes a draft "Welcome sequence" with one
   empty email step. Returns the full tree with `revision`.
2. `newsletter_update_automation {automationId, revision, name, exitGoal, trigger?, reentry?, steps[]}` -
   saves the **whole** tree. Always send the latest `revision`.
   - Steps (max 50): `email` (subject, preheader, bodyHtml), `wait` (1-525,600 minutes),
     `condition` (`has_tag`, `in_list`, `custom_field_equals`, `clicked_any_email`; yes/no branches,
     not nested), `tag`, `untag`, `webhook`, `exit`.
   - Trigger and re-entry change **only while draft**. After activation only content can change.
3. `newsletter_validate_automation {automationId}` - shows the checklist and the tree. Activates nothing.
4. `newsletter_activate_automation` / `newsletter_pause_automation` (scope `newsletter:automations`).
5. `newsletter_enroll {automationId, subscriptionId, eventKey}` - put one active subscriber in by hand.

Triggers: `subscriber.confirmed` (default), `tag.added`, `list.joined`, `date.annual`, `date.specific`,
`api.event`, `stripe.event`, `field.matches`. Free allows only `subscriber.confirmed` with re-entry `never`.

- `field.matches`: `{ "type": "field.matches", "fieldKey": "plan", "op": "equals", "value": "paid", "includeExisting": false }`.
  `op` is `equals`, `not_equals` or `is_not_set` (value `""`). `includeExisting: true` also enrolls
  people who already match. Checked every minute.
- **Imports and MCP upserts never fire `subscriber.confirmed`.** For imported people use `list.joined`
  or `newsletter_enroll`.
- Date triggers fire at 09:00 in the publication's time zone.

## Webhooks (admin only)

`newsletter_create_webhook {url, events}` - `events` is `"*"` or a comma list of `subscriber.created`,
`subscriber.confirmed`, `subscriber.unsubscribed`, `broadcast.sent`, `automation.completed`,
`automation.webhook`, `transactional.sent`, `transactional.failed`. The signing secret is shown **once**;
hand it to the user. `newsletter_list_webhooks`, `newsletter_delete_webhook`.

## Diagnose

| Symptom | Check |
|---|---|
| `no_mailbox`, or `mailbox` checklist row fails | `newsletter_list_mailboxes`. Empty = Newsletter access not ticked (web app). `connected: false` = reconnect in web app. |
| 0 recipients | `newsletter_list_subscribers` `counts.byStatus` (many `pending`?). Segment via `newsletter_preview_segment` + `status is active`. Overlapping `excludeListIds`. `excludedCount` = blocked addresses. |
| `stale` on schedule | Something changed after prepare. Prepare again, use the new `preparedHash`. |
| `operation_conflict` | `operationId` belongs to an older prepare or an unscheduled send. Use a new one. |
| Stuck at `sending` | Report `progress`: `queued` falling slowly = daily budget (compare `daysNeeded`) or a throttle pause. No movement = mailbox disconnected, postal address cleared, or plan lost Pro. |
| Scheduled at the wrong time | `sendAt` sent without `Z` or an offset. |
| One person got nothing | `newsletter_get_subscriber_status` (must be `active`). Blocklist (web app). Recipients are fixed at prepare - added later = not included. |
| Import/upsert did nothing | Already existed, blocked, unsubscribed, or over the plan cap (`newsletter_get_usage`). Error counts in `newsletter_get_import`. |
| `newsletter_set_attributes` changed 0 | `notFound` = those addresses aren't subscribers of this publication. `invalid` = not email addresses. Already had those values = nothing to change. |
| Automation enrolls nobody | Not active (`newsletter_validate_automation`). Imported people + `subscriber.confirmed` trigger. `includeExisting: false`. Re-entry `never`. Free plan. |

## Web app only

Tell the user to do these in the Terlido web app: connect mailboxes and grant Newsletter access; DNS
(SPF/DKIM/DMARC) and own domain; signup forms, pop-ups, magic links; image and logo uploads; the
visual editor; polls; RSS-to-email; Sendy migration; bulk unsubscribe/delete and CSV export; Blocklist
editing; Stripe and API keys; renaming a publication; creating tags; changing a custom field's type
or label.

## Rules

- Never import or add people without consent, and never pick a `consentSource` that isn't true.
- Get a clear yes from the user before `newsletter_schedule_broadcast`, `newsletter_cancel_broadcast`,
  `newsletter_delete_subscriber`, `newsletter_unsubscribe_subscriber` and `newsletter_activate_automation`.
- Reuse the same `operationId` / `eventKey` when retrying after an uncertain result. Never invent a new
  one to "try again".
- Report numbers from the tools as they are. Opens are estimates; clicks are verified.
