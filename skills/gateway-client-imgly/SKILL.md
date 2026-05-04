---
name: gateway-client-imgly
description: Build IMG.LY-hosted apps that call the IMG.LY AI Gateway using the end user's IMG.LY/Clerk session — no API key on the client, no token-mint backend. Activate ONLY when the user is building inside the IMG.LY hosted app surface (e.g. an app deployed under `*.img.ly`, an IMG.LY-managed Clerk session, or explicitly mentions wanting to authenticate gateway calls with a Clerk JWT issued by IMG.LY's auth setup). For external integrators bringing their own auth, use the `gateway-client` skill instead.
---

# IMG.LY Gateway — Client Integration for IMG.LY-Hosted Apps

You are helping a developer who is building an app **hosted by IMG.LY** (deployed under the IMG.LY product surface, using IMG.LY's Clerk-managed end-user auth). This is a narrow audience — it does not apply to external integrators using the gateway from their own infrastructure.

If you are not sure the user is in this audience, **stop and use the `gateway-client` skill instead.** That skill covers the standard integration paths (API key, gateway JWT mint flow) used by the vast majority of integrators.

## When this skill applies

Activate only when **all** of the following are true:

1. The app being built will be deployed under IMG.LY hosting (e.g. an IMG.LY-managed subdomain, an IMG.LY-published product surface).
2. End users authenticate via IMG.LY's Clerk integration — there is no separate IMG.LY API key in play; the app relies on the user's IMG.LY session.
3. The developer wants to call `https://gateway.img.ly` from the client using the user's session JWT directly.

If the developer is bringing their own API key, building outside the IMG.LY hosting surface, or running their own backend that mints gateway JWTs, this skill does not apply — point them at the `gateway-client` skill.

## How auth works in this mode

The user's session is represented by a Clerk-issued JWT. The gateway accepts these JWTs and bills generation against the user's IMG.LY account.

The client sends the Clerk session JWT directly to the gateway:

```
Authorization: Bearer <clerk-session-jwt>
```

There is no token-mint hop, no `POST /v1/tokens`, no API key on the client.

### Wiring this up in code

```typescript
import { useAuth } from "@clerk/clerk-react"; // or your framework's Clerk SDK

function useGatewayFetch() {
  const { getToken } = useAuth();

  return async function gatewayFetch(path: string, init: RequestInit = {}) {
    const token = await getToken(); // Clerk session JWT
    const headers = new Headers(init.headers);
    headers.set("Authorization", `Bearer ${token}`);
    headers.set("Content-Type", "application/json");
    return fetch(`https://gateway.img.ly${path}`, { ...init, headers });
  };
}
```

For server-side rendering or server actions, retrieve the token from the Clerk server SDK (`auth().getToken()` in Next.js, equivalent helpers elsewhere) and forward it on the gateway request.

## What's the same as the standard integration

Everything else: endpoints, request shapes, SSE event names, error codes, schema-driven inputs, model discovery, asset upload flow.

For all of those, **fetch the live reference**:

```
WebFetch https://gateway.img.ly/llms.txt
```

That document is the source of truth for endpoints and SSE — read it before answering specific questions about generation, models, schemas, or uploads. The only thing that document does not cover is the Clerk JWT auth path described here, because that path is specific to the IMG.LY hosting surface and not part of the public integration guide.

## Things to be aware of

1. **Tokens still expire.** Clerk session JWTs are short-lived. The Clerk SDK's `getToken()` handles refresh automatically when called per-request — don't cache it across long-running operations.
2. **The user's scopes are managed by IMG.LY.** A Clerk-authenticated user gets a default scope set on the gateway side; it cannot be widened from the client. If a request fails with `provider_not_authorized`, the gateway is signalling that this user's account is not entitled to call that model — surface a clear error rather than retrying.
3. **Credits are billed to the IMG.LY account associated with the user.** Treat `insufficient_credits` (402) as an account-level state and direct the user to the IMG.LY dashboard to top up.
4. **Don't put an IMG.LY API key (`sk_...`) in this kind of app.** This auth model exists specifically so that you don't need to. If you find yourself reaching for `POST /v1/tokens`, you're in the wrong skill — the user is probably in the standard integration path and `gateway-client` applies.
5. **CORS, cookies, and origin restrictions** are configured for the IMG.LY hosting surface. If your app is being served from somewhere else, this auth path may not work — fall back to the standard integration.

## What this skill does NOT cover

- Provisioning Clerk apps, configuring IMG.LY hosting, deploying under IMG.LY domains — those are platform concerns handled outside the gateway.
- The standard public integration paths (API key, gateway JWT mint) — see `gateway-client`.
- Internal gateway architecture or operations — not relevant to client integration.
