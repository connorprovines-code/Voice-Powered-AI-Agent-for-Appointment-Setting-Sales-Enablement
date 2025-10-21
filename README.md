# AI Voice Receptionist Implementation Guide

## Overview

This system creates an intelligent phone receptionist that can handle conversations, answer FAQs, and schedule appointments. The architecture uses:

- **Twilio** - Handles incoming phone calls
- **ElevenLabs** - Manages voice AI, basic conversation, and FAQ responses
- **n8n** - Processes complex scheduling logic and calendar operations

This separation keeps responses snappy by only invoking n8n when calendar operations are needed.

## Prerequisites

You'll need accounts and credentials for:

- Twilio (phone number)
- ElevenLabs (Conversational AI)
- n8n (workflow automation)
- Google Calendar (OAuth2 credentials)
- Redis (memory/session management)
- Anthropic API (Claude AI model)

---

## Part 1: System Connection Flow

### Step 1: Connect Twilio to ElevenLabs

**In Twilio Console:**
1. Navigate to your phone number settings
2. Under "Voice & Fax" → "A Call Comes In"
3. Select "Webhook" and enter your ElevenLabs phone number webhook URL
4. Set HTTP method to POST

**In ElevenLabs:**
1. Create a new Conversational AI agent
2. Configure your agent's voice and personality
3. Upload any FAQ documents or knowledge base files
4. Note your agent's webhook URL for n8n connection

### Step 2: Connect ElevenLabs to n8n

**In n8n workflow:**
1. Find the "Webhook: Receive User Request (ElevenLabs)" node
2. Replace `REPLACE ME` in the path with a secure endpoint (e.g., `/elevenlabs-voice-scheduler`)
3. Copy the full webhook URL (e.g., `https://your-n8n-instance.com/webhook/elevenlabs-voice-scheduler`)

**In ElevenLabs Agent Settings:**
1. Go to "Tools" or "Custom Actions"
2. Add "Client Tools" for calendar operations
3. Configure tool calling to use your n8n webhook URL
4. Enable tools for: checking availability, creating appointments, updating appointments

---

## Part 2: n8n Workflow Configuration

### Step 3: Configure Google Calendar Integration

Replace all instances of `REPLACE ME` in these nodes:
- Calendar: Check Availability
- Calendar: Create Appointment
- Update an event in Google Calendar

**Your Calendar ID format:**
- Personal: `your-email@gmail.com`
- Shared calendar: `xxxxx@group.calendar.google.com`

**To find your Calendar ID:**
1. Google Calendar → Settings → Select calendar
2. Scroll to "Integrate calendar"
3. Copy the Calendar ID

### Step 4: Set Up Redis Memory

Configure the Redis connection in the "Redis Chat Memory" node:
- Host, port, and password from your Redis provider
- Memory stores last 20 messages per session (adjustable via `contextWindowLength`)
- Critical for maintaining conversation state without lag

### Step 5: Customize the Scheduling Prompt

In the "Voice AI Agent" node, replace these bracketed placeholders:

**Business Operations:**
- `[TIMEZONE]` → Your timezone (e.g., PST, EST)
- `[START_TIME]` & `[END_TIME]` → Business hours (e.g., 9am & 5pm)
- `[OPERATING_DAYS]` → Days open (e.g., Monday through Friday)
- `[BLOCKED_DAYS]` → Never available (e.g., weekends)
- `[MINIMUM_LEAD_TIME]` → Buffer before appointments in minutes (e.g., 60)

**Appointment Details:**
- `[APPOINTMENT_DURATION]` → Default length in minutes (e.g., 30)
- `[SERVICE_TYPE]` → Your service name (e.g., "HVAC Service", "Consultation")
- `[PRIMARY_IDENTIFIER]` → What identifies the appointment (e.g., [Address], [Customer Name])

**Information Collection:**
- `[REQUIRED_FIELDS]` → Data to collect (e.g., email, phone, address)
- `[REQUIRED_FIELDS_NATURAL_LANGUAGE]` → How to ask (e.g., "your email and service address")
- `[REQUIRED_FIELDS_LIST]` → List format (e.g., "email, phone, address")
- `[REQUIRED_NOTES_FIELDS]` → What to save in notes (e.g., "customer email, phone, problem description, service address")

**⚠️ Do NOT modify these dynamic variables:**
- `{{ $now }}`
- `{{ $json.body.sessionId }}`
- `{{ $json.body.utterance }}`
- `{{ $json.body.system_caller_id }}`
- `{{ $json.body.call_log }}`

---

## Part 3: Testing & Deployment

### Step 6: Test the Workflow

1. Activate the n8n workflow
2. Test webhook using the ElevenLabs tool tester
3. Make a test call to your Twilio number
4. Verify the AI can:
   - Check calendar availability
   - Collect required information
   - Create appointments
   - Send calendar invites

### Step 7: Monitor & Optimize

- Check n8n execution logs for errors
- Review ElevenLabs conversation transcripts
- Adjust prompt instructions for better clarity
- Fine-tune `contextWindowLength` if needed (higher = more context, slower response)

