---
name: gateway-client
description: Build clients that integrate with the IMG.LY AI Gateway (`gateway.img.ly`) — discovering models, generating images/video/audio/text, uploading inputs, and consuming the SSE response stream. Use whenever the user wants to call, integrate with, or build a client for the IMG.LY Gateway, or mentions `gateway.img.ly`, `IMG.LY Gateway`, IMG.LY AI generation, or an IMG.LY API key starting with `sk_`.
---

# IMG.LY Gateway — Client Integration

You are helping a developer integrate with the IMG.LY AI Gateway. The gateway is a unified REST + SSE API that routes requests to AI providers for image, video, audio, and text generation. Clients only deal with one API — auth, billing, asset storage, and provider routing are handled server-side.

## Always start by fetching the live integration guide

The gateway publishes a current, machine-readable integration reference at:

**`https://gateway.img.ly/llms.txt`**

Fetch this **before answering specific questions** about endpoints, request shapes, SSE event names, error codes, or available capabilities. The reference is updated as the gateway evolves and supersedes anything you may have learned from training data.

```
WebFetch https://gateway.img.ly/llms.txt
```

Use the live reference as the source of truth. The notes below are stable mental-model context that doesn't change with releases.

## Mental model

- **One endpoint for everything.** All generation goes through `POST /v1/responses`. The request body shape varies by model type (media models take `prompt` + model-specific properties; text models take `messages` in OpenAI chat-completions format), but the route, auth, and response transport are uniform.
- **All responses are SSE streams.** Even for fast image models, the response is `text/event-stream`. Always parse as SSE — never try to read it as a single JSON body. Events: `generation.status`, `generation.delta` (streaming text), `generation.completed`, `generation.failed`.
- **Inputs are schema-driven.** Every model declares its input schema at `GET /v1/models/schema?model=<id>`. Use this to build forms or validate inputs — do not hardcode model-specific fields. The schema includes IMG.LY UI extensions (`x-imgly-builder`, `x-imgly-enum-labels`, `x-imgly-enum-icons`, `x-property-order`) that hint at how the parameter should be rendered.
- **Model IDs are `creator/model`** (e.g. `bfl/flux-2`, `google/veo-3.1-fast`). The catalog is dynamic — fetch `GET /v1/models` at runtime instead of hardcoding model lists.
- **Output URLs are gateway-hosted.** `generation.completed` events contain URLs under `gateway.img.ly/v1/assets/...`. They redirect to short-lived signed URLs and don't require authentication.

## Two auth methods — pick the right one

| Method | When to use | Header |
|---|---|---|
| **API key** (`sk_...`) | Server-side code; never in browsers | `Authorization: Bearer sk_live_...` |
| **Gateway JWT** | Browser/client code. Mint via `POST /v1/tokens` from your backend (5–15 min TTL) | `Authorization: Bearer eyJ...` |

For apps with a browser frontend, the **JWT mint flow** is the recommended pattern: keep the API key server-side, mint short-lived tokens for the browser, refresh on expiry. The full flow with code is in the live reference (Example 2).

For server-to-server attribution, set `X-End-User-Id: <your-user-id>` alongside the API key.

## Common decision points

**"What models can I use?"** → `GET /v1/models`. Filter with `?capability=text2image|image2image|text2video|image2video|text2speech|text2text`. Group with `?groupBy=capability`.

**"How do I build a UI for this model's options?"** → `GET /v1/models/schema?model=<id>`. Render the `input_schema.properties` in `x-property-order`, using `x-imgly-builder` hints for component selection and `x-imgly-enum-labels`/`x-imgly-enum-icons` for enum presentation.

**"How do I send an input image?"** → `POST /v1/uploads` to get a presigned URL → `PUT` the image bytes directly to that URL → use the returned `asset_url` in the `image_urls` field of your generation request.

**"My fetch hangs / I don't see the result."** → You're probably reading the response as JSON. It's SSE — you must read the body as a stream and parse `event:` / `data:` frames split by `\n\n`. The reference has a reusable `readGatewaySSE` helper.

**"How do I handle credits / billing?"** → Don't manage them client-side. The gateway accounts for credits automatically. Just handle a `402 insufficient_credits` error gracefully (prompt the user to top up).

**"Video is slow."** → Expected. Video generations emit multiple `generation.status` events (often 30s–5min) before `generation.completed`. Show progress from `data.progress` if present.

## Key gotchas

1. **Always parse as SSE.** Even for sub-second generations.
2. **Tokens expire fast.** Cache them but refresh before `exp`. Don't reuse a JWT across requests after it expires — the gateway returns 401.
3. **Image-to-image models require `image_urls`.** Upload via `/v1/uploads` first if you have local files; you cannot send raw bytes to `/v1/responses`.
4. **Don't hardcode model lists or schemas.** Both are dynamic. Build your UI to read from `/v1/models` and `/v1/models/schema`.
5. **Errors are structured JSON.** `{ "error": { "code": "...", "message": "..." } }` — not plain text. Read `error.code` to drive retry/UI logic.
6. **Output URLs are stable handles.** They redirect to short-lived signed URLs internally, but the URL you receive in `generation.completed` is the one to keep — it stays valid.

## When the user wants code

After fetching the live reference, prefer adapting the examples already in `llms.txt` (server-side API key, client-side JWT flow, image-to-image with upload, video, text streaming, schema-driven UI). Match the user's language/framework. The SSE parser is the most error-prone part — use the `readGatewaySSE` helper from the reference rather than rewriting the loop.

## What this skill does NOT cover

- Provisioning API keys, billing, account setup → those happen in the IMG.LY dashboard.
- Internal gateway architecture and operations → not relevant to client integration.
- Building/extending the gateway itself → unrelated; this skill is for *consumers* of the gateway.
