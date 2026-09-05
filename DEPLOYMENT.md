# Deployment Notes

This project runs in demo form with n8n exposed through an ngrok tunnel,
Vapi as the voice layer, and a Lovable app for the dashboard. Moving it to
a real client engagement changes several things.

## n8n hosting and webhook exposure

- **The ngrok tunnel used here is a dev convenience, not a production
  endpoint.** Free-tier ngrok URLs aren't guaranteed stable across
  restarts, and Vapi's tool Server URLs and assistant Server URL all need
  to keep pointing at whatever's currently live. Production needs either
  n8n Cloud's own stable webhook URL, or self-hosted n8n behind a real
  domain with a reverse proxy (Caddy/nginx) and a real TLS certificate.
- **Vapi's `Server Messages` restriction (Advanced tab) has to be
  reapplied on the production assistant**, not assumed to carry over. Left
  at the default, Vapi sends a dozen-plus event types per call to the
  Server URL, which is real, unnecessary API and compute cost at
  production call volume, not just development noise.

## Secrets

- **Every credential in this project (Cal.com API key, Twilio Auth Token,
  Airtable Personal Access Token) belongs in n8n's built-in credential
  store**, never in the workflow file or a config committed to the repo.
  This build's own `workflow.json` uses
  `REPLACE_WITH_YOUR_*_CREDENTIAL_ID` placeholders for exactly this
  reason.
- **Twilio's Auth Token used here has full account access if it leaks.** A
  real deployment should use a scoped API Key instead, limited to this
  specific number/use case and individually revocable.
- **Vapi's private API key and the Cal.com API key are both live secrets**
  with real billing behind them (Vapi bills per-minute, Cal.com and
  Twilio both have their own usage costs) — rotate them if this repo, or
  any screen recording of the build process, is ever shared more widely
  than intended.

## Twilio and SMS delivery

- **SMS from a new Twilio number needs A2P 10DLC registration completed
  before it reliably delivers to US recipients.** This build's missed-call
  SMS confirmed working to an international number before that
  registration cleared, but real US-to-US delivery depends on it. Start
  that registration on day one of a real client deployment, since it can
  take several days to approve.
- **Business hours and the after-hours "treat as urgent" instruction in
  the system prompt need to match the client's real hours**, not the
  Harborview demo's Monday-Saturday 8am-6pm.

## Cal.com

- **The Event Type used for live availability must be a genuine recurring
  Event Type** (found under Cal.com's "Links" section), not the newer
  one-time "Events" feature — the two can share a similar name and slug,
  but only the former supports the `/v2/slots` API this build depends on.
  See the README's troubleshooting notes for how this was caught.
- **Calendar-conflict checking against a real staff calendar should stay
  on for a real deployment** — it was turned off in this demo specifically
  to stop the developer's own personal calendar from blocking test
  bookings, which isn't a concern once this points at a real business
  calendar with real staff availability.
- **`cal-api-version` header values are pinned per endpoint** in the n8n
  workflow (`/v2/slots` and `/v2/bookings` use different version strings).
  Cal.com versions endpoints independently; confirm these still match
  Cal.com's current documented values before a production launch, since
  they're the kind of thing that changes without much warning.

## The Lovable dashboard's data source

The dashboard deployed for this portfolio build (`harborview-call-log.lovable.app`)
reads live from the same Airtable "Call Log" base n8n actually writes to —
a server function fetches Airtable's REST API directly on page load, using
a read-only token stored as a platform secret, never exposed to the
browser. No ngrok tunnel or n8n uptime dependency: Airtable's API is
reachable independent of whether the local n8n/Docker stack happens to be
running. For a real client, this same pattern (a server-side read against
their actual system of record, scoped to a read-only credential) is the
right approach, just pointed at whatever CRM or database they actually use.

## Before going live for a real client

1. Confirm the Event Type, availability schedule, and business hours all
   match the client's actual operation, not the Harborview demo values.
2. Confirm A2P 10DLC registration has cleared for the client's Twilio
   number before relying on SMS delivery to US numbers.
3. Run at least one full test call through the entire path (intake →
   booking → Airtable log) with the client present, so they've seen it
   work before it starts handling real callers.
4. Confirm the Vapi assistant's phone number is the actual number the
   client will publish/forward to, not a leftover test number.

## Monitoring after launch

Vapi's own call logs and cost breakdown cover per-call debugging. For
ongoing reliability monitoring across this workflow specifically (webhook
failures, a stalled n8n instance, a stale credential), pair this with the
same pattern used in the "Workflow Health & Credential Monitor" project
elsewhere in this portfolio — that build generalizes exactly this kind of
check into a standalone daily health digest.
