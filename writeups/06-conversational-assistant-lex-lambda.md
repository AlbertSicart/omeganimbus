# OmegaNimbus Day 6: Conversational AI Assistant with Amazon Lex, Lambda & API Gateway

**Author:** Albert Sicart  
**Domain:** omeganimbus.com  
**Stack:** Amazon Lex V2 · Lambda · API Gateway · CloudFront · S3  
**Date:** May 2026

---

## Overview

This session added a fully serverless conversational assistant to the OmegaNimbus portfolio. The goal was to give recruiters and visitors an interactive way to explore the site — asking about skills, projects, certifications, and contact information — without navigating manually.

Key components built:

1. **Amazon Lex V2 bot** — `omeganimbus-assistant` with 6 custom intents covering the core portfolio narrative
2. **Lambda handler** — Python function invoking the Lex V2 runtime API and proxying requests from the web
3. **API Gateway endpoint** — `POST /chat` added to the existing `omeganimbus-api-cfn` HTTP API
4. **Frontend widget** — floating chat button integrated into all portfolio pages, styled to match the OmegaNimbus dark tech aesthetic

---

## Architecture

```
Browser (POST /chat with message + sessionId)
         │
         ▼
API Gateway HTTP API (omeganimbus-api-cfn) — eu-north-1
         │  POST /chat
         ▼
Lambda (omeganimbus-lex-handler) — Python 3.12 / arm64 — eu-west-1
         │
         └── Lex V2 Runtime (recognize_text) — eu-west-1
               └── omeganimbus-assistant bot
                     ├── WhatDoYouDo
                     ├── ShowProjects
                     ├── GetCertifications
                     ├── HowToContact
                     ├── WhatIsOmegaNimbus
                     └── SecuritySkills
                         │
                         ▼
Returns JSON { reply, sessionId, intent } → rendered in chat widget
```

**Note on regions:** The Lambda runs in eu-west-1 (Ireland) to minimize latency to the Lex V2 runtime, which also runs in eu-west-1. API Gateway remains in eu-north-1 (Stockholm) alongside the rest of the infrastructure.

---

## Part 1 — Amazon Lex V2 Bot

**Bot name:** `omeganimbus-assistant`  
**Bot ID:** `H6Y3EYRGEZ`  
**Language:** English (US)  
**Session timeout:** 5 minutes  
**IAM:** Auto-created service role with basic Lex permissions

### Intents

| Intent | Sample Utterances | Purpose |
|---|---|---|
| `WhatDoYouDo` | "Who are you", "What do you do", "Tell me about yourself" | Profile overview |
| `ShowProjects` | "Show me your projects", "What have you built", "What's in your portfolio" | Portfolio summary |
| `GetCertifications` | "What certifications do you have", "Are you AWS certified" | Certification roadmap |
| `HowToContact` | "How can I contact you", "I want to get in touch" | Contact info + links |
| `WhatIsOmegaNimbus` | "What is OmegaNimbus", "Tell me about this site" | Infrastructure explanation |
| `SecuritySkills` | "What are your security skills", "What do you know about cybersecurity" | Security background |
| `FallbackIntent` | *(auto)* | Unrecognized input handling |

Each intent uses a **Closing response** with a static text message — no slots, no Lambda fulfillment required. The bot is stateless at the Lex level; session continuity is handled by passing `sessionId` from the frontend.

### Bot versioning

Draft was published as **Version 1** and associated with the `prod` alias (`DJ65YZCGHP`) for stable invocation from Lambda.

---

## Part 2 — Lambda Handler

**Function name:** `omeganimbus-lex-handler`  
**Runtime:** Python 3.12 / arm64  
**Region:** eu-west-1  
**IAM policy added:** `AmazonLexRunBotsOnly`

```python
import boto3
import json
import uuid

client = boto3.client('lexv2-runtime', region_name='eu-west-1')

BOT_ID = 'H6Y3EYRGEZ'
BOT_ALIAS_ID = 'DJ65YZCGHP'
LOCALE_ID = 'en_US'

def lambda_handler(event, context):
    body = json.loads(event.get('body', '{}'))
    user_message = body.get('message', '')
    session_id = body.get('sessionId', str(uuid.uuid4()))

    response = client.recognize_text(
        botId=BOT_ID,
        botAliasId=BOT_ALIAS_ID,
        localeId=LOCALE_ID,
        sessionId=session_id,
        text=user_message
    )

    messages = response.get('messages', [])
    reply = messages[0]['content'] if messages else "I didn't understand that."

    return {
        'statusCode': 200,
        'headers': {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Content-Type',
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'reply': reply,
            'sessionId': session_id,
            'intent': response.get('sessionState', {}).get('intent', {}).get('name', '')
        })
    }
```

