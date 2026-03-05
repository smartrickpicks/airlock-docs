# Web-to-iMessage Handoff: First Touch Design

> **Status:** BRAINSTORM
> **Problem:** When someone's on our website, the message they send should turn into a text — and they should know they're going to receive a text for further instructions. It needs to be done for them so we get the interaction the way we want.
> **Core constraint:** The prospect must initiate the iMessage conversation (not us texting them cold), which also solves the iMessage rate limit / "Report Junk" problem entirely.

---

## The Design Challenge

Static web forms are a dead end. You collect a name + email, throw it into a CRM, and the lead sits in a queue. Response time is hours or days. By then, the prospect's intent has cooled.

What we want:
1. Visitor is on the website, interested in something
2. They engage with a widget / CTA / chat bubble
3. Within 30 seconds, the conversation moves to iMessage
4. The qualification workflow runs via text — conversational, personal, immediate
5. The visitor never filled out a boring form. They just had a conversation.

The trick: **the prospect sends the first text**. This makes it prospect-initiated contact, which means:
- No "Report Junk" link (Apple only shows this for unsolicited messages)
- No risk of hitting iMessage rate limits
- The conversation feels natural, not spammy
- We get the phone number verified (they texted from it)

---

## The Three Handoff Patterns

### Pattern 1: The Smart Widget (Primary — Desktop + Mobile)

A floating chat-style widget on the website that collects just enough context to bridge to iMessage.

```
+------------------------------------------+
|  WEBSITE PAGE                            |
|                                          |
|  ... content ...                         |
|                                          |
|                       +----------------+ |
|                       | Got questions? | |
|                       | Text us.       | |
|                       |      [->]      | |
|                       +----------------+ |
+------------------------------------------+
```

**Click/tap opens the widget:**

```
+-------------------------------+
| Text Us                    [x]|
|-------------------------------|
|                               |
| We respond faster over text   |
| than email. Drop your number  |
| and we'll pick up the         |
| conversation right there.     |
|                               |
| +---------------------------+ |
| | (___) ___-____            | |
| +---------------------------+ |
|                               |
| What's this about?            |
| +---------------------------+ |
| | Contracts  | Licensing    | |
| | Pricing    | Partnership  | |
| | Other                     | |
| +---------------------------+ |
|                               |
| [ Text me ->]                 |
|                               |
| We'll text you from           |
| (310) 555-0142                |
| Normal messaging rates apply  |
+-------------------------------+
```

**What happens when they click "Text me":**

1. Widget shows: "Check your texts! We just sent you a message."
2. Backend fires a POST to the iMessage bridge: sends one message from the Smart Line to the phone number they entered
3. The text reads:

```
Hey! This is [Company Name]. You were just on our site
looking at [topic they selected]. I'm here to help.

What's the best way I can assist you today?
```

4. The Conversational Ask workflow activates — the prospect is now in a live iMessage thread with the Smart Line
5. AI handles the intake. Asks qualifying questions. Routes to a human when ready.
6. The web widget transitions to a confirmation state:

```
+-------------------------------+
| You're all set             [x]|
|-------------------------------|
|                               |
|  [checkmark icon]             |
|                               |
|  We texted you at             |
|  (***) ***-4521               |
|                               |
|  Reply to that text and       |
|  we'll take it from there.    |
|                               |
|  If you didn't get it,        |
|  text us directly:            |
|  (310) 555-0142               |
|                               |
+-------------------------------+
```

