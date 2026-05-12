# Make.com Scenarios

This folder holds the JSON blueprints for the three Make.com scenarios that power Sage's tool calls.

> **HIPAA flag:** Make.com signs a BAA only on its Enterprise tier. This demo runs on **developer-tier Make.com** with synthetic test data — it is **not** a HIPAA-compliant deployment. The production swap is AWS Lambda + API Gateway under the AWS BAA, documented in [`../BAA_VENDOR_MATRIX.md`](../BAA_VENDOR_MATRIX.md). Treat anything in this folder as the *shape* of the production integration, not the production integration itself.

## Files

- [`checkAvailability.blueprint.json`](./checkAvailability.blueprint.json) — returns 2–3 open appointment slots
- [`bookAppointment.blueprint.json`](./bookAppointment.blueprint.json) — writes a PHI-minimal event to Google Calendar
- [`secureMessage.blueprint.json`](./secureMessage.blueprint.json) — enqueues refill / triage / billing requests for the clinical or billing team

## How to export from Make.com

1. Open the scenario
2. Click the ⋮ (three-dots) menu in the bottom toolbar
3. Click **Export Blueprint**
4. Save the JSON here under the filenames above

## How to import into your own workspace

1. New Scenario → ⋮ → **Import Blueprint** → upload the JSON
2. Reconnect Google Calendar (for `bookAppointment`) and Gmail (for `secureMessage`) to your own accounts
3. Update Vapi tool URLs to point at your scenarios' webhook URLs

## What each scenario does

### `checkAvailability`

| | |
|---|---|
| Trigger | Custom webhook (called by Vapi tool) |
| Input | `{ day: string, timeOfDay: string, visitType: string }` |
| Output | `{ slots: string[] }` |
| Demo behavior | Returns hardcoded slots, voice-friendly ("Tuesday at ten thirty in the morning"). When `timeOfDay="urgent"`, returns same-day or next-morning slots only. |
| Production behavior | Query Google Calendar free/busy on the practice calendar; return up to 3 open slots within M–F 8 AM–6 PM Central. `visitType` controls slot duration (15 min sick visit, 30 min follow-up, 60 min new patient). |
| HIPAA | No PHI in inputs or outputs — this scenario is safe to run on non-BAA infrastructure. |

### `bookAppointment`

| | |
|---|---|
| Trigger | Custom webhook (called by Vapi tool) |
| Input | `{ callerName, callerDOB, callerPhone, visitType, chosenSlot, startTimeISO }` |
| Action | Google Calendar → "Create an Event" (Detail mode, **not** Quickly mode — natural-language date parsing was unreliable on the Bright Smile build) |
| Output | `{ success: true, eventId }` |
| Demo behavior | Writes an event with title `[Visit type] — [first name only]`, no DOB in the event body. Synthetic data only. |
| Production behavior | Look up patient by name + DOB against an internal registry, write event with **internal patient ID only** in title and body — name and DOB never enter the calendar. |
| HIPAA | Demo writes synthetic PHI to non-BAA Google Calendar. Production swaps to Google Workspace Business Plus (BAA) or EHR-native scheduling. |

### `secureMessage`

| | |
|---|---|
| Trigger | Custom webhook (called by Vapi tool) |
| Input | `{ callerName, callerDOB, messageType: "refill"\|"triage"\|"billing", payload: object }` |
| Action | Demo: enqueue to a single Gmail inbox the clinical team checks. Production: write to an SQS queue with WORM-S3 archive and IAM-scoped clinical-team access. |
| Output | `{ success: true, messageId }` |
| HIPAA | The whole point of this tool is to *avoid* SMS / Slack / unencrypted email. Demo uses Gmail with synthetic data; production uses an encrypted, audit-logged queue. |
| Critical | Sage never confirms a refill as "done" via this tool — only that the **request was sent**. Refill confirmation is the clinical team's responsibility, not the agent's. |

## Production migration

Each scenario has a corresponding AWS Lambda equivalent in the production migration plan. The Lambda handlers would:

- Validate the Vapi webhook signature
- Look up patient identifiers against a registry (instead of accepting raw name + DOB in event bodies)
- Write events / queue messages with PHI-minimal payloads
- Emit a CloudWatch audit-log entry per call, with PII redacted

See [`../BAA_VENDOR_MATRIX.md`](../BAA_VENDOR_MATRIX.md) § "Recommended HIPAA stack" for the full Lambda + API Gateway + SQS topology.
