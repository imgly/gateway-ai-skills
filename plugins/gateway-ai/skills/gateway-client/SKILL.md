---
name: gateway-client
description: Build clients that integrate with the IMG.LY AI Gateway (`gateway.img.ly`) — discovering models, generating images/video/audio/text, uploading inputs, consuming the SSE response stream, and picking up long-running generations by request id. Use whenever the user wants to call, integrate with, or build a client for the IMG.LY Gateway, or mentions `gateway.img.ly`, `IMG.LY Gateway`, IMG.LY AI generation, or an IMG.LY API key starting with `sk_`.
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
- **Responses are SSE streams by default.** Even for fast image models, the response is `text/event-stream`. Parse a `200` as SSE — never try to read it as a single JSON body. Events: `generation.status`, `generation.delta` (streaming text), `generation.completed`, `generation.failed`, `generation.detached`.
- **A generation is not tied to its stream.** Every generation has a `request_id` (in each event and in the `x-gateway-request-id` header). Most image, video and audio models run as background jobs, and for those `GET /v1/responses/<request_id>` returns the state — `working`, `completed` with the result, or `failed` — at any time: after a dropped connection, after a `generation.detached` event, or instead of a stream altogether. Text generations exist only on their stream.
- **No stream at all, if you prefer.** Send `Prefer: respond-async` with `POST /v1/responses` and a background-job model answers `202` with the `request_id` as soon as the job is submitted; poll the status endpoint for the result. Where the preference cannot be honored (text models, a few stream-only media models) the answer is the usual `200` stream — check the status code.
- **Inputs are schema-driven.** Every model declares its input schema at `GET /v1/models/schema?model=<id>`. Use this to build forms or validate inputs — do not hardcode model-specific fields. The schema includes IMG.LY UI extensions (`x-imgly-builder`, `x-imgly-enum-labels`, `x-imgly-enum-icons`, `x-property-order`) that hint at how the parameter should be rendered.
- **Model IDs are `creator/model`** (e.g. `bfl/flux-2`, `google/veo-3.1-fast`). The catalog is dynamic — fetch `GET /v1/models` at runtime instead of hardcoding model lists.
- **Provider passthrough reaches models the catalog does not wrap**, under `@<provider>/<native id>` (e.g. `@falai/fal-ai/flux-2`). The provider's own fields go in `provider_input: { … }` next to `model`, untranslated; the schema endpoint serves the provider's schema for such ids.
- **Output URLs are gateway-hosted.** `generation.completed` events and status results contain URLs under `gateway.img.ly/v1/assets/...`. They redirect to short-lived signed URLs and don't require authentication.

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

**"Video is slow."** → Expected. Video generations emit multiple `generation.status` events (often 30s–5min) before `generation.completed`. Show progress from `data.progress` if present. A job the gateway can no longer follow ends its stream with `generation.detached` — that is "still working", not an error: poll `GET /v1/responses/<request_id>` after `poll_after_ms`.

**"The connection dropped — is my generation lost?"** → No, if it was an image, video or audio job. Keep the `request_id` from the first event (or the response header) and poll `GET /v1/responses/<request_id>` until `status` is `completed` or `failed`, waiting `poll_interval_ms` between requests. The result holds the same output URLs the stream would have delivered.

**"My server can't hold a connection open for minutes."** → Send `Prefer: respond-async`: a `202` with the `request_id` comes back at once, then poll as above. The reference has a complete example (submit, poll, handle `working` / `completed` / `failed`).

## Key gotchas

1. **Parse a `200` as SSE.** Even for sub-second generations. Only a `202` — which you get solely after sending `Prefer: respond-async` — is a JSON body.
2. **Tokens expire fast.** Cache them but refresh before `exp`. Don't reuse a JWT across requests after it expires — the gateway returns 401.
3. **Image-to-image models require `image_urls`.** Upload via `/v1/uploads` first if you have local files; you cannot send raw bytes to `/v1/responses`.
4. **Don't hardcode model lists or schemas.** Both are dynamic. Build your UI to read from `/v1/models` and `/v1/models/schema`.
5. **Errors are structured JSON.** `{ "error": { "code": "...", "message": "..." } }` — not plain text. Read `error.code` to drive retry/UI logic.
6. **Output URLs are stable handles.** They redirect to short-lived signed URLs internally, but the URL you receive in `generation.completed` is the one to keep — it stays valid.
7. **Closing the stream does not cancel a background job.** An image, video or audio generation runs to its end once submitted and is charged like any completed one; there is no cancel. Don't abort a request to "save" a generation the user navigated away from — fetch its result later, or accept the charge. Failed jobs are not charged.
8. **`generation.detached` is not a failure.** Don't surface it as an error or retry the generation (that would pay twice) — poll for the result.
9. **Results are not stored forever.** Read a finished generation soon and download what you need to keep. A `410 result_unavailable` on the status endpoint means the result was only ever delivered on its stream.
10. **Mint end-user JWTs with a `sub`.** A token with a `sub` can read only that end user's generations by request id; one without can read the whole account's.

## When the user wants code

After fetching the live reference, prefer adapting the examples already in `llms.txt` (server-side API key, client-side JWT flow, image-to-image with upload, video, text streaming, schema-driven UI, provider passthrough, long-running generation without a stream). Match the user's language/framework. The SSE parser is the most error-prone part — use the `readGatewaySSE` helper from the reference rather than rewriting the loop.

## What this skill does NOT cover

- Provisioning API keys, billing, account setup → those happen in the IMG.LY dashboard.
- Internal gateway architecture and operations → not relevant to client integration.
- Building/extending the gateway itself → unrelated; this skill is for *consumers* of the gateway.
