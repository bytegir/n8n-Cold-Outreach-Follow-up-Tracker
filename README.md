# n8n-Cold-Outreach-Follow-up-Tracker

> An n8n automation that turns a messy outreach spreadsheet into a daily, ready-to-send follow-up digest — while keeping every message under human control.

---

## Problem

Cold outreach on WhatsApp (leads sourced from Google Business research) was being sent without any tracking system. Out of 63 leads contacted, a second follow-up had never happened once — even though most target follow-up dates, calculated manually in a spreadsheet, had already passed by months.

This wasn't purely forgetfulness. Two things were happening at once:

1. **Execution gap** — no trigger existed to notify when a follow-up date passed. The plan existed; the execution never fired.
2. **Emotional friction** — hesitation to follow up for fear of being annoying, especially since most leads never replied to the first message.

A solution needed to solve both: not just a reminder, but something that removes the friction of "what do I even say."

---

## Solution

A scheduled n8n workflow reads the existing leads spreadsheet every morning, identifies which leads are due for a follow-up (first or second), drafts a ready-to-send message for each, and delivers all of them as a single digest to Slack.

The owner reviews the digest, copies/tweaks the message, sends it manually via WhatsApp, and updates the spreadsheet by hand. No message is ever sent automatically.

---

## How It Works

```
Scheduler (daily, single run)
        ↓
Google Sheets — get rows where status = "SENT" OR "FU1"
        ↓
Code node — route + draft
  · check status + days elapsed since last touch
  · assign category: KIRIM FU1 / KIRIM FU2 / ANOMALI
  · generate ready-to-send draft message per lead
        ↓
Aggregate — combine all drafts into one digest
        ↓
Slack — post digest to #leads-follow-up
        ↓
Manual — owner reviews, sends via WhatsApp, updates sheet
```

| Step | Node | Purpose |
|---|---|---|
| 1 | **Scheduler** | Runs once a day. V1 only produces drafts, so one pass is enough — the owner decides when to actually send. (Splitting into multiple sessions only becomes relevant in V3, once sending is automated.) |
| 2 | **cekDB** (Google Sheets) | Reads the leads sheet, filtered to `status = SENT OR status = FU1`. Anything already `RESPONDED`, `CLOSED_*`, or `INVALID` is excluded here. |
| 3 | **cekDate** (Code) | For each row, checks status and days elapsed since the relevant date (`Sent_Date` for SENT, `FU1_Date` for FU1), decides whether it's due for FU1, FU2, or neither, and builds the draft message. Includes a guard against invalid or logically impossible dates (see [Design Decisions](#design-decisions)). |
| 4 | **Aggregate** | Collects every generated draft into a single array. |
| 5 | **Notif** (Slack) | Posts the full digest as one message to a dedicated Slack channel. |

<img width="1872" height="703" alt="Screenshot 2026-08-17 175615" src="https://github.com/user-attachments/assets/2abe8da1-ab43-47e5-9db5-789c48d1a391" />
*Screenshot: full n8n canvas — Scheduler → Google Sheets → Code (routing + drafting) → Aggregate → Slack.*

---

## Why Slack, Not Telegram

The original design used Telegram (simpler setup, faster to prototype). It was switched to Slack for two reasons:

- **Message length** — Telegram has a character limit that gets restrictive once a digest contains several draft messages in one push.
- **Organization** — Slack allows a dedicated channel (`#leads-follow-up`) just for this workflow, separate from personal chat, which keeps the digest easy to find and doesn't clutter a personal Telegram account.

---

## Follow-up Cadence

| Stage | Timing | Assumption |
|---|---|---|
| FU1 | H+2 from `Sent_Date` | Lead is likely busy or missed the message, not uninterested |
| FU2 | H+5 from `FU1_Date` | If still silent, likely genuinely not interested — exit gracefully, don't push |

After FU2 with no response, status is manually changed to `CLOSED_LOST` and the lead exits the reminder cycle.

---

## Data Schema

Existing sheet columns (no new sheet or form needed):

`ID | nama | niche | link IG | No WA | status | Sent_Date | FU1_Date | FU2_Date | Respon | tipe | Closing`

**Status values** (standardized going forward — old rows are left as-is):

| Status | Replaces |
|---|---|
| `SENT` | SENT, sent |
| `FU1` | FU1 |
| `FU2` | FU2 |
| `RESPONDED` | ON PROCESS, "ada respon..." |
| `CLOSED_WON` | Closing = True |
| `CLOSED_LOST` | CLOSED - NO NEED, Salah target market |
| `INVALID` | Nomer salah, Nomer tidak terdaftar, Double leads |

---

## Setup

1. Import the workflow JSON into n8n.
2. Connect a **Google Sheets** service account credential and point the `cekDB` node at your leads sheet.
3. Connect a **Slack** credential and set the target channel on the `Notif` node.
4. Confirm the sheet's `status` column uses the standardized values above (at least for new rows going forward).
5. Set the **Scheduler** trigger to your preferred daily run time.
6. Run once manually to confirm the digest posts correctly, then activate the workflow.

---

## Output Example

![Slack Digest Example](./assets/slack-digest-example.png)
*Screenshot: a sample morning digest posted to Slack — one entry per lead due for FU1 or FU2.*