---

## Frequently Asked Questions

### General

**Q: Why use three separate services instead of one AI solution?**

A: This architecture optimizes for speed. ElevenLabs handles instant responses for casual chat and FAQs, only calling n8n when complex calendar operations are needed. This prevents lag in voice conversations.

**Q: Can I use a different AI model instead of Claude?**

A: Yes, but Claude 3.5 Sonnet is recommended for best results. Other models may require prompt adjustments and could struggle with complex scheduling logic. The "think" tool is critical regardless of model choice.

**Q: Is Redis absolutely necessary?**

A: Yes. Without Redis memory, the AI must reparse the entire conversation on every turn, causing severe lag that breaks the real-time voice experience.

### Configuration

**Q: What happens if I don't replace all the [PLACEHOLDERS]?**

A: The AI won't know your business rules and will fail to schedule properly. All bracketed items must be customized for your specific use case.

**Q: Can I collect different information than the default fields?**

A: Absolutely. Modify `[REQUIRED_FIELDS]`, `[REQUIRED_FIELDS_NATURAL_LANGUAGE]`, and `[REQUIRED_NOTES_FIELDS]` to match what you need (e.g., vehicle make/model, pet name, insurance info).

**Q: How do I prevent bookings outside business hours?**

A: Set `[START_TIME]`, `[END_TIME]`, `[OPERATING_DAYS]`, and `[BLOCKED_DAYS]`. The AI will reject any booking attempts outside these parameters, even for "emergencies."

**Q: What's the minimum lead time for?**

A: `[MINIMUM_LEAD_TIME]` prevents same-hour bookings. If set to 60 minutes, callers can't book appointments within the next hour, giving you preparation time.

### Troubleshooting

**Q: The AI isn't booking appointments. What's wrong?**

A: Check:
1. All `REPLACE ME` values are set (webhook path, calendar IDs)
2. Google Calendar OAuth2 is connected
3. Placeholder brackets are replaced in the prompt
4. Redis is connected and functioning
5. ElevenLabs tools are properly configured to call n8n webhook

**Q: Calendar invites aren't being sent to customers.**

A: Ensure the "Calendar: Create Appointment" node has `sendUpdates: all` enabled and that customer email is added as an attendee. Check the prompt includes instructions to add email as attendee.

**Q: Responses are too slow.**

A: This usually means Redis isn't working. Verify Redis connection and check that `sessionId` is properly passed from ElevenLabs. Also reduce `contextWindowLength` if set very high.

**Q: The AI asks for the same information multiple times.**

A: The prompt instructs the AI to check `call_log` for previously mentioned info. Ensure:
1. Redis memory is functioning
2. ElevenLabs is passing `call_log` in the webhook payload
3. Session ID remains consistent throughout the call

### Customization

**Q: Can I add SMS confirmations or email receipts?**

A: Yes! Add n8n nodes after "Calendar: Create Appointment" to send messages via Twilio SMS, SendGrid, or similar services. Pull appointment details from the calendar event data.

**Q: How do I integrate with my CRM?**

A: Add CRM nodes (Salesforce, HubSpot, etc.) to the workflow. You can create/update contacts when appointments are booked. Connect them after the calendar creation step.

**Q: Can I add payment collection?**

A: You could integrate Stripe/payment nodes, but phone-based payment collection raises PCI compliance concerns. Consider sending payment links via SMS instead.

**Q: How do I handle cancellations and rescheduling?**

A: The workflow includes an "Update an event in Google Calendar" node. Enhance the prompt with cancellation/reschedule logic, and ensure ElevenLabs tools are configured to trigger these operations.

### ElevenLabs Specific

**Q: What should I put in the ElevenLabs knowledge base?**

A: Upload FAQs, pricing info, service descriptions, policies - anything the AI needs to answer without invoking n8n. Keep calendar-specific logic in n8n.

**Q: How does ElevenLabs know when to call n8n?**

A: Configure "Client Tools" in ElevenLabs that match the n8n tools (check availability, create appointment, update appointment). ElevenLabs will automatically invoke n8n when calendar operations are needed.

**Q: What's the latency like?**

A: Typical response times:
- ElevenLabs-only responses: <1 second
- n8n tool calls: 2-4 seconds (depends on Redis, AI model speed)
- First response with memory: Slightly slower as context loads

---

## Quick Start Checklist

- [ ] Twilio number connected to ElevenLabs
- [ ] ElevenLabs agent configured with knowledge base
- [ ] ElevenLabs tools point to n8n webhook
- [ ] n8n webhook path configured (replace `REPLACE ME`)
- [ ] Google Calendar IDs set (all 3 nodes)
- [ ] Redis connection configured
- [ ] All `[PLACEHOLDER]` values customized in prompt
- [ ] Anthropic API credentials added
- [ ] Test call completed successfully
- [ ] Calendar invite received by test email

---

## Support

Need help? Check n8n execution logs and ElevenLabs conversation history to debug issues. Most problems stem from missing credentials or unreplaced placeholder values.
