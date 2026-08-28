# AI Voice Receptionist / Missed-Call Recovery

## The problem this solves

A home service business misses revenue every time the phone rings and nobody
picks up. The technician is on a job, it's after hours, or the call comes in
during another call. Voicemail catches some of that, but most callers just
hang up and call the next company on the list instead of leaving a message.

This system answers every call live, with a real conversation, not a menu
tree. It collects the caller's details, classifies the urgency, checks a
real calendar, and books the appointment on the spot, all while the caller
is still on the phone. If it can't book (nothing available, a system error,
or the request falls outside what it handles), it never leaves the caller
with nothing. It tells them a team member will call back within the hour,
logs the call, and texts a follow-up.

## Architecture

Three separate things are happening here, and they run on different timing:

1. **The voice agent itself (Vapi)**: answers the call, holds the
   conversation, and decides when to call out to n8n mid-call.
2. **Two live, synchronous tools** (`check_availability`, `book_appointment`):
   the agent calls these *while the caller is on hold for the response*, so
   they hit Cal.com's API directly through n8n and return fast. The caller
   hears a short filler line ("One moment, let me check our availability")
   while this happens.
3. **One fire-and-forget webhook** (the end-of-call report): Vapi sends this
   after the call is already over. It logs the call to Airtable and, if the
   call never resulted in a booking, sends a missed-call text.

```
Inbound call → Vapi assistant
  ├─ mid-call: check_availability  → n8n → Cal.com (live availability)
  ├─ mid-call: book_appointment    → n8n → Cal.com (creates the booking)
  └─ after call ends: end-of-call-report → n8n → Airtable log + missed-call SMS
```

## Why this calls Cal.com live, during the call, instead of texting a booking link

This was a deliberate scope decision, not the only valid approach. Texting a
self-serve booking link after the call is simpler to build and would have
been a reasonable choice. Live in-call booking is more impressive and more
useful to the caller (they hang up with a confirmed time, not a link to go
click), but it's real added complexity: the agent has to hold state across
multiple tool calls in one conversation, handle a slot becoming unavailable
mid-call, and never leave the caller in dead air while an API call is in
flight. That's the harder, more valuable version, and it's the one this
project set out to prove.

## Design decisions worth knowing before reading the code

- **The booking attendee email is a single fixed address, not collected per
  caller.** Email is painful to collect over voice, and Cal.com's booking
  API requires *some* email to exist even though the caller never sees or
  uses it. The caller's real confirmation is the voice read-back on the
  call itself. For a real client this would be the business's own
  booking-notifications inbox, not something gathered from callers.
- **The AI calls OpenAI-style function tools directly against Cal.com's REST
  API through n8n, not a no-code Cal.com connector.** This is the same
  "prove I can integrate at the API level" choice made in the Maintenance
  Request Routing project, applied here to live, mid-call tool orchestration
  instead of a post-hoc classification step.
- **Every tool-call webhook and the end-of-call webhook parse Vapi's raw
  payload defensively** (try/catch, safe fallback defaults) rather than
  assuming a fixed shape. Vapi's payload structure genuinely shifted between
  what the docs suggested and what the live API actually sent during this
  build (see Troubleshooting below) — the code is written to degrade
  gracefully rather than crash the call if that happens again.
- **`onError: continueRegularOutput` is set on both Cal.com HTTP nodes and
  the Airtable log node.** A failed calendar call becomes a spoken fallback
  line to the caller, not a dead call. A failed Airtable write doesn't block
  the missed-call SMS check downstream of it.

## Known limitations (honest, not hidden)

- **Full caller details (name, address, issue) are only captured for calls
  that reach the booking step.** They're extracted from the arguments of
  the `check_availability` tool call, since that's the point in the
  conversation where the AI has collected everything. A call that's cut
  short before that point (hang-up, silence timeout, an off-script tangent)
  logs with just a phone number and a "Missed Call" outcome, not the
  partial details the caller may have already given. Closing this fully
  would mean adding an incremental "save progress" tool called after each
  piece of information rather than one tool called once at the end — a
  real next step, not built in this version.
- **No separate flow for "existing customer following up."** The assistant
  asks whether it's a new request or a follow-up, but only the new-request
  path is actually built out.
- **A caller booking the last open slot at the exact moment someone else
  does isn't handled with custom logic** — Cal.com's own booking API
  rejects a slot that's no longer free at the moment of the request, and
  the existing "booking failed → we'll call you back" fallback already
  covers that case correctly, just not with a bespoke race-condition
  message.

## Setup

1. **Cal.com**: create a real (not "Events"-feature) Event Type — see the
   troubleshooting note below on this distinction, it cost real time to
   discover. Set its duration and a Monday-Saturday weekly availability
   schedule. Turn off calendar-conflict checking against any personal
   calendar you don't want blocking test bookings. Grab the numeric event
   type ID and an API key.
2. **Twilio**: buy a local number, import it into Vapi. SMS from this
   number needs A2P 10DLC registration completed before it reliably
   delivers to US numbers; international delivery worked in testing without
   waiting on that.
3. **Vapi**: create the assistant with the system prompt in
   `vapi-assistant-config.md`, plus the three tools (`check_availability`,
   `book_appointment`, the built-in `endCall` tool) with the schemas and
   Messages configuration documented there. Restrict **Server Messages**
   (Advanced tab) to just `end-of-call-report` — left at the default, Vapi
   sends a dozen-plus event types per call to your Server URL.
4. **n8n**: import `workflow.json`, reconnect credentials (Airtable PAT,
   Cal.com Header Auth, Twilio), expose it via a public tunnel (ngrok or
   similar), and point Vapi's tool Server URLs and assistant Server URL at
   the three webhook paths.
5. **Airtable**: a Call Log table (schema in `workflow.json`'s Airtable
   node) to receive the post-call record.

## Troubleshooting notes from building this

- **Cal.com has two different features that can share a name and a
  similar slug**: the classic recurring **Event Type** (what this project
  needs, listed under "Links" in Cal.com's sidebar) and a newer one-time
  **Events** feature (webinar/registration style, with a fixed date and a
  "Register" button). Creating the wrong one looks fine right up until you
  call the `/v2/slots` API against it and get a `NOT_FOUND`, even though
  the object clearly exists in the UI. Check via the API
  (`GET /v2/event-types`) for `"isCalEvent": false` to confirm you have the
  right kind.
- **Cal.com versions its API endpoints independently per endpoint** —
  `/v2/slots` and `/v2/bookings` needed two *different* `cal-api-version`
  header values in this build. A wrong version doesn't always error
  clearly; it can just 404 as if the route doesn't exist.
- **A stray one-time "Events" object with a broken multi-day duration**
  (from an early setup mistake) silently blocked all real availability for
  its entire date range, since Cal.com's own conflict tracking counted it
  against the host regardless of whether it showed up as a normal Google
  Calendar entry.
- **The exact shape of Vapi's tool-call and end-of-call-report webhooks**
  took direct API inspection to nail down — `toolCalls` (not
  `toolCallList`) inside `artifact.messages`, and `arguments` arrives as a
  JSON *string* that needs parsing, not an object.
- **`endCallAfterSpokenEnabled` on the endCall tool's completion message
  can silently persist as `false`** even when the dashboard checkbox looks
  checked. If a goodbye message isn't playing before hangup, verify this
  field directly in the tool's raw JSON/code view, not just the checkbox.
- **A too-rigid "collect one thing at a time" instruction backfires** when
  a real caller gives everything in one breath — the assistant re-asked for
  information it already had. The fix was an explicit instruction to check
  what's already been provided before asking for anything.
