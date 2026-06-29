#  Dental Clinic AI Voice Agent — Full Setup Guide

> **Stack:** Twilio (calls) → n8n (workflow) → Gemini AI (brain) → Airtable (database)  
> **Cost:** Free trial on all platforms — no credit card needed to start testing

---

##  Files in This Package

| File | Purpose |
|------|---------|
| `dental-agent-DUMMY.json` | n8n workflow (import this first — works without any API keys) |
| `README.md` | This guide |

---

##  How It Works (Big Picture)

```
Patient calls Twilio number
        ↓
Twilio sends POST request → n8n Webhook
        ↓
n8n parses call data (step, speech, collected fields)
        ↓
Gemini AI generates natural response + extracts patient info
        ↓
IF all 4 fields collected (name, phone, datetime, reason)
    → Save to Airtable → Say confirmation → Hang up
ELSE
    → Say next question → Listen again → Loop back
```

**The 4 fields collected:**
1. Patient full name
2. Phone number (auto-captured from Twilio)
3. Preferred appointment date & time
4. Reason for visit

---

##  PHASE 1 — Test with Dummy Workflow (No APIs Needed)

### Step 1: Import Workflow into n8n

1. Open your n8n instance (local or Railway)
2. Click **"+"** → **"Import from file"**
3. Upload `dental-agent-DUMMY.json`
4. You should see 9 nodes appear
5. Click **"Activate"** toggle (top right)

### Step 2: Get Your Webhook URL

1. Click on the **" Incoming Call Webhook"** node
2. Copy the **Production URL** shown — it looks like:
   ```
   https://YOUR-N8N-URL/webhook/dental-call
   ```
3. Save this URL — you'll paste it into Twilio later

### Step 3: Update the Webhook URL Inside Workflow

1. Click on node **" Build Continue TwiML"**
2. Find this line in the code:
   ```javascript
   const N8N_BASE_URL = 'https://YOUR-N8N-URL';
   ```
3. Replace `YOUR-N8N-URL` with your actual n8n URL
4. Click **"Save"**

### Step 4: Test Without Twilio (HTTP Test)

You can test the workflow directly using a browser or Postman:

**Send a POST request to your webhook URL:**
```
POST https://YOUR-N8N-URL/webhook/dental-call
Content-Type: application/x-www-form-urlencoded

Body:
CallSid=TEST123&Caller=%2B923001234567&SpeechResult=My+name+is+Ahmed
```

Or use curl:
```bash
curl -X POST https://YOUR-N8N-URL/webhook/dental-call \
  -d "CallSid=TEST123" \
  -d "Caller=+923001234567" \
  -d "SpeechResult=My name is Ahmed"
```

You should get back XML like:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Say voice="Polly.Joanna">Thank you Ahmed! What date and time works best?</Say>
  <Gather input="speech" action="..." method="POST" ...>
  </Gather>
</Response>
```

Check n8n **Executions** tab to see full logs of each run.

---

##  PHASE 2 — Connect Real Services

### A) Set Up Twilio (Free Trial)

1. Go to [twilio.com](https://twilio.com) → **Sign Up Free**
2. Verify your phone number (required for trial)
3. Go to **Console → Phone Numbers → Manage → Buy a Number**
   - With free trial credit (~$15) you can get a number for $1/month
4. Click your new number → **Voice & Fax** section
5. Set **"A call comes in"** → **Webhook** → paste your n8n URL:
   ```
   https://YOUR-N8N-URL/webhook/dental-call
   ```
6. Method: **HTTP POST**
7. Click **Save**

>  **Trial Limitation:** Twilio will play *"This is a Twilio trial call"* before your agent speaks. This goes away when you add $5 to your account.

>  **Trial Limitation:** You can only call **verified numbers** during trial. Go to Twilio Console → Verified Caller IDs to add your phone.

---

### B) Set Up Google Gemini (Free)

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with Google account
3. Click **"Get API Key"** → **"Create API Key"**
4. Copy the key (starts with `AIza...`)
5. In n8n → **Settings → Credentials → Add Credential**
6. Search for **"Google Gemini"**
7. Paste your API key → Save

**Free limits:** 15 requests/minute, 1 million tokens/day — more than enough for testing.

---

### C) Set Up Airtable (Free)

#### Create the Base:
1. Go to [airtable.com](https://airtable.com) → Sign Up Free
2. Click **"Add a base"** → **"Start from scratch"**
3. Name it: `Dental Clinic`
4. Rename the default table to: `Appointments`

#### Add these exact columns:

| Field Name | Field Type |
|-----------|-----------|
| `Patient Name` | Single line text |
| `Phone` | Phone number |
| `Appointment Date & Time` | Single line text |
| `Reason for Visit` | Single line text |
| `Status` | Single select (add: Pending, Confirmed, Cancelled) |
| `Booked At` | Date (include time) |
| `Call SID` | Single line text |

#### Get Your API Credentials:
1. Go to [airtable.com/create/tokens](https://airtable.com/create/tokens)
2. Click **"Create new token"**
3. Name: `n8n-dental-agent`
4. Scopes: check `data.records:write` and `data.records:read`
5. Access: select your `Dental Clinic` base
6. Click **Create** → copy the token

#### Get Your Base ID:
1. Open your base in Airtable
2. Look at the URL: `https://airtable.com/appXXXXXXXXX/tblXXX/...`
3. The Base ID starts with `app` — copy it

---

## 🔧 PHASE 3 — Replace Dummy Nodes with Real Ones

### Replace DUMMY AI node with Gemini:

