---
name: trugen-ai
description: Build, configure, and embed TruGen AI conversational video agents via the REST API — create agents, ground them with knowledge bases and slide decks, attach guardrails and tools, run real-time video proctoring/monitoring with Hawkeye vision sessions, send agents to external meetings, persist user memory, subscribe to webhook events, reuse templates, and ship them into a React (or any) frontend via widget/iframe/SDK — or add a TruGen avatar to an existing LiveKit voice agent. Use when a user wants to create or configure a TruGen agent, build an end-to-end product with one (interview agent with proctoring, sales SDR, support agent, coach, intake assistant, proctor), wire up RAG/guardrails/tools/memory, embed a live avatar conversation, or make a LiveKit voice agent avatar-ready.
license: MIT
compatibility: Requires a TruGen API key (server-side `x-api-key` header) from https://app.trugen.ai. REST base URL `https://api.trugen.ai/v1` (vision sessions and custom avatars use `/v2`). Any HTTP client works; JS (`@trugen/js-sdk`, `@trugen/sdk`, `@aiteammate/agent-widget`) and Python SDKs available for client-side embedding.
metadata:
  author: trugen.ai
  version: "1.1"
---

TruGen AI is a platform for building and deploying real-time conversational video agents. Agents are powered by proprietary foundation models — Huma-1 and Huma-2 for expressive avatar rendering (Huma-2 delivers sub-500ms end-to-end latency) and Hawkeye-1 for real-time vision understanding — and respond in under one second end to end. Everything is driven by a REST API; agents are embedded in a frontend via widget, iframe, or SDK, and can also join external meetings (Google Meet, Zoom, Teams, Webex) like a person.

## Authentication

TruGen uses two credentials for two different purposes:

| Credential | Where it lives | What it can do |
| --- | --- | --- |
| **API Key** | Your server, sent as the `x-api-key` header | Every REST endpoint: create agents, manage templates, start vision sessions, read conversations |
| **Session Token** | Passed to the browser | Join one live conversation (short-lived JWT, ~5 minutes) |

The API key is server-only — never embed it in browser or mobile client code. For client-side use, your server exchanges it for a session token:

```bash
POST https://api.trugen.ai/v1/auth/conversation
Headers: X-API-Key: <api-key>, Content-Type: application/json
Body: { "agentId": "<agent-id>" }
Response: { "token": "<jwt>" }
```

The client then initializes the JS SDK with `createClient({ token })` — no API key ever reaches the browser. See `/api-reference/overview`, `/docs/sdks/javascript/authentication`, `/docs/sdks/python/authentication`.

Base URL: `https://api.trugen.ai/v1`. Vision sessions use `https://api.trugen.ai/v2/vision`. All responses are JSON. Errors return `{"error": "..."}`. List endpoints paginate via `offset`/`perpage` query params, with total count in the `X-Total-Count` response header.

## Resource model

| Resource | Purpose |
| --- | --- |
| **Agents** | The primary object: a configured conversational video agent with a prompt, avatar, voice, LLM, tools, widget config, and webhook config |
| **Templates** | Reusable blueprints: share prompt, knowledge, and behavior across many agents |
| **Knowledge Bases** | Document collections agents ground answers in via RAG |
| **Presentations** | PDF/PPTX slide decks agents semantically search and *show* during a conversation (`visual_presentations`) |
| **Tools** | Callable actions agents invoke during conversations (webhooks, MCP, integrations, client-side UI actions) |
| **Guardrails** | Safety-check rules agents can call as a tool: canned responses, moderation checks, trigger webhooks |
| **MCPs** | Model Context Protocol server configurations |
| **Providers** | Available LLM, STT, and TTS providers (plus BYO OpenAI-compatible LLM) |
| **Avatars** | Visual identities: stock (any Huma model) or custom (Huma-3 only) |
| **Vision Sessions** | Hawkeye-1 real-time video analysis on a LiveKit room: emotions, gaze, head pose, face count |
| **Memory** | Per-user recall across conversations, driven by custom instructions |
| **Conversations** | Session records: chat history, feedback, usage |

Two agent capabilities are configured per agent but don't have their own resource: **External Meetings** (the agent joins Google Meet/Zoom/Teams/Webex via a dedicated `@agent.trugen.ai` email) and **Text to Video** (render an avatar video from a script; private preview).

