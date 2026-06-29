# Integrating @nizam-os/operator-sdk — multi-tenant scope

A Nizam user can belong to **more than one organization**. Your bearer token
proves *who you are*; it does not pin *which organization* a request applies
to. The active organization is resolved server-side and validated against
your live memberships on every call, so switching organizations is instant
(no token re-issue) and revocation takes effect on the next request.

Full guide: <https://docs.nizam.ai/getting-started/multi-tenant-scope>

## Single-organization users

No header needed — the API auto-resolves the caller's one organization.

```ts
import { NizamOperatorClient } from '@nizam-os/operator-sdk';

const client = new NizamOperatorClient({ token: accessToken });
```

## Multi-organization users

Send `X-Nizam-Organization: <slug>` on every request — either as a default
header on the client, or per call.

```ts
// Option A — default header (one client always acts as one organization)
const client = new NizamOperatorClient({
  token: accessToken,
  headers: { 'X-Nizam-Organization': activeOrgSlug },
});

// Option B — per call (one client acting across organizations)
await client.activeOrganization.getActiveOrganization({
  headers: { 'X-Nizam-Organization': activeOrgSlug },
});
```

The slug is a routing hint, not an authority: the server validates it against
your active memberships on every request and enforces tenant isolation
regardless of the input — a slug you aren't a member of can never widen access.

## Responses to handle

| Status | Code | Meaning | Action |
|---|---|---|---|
| `400` | `tenant.scope_required` | Multi-org caller sent no header; the body carries a `memberships` array | Render an organization picker from `memberships`, remember the slug, resend with the header |
| `403` | `tenant.not_a_member` | The header named an organization the caller isn't a member of | Drop the slug, re-prompt, retry |
| `403` | `tenant.rls_rejected` | The data layer rejected the operation in the current tenant context | Refresh the token and retry once |

See <https://docs.nizam.ai/errors/tenant/scope_required> for the picker payload
and the full recovery flow.
