# Business Case

## Problem

Home service businesses lose real revenue specifically because they can't
answer the phone. Technicians are on job sites, not at a desk, and HVAC,
plumbing, and electrical companies miss an estimated **40-70% of inbound
calls** as a result ([Automatdo](https://automatdo.com/blog/the-real-cost-of-missed-calls-for-home-services-companies/)).

Voicemail doesn't recover that gap. **80-85% of callers who reach
voicemail hang up without leaving a message**, and of those, most don't
call back at all — they call the next company on the list instead
([ContractorInCharge](https://contractorincharge.com/blog/missed-call-statistics-for-home-service-companies)).
Industry estimates put the cost of this at **$125-350 in lost revenue per
missed call**, and **$50,000-200,000+ a year** for a small service
business, depending on call volume ([Phone2](https://www.phone2.io/post/true-cost-of-missed-calls)).

There's a second, quieter problem behind the first: **78% of deals go to
whoever responds first** ([Phone2](https://www.phone2.io/post/true-cost-of-missed-calls)).
A callback an hour later, even a fast one, is often already too late —
the caller has already booked with a competitor who picked up live.

## Solution

This system answers every call live and tries to close the loop inside
the call itself, not after it, described in full in
[ARCHITECTURE.md](ARCHITECTURE.md):

- **A real conversation, not a menu tree or voicemail.** The AI collects
  name, address, and issue the same way a human receptionist would,
  adapting to however the caller actually talks (one detail at a time, or
  everything in one breath).
- **Live urgency classification** (Emergency vs. Routine, by category)
  means a caller with an active leak or a gas smell isn't queued behind a
  routine dripping-faucet request the way a first-come voicemail box
  would treat them identically.
- **Booking happens on the call, against a real calendar**, not through a
  follow-up text with a link the caller may never click. The caller hangs
  up with a confirmed time, closing the "first responder wins the deal"
  window immediately instead of leaving it open for a competitor to fill.
- **A graceful fallback, not a dead end, when booking can't happen** — no
  available slot, a calendar API failure, or a request outside the
  system's scope all result in an explicit "a team member will call you
  back within the hour" instead of silence or a confusing error.
- **Every call is logged**, and any call that doesn't end in a booking
  triggers an automatic follow-up text, so a missed connection doesn't
  quietly disappear the way an unanswered voicemail does.
- **A call log dashboard** gives the business owner a plain view of what
  the system actually did, call volume, booked rate, missed calls,
  without needing to check n8n or a spreadsheet.

## Who this is for

Any home service business (HVAC, plumbing, electrical, and similar
trades) that misses calls because technicians are in the field, and
either has no after-hours coverage or relies on voicemail today. Also a
fit for businesses that already use a booking calendar (Cal.com, Calendly,
or similar) and want inbound calls to book directly against it instead of
funneling through a human scheduler for every request.

## Expected impact

This is a portfolio build, not a live client deployment, so there's no
real production call volume to report. What follows is industry benchmark
data, not a guaranteed or achieved result for this specific
implementation:

- At an estimated $125-350 lost per missed call and 40-70% of calls
  currently going unanswered, a business fielding even 100 calls a month
  is looking at a plausible **$5,000-24,500/month** in calls that never
  got a live answer, using the low end of that per-call estimate
  ([Phone2](https://www.phone2.io/post/true-cost-of-missed-calls); [Automatdo](https://automatdo.com/blog/the-real-cost-of-missed-calls-for-home-services-companies/)).
- Recovering even a fraction of that gap by answering live instead of
  voicemailing represents a direct, attributable revenue line, not a
  marginal optimization, especially given how few of those callers ever
  call back on their own (as few as 15%, per the missed-call statistics
  above).

The honest framing for a prospective client: this system is built to
close the specific gap between "the phone rang and nobody picked up" and
"the caller had to leave a message and hope," not to replace live human
staff entirely. Actual recovery rate depends on the business's existing
missed-call volume, call quality, and how well the AI's script matches
what a real caller expects to hear, and should be measured against that
business's own baseline once live.