## Common workflows

**Create an agent and embed it**
1. `POST /v1/ext/agent` with `agent_name`, `agent_system_prompt`, and an `avatars` array (`avatar_key_id` from the Avatar Gallery) — get back an `id`. Platform defaults: Groq `openai/gpt-oss-120b` LLM, Deepgram STT, ElevenLabs TTS, English, turn detection on, max call 300s, recording off.
2. Render it in your frontend: React widget (`@aiteammate/agent-widget`), an iframe (`https://app.trugen.ai/embed/{AGENT_ID}?username=&id=&context=`), or the JS/Python SDK for full session control. See the "Build the frontend" section below.
3. `PATCH /v1/ext/agent/{id}` to change behavior later — no redeploy needed, the next session picks it up.
See `/docs/quickstart`, `/api-reference/endpoint/agentcreate`. Note: some older examples use the legacy `POST /v1/agent/api` shape (`name`, `system_prompt`, `avatar_ids`, `config.timeout`) — prefer `/v1/ext/agent`.

**Personalize per user**
Pass the user into every session so the agent can address them and personalize: the React widget takes `agentId`, `userName`, `userId`, `context` props; the iframe takes the same as `?username=USER&id=USER_ID&context=CONTEXT` query params; the SDK takes them when minting the session token. See `/docs/integrations/widget-reference`.

**Reuse a template across many agents**
1. `POST /v1/ext/template` with the shared prompt, knowledge, and guardrail config.
2. `POST /v1/ext/agentbytemplate` with that `template_id` to spin up a new agent instantly.
See `/docs/agents/templates`, `/api-reference/endpoint/templatecreate`.

**Ground an agent in your data (RAG)**
1. Create a Knowledge Base (`/docs/agents/knowledge/creating`).
2. Add documents (PDF/DOCX), plain text, or URLs — content is extracted, chunked, and indexed automatically (`/docs/agents/knowledge/uploading`).
3. Attach the Knowledge Base to one or more agents/templates by ID (`/docs/agents/knowledge/attaching`).
Choose **Agentic RAG** (LLM decides when to search — more precise) or **Traditional RAG** (always searched — simpler, more predictable).

**Give the agent a slide deck (Visual Presentations)**
1. Upload and index a deck: `POST /v1/ext/presentation` with `multipart/form-data` — `name`, `description`, `input` (`.pdf` or `.pptx`, max 100 MB). Each page/slide is indexed; status moves `pending` → `processing` → `completed`.
   ```bash
   curl --location 'https://api.trugen.ai/v1/ext/presentation' \
     --header 'x-api-key: YOUR_API_KEY' \
     --form 'name="Product Pitch Deck"' \
     --form 'description="Use when asked about features, pricing, architecture, or product demos."' \
     --form 'input=@/path/to/deck.pdf'
   ```
2. Attach it when creating/updating an agent via the `visual_presentations` array: `"visual_presentations": [{ "id": "<presentation-id>" }]`.
3. During a session, when a user question matches, the agent **speaks the answer and presents the relevant slide** in the widget — it decides via semantic search, no scripting needed. If nothing matches well, it answers normally.
The `name` and `description` drive retrieval routing — write them like routing instructions ("Use when asked about X"). Manage with `GET`/`PATCH`/`DELETE` on `/v1/ext/presentation`. See `/docs/presentations/overview`, `/api-reference/endpoint/presentationcreate`.

