---
name: terlido-outreach
description: How Terlido Outreach (cold email) works and how to set it up correctly through the Terlido MCP server - mailboxes, warmup, DNS, contacts, lists, segments, campaigns, sequences, schedules, senders, safe enrollment and launch, the review queue, inbox replies, and diagnosing campaigns that don't send. Use whenever an agent works with Terlido Outreach or tools like `create_campaign`, `set_campaign_sequence`, `launch_campaign`, `import_leads`, `send_reply` - "set up a cold email campaign", "add these leads", "launch it", "why is my campaign not sending" - even if the word "skill" is never used.
---

# Terlido Outreach

Terlido Outreach sends cold email sequences from the workspace's **own connected mailboxes**. Each
mailbox has one daily send budget shared by warmup, Outreach and Newsletter. Contacts, tags, custom
fields and the Blocklist are shared with Terlido Newsletter.

## Start every session

1. Call `whoami`. It returns the workspace and the granted `scopes`.
   - Read scopes: `campaigns:read`, `leads:read`, `sequences:read`, `inbox:read`, `mailboxes:read`, ...
   - Write scopes: `campaigns:write`, `leads:write`, `sequences:write`, `lists:write`, `tags:write`, ...
   - Sensitive: `campaigns:launch` (launch, review), `inbox:send` (real email), `warmup:write`.
   - A missing scope means the user must reconnect the agent and tick it. Don't work around it.
2. Read before you write: `list_mailboxes`, `list_campaigns`, `list_custom_fields {entity: "lead"}`.
3. Limits: 100 requests/minute. 50 agent emails (`send_reply` + `send_email`) per UTC day.
   Campaign sends don't count against the 50.

## Words

- **Mailbox** - Gmail, Microsoft or IMAP/SMTP. `dailyVolume` is its total daily budget
  (warmup + Outreach + Newsletter). A disconnected mailbox pauses its campaigns.
- **Warmup** - automatic peer sending that builds reputation. Needs no campaign.
- **Contact (lead)** - email unique per workspace, names, title, company, phone, timezone,
  `emailStatus` (`valid|risky|unknown|invalid`), custom fields, tags.
- **List** - static group. **Linking** a list to a campaign auto-enrolls people added later.
- **Segment** - saved filter. Enrolling from it is a one-time snapshot.
- **Campaign** - `draft` -> `running` <-> `paused` -> `finished` (final). Every campaign write bumps
  its `editRevision`; always pass the latest one.
- **Sequence** - ordered steps: `email`, `condition`, `call`, `manual_task`, LinkedIn steps (become
  manual tasks), `whatsapp_message`, `api_call`, `send_to_campaign`.
- **Schedule** - send days and hours in an IANA timezone, applied in **each contact's own timezone**
  when known. No schedule = Mon-Fri 08:00-18:00 in the workspace timezone.
- **Senders** - mailboxes the campaign rotates through. Each contact keeps the same sender.
- **Blocklist (Suppression)** - emails, domains or phones that never get mail. Unsubscribes, hard
  bounces and newsletter opt-outs also stop campaign email.

## Setup order for a first campaign

Steps marked **web app** cannot be done over MCP - tell the user exactly what to do.

1. **Mailbox (web app).** Connect Gmail, Microsoft or IMAP/SMTP. Confirm with `list_mailboxes`:
   `connected: true` and `lastError` null.
2. **DNS (web app / DNS provider).** SPF, DKIM and DMARC for every sender domain
   (Deliverability -> Domains & DNS).
   - SPF or DMARC failing **blocks launch** and auto-pauses a running campaign.
   - Missing DKIM **throttles** that mailbox to 10 Outreach emails a day until fixed.
3. **Warmup.** `set_warmup_settings {mailboxId, warmupEnabled: true, dailyVolume, sendWindowStart, sendWindowEnd}`.
   A new mailbox should warm for a few weeks before real volume. Tell the user this.
4. **Tracking domain (web app, recommended).** A custom tracking domain (CNAME) helps deliverability.
5. **Contacts.**
   - Check fields first: `list_custom_fields {entity: "lead"}`.
   - Many: `import_leads {leads[1..500], dedupeMode: "skip"|"update", verifyOnImport?, leadListId?}` (runs in the background).
   - One: `create_lead {email, firstName, lastName, company, title, customFields}`.
   - Read one: `get_lead {leadId}` (includes custom fields).
   - Change: `update_leads` (max 200 rows; custom field keys must already exist).
   - Optional verify: `enrich_lead {kind: "verify"}`.
   - **Tag every new contact with the product it was imported for** - see "Product tags" below.
   - Organise: `create_lead_list` + `add_leads_to_list`, `create_tag` + `tag_leads`, `search_leads`.
