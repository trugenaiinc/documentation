---
name: trugen-ai
description: Build, configure, and embed TruGen AI conversational video agents via the REST API — create agents, ground them with knowledge bases, attach guardrails and tools, reuse templates, and drop them into a page via widget/iframe/SDK, or add a TruGen avatar to an existing LiveKit voice agent. Use when a user wants to create or configure a TruGen agent, wire up RAG/guardrails/tools, embed a live avatar conversation, or make a LiveKit voice agent avatar-ready.
license: MIT
compatibility: Requires a TruGen API key (server-side `x-api-key` header) from https://app.trugen.ai. REST base URL `https://api.trugen.ai/v1`. Any HTTP client works; JS (`@trugen/js-sdk`, `@aiteammate/agent-widget`) and Python SDKs available for client-side embedding.
metadata:
  author: trugen.ai
  version: "1.0"
---

TruGen AI is a platform for building and deploying real-time conversational video agents. Agents are powered by proprietary foundation models — Huma-1 and Huma-2 for expressive avatar rendering (Huma-2 delivers sub-500ms end-to-end latency) and Hawkeye-1 for vision-based action recognition — and respond in under one second end to end. Everything is driven by a REST API; agents are embedded in a frontend via widget, iframe, or SDK.

## Authentication

TruGen uses two credentials for two different purposes:

| Credential | Where it lives | What it can do |
| --- | --- | --- |
| **API Key** | Your server, sent as the `x-api-key` header | Every REST endpoint: create agents, manage templates, read conversations |
| **Session Token** | Passed to the browser | Join one live conversation |

The API key is server-only — never embed it in browser or mobile client code. For client-side use, exchange it for a short-lived session token via the JS or Python SDK auth flow. See `/api-reference/overview`, `/docs/sdks/javascript/authentication`, `/docs/sdks/python/authentication`.

Base URL: `https://api.trugen.ai/v1`. All responses are JSON. Errors return `{"error": "..."}`. List endpoints paginate via `offset`/`perpage` query params, with total count in the `X-Total-Count` response header.

## Resource model

| Resource | Purpose |
| --- | --- |
| **Agents** | The primary object: a configured conversational video agent with a prompt, avatar, voice, LLM, and tools |
| **Templates** | Reusable blueprints: share prompt, knowledge, and behavior across many agents |
| **Knowledge Bases** | Document collections agents ground answers in via RAG |
| **Tools** | Callable actions agents invoke during conversations (webhooks, MCP, integrations, client actions) |
| **Guardrails** | Safety-check rules agents can call as a tool: canned responses, moderation checks, trigger webhooks |
| **MCPs** | Model Context Protocol server configurations |
| **Providers** | Available LLM, STT, and TTS providers |
| **Avatars** | Visual identities: stock (any Huma model) or custom (Huma-3 only) |
| **Conversations** | Session records: chat history, feedback, usage |

## Common workflows

**Create an agent and embed it**
1. `POST /v1/ext/agent` with `agent_name`, `agent_system_prompt`, and an `avatars` array — get back an `id`.
2. Render it with the React widget (`@aiteammate/agent-widget`), an `<iframe src="https://app.trugen.ai/embed/<AGENT_ID>">`, or the JS/Python SDK for full session control.
3. `PATCH /v1/ext/agent/{id}` to change behavior later — no redeploy needed, the next session picks it up.
See `/docs/quickstart`, `/api-reference/endpoint/agentcreate`.

**Personalize per user**
Mint a session token with `userId`, `userName`, and `context` on each session; the agent sees that context and personalizes its responses. See `/docs/sdks/javascript/authentication`.

**Reuse a template across many agents**
1. `POST /v1/ext/template` with the shared prompt, knowledge, and guardrail config.
2. `POST /v1/ext/agentbytemplate` with that `template_id` to spin up a new agent instantly.
See `/docs/agents/templates`, `/api-reference/endpoint/templatecreate`.

**Ground an agent in your data (RAG)**
1. Create a Knowledge Base (`/docs/agents/knowledge/creating`).
2. Add documents (PDF/DOCX), plain text, or URLs — content is extracted, chunked, and indexed automatically (`/docs/agents/knowledge/uploading`).
3. Attach the Knowledge Base to one or more agents/templates by ID (`/docs/agents/knowledge/attaching`).
Choose **Agentic RAG** (LLM decides when to search — more precise) or **Traditional RAG** (always searched — simpler, more predictable).

**Attach a guardrail**
1. Create a guardrail with a `name`, a `prompt` describing what should trigger it, and a `response_message`.
2. Attach it to one or more agents.
3. The agent's LLM decides whether to call it each turn, exactly like any other tool. When it fires, the agent speaks the `response_message` and, if configured, POSTs a `guardrail_triggered` event to a webhook.
See `/docs/agents/guardrails/overview`, `/api-reference/endpoint/guardrailcreate`.

**Give the agent an action**
Register a Tool so the agent can act, not just respond: an API/webhook call, an MCP server, a Composio integration (1,000+ apps), or a client-side tool that triggers a UI event in your frontend. See `/docs/agents/tools/overview`.

**Switch an agent's LLM, STT, or TTS provider**
1. `GET /v1/providers` to list every currently available provider and model, grouped by type (`llm`, `stt`, `tts`).
2. `PATCH /v1/ext/agent/{id}` with an `avatars` entry setting `llm.provider`/`llm.model` (add `url`/`token` if `provider` is `custom`), `stt.provider`/`stt.model`, or `tts.provider`/`tts.model_id`/`tts.voice_id` — send only the block you're changing.
3. The change takes effect on the agent's next session; end active sessions first (`DELETE /v1/conversation/{conversationId}`) to force it immediately.
Always re-fetch `/v1/providers` before hardcoding a model identifier — new providers and models are added regularly. See `/docs/providers/overview`, `/api-reference/endpoint/providerget`, `/api-reference/endpoint/agentupdate`.

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

## Constraints and known limitations

- API keys are server-only; exchange for a session token before touching a browser.
- A guardrail only fires if the LLM decides to call it — there's no code-level enforcement. Reliability depends on the system prompt explicitly instructing the agent to check its guardrails.
- Guardrails support `POST` (create), `GET` (list all), `PUT` (update), and `DELETE` only — there's no single-item `GET /ext/guardrail/{id}`.
- Detaching one guardrail from an agent means `PUT`-ing the full attached list minus that entry; there's no partial detach.
- There's no guardrail test sandbox — verify by attaching to a test agent and talking to it directly.
- Custom avatars require the **Huma-3** model and are in private preview; they are not available on Huma-1 or Huma-2 agents.
- LLM/STT/TTS model identifiers change as providers update their lineups — always call `GET /v1/providers` rather than hardcoding a model string.
- The LiveKit plugin is a separate integration path from the REST API: it only adds an avatar track to a room you already run in LiveKit, and doesn't create a TruGen `agent` resource or go through `/v1/ext/agent`.

## Further reading

- `/docs/quickstart` — first agent in under five minutes
- `/api-reference/overview` — full REST API reference
- `/docs/agents/conversational-video-agents` — agent architecture and pipeline
- `/docs/agents/templates`, `/docs/agents/guardrails/overview`, `/docs/agents/knowledge/overview`, `/docs/agents/tools/overview`
- `/docs/voice-to-video/livekit` — add a TruGen avatar to an existing LiveKit voice agent
- `/llms.txt` — full documentation directory
