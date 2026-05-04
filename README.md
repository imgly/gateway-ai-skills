# IMG.LY Gateway — AI Skills for Claude Code

Claude Code plugin that helps developers integrate clients with the [IMG.LY AI Gateway](https://gateway.img.ly) — a unified API for image, video, audio, and text generation.

When this plugin is installed, Claude Code will automatically activate the relevant skill whenever you ask about integrating with `gateway.img.ly`, calling IMG.LY AI generation, parsing the gateway's SSE stream, or using IMG.LY API keys (`sk_...`). Skills load the live integration reference at use-time, so guidance stays current as the gateway evolves.

## Install

In Claude Code, add the marketplace and install the plugin:

```
/plugin marketplace add imgly/gateway-ai-skills
/plugin install gateway-ai@gateway-ai-skills
```

That's it. Ask Claude something like *"Help me call the IMG.LY Gateway from my Next.js app"* and the skill will activate automatically.

## What's included

- **`gateway-client`** — The default skill for everyone integrating the gateway from their own infrastructure. Covers API key (server-side) and gateway JWT (browser) auth, model discovery, schema-driven inputs, generation, SSE parsing, and uploads. Activates on general gateway integration questions.
- **`gateway-client-imgly`** — A narrower skill for developers building apps **hosted on the IMG.LY product surface** that authenticate end users via IMG.LY's Clerk integration and call the gateway with the user's session JWT directly (no API key on the client, no token-mint backend). Activates only when the user is clearly in this hosted-app context. Most integrators do **not** need this skill.

## What it covers

- Auth methods (API key, gateway JWT) and when to use each
- Discovering models (`GET /v1/models`) and reading per-model input schemas (`GET /v1/models/schema`)
- Generating content via `POST /v1/responses` and consuming the SSE stream (`generation.status`, `generation.delta`, `generation.completed`, `generation.failed`)
- Uploading input images (`POST /v1/uploads`) for image-to-image and image-to-video workflows
- Schema-driven UI generation using IMG.LY's OpenAPI extensions (`x-imgly-builder`, `x-imgly-enum-labels`, etc.)
- Common gotchas (SSE parsing, token expiry, structured error handling)

## What it does NOT cover

- Provisioning API keys, billing, or account setup (handled in the IMG.LY dashboard)
- Building or extending the gateway itself

## Using without Claude Code

The same integration reference this skill points at is available directly:

- **`https://gateway.img.ly/llms.txt`** — machine-readable integration guide for any LLM tool

If you're using Cursor, Cline, Continue, or any other agent that supports `llms.txt`, point it at that URL.

## License

MIT. See [LICENSE](LICENSE).