6. **Campaign.** `create_campaign {name}` -> draft. Get `editRevision` from `list_campaigns` or `get_campaign`.
7. **Sequence.** `set_campaign_sequence {campaignId, expectedRevision, steps[]}` (see below). Keep the new `editRevision`.
8. **Senders.** `set_campaign_senders {campaignId, expectedRevision, mailboxes: [{mailboxId, dailyCap, enabled}]}`.
   Replaces the whole pool. Every enabled mailbox must be connected. `dailyCap` default 40.
9. **Schedule.** `set_campaign_schedule {campaignId, expectedCampaignRevision, expectedScheduleRevision, name, timezone, days, startMinute, endMinute}`.
   Pass `expectedScheduleRevision: null` when the campaign has no schedule yet. `list_schedules`
   shows existing schedules with their `editRevision` and how many campaigns share each. Minutes count from
   midnight (09:00 = 540). If the schedule is shared with other campaigns, it refuses unless
   `applyToSharedSchedule: true` - ask the user first.
10. **Options.** `update_campaign {campaignId, expectedRevision, ...}`:
    `dailyLimit` (default 120), `maxNewLeadsPerDay` (default 40), `domainThrottle` (default 2 per
    company domain per day), `maxBounceRatePct` (default 5, auto-pauses above it), `stopOnReply`
    (default on), `stopOnClick`, `trackOpens`, `trackClicks`, `reviewBeforeSend`.
11. **Enroll** - guarded flow below.
12. **Optional inbox test.** `run_inbox_check {campaignId, senderMailboxId, receiverMailboxId, subject, bodyHtml, variables}`
    sends one real email (20 per 24 h), then poll `get_inbox_check` for spam score and folder.
13. **Launch** - guarded flow below. Or `schedule_campaign_launch`.

## Product tags

One workspace often sells more than one product. Every contact you add must carry a tag that says
which product it was imported for, so the user can filter, segment and report by product later.

- **Name:** `Product: <Name>`, e.g. `Product: PageDuel`. One tag per product. Use the same spelling
  every time (tag names are unique, ignoring case, max 60 characters).
- **Which product:** take it from the user's request. If it isn't clear, ask. Never guess.
- **Find or create the tag:** `list_tags` first and reuse the existing `Product: ...` tag. Only call
  `create_tag {name: "Product: <Name>"}` if none exists.
- **Tag the contacts:**
  - `create_lead` returns the lead id - call `tag_leads {tagId, leadIds}` right after.
  - `import_leads` has no tag field and runs in the background. Pass a `leadListId`, then page
    through `get_lead_list` for the member ids once the import is done, and call `tag_leads` in
    batches of up to 500. People skipped as duplicates also join the list, so they get the tag too.
  - A contact that already has another product's tag keeps it and gets this one as well. Never
    remove a product tag, except to fix one you just put on by mistake (`untag_leads {tagId, leadIds}`,
    or `untag_campaign {campaignId, tagId}` for a campaign).
- **Tag the campaign too:** `tag_campaign {campaignId, tagId}` with the same tag, so
  `list_campaigns {tagIds}` shows each product's campaigns.
- **Check:** `search_leads {tagIds: [tagId]}` should return the imported contacts.

## Sequence format

`set_campaign_sequence` **replaces the whole sequence**. A step you leave out is deleted with its stats.
Read `get_campaign` first and reuse step ids and variant ids when editing.

To draft a sequence instead of writing it by hand: `generate_sequence {channels, goal?, stepCount?,
tone?, icp?, offer?}` - 1 AI credit, saves nothing. It returns `steps` already in the
`set_campaign_sequence` shape (variables with fallbacks, `{{unsubscribe}}` at the end of each email)
and a `spamSignalCount`. Show the user the copy, then pass the steps to `set_campaign_sequence`.