**Run real-time proctoring or monitoring (Vision Understanding)**
Hawkeye-1 joins a LiveKit room as a subscriber and analyses the target participant's video in real time — **enterprise-only, billed flat 1¢/minute per session**.
1. Have (or create) the LiveKit room the participant's video flows through, with a `wss://` URL and a LiveKit access token (with `roomJoin: true`) for that room.
2. Start the session:
   ```bash
   curl --request POST \
     --url https://api.trugen.ai/v2/vision \
     --header 'Content-Type: application/json' \
     --header 'x-api-key: YOUR_API_KEY' \
     --data '{
       "livekit_url": "wss://your-domain.livekit.cloud",
       "access_token": "<livekit-jwt>",
       "participant": "candidate-identity",
       "callback_url": "wss://your-ws-endpoint.example.com/vision",
       "max_duration": 10,
       "modules": {
         "face_pose_detection": ["Looking Left", "Looking Right"],
         "emotion_recognition": ["angry", "sad", "happy", "surprise", "neutral"],
         "eyegaze_tracking": ["Looking left", "Looking right"],
         "face_count": ["2"],
         "face_out_of_focus": ["out_of_frame"]
       }
     }'
   ```
   Rules: omit `modules` for all defaults; an empty class list runs that module's full defaults; unknown class names are rejected with `400`. `max_duration` (minutes) is a hard cap — Hawkeye disconnects when reached. `participant` is a LiveKit **identity**, default `"auto"` (first non-agent participant).
3. Consume events in real time — every detection arrives on both channels:
   - WebSocket to your `callback_url`: `{"session_id": "...", "module": "emotion_recognition", "class": "angry", "timestamp": "..."}`.
   - LiveKit data channel (always on): `publish_data` with `reliable=True` on the **`hawkeye-events`** topic — any client in the room (your React proctor dashboard, a moderator view) filters for that topic and decodes `{"type": "alert", "payload": {...same fields...}}`. Best-effort delivery.
4. Poll `GET /v2/vision/{session_id}` for lifecycle (`IN_QUEUE` → `IN_PROGRESS` → `COMPLETED`/`FAILED`/`CANCELLED`), active modules, and elapsed billing duration. No need to poll faster than every few seconds.
Use it for exam proctoring, interview monitoring, engagement tracking, compliance monitoring. See `/docs/agents/vision/overview`, `/docs/agents/vision/start-session`, `/api-reference/endpoint/visionstart`.

**Attach a guardrail**
1. Create a guardrail with a `name`, a `prompt` describing what should trigger it, and a `response_message`.
2. Attach it to one or more agents.
3. The agent's LLM decides whether to call it each turn, exactly like any other tool. When it fires, the agent speaks the `response_message` and, if configured, POSTs a `guardrail_triggered` event to the guardrail's own `callback_url`.
See `/docs/agents/guardrails/overview`, `/api-reference/endpoint/guardrailcreate`.

**Give the agent an action (Tools)**
Register a Tool so the agent can act, not just respond. Three custom types:
- **API/Webhook**: function-calling against any HTTP endpoint — `"type": "tool.api"` with a JSON-schema `schema` (name/description/parameters) and a `request_config` (method, url, headers), plus optional `event_messages` (`on_start`/`on_success`/`on_delay`/`on_error`).
- **MCP**: point at your own MCP server (`shttp` or `sse` transport); it must be publicly reachable.
- **Client tool**: executes in your frontend — the agent triggers a UI event in your React app. Define it in the Studio, then handle it in your code: `client.on('tool:openHelpPanel', (params) => ...)` via `@trugen/sdk` `TrugenClient`. The tool name must exactly match the handler event name, including casing. Use the Studio's **Test** button to fire it with sample inputs.
Plus 1,000+ prebuilt apps via Composio integrations. Tool descriptions are the agent's only signal for when to call — write them like instructions to a smart colleague. See `/docs/agents/tools/overview`, `/docs/agents/tools/custom-tools`.

**Subscribe to live conversation events (webhooks)**
Set `callback_url` (public HTTPS) and `callback_events` on the agent (or its template):
```json
"callback_url": "https://yourdomain.com/webhooks/trugen",
"callback_events": [
  "agent.interrupted", "agent.started_speaking", "agent.stopped_speaking",
  "call_ended", "max_call_duration_timeout", "max_call_duration_warning",
  "participant_left", "tool_call", "utterance_committed",
  "user.started_speaking", "user.stopped_speaking"
]
```
Each event POSTs `{timestamp, conversation_id, type: "pipeline", event: {name, payload}}` — e.g. `utterance_committed` carries the finalized transcript text, `tool_call` carries `tool_name`/`parameters`/`result`, `call_ended` fires post-call workflows. Return 2xx fast, be idempotent. Guardrail webhooks are **separate**: configured per guardrail, not via `callback_events`. See `/docs/agents/callback`.

