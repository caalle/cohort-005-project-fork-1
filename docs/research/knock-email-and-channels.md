# How Knock handles email and coordinates Channels

Research for issue #3, part of the map in issue #1. Researched 2026-11-10 from public documentation only; nobody has a Knock account during planning, so nothing here was tested.

**Terms.** Student, Notification and Channel are used as in `GLOSSARY.md`. Knock's own vocabulary differs: its "recipient" is a Student here, and a Knock "channel" is one configured provider connection (for example, one Postmark setup), which is narrower than a Cadence Channel. Knock's word is used only for its product features and field names.

**The legal sections are a reading of regulator material for planning purposes, not legal advice.**

## Answer in brief

- **Postmark:** Yes. Knock has a first-party Postmark integration that needs a Postmark server API key, a from address and a verified sender signature or domain. Knock sends on Postmark's `outbound` transactional stream unless a payload override sets `MessageStream`.
- **Skipping email if the in-app Notification was read:** Yes. Use an in-app step, then a delay step, then an email step whose condition is `refs.<in_app_ref>.engagement_status not_contains $message.read`. The delay is required, because conditions are evaluated at the moment the step runs.
- **Batching and digests:** Batch steps group activity per Student over a window. Knock says batching is not a substitute for recurring digests, which it builds from schedules plus a fetch step calling an HTTP endpoint on your side. Whether per-event in-app delivery combines with batched email and a read check is **not documented**.
- **Preferences:** Per Channel type, per specific Knock channel, per workflow and per category. Separately, a single `commercial_subscribed` flag covers all commercial messages and is what the unsubscribe link changes. It is not per category or per Channel.
- **Email rules:** The line between commercial and transactional depends on the message content, and the US and UK tests differ. A message that encourages a Student to come back is at real risk of being marketing, particularly under UK guidance. Cadence should plan on treating re-engagement email as commercial.

## 1. Sending through Postmark

### What it requires

