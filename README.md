# AI Content Automation Workflow (n8n)

An n8n workflow that turns a short chat prompt into a ready-to-review social media post — caption text, a cartoon/claymation-style illustration, and a Trello card for human sign-off. Built for a UK children's social care organisation, but the pattern generalizes to any brand that needs AI-drafted content with a mandatory human review step before publishing.

> ⚠️ **Note:** n8n workflows can't be "run" directly from a GitHub repo — GitHub just stores the exported JSON. To use this workflow, you need to import the JSON into your own n8n instance (cloud or self-hosted). See **Setup** below.

## What it does

1. **Chat Trigger** — user types a topic/prompt into n8n's chat interface (e.g. *"Sparks Fostering info session about faith-friendly placements"*).
2. **Prep Prompt** — a Code node cleans the input and extracts a brand name.
3. **Generate Captions (Groq)** — calls Groq's `llama-4-scout-17b-16e-instruct` model to draft an Instagram caption, Facebook caption, headline, hashtags, and a sensitivity flag — returned as strict JSON.
4. **Parse Caption Response** — parses/validates that JSON, with a safe fallback if the model returns malformed output.
5. **AI Agent – Write Illustration Brief** — an n8n LangChain agent (using `gpt-4.1-mini`) turns the topic into a detailed image-generation prompt, enforcing strict rules: cartoon/claymation style only, **never** a realistic human, warm and non-sensationalised tone.
6. **Submit Illustration Job (Kie.ai)** — sends that brief to Kie.ai's image generation API (`gpt-image/1.5-text-to-image`).
7. **Capture Task ID → Wait → Check Illustration Status → Image Ready?** — polls Kie.ai every 10 seconds until the image job succeeds (loops back if still processing).
8. **Format Illustration Result / Merge Captions + Illustration** — combines the generated captions with the final image URL.
9. **Create Trello Sign-off Card** — posts everything (captions, hashtags, image, the exact illustration brief used, sensitivity flag) as a Trello card in an "Awaiting Sign-off" list.
10. **Reply to Chat** — confirms back to the user in the n8n chat that a draft is ready for human review.

**Key design principle:** nothing in this workflow auto-publishes. Every output lands in Trello for a person to review and approve first.

## Tech stack

- **n8n** (workflow engine, self-hosted or cloud)
- **Groq API** — caption generation (`llama-4-scout-17b-16e-instruct`)
- **OpenAI** — `gpt-4.1-mini` powering the illustration-brief agent
- **Kie.ai** — text-to-image generation
- **Trello** — human review/approval queue

## Setup

1. **Import the workflow**
   - Open your n8n instance → **Workflows → Import from File** → select `ai-content-automation-workflow.json`.

2. **Add credentials** (n8n stores these separately from the workflow JSON — they are *not* included in this file):
   - **Groq**: create an HTTP Header Auth credential named to match the "Generate Captions (Groq)" node, with header `Authorization: Bearer YOUR_GROQ_API_KEY`.
   - **OpenAI**: add your OpenAI API key credential for the "OpenAI Chat Model" node.
   - **Kie.ai**: open the "Submit Illustration Job (Kie.ai)" and "Check Illustration Status" nodes and replace `YOUR_KIE_AI_API_KEY` in the Authorization header with your real key (or better, convert this to an n8n credential instead of a hardcoded header — see **Security note** below).
   - **Trello**: connect your Trello account credential, and update the `listId` in "Create Trello Sign-off Card" to match your own Trello board's list ID.

3. **Activate the workflow** and open the chat trigger to test it with a sample prompt.

## Security note

The original working copy of this workflow had the Kie.ai API key hardcoded directly in the node parameters. **That version must never be pushed to a public (or even private) GitHub repo**, since anyone with repo access — or anyone who finds it in your commit history — could read the key. This repo's copy has that key redacted to `YOUR_KIE_AI_API_KEY`.

Recommended fix going forward: move the Kie.ai auth into a proper n8n **HTTP Header Auth credential** (same pattern already used for Groq), so no key ever appears in the exported JSON.

## Repo structure

```
.
├── README.md
└── ai-content-automation-workflow.json
```

## License

MIT License