**Tune turn-taking and interruptions**
Set on `avatars[0].config` when creating/updating an agent:
- `turn_handling`: `proactive` (drives the conversation, default), `balanced` (natural back-and-forth), or `deliberate` (waits and speaks only when clearly prompted).
- `interruptability`: `low` (finishes its thought before yielding), `medium` (default), `high` (yields at the slightest sign of speech).
For interview or assessment agents, `deliberate` + `medium` usually feels right; for a sales SDR, `proactive`. See `/docs/agents/pipeline`.

**Persist user details (Memory)**
Enable Memory on the agent in the Studio (**Memory** tab → enable **Custom Instruction**) and write an instruction describing what to save (categories, timestamps, update rules). At the start of a conversation the agent loads that user's prior history and recalls relevant context; new details are stored as the conversation progresses. The docs include full example memory instructions for a sales agent and an educational mentor (`/docs/agents/memory`). Get user consent before storing sensitive details.

**Send the agent to external meetings**
Enable **Join external meetings** in the agent's **Advanced** settings in the Studio — the agent gets a dedicated email like `alex@agent.trugen.ai` (you pick the prefix). Invite that email as a guest in Google Calendar/Outlook, and the agent joins the Meet/Zoom/Teams/Webex call on time, handles waiting rooms and lobbies, participates with the proprietary **Chime In** turn-taking model, and stores the full transcript and recording when the call ends. With Vision enabled it can analyse the live meeting video too. See `/docs/agents/meetings/overview`, `/api-reference/endpoint/externalmeeting`.

**Switch an agent's LLM, STT, or TTS provider**
1. `GET /v1/providers` to list every currently available provider and model, grouped by type (`llm`, `stt`, `tts`).
2. `PATCH /v1/ext/agent/{id}` with an `avatars` entry setting `llm.provider`/`llm.model` (add `url`/`token` if `provider` is `custom`), `stt.provider`/`stt.model`, or `tts.provider`/`tts.model_id`/`tts.voice_id` — send only the block you're changing.
3. The change takes effect on the agent's next session; end active sessions first (`DELETE /v1/conversation/{conversationId}`) to force it immediately.
**BYO LLM**: any OpenAI-compatible endpoint works — `"llm": {"provider": "custom", "model": "...", "url": "...", "token": "..."}`. It must implement `POST /chat/completions` with SSE streaming; add `fallback_provider`/`fallback_model` for production resilience. Aim for <500ms time-to-first-token, <50ms network latency to TruGen. See `/docs/resources/byo-llm`, `/docs/providers/overview`, `/api-reference/endpoint/providerget`.

**Create a custom avatar (Huma-3 only)**
Custom avatars are only available on the **Huma-3** model — Huma-1 and Huma-2 agents cannot use a custom avatar.
1. Prepare a well-lit, front-facing source photo per `/docs/avatars/best-practices`.
2. Create the avatar in the Developer Platform (**Avatars → Create Avatar**) or programmatically with `POST /v2/custom-avatar` (base64-encoded `input_image`, `avatar_name`, `gender`).
3. Preview it in a live conversation and save it — you get back an `avatar_id` to use in any agent's `avatars` array, exactly like a stock avatar.
See `/docs/avatars/custom`. Custom avatars are in private preview; the full Huma-3 API surface isn't documented yet, so confirm current availability with TruGen before building against it.