```json
{ "id": "step-1", "orderIndex": 0, "type": "email", "waitDays": 0,
  "parentStepId": null, "parentBranch": null,
  "variants": [
    { "id": "v-a", "label": "A", "subject": "Quick question, {{firstName}}", "bodyHtml": "<p>Hi {{firstName | \"there\"}},</p>...", "weight": 100 },
    { "id": "v-b", "label": "B", "subject": "Idea for {{company}}", "bodyHtml": "<p>...</p>", "weight": 100 }
  ] }
```

- **Linear sequence: every step has `parentStepId: null` and `parentBranch: null`, ordered by
  `orderIndex`.** Do NOT chain steps by pointing `parentStepId` at the previous step. Use
  `parentStepId` + `parentBranch: "yes"|"no"` only to put a step inside a `condition` branch.
- `waitDays` (0-365) is the wait **before** that step.
- **Follow-up in the same thread:** `subject: null`. Only allowed when an email was already sent on
  every path into that step.
- **A/B test:** 1-20 `variants` with relative `weight`. Each contact always gets the same variant.
  Aim for 100+ sends per variant before judging.
- **Conditions:** `conditionType` = `opened_email`, `clicked_link`, `replied`, `has_linkedin_url`,
  `booked_meeting`, `custom_field {field, operator, value}`, `email_status {status}`, `has_score {operator, threshold}`.
- **Running campaign:** only text edits to existing variants. For anything else, `pause_campaign`
  first, edit, then `launch_campaign` to resume.

## Personalization

- `{{firstName}}`, `{{lastName}}`, `{{email}}`, `{{company}}`, `{{title}}`, `{{senderName}}`,
  `{{senderFirstName}}`, `{{customFields.key}}` (or bare `{{key}}`), `{{company.key}}` (company field).
- Fallback: `{{firstName | "there"}}`. Without one, a missing value becomes empty text - always add a
  fallback for optional fields.
- Spintax: `{Hi|Hello|Hey}` - one option per contact, stable.
- `{{unsubscribe}}` places the opt-out link. Without it a footer is added. One-click unsubscribe
  headers are always sent.
- Body is HTML and gets sanitized (scripts, styles, event handlers removed). Keep cold email short
  and plain - few links, no images, no attachments on the first email.

## Safe enrollment and launch

Always use the guarded tools for automation. They stop if anything changed since the user approved.

1. `get_campaign_execution_scope {campaignId}` -> keep `campaign.editRevision`, `campaign.status`, `scopeHash`.
   Show the user what will happen: senders, schedule, daily limits, number of contacts.
2. Get the user's clear yes.
3. `add_campaign_leads_guarded {campaignId, leadIds[<=500], expectedRevision, expectedStatus, expectedScopeHash, operationId}`.
   All or nothing. If the answer is unclear (timeout), call `check_campaign_leads {campaignId, leadIds}`
   and retry with the **same** `operationId` only if it says `complete: false`.
   On a busy running campaign the hash moves often - read the scope again and retry; if settings
   changed, ask the user again.
4. `get_campaign_execution_scope` again (enrolling changed the hash), then
   `launch_campaign {campaignId, expectedRevision, expectedStatus: "draft", expectedScopeHash, operationId}`.
   Send all four guard fields or none.

Other ways to enroll (no guard): `add_campaign_leads` with exactly one of `leadIds`, `listId` (links
the list - future members auto-enroll) or `segmentId` (snapshot). Already-enrolled people are skipped.

Launch at a set time: `schedule_campaign_launch {campaignId, at: {year, month, day, hour, minute}}` -
wall-clock time in the **workspace timezone**, not ISO. `at: null` clears it.

If launch is refused, the reason is one of: `no_sequence`, `invalid_sequence_content`,
`no_connected_sender`, `handoff_target_unavailable`, `sender_domain_unauthenticated`,
`no_actionable_contacts`, `no_sendable_contacts`, a plan limit, or the active-campaign limit.

## Running a campaign

- Pause: `pause_campaign {campaignId, reason?}`. Resume: `launch_campaign`.
- Stop for good: `finish_campaign` - **cannot be resumed**. Ask first.
- `delete_campaign` - permanent, admin only, pause or finish first. Ask first.
- `duplicate_campaign` - new draft with sequence, settings, schedule and senders (no contacts).
- Progress: `get_campaign_report`, `list_campaign_leads {campaignId, status?}` (shows `nextActionAt`,
  `pauseReason`, `statusNote`), `get_workspace_analytics {days}`.
