# Architecture

Three separate paths run on different timing, all tied to one Vapi
assistant. See [README.md](README.md) for setup and troubleshooting, and
[vapi-assistant-config.md](vapi-assistant-config.md) for the full system
prompt and tool definitions.

```mermaid
flowchart TD
    subgraph Call["Inbound Call — Vapi Assistant"]
        A1((Caller dials in)) --> A2[Vapi greets, collects intake]
        A2 --> A3{Has name, address,<br/>issue, phone, category,<br/>urgency?}
    end

    subgraph Live["Live Tools — caller on hold for the response"]
        A3 -->|yes| B1[Tool call: check_availability]
        B1 --> B2[n8n: Webhook receives tool call]
        B2 --> B3[Cal.com: GET available slots]
        B3 --> B4{Slots returned?}
        B4 -->|yes| B5[Respond with times + exact ISO values]
        B4 -->|no / error| B6[Respond with fallback line]
        B5 --> C1[Caller picks a time]
        C1 --> C2[Tool call: book_appointment]
        C2 --> C3[n8n: Webhook receives tool call]
        C3 --> C4[Cal.com: POST create booking]
        C4 --> C5{Booking succeeded?}
        C5 -->|yes| C6[Respond: confirmed time]
        C5 -->|no| C7[Respond with fallback line]
    end

    subgraph Fallback["Fallback Path"]
        B6 --> D1[Vapi tells caller: team will call back]
        C7 --> D1
        A3 -->|no, call ends early| D1
    end

    subgraph PostCall["After the Call Ends — fire and forget"]
        D2((Vapi: end-of-call-report)) --> E1[n8n: Webhook receives report]
        E1 --> E2{message.type ==<br/>'end-of-call-report'?}
        E2 -->|no| E3[Ignored]
        E2 -->|yes| E4[Parse transcript for intake<br/>+ booking outcome]
        E4 --> E5[(Airtable: create Call Log row)]
        E5 --> E6{Call outcome ==<br/>'Missed Call'?}
        E6 -->|yes| E7[Twilio: send missed-call SMS]
        E6 -->|no| E8[Done]
    end

    C6 -.call ends.-> D2
    D1 -.call ends.-> D2

    F1[Lovable: Call Log Dashboard] -.reads.-> F2[(Call records)]
```

## Why live tool calls instead of a callback link

`check_availability` and `book_appointment` run *while the caller is still
on the phone*, not as a follow-up text with a booking link. That's a
deliberate scope decision (see the README's "Why this calls Cal.com live"
section) — it means the agent has to hold state across two sequential tool
calls in one conversation, survive a slot becoming unavailable mid-call,
and never leave the caller in dead air while an API round-trip is in
flight. The `request-start` filler message on each tool ("One moment, let
me check our availability") exists specifically to cover that gap.

## Why the post-call webhook is filtered by message type

Vapi's Server URL receives every lifecycle event during a call by
default, not just the final report — transcript updates, status changes,
speech events, a dozen-plus messages per call. The `IF: Is End-of-Call
Report` node right after the webhook checks `message.type ==
'end-of-call-report'` and drops everything else. Without it, every one of
those in-call events would trigger a full Airtable-write attempt, which is
exactly what happened during testing before this filter was added (see
the README's troubleshooting notes). Restricting Vapi's own **Server
Messages** setting to just `end-of-call-report` closes this at the source
too; the filter node is the backup, not the only defense.

## Why the post-call parse and the live tools don't share code

`check_availability` and `book_appointment` each format their own
response for the AI to speak. The post-call parser reads the *entire*
call transcript after the fact and reconstructs what happened from the
tool-call arguments embedded in it — a different extraction problem, since
by the time it runs, the call is already over and nothing can be asked
again. That's also why intake details (name, address, issue) are only
reliably captured for calls that reach `check_availability`: that's the
one place in the whole conversation where the full intake gets stated as
structured arguments instead of only appearing in free-text speech.