**Why this works:**
- Minimal friction: phone number + topic selection. 5 seconds max.
- The prospect knows exactly what's happening: "they're going to text me"
- The first message from us is contextual (references what they were looking at)
- If the outbound text fails delivery, the fallback CTA gives them the number to text us first (prospect-initiated)
- The topic selection pre-seeds the workflow with intent classification
- We captured a verified phone number (they'll reply from it)

**Mobile optimization:** On mobile, the "Text me" button can instead open the native Messages app with a pre-filled message via `sms:` URI:

```
sms:+13105550142&body=Hi! I was on your site looking at [topic]
```

This makes the prospect send the first text — fully prospect-initiated. The workflow triggers on inbound message receipt. Even better for rate limits.

---

### Pattern 2: The QR Bridge (Events, Print, In-Person)

For situations where someone's not on a phone screen — trade shows, printed materials, office signage, business cards, slide decks.

```
+------------------------------------------+
|                                          |
|  Scan to text us                         |
|                                          |
|  +----------+                            |
|  |  [QR]    |   Questions? Deals?        |
|  |  [CODE]  |   Contracts?               |
|  |  [HERE]  |                            |
|  +----------+   Scan this and we'll      |
|                  pick up in your texts.   |
|                                          |
|  Or text (310) 555-0142 directly         |
|                                          |
+------------------------------------------+
```

**The QR code encodes:**

```
sms:+13105550142&body=Hi, I scanned your QR at [event/location]
```

When scanned on iPhone:
1. Opens Messages app
2. Pre-fills the recipient (Smart Line number) and a starter message
3. User taps Send — this is fully prospect-initiated
4. Inbound message hits the Smart Line
5. Workflow activates with context from the pre-filled body text
6. AI responds within seconds with the intake flow

**QR variants by context:**
- Trade show booth: `body=Hi, I'm at [Event Name] and interested in learning more`
- Business card: `body=Hi, we connected at [Event]`
- Slide deck CTA: `body=Hi, I saw your presentation on [Topic]`
- Office lobby: `body=Hi, I'm visiting your office`

Each variant gives the workflow engine context about where this lead came from — source attribution without asking.

---

### Pattern 3: The Embedded CTA (Content Pages, Email Signatures, Social Bios)

For places where a widget doesn't make sense but you want the same handoff.

**On a pricing page:**
```
Ready to talk numbers? Text us: (310) 555-0142
We respond in under 2 minutes. No forms, no waiting.
```

**In an email signature:**
```
Sarah Chen | VP Partnerships
sarah@company.com | Text us: (310) 555-0142
```

**In a social bio:**
```
Text us for deals: (310) 555-0142
```

**In a "Contact Us" page (instead of a form):**
```
+------------------------------------------+
|  Let's talk                              |
|                                          |
|  Skip the form. Text us directly and     |
|  we'll have a real conversation.         |
|                                          |
|  +------------------------------------+ |
|  |  [iMessage icon]                    | |
|  |                                     | |
|  |  Text (310) 555-0142               | |
|  |                                     | |
|  |  Or tap to open Messages:          | |
|  |  [ Open Messages ]                 | |
|  |                                     | |
|  |  We respond in under 2 minutes     | |
|  |  Mon-Fri 8am-6pm PT                | |
|  +------------------------------------+ |
|                                          |
|  Prefer email? hello@company.com         |
|  (but seriously, text is faster)         |
|                                          |
+------------------------------------------+
```

The "Open Messages" button uses the `sms:` URI scheme. On iPhone, this opens Messages with the number pre-filled. The prospect types and sends their first message. Fully initiated by them.

---

## The Workflow Behind the Handoff

Regardless of which pattern brought them in, the same workflow engine handles the conversation.

### Trigger: `inbound_message` on Smart Line

```
Workflow: "Website Lead Intake"
Trigger: Inbound Message on Smart Line numbers

Step 1: FETCH — Phone # lookup
    Known contact? → Route to assigned rep (skip intake)
    Unknown? → Continue to Step 2

Step 2: AI CLASSIFY — Parse the first message
    Extract: source (web widget, QR, direct text)
    Extract: topic/intent (from pre-filled body or free text)
    Extract: urgency signals

Step 3: CONVERSATIONAL ASK — "What's your name?"
    AI: "Thanks for reaching out! I'm [Company]'s assistant.
         What's your name and what company are you with?"
    Parse response → create vault_contact

Step 4: BRANCH — Based on topic from Step 2
    Topic == "contracts" → Ask about volume
    Topic == "pricing" → Ask about current solution
    Topic == "partnership" → Ask about company size
    Topic == "other" → Open-ended: "Tell me more about what you need"

Step 5: CONVERSATIONAL ASK — Qualifying question (topic-specific)
    AI: "How many contracts does your team manage monthly?"
    Parse response → set qualification_score modifier

Step 6: AI CLASSIFY — Score the lead
    Inputs: company name, role, volume, topic, urgency
    Output: lead_score (0-100)

Step 7: BRANCH — Route based on score
    Score >= 80 → Create vault (lifecycle_stage='sql'), assign to AE
    Score 50-79 → Route to SDR pool for further qualification
    Score < 50 → Send resource link, add to nurture sequence

Step 8: SEND MESSAGE — Confirmation
    High score: "Great news — I'm connecting you with [AE name].
                 They'll text you within [SLA time]. In the meantime,
                 here's [relevant resource link]."

    Medium score: "Thanks [name]! One of our team members will follow
                   up shortly. They'll text you from this same number."

    Low score: "Thanks for reaching out, [name]! Based on what you
               described, [resource link] might be a great starting
               point. We're here when you need us."

Step 9: CREATE TASK — For the assigned human
    Title: "Follow up with [name] from [company] — [topic]"
    Context: Full conversation transcript + lead score + source
    Due: Within SLA window
    Module: CRM
```

### The Key Conversational Ask Difference

Static web forms ask everything upfront. Our approach:

| Static Form | Conversational Intake |
|---|---|
| 8 fields to fill out | 2-3 natural questions |
| All fields required | Questions adapt based on answers |
| Submit button → "we'll get back to you" | Each answer moves the conversation forward |
| Response in 24-48 hours | First AI response in under 30 seconds |
| Lead sits in a queue | Lead is being qualified in real-time |
| No context on intent | Every message enriches the CRM record |
| One-size-fits-all | Routing changes based on who they are |

---

## The Web Widget: Technical Spec

### Widget Architecture

The widget is a lightweight JS embed that lives on the customer's website. It doesn't need our full app — just a small script tag.

```html
<!-- Airlock Smart Widget -->
<script
  src="https://widget.airlock.app/v1/embed.js"
  data-smart-line="+13105550142"
  data-workspace="acme-corp"
  data-theme="dark"
  data-position="bottom-right"
></script>
```

### Widget States

**State 1: Collapsed (default)**
```
Floating button, bottom-right corner.
Subtle pulse animation every 30 seconds.

+------------+
| Text us  > |
+------------+
```

**State 2: Expanded (clicked)**
```
+-------------------------------+
| Text Us                    [x]|
|-------------------------------|
|                               |
| Skip the form.               |
| We'll text you instead.      |
|                               |
| +---------------------------+ |
| | Phone number              | |
| +---------------------------+ |
|                               |
| What's this about?            |
|  [Contracts] [Pricing]        |
|  [Partnership] [Other]        |
|                               |
| [  Send me a text  ->]       |
|                               |
| We text from (310) 555-0142  |
+-------------------------------+
```

**State 3: Sending (processing)**
```
+-------------------------------+
| Text Us                       |
|-------------------------------|
|                               |
|  [spinner]                    |
|  Sending you a text...        |
|                               |
+-------------------------------+
```

**State 4: Sent (confirmation)**
```
+-------------------------------+
| You're set                 [x]|
|-------------------------------|
|                               |
|  [check icon]                 |
|                               |
|  Check your messages.         |
|  We just texted you at        |
|  (***) ***-4521               |
|                               |
|  Reply to continue the        |
|  conversation there.          |
|                               |
|  Didn't get it? Text us:      |
|  (310) 555-0142               |
|                               |
+-------------------------------+
```

**State 5: Mobile redirect (iOS detected)**

On iOS devices, skip the phone number field entirely. The button opens Messages directly:

```
+-------------------------------+
| Text Us                    [x]|
|-------------------------------|
|                               |
| What's this about?            |
|  [Contracts] [Pricing]        |
|  [Partnership] [Other]        |
|                               |
| [ Open Messages ->]          |
|                               |
| Opens iMessage with our       |
| number pre-filled.            |
| Just tap send.                |
|                               |
+-------------------------------+
```

The "Open Messages" button links to:
```
sms:+13105550142&body=Hi! I'm on your site, interested in [selected topic].
```

This is the best path because:
- No phone number entry needed (they're texting FROM their number)
- Fully prospect-initiated (zero rate limit risk)
- Opens native Messages app (familiar, trusted UX)
- Pre-filled message gives the workflow engine context immediately

### Mobile Detection Logic

```javascript
function getHandoffStrategy() {
  const isIOS = /iPhone|iPad|iPod/.test(navigator.userAgent);
  const isAndroid = /Android/.test(navigator.userAgent);
  const isMobile = isIOS || isAndroid;

  if (isIOS) {
    // Best path: open Messages app directly with sms: URI
    return 'sms_uri';
  } else if (isAndroid) {
    // Android: sms: URI works too, opens default messaging app
    return 'sms_uri';
  } else {
    // Desktop: collect phone number, we'll send the first text
    return 'collect_phone';
  }
}
```

**On mobile (sms_uri):** No phone number field needed. Topic selection + "Open Messages" button. The prospect sends the first text. Workflow triggers on inbound.

**On desktop:** Phone number field + topic selection + "Send me a text" button. We send the first outbound text to bridge them to iMessage. This is the only path where we send an unsolicited first message — but the user literally just asked us to. The widget submission is their consent.

### Widget API Endpoint

When the desktop widget submits (the `collect_phone` path):

```
POST /api/widget/handoff
{
  "phone": "+15551234567",
  "topic": "contracts",
  "page_url": "https://company.com/pricing",
  "page_title": "Pricing - Company",
  "workspace_id": "acme-corp",
  "smart_line": "+13105550142",
  "utm_source": "google",
  "utm_campaign": "spring-2026",
  "referrer": "https://google.com/search?q=contract+management",
  "device": "desktop",
  "browser": "Chrome 124"
}
```

Backend:
1. Validate phone number (E.164 format)
2. Check if phone number already exists in vault_contacts → if yes, route to assigned rep
3. If new: send iMessage via bridge with contextual opener
4. Create `communications` record (direction: outbound, channel: imessage)
5. Create pending `workflow_run` that activates when the prospect replies
6. Return `{ status: "sent", masked_phone: "(***) ***-4567" }`

### Widget Styling

Matches the Airlock dark theme by default, with a `data-theme` override:

```css
/* Dark theme (default) */
--widget-bg: #0F1219;
--widget-surface: #151923;
--widget-border: #1E2330;
--widget-text: #E2E8F0;
--widget-muted: #64748B;
--widget-accent: #00D1FF;
--widget-cta: #F59E0B;

/* Light theme (for light-themed websites) */
--widget-bg: #FFFFFF;
--widget-surface: #F8FAFC;
--widget-border: #E2E8F0;
--widget-text: #1E293B;
--widget-muted: #64748B;
--widget-accent: #0091EA;
--widget-cta: #F59E0B;
```

CTA button uses `--widget-cta` (amber) — same as Airlock's action color. Stands out on both dark and light backgrounds.

---

## Page-Aware Context

The widget knows what page the visitor is on. This enriches the workflow:

| Page | Pre-filled Topic | First AI Message |
|---|---|---|
| `/pricing` | Pricing | "Saw you checking out our pricing. Happy to walk you through which plan fits..." |
| `/features/contracts` | Contracts | "Interested in our contract management? How many contracts does your team handle monthly?" |
| `/case-studies/sony` | Partnership | "Reading about our Sony case study? Want to hear how we could do something similar for you?" |
| `/blog/ai-contract-review` | Contracts | "That article on AI contract review caught your eye. Want to see it in action?" |
| `/demo` | Demo request | "Ready for a demo? Let me find the right person to walk you through it..." |
| Generic / home | Other | "Thanks for reaching out! What brings you to [Company] today?" |

The page URL and title get passed through the widget payload, and the workflow's first Conversational Ask node adapts its opener based on `trigger_data.page_url`.

### UTM Attribution

The widget captures UTM parameters from the page URL. These flow through to the CRM record:

```
utm_source   → lead.source (google, linkedin, referral)
utm_medium   → lead.medium (cpc, organic, social)
utm_campaign → lead.campaign (spring-2026, product-launch)
utm_content  → lead.content (pricing-cta, hero-banner)
```

This means when a rep picks up the conversation, they already know: "This lead came from a Google Ads campaign targeting contract management, landed on the pricing page, and selected 'Contracts' as their topic." Full context before the first human interaction.

---

## The Fallback Chain

Not every handoff will succeed on the first try. The system handles failures gracefully:

```
Step 1: Try iMessage via bridge
    Success? → Workflow activates, conversation begins

Step 2: iMessage delivery failed (not an Apple device, or bridge down)
    → Try SMS via Twilio to the same number
    → Same message content, just green bubble instead of blue

Step 3: SMS delivery failed (invalid number, carrier block)
    → Widget shows: "We couldn't reach that number.
       Text us directly at (310) 555-0142
       or email hello@company.com"
    → Create task: "Manual follow-up needed — delivery failed"

Step 4: Prospect never replies (sent text, no response after 24h)
    → Send one follow-up: "Hey [if we have name, else 'there']!
       Just checking in from [Company]. Still interested in
       [topic]? No pressure — text back anytime."
    → After 48h of no reply: mark lead as 'unresponsive',
       add to email nurture sequence if we have their email
    → No further texts (respect the silence)
```

**Critical: Max 2 outbound texts without a reply.** After the initial bridge text and one follow-up, we stop. This protects the Apple ID from being flagged and respects the prospect's attention. If they want to engage, they know the number.

---

## After-Hours Handling

The Smart Line is always on, but humans aren't always available.

```
Prospect texts at 11pm:

AI: "Hey! Thanks for reaching out to [Company].
     Our team is offline right now (we're back at 8am PT),
     but I can help with a few things:

     1. Send you info about [topic they mentioned]
     2. Schedule a time to chat tomorrow
     3. Answer quick questions about what we do

     What works for you?"

→ If they pick 1: Send a relevant resource link immediately
→ If they pick 2: Show available time slots (Calendly-style, via text)
→ If they pick 3: AI handles FAQ-level questions
→ If they say something else: AI notes it, creates priority task for morning

Next morning at 8am:
    Task fires: "Follow up with [name] — texted at 11pm about [topic]"
    Assigned rep sees full conversation transcript
    Rep texts from the same Smart Line number
    Prospect sees a seamless continuation
```

---

## Analytics & Tracking

### Widget Analytics

Track these events for the widget:

| Event | What It Tells You |
|---|---|
| `widget.impression` | How many visitors see the widget |
| `widget.opened` | How many click/tap to expand |
| `widget.topic_selected` | What topics are most popular |
| `widget.submitted` | How many complete the handoff |
| `widget.text_sent` | How many outbound texts were delivered |
| `widget.text_failed` | Delivery failures (wrong number, bridge down) |
| `widget.reply_received` | How many prospects replied (conversion!) |
| `widget.time_to_reply` | How fast prospects respond after handoff |

**Key metric: Widget-to-Reply Rate.** If 100 people open the widget and 40 end up in an iMessage conversation, that's a 40% conversion rate. Compare that to a typical web form's 2-5% conversion rate.

### Funnel Dashboard (in CRM Discover section)

```
Website Traffic
    |
    v
Widget Impressions -------- 10,000/mo
    |
    v
Widget Opened ------------- 2,400 (24% open rate)
    |
    v
Handoff Submitted --------- 960 (40% of opened)
    |
    v
Text Delivered ------------ 912 (95% delivery)
    |
    v
Prospect Replied ---------- 638 (70% reply rate!)
    |
    v
Qualified (MQL) ----------- 287 (45% of replies)
    |
    v
Sales Accepted (SAL) ------ 143 (50% of MQL)
    |
    v
Sales Qualified (SQL) ----- 72 (50% of SAL)
    |
    v
Opportunity Created ------- 54 (75% of SQL)
```

These numbers are aspirational but grounded in SMS response rate data (45% response rate for SMS, 98% open rate). iMessage should perform even better due to blue bubbles, read receipts, and richer media.

---

## Multiple Smart Lines (Multi-Product / Multi-Region)

When the business scales, different Smart Lines serve different purposes:

| Smart Line | Number | Purpose | Widget Config |
|---|---|---|---|
| General | (310) 555-0142 | Main intake, all topics | Default on homepage |
| Contracts | (310) 555-0143 | Contract-specific inquiries | On `/features/contracts` pages |
| Support | (310) 555-0144 | Existing customer support | In customer portal |
| Enterprise | (310) 555-0145 | Enterprise sales | On `/enterprise` page |

The widget's `data-smart-line` attribute controls which number it uses. The workflow engine has separate intake workflows per Smart Line. But the CRM sees everything — all conversations, all contacts, all context — unified in the vault hierarchy.

---

## Data Flow Summary

```
1. Visitor lands on website
       |
2. Widget renders (page-aware, device-aware)
       |
3. MOBILE: "Open Messages" → sms: URI → Messages app
   DESKTOP: Phone number + topic → "Send me a text"
       |
4. Prospect sends text (mobile) OR we send bridge text (desktop)
       |
5. Inbound message hits Smart Line
       |
6. Workflow "Website Lead Intake" triggers
       |
7. FETCH: Phone # lookup → known or unknown contact
       |
8. AI CLASSIFY: Parse message for intent, source, topic
       |
9. CONVERSATIONAL ASK: "What's your name?" → create vault_contact
       |
10. BRANCH: Route by topic → topic-specific qualifying questions
       |
11. AI CLASSIFY: Score the lead (0-100)
       |
12. ROUTE: Score >= 80 → AE | 50-79 → SDR Pool | < 50 → Nurture
       |
13. SEND MESSAGE: Contextual confirmation with next steps
       |
14. CREATE TASK: For assigned human with full conversation context
       |
15. CRM RECORD: vault_contact + communications + lead_score + source attribution
```

The visitor went from "browsing a website" to "in a qualified conversation with a human who has full context" in under 5 minutes. No forms. No email black hole. No "someone will get back to you in 24-48 hours."

---

## Demo Priority

For the HTML prototype, build:

1. **The Smart Widget mockup** — all 5 states (collapsed, expanded, sending, sent, mobile redirect) as a demo page
2. **The handoff flow animation** — show the transition from widget → iMessage conversation (mocked text bubbles)
3. **The intake workflow** — mocked Conversational Ask sequence showing 3-4 exchanges
4. **The CRM result** — show what the rep sees: vault_contact with full context, lead score, conversation transcript, source attribution

### Demo Page Location

```
/Features/CRM/web-widget-demo.html
```

This demo should be wirable into the demo shell under CRM > Discover > Inbox or a standalone "Smart Widget" view.

---

## Related Specs

- **[CRM Communications](./crm-comms-brainstorm.md)** — Smart Line architecture, iMessage bridge, pool model
- **[Workflow Engine](../WorkflowEngine/overview.md)** — The automation engine powering the intake flow
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — Where contacts and leads live in the data model
- **[Universal Task System](../TaskSystem/overview.md)** — How follow-up tasks get created
- **[Notifications](../Notifications/overview.md)** — How reps get alerted about new leads