- Remove people: `remove_campaign_leads` (only that campaign). Stop list auto-enroll: `unlink_campaign_list`.
- Block someone everywhere: `suppress_lead {value, type: "email"|"domain"|"phone", reason}` -
  there is **no MCP tool to undo it**. Ask first.

### Review before send

With `reviewBeforeSend: true`, each email waits as `needs_review` - the campaign shows "running" but
sends nothing until approved.
- `list_review_queue {campaignId}` shows the rendered emails.
- `review_job {campaignId, jobId, action: "approve"|"skip", editedSubject?, editedBody?}` one by one.
- `approve_review_jobs {campaignId, jobIds[<=200], dispatch: "schedule"}`. `dispatch: "now"` skips the
  send window and spacing - only if the user asks.
- `update_campaign {reviewBeforeSend: false}` releases everything already waiting.

## Lists, tags and segments

- Lists: `list_lead_lists` (with contact counts), `get_lead_list {leadListId, cursor?, limit<=100}`
  (members, paged), `create_lead_list`, `rename_lead_list {leadListId, name}`,
  `add_leads_to_list` / `remove_leads_from_list {leadListId, leadIds[<=500]}`.
  `delete_lead_list {leadListId}` deletes the list only - the contacts stay. Removing someone from a
  list linked to a campaign does **not** unenroll them.
- Tags (shared by contacts, companies and campaigns): `list_tags`, `create_tag {name, color?}`,
  `tag_leads` / `untag_leads {tagId, leadIds[<=500]}`, `tag_campaign` / `untag_campaign {campaignId, tagId}`.
- Segments: `list_segments` shows saved filters with live counts and `brokenRules` (a rule on a
  deleted field matches nothing). Segments are created in the web app; enroll one with
  `add_campaign_leads {segmentId}`. For a one-off filter use `search_leads {query?, emailStatus?, tagIds?, filter?}`.

## Companies

Contacts link to a company by name.
- `list_companies {search?, cursor?, limit<=100}`, `get_company {companyId}`.
- `update_company {companyId, ...}`: `name`, `domain`, `website`, `status` (`new`, `lead`, `contacted`,
  `opportunity`, `customer`, `evangelist`), `industry`, `size`, `location`, `hq`, `specialties`,
  `tagline`, `founded`, `linkedinUrl`, `ownerId`, `aiContext` (notes about the prospect the AI uses
  when writing to them), `customFields`. `null` clears a field.
- `add_company_note {companyId, body}` - notes **cannot be edited or deleted** over MCP.

## Tasks

Calls, LinkedIn steps and manual steps in a sequence create tasks; you can also add your own.
- `list_tasks {status?, ownerId?, leadId?, campaignId?, limit?}` - without `status` it returns what
  needs doing now (open, plus snoozed tasks that are due again), earliest due first.
- `create_task {type, title, note?, url?, priority?, ownerId?, leadId?, dueAt?}` - `type` is `call`,
  `email`, `linkedin`, `whatsapp` or `manual`; `priority` is `low`, `normal` (default), `high` or
  `urgent`; `dueAt` is an ISO datetime. The owner defaults to the connected user.
- `update_task {taskId, status?, outcome?, snoozeUntil?, title?, note?, url?, ownerId?, dueAt?}` -
  `status` is `open`, `done`, `ignored` or `snoozed`. `outcome` is `connected`, `voicemail`,
  `no_answer`, `wrong_number`, `interested`, `not_interested` or `replied`; the last three **stop the
  contact's sequence**. Changing someone else's task needs its owner or an admin.
- To never contact someone again use `suppress_lead`, not a task outcome.

## Templates

Reusable copy for sequences and replies.
- `list_templates {type?: "email"|"linkedin"|"call"}`, `get_template {templateId}`.
- `create_template {name, subject?, bodyHtml}` - email templates; the HTML is sanitized.
- `update_template {templateId, name?, type?, subject?, bodyHtml?}` - email, LinkedIn or call.
- `set_campaign_sequence` takes text, not a template id: read the template with `get_template` and
  put its subject and body into the step's variant.

## Webhooks

