🏨 AI Hotel Front-Desk & Payment Verification Bot — n8n Workflow

An AI-powered hotel front-desk assistant built with n8n. Guests chat to check room availability and book rooms live via Google Sheets, upload a payment receipt for AI-powered verification via Groq's vision model, and a human operator gives final YES/NO approval by replying to an email — which then triggers the guest's confirmation or rejection email automatically.

## 🔄 Workflow Overview

```
Chat Trigger → AI Agent (Check Availability / Update Booking) → Guest uploads receipt
                                                                        ↓
                                                    Convert Image → Detect Format
                                                                        ↓
                                                    Groq Vision Model → Parse Response
                                                                        ↓
                                                    Payment Verified? ──┬── Match → Email Operator (Verify Payment)
                                                                        └── Mismatch → Email Operator (Suspicious Receipt)
                                                                        
Gmail Trigger (watches operator reply) → Parse YES/NO → Find Guest Session
                                                                ↓
                                            Is Reply YES? ──┬── Yes → Confirmation Email to Guest
                                                             └── No → Rejection Email to Guest
```

## 🧩 Nodes

| Node | Type | Purpose |
|---|---|---|
| Chat Trigger | Chat Trigger | Starts guest conversation, accepts file uploads |
| AI Agent | LangChain Agent (`gpt-4.1-mini`) | Handles greeting, availability, and booking logic |
| Check Availability | Google Sheets Tool | Live read of room inventory for the AI Agent |
| Update Booking | Google Sheets Tool | Live write — marks a room booked, matched by room + availability |
| Window Buffer Memory | Memory (LangChain) | Keeps last 10 messages of conversation context |
| Has Receipt Image? | IF | Branches based on whether a file was uploaded |
| Convert Image to Base | Extract From File | Converts uploaded receipt to base64 |
| Detect Image Format | Code (JS) | Builds a correct base64 data URL with MIME type |
| Lookup Booked Room Price | Google Sheets | Fetches all rows to find the guest's expected price |
| Extract Price from Sheet | Code (JS) | Matches session ID to expected price, name, email |
| HTTP Request | HTTP Request (Groq vision) | Sends receipt image to `llama-4-scout-17b-16e-instruct` for verification |
| Parse Vision Response | Code (JS) | Parses AI response, strict numeric match check |
| Payment Verified? | IF | Routes matched vs mismatched receipts |
| Send Receipt Email to Hotel | Gmail | Notifies operator to manually confirm a matched payment |
| Send Suspicious Receipt Email | Gmail | Immediately flags a mismatched/suspicious receipt |
| Gmail Trigger - Watch Your Reply | Gmail Trigger | Watches operator inbox for YES/NO replies |
| Parse Your YES or NO Reply | Code (JS) | Extracts session ID and YES/NO verdict from reply |
| Find Matching Session | Code (JS) | Matches reply back to the correct guest booking |
| Is Reply YES? | IF | Routes confirmed vs rejected payments |
| Send Confirmation Email to Guest | Gmail | Tells guest booking is fully confirmed |
| Send Rejection Email to Guest | Gmail | Tells guest payment wasn't found, asks to re-upload |

## ⚙️ Setup

### 1. Prerequisites
- n8n (self-hosted or cloud)
- Groq API key — console.groq.com
- OpenAI API key (for the front-desk agent model)
- Google Sheets OAuth2 credentials
- Gmail OAuth2 credentials

### 2. Import Workflow
- Open your n8n instance
- Go to **Workflows → Import**
- Upload `hotel-ai-booking-workflow.json`

### 3. Configure Credentials

**Groq API Key**
In the "HTTP Request" node, set up an HTTP Header Auth credential:
```
Authorization: Bearer YOUR_GROQ_API_KEY
```

**OpenAI**
Connect your OpenAI API key credential to the "OpenAI Chat Model2" node.

**Google Sheets**
- Connect via OAuth2 in n8n credentials
- Point `documentId` at your own spreadsheet with columns: `Standard Room`, `Availability`, `Price`, `Guest Name`, `Guest Email`, `Booked At`, `session id`

**Gmail**
- Connect your Gmail account via OAuth2 (used for both the trigger and the send nodes)
- Update `sendTo` in "Send Receipt Email to Hotel" and "Send Suspicious Receipt Email" to your own operator inbox
- Update the `sender` filter in "Gmail Trigger - Watch Your Reply" to match that same inbox

## ✅ Booking & Verification Logic

| Stage | Condition | Result |
|---|---|---|
| Availability | Sheet value = Yes/yes/Y (case-insensitive) | Room shown to guest |
| Availability | Sheet value = No/no/N (case-insensitive) | Room hidden from guest |
| Receipt check | AI-extracted amount exactly matches expected price | ✅ Sent to operator for final confirmation |
| Receipt check | AI-extracted amount differs, even slightly | ⚠️ Flagged as suspicious, operator alerted immediately |
| Operator reply | Replies YES | ✅ Guest sent confirmation email |
| Operator reply | Replies NO | ❌ Guest sent rejection email, asked to re-upload |

## 📍 Location / Business Data

Currently configured for a specific hotel's spreadsheet (real Google Sheet ID) and a real personal Gmail inbox for operator alerts. To reuse this workflow for a different property:
- Replace the Google Sheet `documentId` with your own
- Replace the operator email in the Gmail nodes with your own

## 📦 Dependencies

- **Groq API** — LLaMA 4 Scout vision model for receipt verification
- **OpenAI API** — `gpt-4.1-mini` powering the front-desk agent
- **Google Sheets** — live room inventory and booking database
- **Gmail** — operator alerts, guest confirmations, and reply-based approval
- n8n built-in nodes: Chat Trigger, Google Sheets, Gmail, HTTP Request, Code, IF

## 🔐 Security Note

Never commit real API keys to version control. Use n8n's built-in credentials manager for Groq, OpenAI, Google Sheets, and Gmail. This workflow's exported JSON also contains a real Gmail address and Google Sheet ID — replace both before reusing for a different business.

## 📄 License

MIT