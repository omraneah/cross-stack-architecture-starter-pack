# Worked Example: Auth Boundary Applied

One boundary applied to one stack to make the abstract concrete. The stack here is TypeScript on a typical web backend, but the shape ports identically to Python, Go, Ruby. The framework-specific syntax (decorators, dependency injection style, ORM choice) is incidental; the structure is the point.

**Principle:** `boundaries/identity-access/auth-boundaries.md`.
**Doctrine:** the author's specific choice on each trade-off lives in `DOCTRINE.md` under *Auth & Identity*.

## The stack illustrated

- Backend: TypeScript with a typed query layer (no specific ORM)
- Auth provider: opaque (Auth0, Okta, Clerk, Firebase, custom — the boundary doesn't care)
- Database: PostgreSQL
- Transport: HTTP. Queue workers and CLI tools follow the same shape — the boundary is the function, not the transport.

## The wrong shape (what most early codebases look like)

```typescript
// in services/order.service.ts
import { ProviderSdk } from 'auth-provider-sdk';      // VIOLATION: SDK in business module

export class OrderService {
  async createOrder(token: string, items: OrderItem[]) {
    const claims = await ProviderSdk.verify(token);    // VIOLATION: verify in business code
    const userId = claims.sub;                          // VIOLATION: provider ID used as internal ID
    const role = claims.groups[0];                      // VIOLATION: role from claims

    if (role !== 'CUSTOMER') throw new Forbidden();

    return this.orderRepo.create({
      userId,                                           // VIOLATION: provider ID stored on business table
      items,
    });
  }
}
```

Four violations in eight lines. Each is the seed of a different failure the boundary names.

## The right shape

**1. The provider wrapper — the only file that imports the SDK**

```typescript
// in common/auth/provider-wrapper.ts
import { ProviderSdk } from 'auth-provider-sdk'; // ONLY HERE

export interface ProviderClaims {
  providerUserId: string;
  email?: string;
  expiresAt: number;
}

export class ProviderWrapper {
  async verify(token: string): Promise<ProviderClaims> {
    const decoded = await ProviderSdk.verify(token);
    return {
      providerUserId: decoded.sub,
      email: decoded.email,
      expiresAt: decoded.exp,
    };
  }
}
```

**2. The auth guard — translates provider ID → internal userId, attaches context**

```typescript
// in middleware/auth-guard.ts
export async function authGuard(request: Request): Promise<RequestContext> {
  const token = extractBearer(request);
  const claims = await providerWrapper.verify(token);

  // Translation: provider ID → internal userId
  const userId = await authMappingRepo.findUserIdByProviderId(claims.providerUserId);
  if (!userId) throw new Unauthorized();

  // Single load: user record is the source of truth for tenant + role
  const user = await userRepo.findById(userId);
  if (!user) throw new Unauthorized();

  // Role derived from DB profile, NOT from claims
  const role = deriveRoleFromProfile(user.profileType);

  return { userId: user.id, tenantId: user.tenantId, role };
}
```

**3. The provider-mapping table — the only place provider IDs are stored**

```sql
CREATE TABLE auth_provider_mappings (
  id            UUID PRIMARY KEY,
  provider_id   TEXT NOT NULL UNIQUE,    -- provider's subject claim
  user_id       UUID NOT NULL REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**4. The business service — sees only the request context**

```typescript
// in services/order.service.ts (NO provider SDK import anywhere in this file)
export class OrderService {
  async createOrder(context: RequestContext, items: OrderItem[]) {
    if (context.role !== Role.CUSTOMER) throw new Forbidden();

    return this.orderRepo.create({
      userId: context.userId,    // internal UUID, not provider ID
      tenantId: context.tenantId,
      items,
    });
  }
}
```

## What this buys you

- **Provider migration is a one-file change.** Swap `ProviderWrapper`'s implementation; nothing else moves.
- **Roles change without redeploys.** Update the `profileType → role` mapping in one place; the next request sees the new role.
- **Audits are answerable.** "Who has which role" is one SQL query.
- **Provider outages don't cascade.** Only the auth boundary fails; business modules don't know the provider exists.

## What this costs you

- One extra table (`auth_provider_mappings`).
- One extra DB lookup per authenticated request (mitigated by a short-TTL cache if the lookup becomes hot).
- More files than the "wrong shape."

The cost is paid every day. The benefit is paid the day the provider raises prices, has an outage, or gets acquired. The author has yet to see a system where that day didn't arrive.

## Porting to other stacks

The shape doesn't change. The implementation surface does.

- **FastAPI / Python:** `ProviderWrapper` class + a dependency-injected auth dependency + a Pydantic `RequestContext` model + SQLAlchemy mapping table.
- **Rails / Ruby:** `ProviderWrapper` service + `before_action` + `ApplicationController` context + ActiveRecord mapping model.
- **Go:** `ProviderWrapper` struct + middleware function + `context.Context` with typed values + `sqlx` mapping repository.
- **Express / Fastify (without DI):** `ProviderWrapper` module + middleware function + `req.context` typed extension.

The framework changes. The four boxes — wrapper, guard, mapping table, business service — do not.