The handler extracts `message` and `sessionId` from the request body, calls `recognize_text` on the Lex V2 runtime, and returns the bot reply along with the matched intent name. The `sessionId` is passed back to the frontend so the browser can maintain conversational context across multiple messages.

---

## Part 3 — API Gateway

Added route `POST /chat` to the existing `omeganimbus-api-cfn` HTTP API (ID: `judzwkiy9h`) in eu-north-1. Lambda proxy integration pointing to `omeganimbus-lex-handler` in eu-west-1 — API Gateway supports cross-region Lambda integrations natively.

**Full endpoint:** `https://judzwkiy9h.execute-api.eu-north-1.amazonaws.com/chat`

Validated with CloudShell:

```bash
curl -X POST https://judzwkiy9h.execute-api.eu-north-1.amazonaws.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "who are you", "sessionId": "test-abc"}'
```

Response:
```json
{
  "reply": "I'm Albert Sicart, a Cloud Security Engineer based in Barcelona...",
  "sessionId": "test-abc",
  "intent": "WhatDoYouDo"
}
```

---

## Part 4 — Frontend Widget

A floating chat button integrated into all portfolio pages (`index.html`, `security.html`, `rekognition.html`, `notes.html`). Styled with the OmegaNimbus design system — Bebas Neue headers, Share Tech Mono labels, dark surface palette, red accent.

**Features:**
- Floating toggle button (bottom-right, red circle)
- Chat window with open/close animation
- Greeting message on first open
- Quick suggestion buttons (Who are you · Projects · Certifications · Contact)
- Typing indicator (`// thinking...`) while waiting for API response
- Session ID generated once per page load — maintains conversational context
- Suggestions hidden after first interaction

**Key frontend logic:**

```javascript
const CHAT_API = 'https://judzwkiy9h.execute-api.eu-north-1.amazonaws.com/chat';
let sessionId = 'session-' + Math.random().toString(36).substr(2, 9);

async function sendMessage() {
  const text = input.value.trim();
  if (!text) return;

  addMessage('user', text);
  const typing = addMessage('typing', '// thinking...');

  const res = await fetch(CHAT_API, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: text, sessionId })
  });
  const data = await res.json();
  typing.remove();
  addMessage('bot', data.reply);
}
```

---

## Debugging Notes

**Lambda created in wrong region (us-west-1)**  
Initial Lambda was accidentally created in N. California instead of eu-west-1. Lex V2 runtime is not available in us-west-1. Deleted and recreated in eu-west-1.

**Lex alias had no associated version**  
The `prod` alias was created before publishing a bot version, so it had no version to point to. Fixed by publishing Draft as Version 1 via Bot versions → Create version, then associating it with the alias.

**Route created as ANY instead of POST**  
API Gateway route `/chat` was initially created with method ANY. Changed to POST for consistency with the rest of the API routes.

**Nav item missing in desktop menu**  
`rekognition.html` link (Analyze) was present in the mobile menu of `index.html` but missing from the desktop nav. Added `<li><a href="rekognition.html">Analyze</a></li>` between SecOps and Notes.

---

## Cost Summary

| Service | Usage | Cost |
|---|---|---|
| Amazon Lex V2 | $0.004 per text request | ~$0 at portfolio scale |
| Lambda | <1M requests/month | ~$0 |
| API Gateway | <1M calls/month | ~$0 |

**Total: ~$0/month at portfolio traffic levels**  
Lex V2 free tier: 10,000 text requests/month for the first 12 months.

---

## Key Concepts Practiced

- Amazon Lex V2: bot creation, intent design, utterance training, closing responses
- Lex V2 versioning and alias management
- `lexv2-runtime` Boto3 client: `recognize_text` API
- Session management in stateless serverless architectures
- Cross-region Lambda integration with API Gateway
- IAM least privilege: `AmazonLexRunBotsOnly` scoped policy
- Frontend chat widget: floating UI, message threading, session continuity
- CloudShell for API endpoint validation

---

## What's Next

| Feature | Details | Priority |
|---|---|---|
| Study Notes polish | CCP notes section improvements, SAA structure prep | Medium |
| WAF + Shield | DDoS protection on CloudFront distribution | Low |
| GuardDuty dashboard improvements | Sample findings generator integration | Low |
| RDS + ElastiCache | Relational database layer | Low |

---

*Albert Sicart · [omeganimbus.com](https://omeganimbus.com) · [github.com/AlbertSicart](https://github.com/AlbertSicart) · [linkedin.com/in/albertsicart](https://linkedin.com/in/albertsicart)*
