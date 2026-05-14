# API Versioning Boundaries

**Every public API endpoint has a stable, versioned contract. Breaking changes require a new version.**

**Applies when:** public or external-consumer API (mobile clients, third-party integrators, SDK consumers, partner backends). Skip for internal-only RPC between services deployed together.

## Why it matters

- Without versioning, every contract change forces simultaneous deploy of every client. App store delays make this fatal for mobile.
- Silent breaking changes (renamed fields, changed enums) surface as runtime errors days later, far from the change.
- Mixed unversioned and versioned routes make deprecation guesswork.

## The judgment

**Where the version lives:**

- URI path (`/api/v1/...`). Visible in logs, monitoring, and client code. Easiest to deprecate. Default for public-facing APIs and APIs consumed by independent clients.
- `/v1/` in the URL works until version 3, when half the customers are on v1, a third on v2, and the routing layer has accreted three years of conditional logic that nobody dares delete. Pick the versioning strategy — and the deprecation discipline — before the first external consumer, not after.
- Header (`Accept-Version` or media type). Wins when the API is consumed by a tightly-coupled client SDK that handles versioning transparently, when content-negotiation is central to the design (HATEOAS, hypermedia-driven), or when the surface must keep clean URIs for SEO or aesthetics. Costlier to monitor and grep, so the trade-off has to justify itself.
- Query parameter. Discouraged — easy to forget, easy to cache wrong.

**Breaking-change protocol:**

- Always create a new version; keep the old version alive until traffic confirms zero usage.
- In-place edits to an existing version are forbidden, even for "obviously safe" renames.

**Naming convention at the boundary:**

- One convention (`camelCase` or `snake_case`) used consistently. No transformation layer at the API boundary.
- Transformation interceptors are technical debt — they fork the API surface and accumulate exceptions.

## Signals of violation in an audited codebase

- A controller without an explicit version declaration.
- A breaking change applied in place to an existing version's DTO.
- A test or client SDK base URL that omits the version segment.
- An unversioned route in production with no documented deprecation date or zero-traffic confirmation.
- Mixed naming conventions on the same response payload.

## Minimum viable shape

```
Controller declares version v1 explicitly
  → New endpoint or additive field? → stays in v1
  → Removal, rename, type change? → new v2 controller; v1 untouched
  → Tests, clients, docs all reference the versioned base URL
  → Deprecation = announce → monitor traffic → remove only at zero usage
```

**Severity floor if violated:** P1 — unversioned breaking changes compound into routing-layer accretion that nobody dares delete. P0 if the API is consumed by mobile or third-party clients (app-store-delay = deploy-and-pray surface). May step down by one tier for internal-only APIs with a single tightly-coupled client.
