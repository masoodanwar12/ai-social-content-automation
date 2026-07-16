# 🎨 AI Social Content Automation Bot — n8n Workflow

An n8n workflow that turns a short chat prompt into a ready-to-review social media post: AI-drafted captions, a cartoon/claymation-style illustration, and a Trello card for human sign-off before anything publishes. Built for a UK children's social care organisation, but the pattern works for any brand needing AI-drafted content with a mandatory human review step.

## 🔄 Workflow Overview

```
Chat Trigger → Prep Prompt ──┬── Generate Captions (Groq) → Parse Caption Response ──┐
                              └── AI Agent (Illustration Brief) → Submit Job (Kie.ai)  │
                                                                        ↓               │
                                                    Capture Task ID → Wait → Check Status
                                                                        ↓
                                                        Image Ready? ──┬── Yes → Format Result
                                                                        └── No → Loop Back to Wait
                                                                        ↓
                                            Merge Captions + Illustration → Trello Card → Reply to Chat
```

## 🧩 Nodes

| Node | Type | Purpose |
|---|---|---|
| Chat Trigger | Chat Trigger | Guest/user types a topic to kick off content generation |
| Prep Prompt | Code (JS) | Cleans input, extracts brand name from the prompt |
| Generate Captions (Groq) | HTTP Request (Groq) | Drafts headline, Instagram + Facebook captions, hashtags via `llama-4-scout-17b-16e-instruct` |
| Parse Caption Response | Code (JS) | Parses AI's JSON response, with safe fallback on malformed output |
| AI Agent - Write Illustration Brief | LangChain Agent (`gpt-4.1-mini`) | Turns the topic into a strict cartoon/claymation image-generation brief |
| OpenAI Chat Model | LM Chat Model | Powers the illustration-brief agent |
| Submit Illustration Job (Kie.ai) | HTTP Request | Sends the brief to Kie.ai's text-to-image API |
| Capture Task ID | Code (JS) | Grabs the returned task ID for polling |
| Wait for Image Generation | Wait | Pauses 10 seconds between status checks |
| Check Illustration Status | HTTP Request | Polls Kie.ai for job completion |
| Image Ready? | IF | Routes to formatting once done, loops back if still processing |
| Format Illustration Result | Code (JS) | Combines captions with the final image URL |
| Merge Captions + Illustration | Code (JS) | Passes the combined payload forward |
| Create Trello Sign-off Card | Trello | Posts captions, hashtags, image, and illustration brief to an "Awaiting Sign-off" list |
| Reply to Chat | Set | Confirms back to the user that a draft is ready for review |

## ✅ Content Safety Rules

| Rule | Enforcement |
|---|---|
| Illustration style | Must be cartoon/claymation only — never photorealistic humans |
| Tone | Warm, hopeful, gentle — never dramatic or sensationalised |
| Text overlays / logos | Not generated — added separately by the brand team |
| Publishing | Nothing auto-publishes — every draft lands in Trello for human sign-off |
| Sensitive content | AI flags a `sensitivity_flag` if the topic touches safeguarding or case details |

## ⚙️ Setup

### 1. Prerequisites
- n8n (self-hosted or cloud)
- Groq API key — console.groq.com
- OpenAI API key (for the illustration-brief agent)
- Kie.ai API key — kie.ai
- Trello API credentials

### 2. Import Workflow
- Open your n8n instance
- Go to **Workflows → Import**
- Upload `ai-content-automation-workflow.json`

### 3. Configure Credentials

**Groq**
Set up an HTTP Header Auth credential for the "Generate Captions (Groq)" node:
```
Authorization: Bearer YOUR_GROQ_API_KEY
```

**OpenAI**
Connect your OpenAI API key credential to the "OpenAI Chat Model" node.

**Kie.ai**
In "Submit Illustration Job (Kie.ai)" and "Check Illustration Status," replace `YOUR_KIE_AI_API_KEY` with your real key:
```
Authorization: Bearer YOUR_KIE_AI_API_KEY
```
Recommended: convert this to a proper n8n HTTP Header Auth credential instead of a hardcoded header, so no key ever appears in the exported JSON.

**Trello**
- Connect your Trello account via OAuth2 in n8n credentials
- Update the `listId` in "Create Trello Sign-off Card" to match your own board's list

### 4. Activate the workflow
Test it by typing a sample topic into the chat trigger, e.g. *"info session about faith-friendly placements for Sparks Fostering."*

## 📦 Dependencies

- **Groq API** — LLaMA 4 Scout for caption generation
- **OpenAI API** — `gpt-4.1-mini` powering the illustration-brief agent
- **Kie.ai** — text-to-image generation
- **Trello** — human review/approval queue
- n8n built-in nodes: Chat Trigger, Code, HTTP Request, Wait, IF, Set

## 🔐 Security Note

Never commit real API keys to version control. This repo's copy has the Kie.ai key redacted to `YOUR_KIE_AI_API_KEY` — replace it with your own via n8n's credentials manager rather than pasting it back into the node directly.

## 📄 License

MIT