Configured per environment under the Postmark channel in Knock ([Knock: Postmark](https://docs.knock.app/integrations/email/postmark)):

| Setting | Notes |
| --- | --- |
| API key (required) | The key for a Postmark server. |
| From email address (required) | Supports Liquid. |
| Recipient `email` (required) | The Student must exist in Knock with a valid `email`. |
| Sandbox mode, open and link tracking, CSS inlining | Optional. CSS inlining defaults to on. |

Postmark itself needs a verified sender signature or domain, which is also a listed prerequisite for delivery webhooks. Provider credentials do not support dynamic values ([Knock: email settings](https://docs.knock.app/integrations/email/settings)).

### Delivery status

Delivery status needs a webhook set up by hand: turn on Incoming webhook in Knock, then add that URL in Postmark on the stream in use, for Delivery, Bounce and Spam complaint events. Delivery maps to `delivered` and Bounce to `bounced`. A spam complaint only records a `message.complaint` event and does not change delivery status. Postmark deactivates the address after a complaint, so later sends bounce ([Knock: Postmark](https://docs.knock.app/integrations/email/postmark)).

Knock retries provider failures up to 8 times over 30 minutes ([Knock: delivering notifications](https://docs.knock.app/send-notifications/delivering-notifications)).

### Message streams, the main decision here

Knock does not target a stream, and Postmark falls back to `outbound` when none is given ([Knock: Postmark](https://docs.knock.app/integrations/email/postmark), [Postmark: email API](https://postmarkapp.com/developer/api/email-api)). Another stream takes a payload override at channel or step level: `{ "MessageStream": "broadcasts" }`.

Postmark describes transactional as one recipient, triggered by something a user does or doesn't do, and lists comment notifications as an example. It describes broadcast as multiple recipients on your schedule, and lists product announcements, release notes and **daily digests** ([Postmark: message streams](https://postmarkapp.com/message-streams)). So a single per-event Notification looks transactional to Postmark, and a digest looks like broadcast. Postmark's page does not explicitly ban promotional content on transactional streams, and describes no penalties for misclassification.

The two streams behave differently on unsubscribe:

| | Transactional (`outbound`) | Broadcast |
| --- | --- | --- |
| Unsubscribe link | Optional ([Postmark](https://postmarkapp.com/support/article/1208-how-to-add-an-unsubscribe-link)) | Required, and auto-inserted if absent ([Postmark](https://postmarkapp.com/support/article/1217-why-broadcasts-require-an-unsubscribe-link)) |
| `List-Unsubscribe` headers | Passed through if you add them | Added automatically and not editable while Postmark manages unsubscribes ([Postmark](https://postmarkapp.com/support/article/1299-how-to-include-a-list-unsubscribe-header)) |
| Where an opt-out is recorded | With Knock's link: Knock | Postmark's per-stream Suppressions, unless self-managed |

**Consequence for a broadcast stream.** Sending Knock commercial email on a Postmark-managed broadcast stream would create two unsubscribe mechanisms and two records of opt-out. Postmark's suppression would not update `commercial_subscribed` in Knock. The ways out are:

1. Ask Postmark support to approve "Manage unsubscribes on your own" for the stream, so Knock is the only record. Cadence must then ensure every message has a visible link and the headers ([Postmark](https://postmarkapp.com/support/article/1299-how-to-include-a-list-unsubscribe-header)).
2. Keep Postmark-managed unsubscribes and sync them to Knock using Postmark's Subscription Change webhook ([Postmark](https://postmarkapp.com/support/article/1208-how-to-add-an-unsubscribe-link)).

**Keeping refund email safe.** Postmark suppressions are per stream, and Knock recommends a dedicated channel and, where possible, a separate sending domain or subdomain for marketing mail, to protect transactional reputation ([Knock: broadcasts](https://docs.knock.app/concepts/broadcasts)). That advice is written about frequent or high-volume marketing sends. Sharing `outbound` between Notifications and refund email would mean a complaint about a Notification could suppress a Student's refund email. A separate stream avoids that.

## 2. Skipping email if the in-app Notification was read

Supported through message status conditions ([Knock: step conditions](https://docs.knock.app/designing-workflows/step-conditions)):

```text
1. In-app feed step   (ref: in_app)
2. Delay step         (fixed interval, or a time window)
3. Email step, with condition:
     variable: refs.in_app.engagement_status
     operator: not_contains
     argument: $message.read
```

Details that shape the design:

- **The delay is required.** Knock evaluates status conditions immediately when the step runs, so the Student needs time to read first. A false condition skips that step only.
- **Use `read`, not `seen`.** The React feed marks all items seen when the popover opens by default ([Knock: React feed](https://docs.knock.app/in-app-ui/react/feed)), and `seen` means only that the item was retrieved for display ([Knock: message statuses](https://docs.knock.app/send-notifications/message-statuses)). "Mark all as read" also counts as `read`. A Student who opens the feed but doesn't open the Notification is still emailed.
- **Read state depends on how the feed is built.** Knock's React SDK sets engagement statuses; a custom in-app feed must call the message status API itself ([Knock: tracking](https://docs.knock.app/send-notifications/tracking), [Knock: message statuses](https://docs.knock.app/send-notifications/message-statuses)). Without that, nothing would ever count as read and the email would always send.
- **Keep the delay well inside retention.** If the message has passed the retention window, a status condition always evaluates false, which would skip the email. Retention is 30 days, or 90 on Enterprise ([Knock: data retention](https://docs.knock.app/manage-your-account/data-retention)).
- **Delays have limits.** A delay can run up to 365 days, and a missing dynamic value skips the delay rather than failing ([Knock: delay](https://docs.knock.app/designing-workflows/delay-function)).
- **A condition is better than cancellation here.** Cancellation stops the whole run rather than one step, needs a `cancellation_key` and should not be sent within 5 seconds of the trigger ([Knock: canceling workflows](https://docs.knock.app/send-notifications/canceling-workflows)). It does suit an event in Cadence that makes the Notification moot.

## 3. Batching and digests

### Batch steps

From [Knock: batch function](https://docs.knock.app/designing-workflows/batch-function):

- Batches group per recipient by default. A batch key adds grouping, such as per course.
- Windows can be fixed, dynamic, relative to a dynamic window, or sliding with a maximum extension. A batch can stay open at most 90 days.
- An optional activity limit (2 to 1,000) closes the batch early.
- Templates get `total_activities`, `total_actors`, and up to 10 `activities` and `actors`. Only Enterprise can raise the render limit, to 100.
- The batch's top-level data drops nested objects and truncates strings at 256 characters, so per-event data should come from `activities`.
- "Immediately flush" sends the first item at once. It is then not in `activities`, and if no second item arrives the batch does nothing when it closes.
- Later triggers' runs are terminated once they join a batch.
- Knock advises against batch steps in sequence.

### Combining batching with the read check

**The docs do not settle this.** They don't say whether steps before a batch run for every trigger, or whether a step after a batch can reference a message from before it. Since later runs are terminated when they join, a per-event in-app step followed by batched email plus a read check can't be assumed to work.

The arrangement the documented pieces do support is batch first, then in-app, delay and conditional email. The cost is that the in-app Notification waits for the batch window too, which conflicts with in-app showing what's new promptly. An in-app feed batched this way is also one aggregated item, not several.

### Recurring digests

Knock says batching isn't a substitute for recurring digests ([Knock: batch function](https://docs.knock.app/designing-workflows/batch-function)). Its pattern is a schedule per Student plus a fetch step that calls a **publicly accessible HTTP endpoint** on your service once per Student ([Knock: recurring digests](https://docs.knock.app/tutorials/building-recurring-digests)). `schedules.create` takes up to 100 recipients per call, and schedules can be per-recipient timezone. Knock also recommends your own system as the source of truth for digests at send time.

For Cadence, that means a new endpoint Knock can reach over the public internet, which would need its own access control since Cadence has no real login, and Students synced to Knock with name and email before schedules are created.

## 4. Unsubscribe and preferences

### Model

From [Knock: preferences overview](https://docs.knock.app/preferences/overview):

- A `PreferenceSet` has `channel_types` (such as `email`), `channels` (a specific Knock channel by UUID), `workflows` and `categories`. Each resolves to a boolean, and anything unset defaults to `true`, so Students are **opted in by default**.
- A Notification sends only if every applicable preference is `true`. A `false` category blocks even when the Channel is `true`, and any one `false` category blocks a workflow in several categories.
- A specific `channels` value takes precedence over `channel_types`.
- Merge order without a tenant: the Student's set, then the environment default.
- Preference conditions can be stored as well.
- "Override recipient preferences" on a workflow sends regardless of preferences; Knock intends this for cases like password resets.

### Preference centre

Knock's hosted page needs a signed URL, exposed in templates as `{{vars.manage_preferences_url}}`, and requires no login, as the token identifies the Student and "never expires" ([Knock: hosted preference center](https://docs.knock.app/preferences/hosted-preference-center)). Rows map to a category, workflow or channel, and can each have per-Channel-type toggles, so "per category, per Channel" is supported. Top-level Channel type toggles are not. Alternatively, Cadence can build its own against the preferences API.

That token-only access is worth noting alongside the map's limit that a Student's Notifications are only as private as the Cadence session: anyone holding a forwarded link can change that Student's preferences.

### Commercial unsubscribe

From [Knock: commercial unsubscribe](https://docs.knock.app/preferences/commercial-unsubscribe) and [Knock: broadcasts](https://docs.knock.app/concepts/broadcasts):

- Mark a workflow "Commercial" under Manage workflow. Knock then includes unsubscribe headers in its emails.
- `{{vars.commercial_unsubscribe_url}}` provides the link, and the visual editor can add a "1-click unsubscribe" footer link.
- A click sets `commercial_subscribed` to `false` on the Student's `default` set. That beats tenant and environment defaults and leaves transactional messages unaffected.
- The confirmation page can be customised or redirected.

So one click stops **all** commercial messages. It is a single global flag, separate from the per-category settings. A preference centre is what offers finer control.

## 5. Email rules: re-engagement vs transactional

Planning notes from regulator sources, not legal advice. Which regimes apply depends on where Students are and on Cadence's legal position, which the sources can't determine.

### United States: CAN-SPAM (FTC)

- It looks at a message's primary purpose. Commercial means advertising or promoting a product or service ([FTC guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)).
- Transactional or relationship content is limited to specific categories. Those closest to Cadence are confirming an agreed transaction, notifying of a change in terms or features of an ongoing subscription or account, and delivering goods or services, including updates, the recipient is entitled to under a prior agreement ([16 CFR 316.3](https://www.govinfo.gov/content/pkg/CFR-2024-title16-vol1/xml/CFR-2024-title16-vol1-part316.xml), 2024 edition).
- Having an existing relationship doesn't by itself make a message transactional ([FTC guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)).
- A dual-purpose message is commercial if its subject line would suggest advertising to a recipient, or if the transactional content doesn't come at the beginning of the body ([16 CFR 316.3](https://www.govinfo.gov/content/pkg/CFR-2024-title16-vol1/xml/CFR-2024-title16-vol1-part316.xml)).
- Commercial mail needs accurate headers and subject lines, identification as an advertisement, a valid physical postal address and a working opt-out. Opt-outs must be honoured within 10 business days, and the mechanism must work for at least 30 days after sending. The opt-out must offer a way to stop all marketing ([FTC guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)).

### United Kingdom: PECR (ICO)

- Marketing email to individuals needs consent, unless the "soft opt-in" applies: details obtained during a sale or sale negotiation, marketing only the sender's similar products, and a simple opt-out offered both when details are collected and in every message ([PECR regulation 22](https://www.legislation.gov.uk/uksi/2003/2426/regulation/22)).
- The ICO treats routine service messages as outside direct marketing, including messages about a current contract or past purchase. A message that "actively promotes or encourages" use of a service, offer or upgrade is likely direct marketing, and any marketing element in a service message counts even if it isn't the main purpose ([ICO: identify direct marketing](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/direct-marketing-guidance/identify-direct-marketing/), [ICO: electronic and telephone marketing](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guide-to-pecr/electronic-and-telephone-marketing/)).

The ICO's test matters most for this project, since bringing Students back is the stated purpose of these Notifications.

### European Union: ePrivacy Directive

Article 13 requires prior consent for email direct marketing, with an existing-customer exception for the sender's own similar products and an opt-out at collection and on every message ([Directive 2002/58/EC](https://eur-lex.europa.eu/eli/dir/2002/58/oj/eng)). The Court of Justice held in C-654/23 that an address obtained through a free account can count as obtained in the context of a sale ([judgment](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A62023CJ0654)). Member State implementation varies.

### Canada: CASL (CRTC)

Commercial electronic messages need consent, identification information and an unsubscribe mechanism. Requests must be actioned within 10 business days, and the link must stay valid for at least 60 days ([CRTC FAQ](https://crtc.gc.ca/eng/com500/faq500.htm), [ISED guidance](https://ised-isde.canada.ca/site/canada-anti-spam-legislation/en/getting-consent-send-email), [CASL](https://laws-lois.justice.gc.ca/eng/acts/e-1.6/page-2.html)).

### Mailbox providers

Postmark reports that Gmail and Yahoo require RFC 8058 one-click headers on marketing messages since June 2024, with `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, and that the mailto method alone doesn't satisfy this ([Postmark](https://postmarkapp.com/support/article/1299-how-to-include-a-list-unsubscribe-header)). This is a deliverability requirement rather than law.

### How this applies to Cadence

This is my reading, to test in the spec:

| Email | Likely treatment |
| --- | --- |
| Refund emails (existing mailer) | Transactional. Unaffected, and don't need to move to Knock for this. |
| Neutral, factual update on a course the Student is Enrolled on | Arguable. Could be transactional under the US "updates" category. Becomes marketing under ICO guidance if it encourages the Student to come back. |
| Anything worded to bring the Student back, promote a course, or a digest of such things | Treat as commercial. |

The practical conclusion is to mark Knock workflows that send re-engagement email as commercial, and to include an unsubscribe link and a physical postal address in the email layout. Knock adds the unsubscribe mechanics; **Knock's docs do not mention the postal address**, so Cadence has to supply it.

Consent is a gap of a different kind. UK and EU soft opt-in need an opt-out offered when the Student's details are collected. Cadence has no real sign-up today, so that moment has to exist before real launch, alongside the real-authentication need already noted on the map.

## What the sources do not settle

Items that need a Knock account are candidates for the sign-up task ticket the map allows for.

**Needs a Knock account to test:**

1. Whether in-app before a batch runs per event, and whether a post-batch email condition can reference a pre-batch in-app message.
2. Which headers Knock adds for commercial workflows. The docs say "required unsubscribe headers" and a "1-click unsubscribe header" but never name `List-Unsubscribe`, `List-Unsubscribe-Post` or RFC 8058.
3. Whether `commercial_subscribed` = `false` also suppresses non-email steps, such as the in-app step, in a commercial workflow. The docs talk about emails and about "commercial messages" without saying.
4. Whether preferences are re-checked when an email step runs after a delay, so that an opt-out during the delay is honoured. The channel-step docs imply it but don't state it.
5. What happens at the email step when a Student has no `email`, beyond a step being able to stop early.
6. How long the commercial unsubscribe link stays valid. The preference centre link "never expires", but the docs don't say this about the commercial link. CAN-SPAM needs at least 30 days and CASL at least 60.
7. The exact trigger for automatic `read` in Knock's React feed. One page calls open tracking "implicit" with `KnockFeedProvider`, while the feed page documents only automatic `seen` plus an `onOpen` handler that can mark items read.
8. Whether commercial unsubscribe is limited by plan. The pages read don't say.

**Needs Postmark:**

9. How Knock's commercial headers interact with Postmark on each stream, including whether `outbound` needs anything beyond pass-through, and whether Postmark's own auto-added broadcast headers would conflict with Knock's.
10. Whether Postmark would accept these emails on `outbound`. Its stream definitions are descriptive and the page describes no enforcement for misclassification.
11. Whether self-managed unsubscribes on a broadcast stream appear in Postmark's suppressions or the Subscription Change webhook.

**Needs a legal view:**

12. Whether any given Notification wording counts as marketing, and which regimes apply given where Students are.
13. Whether Cadence's soft opt-in conditions are met for Students who came through a Claimed Seat rather than their own Purchase. The soft opt-in requires details obtained in the course of a sale to that recipient, and the sources read don't address a Seat paid for by a Team Admin.

## How these sources were read

- Pages were read through a fetch tool that returns extracts and summaries rather than raw HTML. Short quotations are as reported by that tool.
- The ICO website refused direct fetches, so ICO points come from search-result excerpts of the cited pages. EU points, including C-654/23, and the CRTC, ISED and CASL points also come from search-result excerpts. Check these against the full pages before relying on them.
- The FTC guide was read directly. eCFR was unavailable, so the CFR citation is the 2024 edition on govinfo.gov and may not reflect later amendments.
- `GLOSSARY.md` was read from the main checkout, because it is not on this branch's base commit. The description of the current mailer comes from the ticket brief. `app/lib/mailer.server.ts` is also absent from this branch's base commit, so it was not inspected.
