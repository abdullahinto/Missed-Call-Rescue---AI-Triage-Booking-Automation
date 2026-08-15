# Missed Call Rescue: AI Triage & Booking Automation

An n8n workflow that automatically handles missed calls for home-service businesses such as HVAC, plumbing, and electrical companies. It receives missed-call events through a webhook, identifies the correct business, classifies the caller's request with AI, and takes an appropriate follow-up action without requiring manual intervention.

## What It Does

The workflow is designed around three primary outcomes:

* **Emergency:** Immediately alert the on-call technician and confirm receipt with the customer.
* **Routine / Same-Day:** Check the business calendar, generate a booking-oriented SMS, and send it to the caller.
* **Spam / Wrong Number:** Log the call without sending a customer message.

Every handled call is also recorded in a Google Sheets call log, and the business owner receives an email notification.

## Workflow

1. **Receive the missed call**

   * A POST webhook at `/missed-call-rescue` receives the call event.
   * The workflow expects `CallSid` and `To` in the payload.

2. **Validate and identify the business**

   * The called number is matched against the `Client Directory` Google Sheet.
   * Business-specific information such as business name, tone notes, emergency keywords, technician phone, calendar ID, and owner email is loaded.
   * Invalid payloads and unknown business numbers are rejected and emailed to the administrator.

3. **Prevent duplicate handling**

   * The workflow checks the `Call Log` sheet for the incoming `CallSid`.
   * Duplicate webhook retries are skipped so customers do not receive duplicate SMS messages.

4. **AI call triage**

   * Gemini 2.5 Flash Lite classifies the voicemail transcript.
   * It determines `service_type`, `urgency`, and a one-sentence summary.
   * Supported urgency levels are emergency, same-day, routine, and spam.

5. **Emergency handling**

   * Emergency calls bypass AI-generated customer copy.
   * A fixed dispatch message is sent to the configured on-call technician through Twilio.
   * The customer receives an emergency confirmation SMS.
   * The event is logged as `emergency_dispatched`.

6. **Routine booking flow**

   * Google Calendar is checked for events during the next three days.
   * GPT-4o-mini drafts a short booking message and identifies potential open windows.
   * Claude Sonnet 4.6 rewrites the message using the business's configured tone.
   * A final code step removes formatting, limits the SMS to 300 characters, and provides a fallback message if necessary.
   * Twilio sends the booking offer to the caller.

7. **Logging and notification**

   * Call details, classification, urgency, action taken, summary, and timestamp are appended to the `Call Log` Google Sheet.
   * The business owner receives an email summarizing how the missed call was handled.

## Required Integrations

The workflow uses:

* **n8n** for orchestration
* **Twilio** for customer and technician SMS
* **Google Sheets** for client configuration and call logging
* **Google Calendar** for availability
* **Google Gemini** for call triage
* **OpenAI GPT-4o-mini** for booking-message drafting
* **Anthropic Claude Sonnet 4.6** for brand-voice editing
* **Gmail** for administrative and owner notifications

## Setup

Before activation, replace the placeholder credentials and IDs, including the Google Sheets document ID, Google Sheets credentials, Gmail credentials, Twilio credentials, and Google Calendar credentials.

The workflow is currently exported as **inactive**, and its credential/template setup is not completed. The `Client Directory` and `Call Log` sheets must also be configured with the expected fields.

The main webhook endpoint is:

`POST /missed-call-rescue`

The automation should be tested thoroughly with emergency, routine, spam, malformed, unknown-number, and duplicate-call scenarios before being enabled in production.

## Important Behavior

The workflow deliberately favors safety and predictable handling. Emergency calls use deterministic SMS content instead of additional AI copywriting, while AI outputs are constrained with structured parsers and reconciled against trusted workflow data.

Calendar availability is inferred from existing events rather than directly creating appointments. The workflow therefore sends booking offers but does not itself complete a calendar booking.