**Make an existing LiveKit voice agent avatar-ready**
For teams that already have a voice-only LiveKit `AgentSession` and want to give it a face without leaving LiveKit or touching the TruGen REST API.
1. Install the latest version of the official plugin: Python `pip install -U "livekit-agents[trugen]"` (or `uv add "livekit-agents[trugen]"`), or JS/TS `npm install @livekit/agents-plugin-trugen@latest`. Don't pin to an old release — the plugin ships frequently.
2. Generate a TruGen API key from the [Developer Platform](https://app.trugen.ai) and set it as `TRUGEN_API_KEY` (plus `TRUGEN_AVATAR_ID` for a specific avatar) in your environment.
3. Instantiate an `AvatarSession` with the avatar ID and start it against your existing agent session and LiveKit room — e.g. Python: `trugen_avatar = trugen.AvatarSession(avatar_id=avatar_id)` then `await trugen_avatar.start(session, room=ctx.room)`, called right before `session.start(...)`. Your LLM, STT, TTS, and tool config stay exactly as already set up in LiveKit — the plugin only adds the rendered avatar track to the room.
4. The avatar session stops automatically when the room disconnects or the primary participant leaves; otherwise close it explicitly with `ctx.room.disconnect()` or `ctx.shutdown()`.
See `/docs/voice-to-video/livekit` for the JS variant (`AvatarSession` options, `LIVEKIT_URL`/`LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET`) and full examples.

**Read back a session**
Handle the `call_ended` webhook (`/docs/agents/callback`) or fetch `GET /v1/ext/conversation/{id}` for `chat_history`, `snippets`, `feedback`, and `usage_json`.

**Generate a video from a script (Text to Video, private preview)**
`POST /v1/script-to-video/createVideo` with `avatar_id`, `voice_id`, `provider_name`, `model_name`, `script`, and `callback_url` — rendering is asynchronous and the finished video link is POSTed to your callback. Poll the companion status endpoint for progress. Contact TruGen for access. See `/docs/agents/text_to_video`, `/api-reference/endpoint/texttovideo`.

## Build the frontend (React)

Every agent ships with a widget and can be embedded three ways. All paths render the same agent and pick up agent-level config changes on the next session.

| | React widget | iFrame | JS SDK |
| --- | --- | --- | --- |
| **Setup** | `npm install`, render component | Paste one HTML tag | Full session control |
| **Best for** | React/Next.js apps | Prototypes, any framework/CMS | Custom UI, fine-grained events, custom pipelines |
| **User context** | Typed props | URL query params | Passed when minting the token |

**React widget** — `npm install @aiteammate/agent-widget`, then:
```tsx
import "@aiteammate/agent-widget/styles.css";
import { TrugenAgentWidget } from "@aiteammate/agent-widget";

export default function TrugenWidget() {
  return (
    <TrugenAgentWidget
      agentId="YOUR_AGENT_ID"
      userName={user.name}
      userId={user.id}
      context="Context the agent sees, e.g. 'Candidate for Senior SWE role, 60-min technical round'"
    />
  );
}
```

**iFrame** — zero setup, works anywhere:
```html
<iframe
  src="https://app.trugen.ai/embed/{agent_id}?username=USER_NAME&id=USER_ID&context=CONTEXT"
  width="100%" height="600" frameborder="0"
  allow="camera; microphone; autoplay"
></iframe>
```

**JS SDK** (`@trugen/js-sdk`) — for custom UIs. Register listeners **before** `connect()`:
```ts
import { createClient, TruGenEvent } from "@trugen/js-sdk";

const session = await createClient({ token: sessionToken }); // from your server
session.on(TruGenEvent.VIDEO_STREAM_STARTED, (track) => track.attach(videoElement));
session.on(TruGenEvent.TEXT_CHUNK_RECEIVED, ({ text }) => appendTranscript(text));
session.on(TruGenEvent.USER_SPEECH_STARTED, () => setListening(true));
session.on(TruGenEvent.AGENT_SPEAKING_STARTED, () => setAvatarSpeaking(true));
session.on(TruGenEvent.CONNECTION_CLOSED, (code, reason) => handleDisconnect(code, reason));
await session.connect();
```
See `/docs/sdks/javascript/reference/list-all-events` for the full event enum.

**Widget appearance is agent config, not frontend code.** The agent's `widget` object (editable via `PATCH /v1/ext/agent/{id}` or the Studio) controls: `position` (`full` for inline panels, `left`/`right` for floating, `center`, `center-mini`), `widget_type` (`dual` Talk+Chat, `talk`, `chat`), theme (`default_theme: dark|light`), colors (`color_mode: solid|gradient`), branding (`company_name`, `company_logo`, `sub_text`), button labels, suggested topics, and `allowed domains` (restrict which origins can embed). See `/docs/integrations/widget-customization`, `/docs/integrations/widget-reference`.

**Agent-triggered UI (client tools).** Register a client tool in the Studio (name, description, parameters), then handle it in your React app with `@trugen/sdk`:
```ts
import { TrugenClient } from '@trugen/sdk';

const client = new TrugenClient({ agentId: 'YOUR_AGENT_ID' });
client.on('tool:openReportPanel', (params: { section: string }) => {
  openPanel(params.section); // agent decides when to trigger this
});
```

**Three event surfaces — pick per consumer:**
- **Server webhooks** (`callback_events`): pipeline events + `call_ended`, for backend logic, transcripts, analytics.
- **JS SDK events** (browser): live UI state — speaking indicators, transcript chunks, connection lifecycle.
- **LiveKit data channel** (`hawkeye-events` topic): Hawkeye vision alerts to any in-room client, no callback server needed.
See `/docs/agents/callback` for the full webhook reference.

## Reference recipes

### AI interview agent with proctoring

A structured interviewer that runs on camera, watches the candidate with Hawkeye, and produces a reviewable report.

1. **Create the interviewer agent.** Start from the example prompt in `/docs/examples/ai-interviewer` (ask role-relevant questions one at a time, follow up, stay neutral, summarize strengths, never make hiring decisions), and consider `turn_handling: "deliberate"`, a guardrail for off-limits questions (e.g. discrimination-adjacent topics), and a client tool like `flagConcern` your React UI can react to.
   ```bash
   curl --request POST \
     --url https://api.trugen.ai/v1/ext/agent \
     --header 'Content-Type: application/json' \
     --header 'x-api-key: YOUR_API_KEY' \
     --data '{
       "agent_name": "AI Interviewer",
       "agent_system_prompt": "You are an AI Interviewer conducting structured, professional interviews...",
       "avatars": [{ "avatar_key_id": "<from-avatar-gallery>" }],
       "callback_url": "https://yourdomain.com/webhooks/trugen",
       "callback_events": ["utterance_committed", "call_ended", "tool_call"]
     }'
   ```
2. **Embed it in your React app** with `TrugenAgentWidget` (full position for a dedicated interview room), passing `context` with the candidate's name, role, and interview stage.
3. **Attach proctoring.** Hawkeye joins the LiveKit room carrying the session — bring your own room (you need its `wss://` URL and an access token with `roomJoin: true`), start `POST /v2/vision` before the interview, and confirm scope with TruGen (enterprise-only). For external interviews the agent can instead join the meeting itself via its `@agent.trugen.ai` email with Vision enabled.
4. **Build the proctor dashboard.** React client subscribes to the `hawkeye-events` data-channel topic (gaze, head pose, emotion, face-count events) and renders live alerts; your server receives the same events via the vision `callback_url` (WebSocket) and the conversation webhooks (`utterance_committed` for the transcript, `call_ended` to finalize).
5. **Assemble the report.** After `call_ended`, fetch `GET /v1/ext/conversation/{id}` for `chat_history`/`feedback`, pair it with the vision event log (time-stamped `module`/`class` detections), and store per candidate. Set `record: true` if you want session recordings (check your privacy policy first).

### Sales SDR agent

A conversion-focused agent for demos, qualification, and objection handling (example prompt: `/docs/examples/sales-agents`).

1. **Create the agent** with a discovery/qualification system prompt, `turn_handling: "proactive"`.
2. **Attach a pitch deck** via `visual_presentations` so it presents the pricing or architecture slide when asked (`/docs/presentations/overview`).
3. **Wire actions**: a `tool.api` tool to log the lead to your CRM, a booking webhook to schedule demos, or a Composio integration. Add guardrails (never misrepresent capabilities, no pricing below floor).
4. **Add memory** with a custom instruction so it remembers each prospect's context across calls.
5. **Embed the widget** on the marketing site (`position: "right"`, branded colors/logo, suggested topics like "See pricing" or "Book a demo"), with `allowed domains` locked to your origins.
6. **Wire webhooks**: `call_ended` → push transcript + lead details to CRM; `tool_call` → audit trail.

### Map any use case to capabilities

| You want... | Reach for |
| --- | --- |
| Exam proctoring, interview monitoring, compliance | Vision Sessions (`POST /v2/vision`) + `hawkeye-events` |
| Agent that presents pitches, tutorials, walkthroughs | Visual Presentations |
| Agent in human meetings (Meet/Zoom/Teams/Webex) | External Meetings (`@agent.trugen.ai` email) |
| Personalized, cross-session rapport | Memory custom instructions |
| Grounded, factual answers | Knowledge Base (RAG) |
| Actions (CRM, booking, search, automations) | Tools (API/webhook, MCP, Composio, client tools) |
| Off-limits topics, canned responses | Guardrails (+ per-guardrail webhook) |
| Own fine-tuned/self-hosted LLM | BYO LLM (`provider: "custom"` + fallback) |
| Branded embed on a site | Agent `widget` config + React widget/iframe |
| Deeper UI integration, custom dashboards | JS SDK + client tools |
| One-off rendered videos (training, outreach) | Text to Video (private preview) |

## Constraints and known limitations

- API keys are server-only; exchange for a session token before touching a browser.
- A guardrail only fires if the LLM decides to call it — there's no code-level enforcement. Reliability depends on the system prompt explicitly instructing the agent to check its guardrails.
- Guardrails support `POST` (create), `GET` (list all), `PUT` (update), and `DELETE` only — there's no single-item `GET /ext/guardrail/{id}`.
- Detaching one guardrail from an agent means `PUT`-ing the full attached list minus that entry; there's no partial detach.
- There's no guardrail test sandbox — verify by attaching to a test agent and talking to it directly.
- Vision Understanding is enterprise-only and billed flat 1¢/minute per session. `POST /v2/vision` attaches to a LiveKit room you bring (URL + access token with `roomJoin: true`); `participant` must match the participant's LiveKit identity (not display name), and if that participant never joins, the session idles until `max_duration` fires. Data-channel delivery is best-effort; the WebSocket callback is the reliable path.
- Presentations accept `.pdf`/`.pptx` only, up to 100 MB, and process asynchronously. The docs conflict on attachment limits — the API takes a `visual_presentations` array, but `/docs/presentations/attaching` states an agent can be configured with one deck at a time in the Studio. Confirm current behavior with TruGen before attaching many decks.
- Vision sessions require a LiveKit room with its own URL and access token; if you're using the managed TruGen embed without your own LiveKit instance, coordinate with TruGen for the room details.
- LLM/STT/TTS model identifiers change as providers update their lineups — always call `GET /v1/providers` rather than hardcoding a model string.
- External Meetings are enabled per agent in the Studio (Advanced section); meeting emails are unique per agent and prefixes are reserved.
- Custom avatars require the **Huma-3** model and are in private preview; they are not available on Huma-1 or Huma-2 agents. Text to Video is also private preview.
- Client tool names must exactly match the handler event names in your frontend, including casing.
- SDK event listeners must be registered before `connect()` — late listeners miss startup events.
- The LiveKit plugin is a separate integration path from the REST API: it only adds an avatar track to a room you already run in LiveKit, and doesn't create a TruGen `agent` resource or go through `/v1/ext/agent`.

## Further reading

- `/docs/quickstart` — first agent in under five minutes
- `/api-reference/overview` — full REST API reference
- `/docs/agents/conversational-video-agents` — agent architecture and pipeline
- `/docs/agents/templates`, `/docs/agents/guardrails/overview`, `/docs/agents/knowledge/overview`, `/docs/agents/tools/overview`, `/docs/agents/memory`, `/docs/agents/pipeline`
- `/docs/agents/vision/overview`, `/docs/agents/vision/start-session` — Hawkeye proctoring and event delivery
- `/docs/presentations/overview`, `/docs/presentations/creating`, `/docs/presentations/attaching` — slide decks
- `/docs/agents/meetings/overview` — external meetings
- `/docs/integrations/widget-integration`, `/docs/integrations/widget-customization`, `/docs/integrations/widget-reference`, `/docs/integrations/embed-via-iFrame` — embedding
- `/docs/agents/callback` — webhook events reference
- `/docs/sdks/javascript/authentication`, `/docs/sdks/javascript/reference/event-handling` — session tokens and SDK events
- `/docs/resources/byo-llm` — bring your own LLM
- `/docs/examples/ai-interviewer`, `/docs/examples/sales-agents` — ready-made system prompts
- `/docs/voice-to-video/livekit` — add a TruGen avatar to an existing LiveKit voice agent
- `/llms.txt` — full documentation directory