1. Delete node **" DUMMY AI Logic (Replace with Gemini)"**
2. Click **"+"** → search **"Google Gemini"**
3. Add it between ` Parse Call Data` and ` Booking Complete?`
4. Configure:
   - **Credential:** your Gemini credential
   - **Model:** `gemini-2.0-flash`
   - **Prompt (User message):**

```
You are Casey, a friendly dental receptionist at Bright Smile Dental Clinic on a phone call.

Your job: Collect these 4 things naturally in conversation:
1. Patient full name
2. Phone number
3. Preferred appointment date and time
4. Reason for visit (cleaning, checkup, toothache, filling, extraction, whitening, etc.)

Current collected data:
- Name: {{ $json.collectedName || "not yet" }}
- Phone: {{ $json.collectedPhone || "not yet" }}
- Date/Time: {{ $json.collectedDatetime || "not yet" }}
- Reason: {{ $json.collectedReason || "not yet" }}

Current conversation step: {{ $json.step }}
What the caller just said: "{{ $json.speechResult }}"

Rules:
- Keep response VERY SHORT — max 2 sentences (this is a phone call)
- Ask for ONE missing piece of info at a time
- Be warm and natural, not robotic
- If step is "1", greet first then ask for name
- Extract any info you detect from what they said
- When ALL 4 fields are collected → end response with exactly: [BOOKING_COMPLETE]
- Always end with: [DATA:{"name":"...","phone":"...","datetime":"...","reason":"..."}]
  (fill in whatever has been collected so far, use empty string if not yet known)
```

5. In the **"Process AI Response"** (you need to add this Code node), parse Gemini's output:

```javascript
const aiText = $input.first().json.text || $input.first().json.content || '';
const prev = $(' Parse Call Data').first().json;

// Extract DATA block
let extracted = {};
const dataMatch = aiText.match(/\[DATA:(.*?)\]/);
if (dataMatch) {
  try { extracted = JSON.parse(dataMatch[1]); } catch(e) {}
}

const isComplete = aiText.includes('[BOOKING_COMPLETE]');

let speechText = aiText
  .replace(/\[BOOKING_COMPLETE\]/g, '')
  .replace(/\[DATA:.*?\]/gs, '')
  .trim();

if (!speechText) speechText = 'I did not catch that, could you please repeat?';

return [{
  json: {
    aiResponse: speechText,
    isComplete,
    name: extracted.name || prev.collectedName || '',
    phone: extracted.phone || prev.collectedPhone || '',
    datetime: extracted.datetime || prev.collectedDatetime || '',
    reason: extracted.reason || prev.collectedReason || '',
    callSid: prev.callSid,
    nextStep: String(parseInt(prev.step || '1') + 1)
  }
}];
```

---

### Replace DUMMY Airtable node with real Airtable node:

1. Delete node **" DUMMY Airtable Save"**
2. Click **"+"** → search **"Airtable"**
3. Add it in the same position
4. Configure:
   - **Operation:** Create Record
   - **Credential:** Add new → paste your Airtable token
   - **Base:** paste your Base ID (`appXXXXXX`)
   - **Table:** `Appointments`
5. Map fields:

| Airtable Field | n8n Expression |
|---------------|---------------|
| Patient Name | `{{ $json.name }}` |
| Phone | `{{ $json.phone }}` |
| Appointment Date & Time | `{{ $json.datetime }}` |
| Reason for Visit | `{{ $json.reason }}` |
| Status | `Pending` |
| Booked At | `{{ new Date().toISOString() }}` |
| Call SID | `{{ $json.callSid }}` |

---

##  Full Test Flow (After Real Connections)

1. Call your Twilio number from your verified phone
2. You hear: *"Hello! Thank you for calling Bright Smile Dental..."*
3. Say your name → AI responds and asks for date
4. Say a date → AI asks for reason
5. Say reason → AI confirms and says goodbye
6. Open Airtable → check **Appointments** table → your record should be there 

---

##  Node Connection Map

```
[ Incoming Call Webhook]
         ↓
[ Parse Call Data]
         ↓
[ Gemini AI] ← replace dummy node
         ↓
[ Booking Complete?]
    ↓ TRUE            ↓ FALSE
[ Airtable]    [ Continue TwiML]
    ↓                  ↓
[ Confirm TwiML] [ Respond to Twilio]
    ↓
[ Respond to Twilio]
```

---

##  Troubleshooting

| Problem | Fix |
|---------|-----|
| Webhook not receiving | Make sure workflow is **Active** (toggle top right) |
| Twilio says "Application Error" | Check n8n logs — likely TwiML is malformed |
| AI not responding | Check Gemini API key is correct in credentials |
| Airtable save fails | Verify Base ID and field names match exactly |
| "Trial call" message | Normal on Twilio free trial — upgrade to remove |
| Caller not recognized | Add caller's number to Twilio Verified Caller IDs |
| Speech not detected | Increase `speechTimeout` value in Continue TwiML node |

---

##  Cost Summary

| Service | Free Tier | Enough For |
|---------|----------|-----------|
| n8n (self-hosted) | Free forever | Everything |
| Twilio | ~$15 trial credit | ~150 min of calls |
| Gemini API | 15 req/min, 1M tokens/day | Hundreds of calls |
| Airtable | 1,000 records/base | ~1,000 appointments |

**Total cost to test: $0** (using trial credits)

---

##  Support

Built for personal testing. Open n8n execution logs to debug — every node prints detailed `console.log` output visible in the **Executions** panel.
Always ready to be a part of such intresting projects . For any query fell free to ask .
