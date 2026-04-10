# Anti-Pattern: Provider ID Leakage

**ARD Source:** `auth-boundaries.md`
**Detection heuristic:** Search for any provider-specific identifier (e.g., `sub`, `cognito_sub`, `firebase_uid`, `auth0_id`, `providerUserId`) outside of `auth/`, `user/`, and `external-services/` directories.

---

## What It Looks Like

```typescript
// VIOLATION 1: Provider identifier in a business entity
@Entity({ name: 'booking' })
export class BookingEntity {
  @Column({ name: 'cognito_sub' })
  cognitoSub: string;  // ← provider identifier in a business table
}

// VIOLATION 2: Provider SDK imported in a business service
import { CognitoIdentityServiceProvider } from 'aws-sdk';  // ← in booking.service.ts
const user = await cognito.getUser({ AccessToken: token }).promise();

// VIOLATION 3: Provider identifier in an API response
return {
  userId: user.id,
  providerUserId: user.cognitoSub,  // ← exposed to client
  bookingId: booking.id,
};

// VIOLATION 4: Role read from provider claims
const role = decodedToken['cognito:groups'][0];  // ← role from provider, not DB

// VIOLATION 5: Provider identifier in a query filter
const user = await this.userRepo.findOne({ where: { cognitoSub: providerUserId } });
// (This one is acceptable ONLY in the user/auth module's provider-mapping table — never elsewhere)
```

---

## Why an Agent Gravitates Toward It

1. **It's the path of least resistance.** The JWT token is available on the request object. Using `token.sub` directly is shorter than looking up the internal userId.

2. **It "works" locally.** Provider IDs are valid identifiers. Queries run. Tests pass. The failure is architectural, not functional — it surfaces only when the provider changes.

3. **Existing code may show partial leakage.** If legacy code already uses a provider ID somewhere, an agent may continue the pattern as "consistent with existing code."

4. **Framework auto-injection makes provider data available everywhere.** JWT strategies in some frameworks populate the request user from the token claims directly, making it easy to use those claims in any service.

---

## What It Breaks

**At provider migration time:** Every business table and service that references the provider ID must be audited and rewritten. A migration affects every row in every affected table. At scale (100K+ users, 10+ business tables), this is a multi-team, multi-month effort.

**At provider outage:** If the provider's verification endpoint is down and business modules make provider calls, those modules fail independently. With the boundary enforced, only the auth zone fails.

**At security audit:** Provider-specific identifiers in business tables are a data governance issue. They reveal which provider is used, potentially expose account enumeration vectors, and create compliance questions about PII handling.

**In testing:** Tests that use real provider IDs are brittle. Mocking the provider SDK in business service tests couples test setup to the provider's interface.

---

## Compliant Alternative

```typescript
// CORRECT STRUCTURE

// 1. Provider ID lives ONLY in the auth mapping table (user module)
@Entity({ name: 'user_auth_mappings' })
export class UserAuthMappingEntity {
  @Column({ name: 'provider_user_id', unique: true })
  providerUserId: string;  // ← ONLY HERE in the entire codebase
  
  @Column({ name: 'user_id' })
  userId: string;          // ← internal ID
}

// 2. Business entities use ONLY internal userId
@Entity({ name: 'booking' })
export class BookingEntity {
  @Column({ name: 'user_id' })
  userId: string;  // ← internal ID, not provider ID ✓
}

// 3. Auth provider SDK ONLY in the wrapper
// auth-provider.wrapper.ts — the ONLY file that imports provider SDK

// 4. Role from database, not from JWT claims
const role = this.roleDerivationService.deriveRole(dbUser.profileType);  // ← from DB ✓

// 5. API responses contain only internal identifiers
return { userId: user.id, bookingId: booking.id };  // ← no providerUserId ✓
```

---

## Self-Review Heuristic

Run a search across all files outside `auth/`, `user/`, `external-services/`:
- Any import of the auth provider SDK
- Any field name matching provider identifier patterns (`_sub`, `provider_id`, `firebase_uid`, etc.)
- Any column in a migration outside the auth mapping table that stores a provider-derived identifier

**Zero matches = compliant. Any match = violation.**