`create_webhook {url, events}` - `events` is `"*"` or a comma list of `lead.replied`,
`lead.interested`, `lead.unsubscribed`, `lead.bounced`, `lead.finished`, `reply.created`,
`task.created`, `task.completed`, `import.completed`, `webhook`, `deliverability.alert.fired`,
`deliverability.alert.resolved`. The signing secret is returned **once** - give it to the user.
`delete_webhook {webhookId}` is permanent; ask first.

## Inbox

- `list_inbox_threads {status?, intent?}`, `get_inbox_thread {threadId}`.
- `draft_reply {threadId, instructions?}` - AI draft, costs 1 AI credit, sends nothing.
- `send_reply {threadId, body, operationId}` - sends **immediately** (no 30-second undo like the app).
  Plain text; `{{firstName}}`, `{{senderFirstName}}` and other variables are filled in.
- `send_email {leadId, mailboxId, subject, body, operationId}` - new 1:1 thread, plain text, no attachments.
  Any connected mailbox works, also a Newsletter-only one; Outreach access is only needed for campaigns.
- Both `send_reply` and `send_email` tag the message as an MCP send. In the app, a thread waiting on
  an answer shows in Inbox > **Sent** with an **MCP** chip, so the user can see what the agent sent.
- Always pass your own `operationId` and reuse it after a timeout. Check with
  `get_reply_operation {threadId, operationId}` (`sent`, `sending`, `delivery_unknown`).
  **Never retry an unclear send with a new id** - the person gets it twice.
- Show the user the exact text and get a yes before any send.

## Diagnose "running but not sending"

| Cause | Check |
|---|---|
| Waiting for review | `get_campaign` -> `reviewBeforeSend`; `list_review_queue` has items. |
| Outside send hours | `list_campaign_leads` -> `nextActionAt` in the future. Schedule is per contact timezone, Mon-Fri by default. |
| Daily limits used up | `dailyLimit`, `maxNewLeadsPerDay`, sender `dailyCap`; `list_mailboxes` -> `outreachUsedToday` vs `dailyVolume` (shared with warmup and Newsletter). Rolls to next day. |
| Mailbox problem | `list_mailboxes` -> not `connected`, `lastError`. Reconnect in the web app, then `launch_campaign`. |
| Auto-paused | `get_campaign` -> status `paused` + `pausedReason` (bounce rate, or SPF/DMARC failure). Fix, then `launch_campaign`. |
| Sender resting | `list_mailboxes` -> `restingSince` set, with `restReason`: the mailbox lands in spam, so campaigns skip it (warmup still runs). A campaign whose only senders rest shows `pausedReason` "Auto-paused · all senders resting". The user resumes the mailbox in the web app (best once `recoveryReady` is true), then `launch_campaign`. |
| DKIM missing | Only 10/day per mailbox. Check the Deliverability tab (web app). |
| Contacts skipped | `list_campaign_leads {status: "skipped"}` -> `pauseReason`/`statusNote`; `list_suppressions`; `emailStatus` invalid. |
| Many people at one company | `domainThrottle` (default 2 per domain per day). |
| Contacts stopped | Status `replied`, `bounced`, `unsubscribed`, or `paused` (out of office). |

## Web app only

Tell the user to do these in the Terlido web app: approve or revoke agent access; connect or
reconnect mailboxes; DNS checks and custom tracking domain; enrichment provider keys; Blocklist
removal and the unsubscribe footer; creating custom-field definitions and saved segments; campaign
options `stopOnMeeting`, `pauseCompanyOnReply`, `skipEmailStatuses`; CSV file import with mapping;
attachments; fixing `delivery_unknown` items and retrying single failed contacts; deleting contacts;
billing and plan.

Lead-finding tools (`get_enrichment_capabilities`, `preflight_enrichment_accounts`,
`search_professional_roles`, `verify_exact_emails`) **never return email addresses**. Addresses must
come from the user's own source; then `import_leads` or `create_lead`.

## Rules

- Only contact people the user has a legitimate reason to email. Never invent or guess addresses.
- Get a clear yes before: enrolling, launching, scheduling a launch, any real send, `finish_campaign`,
  `delete_campaign`, `suppress_lead`, approving review jobs, and changing a shared schedule.
- Always pass the latest `editRevision`. On a revision conflict, read again - don't overwrite.
- Reuse the same `operationId` when retrying an unclear result.
- Suggest safe volume: warm mailboxes first, start around 20-40 new contacts a day per mailbox,
  watch bounces and replies, then grow.
