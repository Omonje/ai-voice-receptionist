# Upwork Portfolio Entry (copy-paste ready, edit before posting)

**Title:** AI Voice Receptionist with Live Calendar Booking (Vapi, n8n, Cal.com)

**Cover image:** a screenshot of the Vapi call transcript showing the full
loop, greeting through a confirmed booked time, next to the matching
Cal.com booking and Airtable call log row.

**Links to include in the listing:** [README.md](README.md),
[vapi-assistant-config.md](vapi-assistant-config.md),
[live call log dashboard](https://harborview-call-log.lovable.app)

**Description:**

Every missed call is a missed job for a home service business. Voicemail
doesn't stop that. Most callers just hang up and call the next company.

Built an AI phone receptionist that answers live, holds a real
conversation, and books a real appointment on a real calendar before the
caller hangs up. It collects the caller's details, classifies the issue by
urgency, checks live calendar availability, and confirms a booked time,
all inside the call itself, not through a follow-up text with a link.

The harder, more valuable version of this is doing the booking live,
mid-call, rather than texting a self-serve link afterward. That means the
AI has to hold state across multiple tool calls in one conversation, never
leave the caller in silence while an API call is in flight, and degrade
gracefully to "a team member will call you back" if the calendar API fails
or nothing's open. All three of those are built in, not assumed.

If the call doesn't end in a booking (no answer engaged, the caller hangs
up, nothing available), the system logs it and sends a follow-up text
automatically, so no lead falls through purely because the AI couldn't
close the loop live.

A call log dashboard gives the business owner a plain view into what the
AI actually did on every call, total calls, booked rate, missed calls, and
a filterable log by category, urgency, and outcome, without needing to
open n8n or a spreadsheet.

**Skills tags:** Vapi, n8n, Cal.com API, Voice AI, LLM Function Calling,
API Integration, Twilio, Real-Time Systems, Airtable, Workflow Automation

**Before posting, confirm:**
- [ ] Vapi call recordings/transcripts used in any demo material feature
      only test data, nothing that reads as a real client's information
- [ ] Loom recorded, walking through one full call: greeting through a
      confirmed live booking, showing the Cal.com booking and Airtable log
      appearing in real time
- [ ] Repo is public and includes the latest workflow.json,
      vapi-assistant-config.md, ARCHITECTURE.md, BUSINESS-CASE.md, and
      DEPLOYMENT.md (premium deliverable bar)
- [ ] Case study description above has no leftover placeholder text
      before copy-pasting into the Upwork listing