---

## Design Decisions

### Why the routing logic lives in a Code node, not a Switch node

The initial design used n8n's Switch node with two date-based rules (`Sent_Date` for FU1, `FU1_Date` for FU2). This broke in a specific way: the Switch node's UI doesn't support combining multiple conditions (e.g. `status equals X` AND `date is before Y`) within a single rule — the underlying schema allows it, but there's no way to add it from the interface.

Without a status condition, a lead still sitting at `SENT` status but long overdue could get misrouted into the FU2 branch, because its precomputed `FU1_Date` (a fixed offset from `Sent_Date`) would already be far in the past — even though it had never actually received a first follow-up. A Code node solves this in one place: it checks `status` and the relevant date together, per row, with full control over the logic.

### Guard against invalid or impossible dates

Because the leads sheet was populated manually over time, some dates are inconsistent (wrong format, or a `FU1_Date` that's earlier than `Sent_Date` — logically impossible, since FU1 can't happen before the first message was sent). Rather than silently skipping these rows, the Code node flags them as `ANOMALI` and includes them in the same Slack digest with a warning label, so they surface for manual review instead of disappearing unnoticed.

This choice — surfacing anomalies in the existing channel rather than building a separate alerting system — keeps the fix proportional to the problem: no new node, no new channel, no added system complexity, just an additional entry in an output that already exists.

### Why status updates stay manual

Early on, the idea of an Approve/Reject button in Slack came up — tap to auto-update the sheet status. This was deliberately rejected, not due to a technical limitation and not deferred to a future version.

There's a gap between "tap Approve" and "the WhatsApp message actually gets sent." If tapping a button is too easy, it can become an escape hatch for the exact pattern this project exists to fix — hesitation to follow up because it feels intrusive. Tapping Approve makes the notification disappear and feels like progress, even if the WhatsApp message was never actually sent.

Left unmanaged, this could mean:
- The sheet says `FU1`, but nothing was actually sent on WhatsApp.
- That lead disappears from the reminder cycle despite never having been genuinely followed up with.
- A silent failure — harder to detect than the current, fully-manual state.

Weighed against the risk of staying manual (forgetting to update → the lead resurfaces in tomorrow's digest → mildly annoying, but nothing is ever lost), a one-tap auto-update is the riskier design for this specific project. The same principle applies to Project #2 (Quote Follow-up): manual approval before sending, manual status update after sending.

**Considered for a later version:** the Slack digest could include a deep link straight to the relevant row in Google Sheets, to make manual updates faster — but still a deliberate, conscious action rather than a single tap.

---

## Current Limitations (V1)

**Included:**
- [x] Reads the existing sheet, filters by status and day threshold
- [x] Daily Slack digest with ready-to-use draft messages
- [x] FU1 (H+2) and FU2 (H+5) cadence
- [x] Anomaly detection for invalid/impossible dates

**Not included (later versions or out of scope):**
- [ ] Automatic status updates in the sheet
- [ ] Automatic WhatsApp sending
- [ ] Backfilling or cleaning old inconsistent status data
- [ ] Automatic handling of duplicate leads
- [ ] LLM-based message personalization (handled in Project #2)

---

## Roadmap

### V2
- Replace the static message templates with a Code node that randomly selects from 3–5 message variants per category, so outbound messages don't all read identically across leads.
- Move digest delivery to Slack's App Builder with a per-lead Approve button — replacing manual copy-paste, though sending itself stays manual (see [Design Decisions](#design-decisions)).

### V2.1
- Once V2 is fully stable (no recurring bugs or edge cases), automate the status + follow-up date update in Google Sheets. Likely implementation: a Wait-for-Response node or a separate subflow triggered by a Slack event subscription (webhook).

### V3 *(requires further research on anti-bot-detection)*
- Actual automated WhatsApp sending via Evolution API — draft becomes send, not just a suggestion. Numbers mapped directly from the sheet.
- Split sending into two sessions per day (12:00 and 16:00), 10–15 leads per session.
- To avoid WhatsApp flagging the number as spam: each send is gated by a Wait node with a randomized 45–90 minute delay per lead, generated via a Code node inside a loop.

### Why this staged approach

*"Don't automate everything at once."* Each version only adds the next layer once the previous one is proven stable. This keeps unnecessary technical complexity out of versions that don't need it yet, and lets the most efficient logic get discovered incrementally rather than guessed upfront.

---

## Open Assumptions — To Validate While Running

- [ ] Whether the H+2 / H+5 thresholds feel right, or need adjusting after a few weeks in production
- [ ] Whether the generic FU1/FU2 templates work across niches (iPhone rental, barbershop, etc.) or need slight variation
- [ ] Whether daily lead volume stays small enough for one morning digest, or needs to be split

---

## Success Criteria

- The system runs without manual intervention to detect who needs a follow-up
- The owner actually follows up based on the digest — not just reads and ignores it
- FU2 starts genuinely happening (from zero prior occurrences)
- End-to-end demo achievable in under a minute: old sheet → digest appears → example follow-up ready to send

---

## Tech Stack

- **n8n** — workflow orchestration
- **Google Sheets** — lead database (existing, no schema changes required)
- **Slack** — digest delivery channel
