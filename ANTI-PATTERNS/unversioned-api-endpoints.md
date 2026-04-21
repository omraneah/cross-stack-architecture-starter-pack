# Anti-Pattern: Unversioned API Endpoints

**ARD Source:** `api-boundaries.md`
**Detection heuristic:** Any `@Controller('path')` without a `version` field, or any test base URL that does not include `/v1` (or the current active version).

---

## What It Looks Like

```typescript
// VIOLATION 1: Controller with no versioning at all
@Controller('orders')
export class BookingController {
  @Get()
  getBookings() { ... }
  // Exposes: GET /orders (unversioned — no stable contract)
}

// VIOLATION 2: Controller with VERSION_NEUTRAL as the only version (permanent state)
@Controller({ path: 'payments', version: VERSION_NEUTRAL })
export class PaymentController {
  @Post()
  createPayment() { ... }
  // Exposes: POST /payments (unversioned — VERSION_NEUTRAL means no version prefix)
}

// VIOLATION 3: Test using unversioned base URL
const response = await request(app)
  .get('/users')                  // ← unversioned
  .set('Authorization', `Bearer ${token}`);
// Tests the unversioned contract, which may differ from the versioned contract

// VIOLATION 4: Client SDK configured with unversioned base URL
const apiClient = new ApiClient({
  baseURL: 'https://api.example.com',  // ← no version segment
});
// All calls go to unversioned routes — will break when unversioned routes are removed

// VIOLATION 5: Breaking change applied in-place to existing version
// v1 BookingResponseDto — BEFORE
export class BookingResponseDto {
  id: string;
  amount: number;       // existing field
}
// v1 BookingResponseDto — AFTER (breaking change in place)
export class BookingResponseDto {
  id: string;
  totalAmount: number;  // ← renamed from 'amount' — breaks all v1 clients
}
```

---

## Why an Agent Gravitates Toward It

1. **Controller creation follows the simplest decorator pattern.** `@Controller('orders')` is the simplest form. Adding `version` requires knowing the current version and the versioning pattern.

2. **Version is an afterthought.** An agent generates the controller, routes, and DTOs — then is told to "add versioning." The version is appended but the dual-support state (`VERSION_NEUTRAL`) is never cleaned up.

3. **In-place modification is simpler than creating a new version.** Renaming a field in an existing DTO is 1 line. Creating a v2 endpoint, a v2 DTO, updating documentation, and adding a deprecation plan is 50+ lines.

4. **Legacy code may show unversioned patterns.** The codebase has unversioned routes for historical reasons. An agent generating new code may follow the existing pattern.

---

## What It Breaks

**No migration path for breaking changes.** Without versioning, changing a field name or removing a field breaks all clients simultaneously on deployment. There is no grace period. Mobile apps cannot be rolled back from the app store.

**No testable contract.** When tests use unversioned URLs, the versioned API has no automated test coverage. A breaking change to the versioned API may not be caught by the test suite.

**Forced coordinated releases.** Every client and the backend must deploy together when any contract change occurs. This eliminates independent deployment of services and clients.

**Audit and monitoring complexity.** Without a versioned contract, monitoring cannot distinguish between "traffic to the stable API" and "traffic to a legacy route." Removing deprecated routes is guesswork.

---

## Compliant Alternative

```typescript
// CORRECT: Controller with explicit versioning from the start

@Controller({ path: 'orders', version: '1' })
export class BookingV1Controller {
  @Get()
  getBookings() { ... }
  // Exposes: GET /api/v1/orders — versioned, stable contract ✓
}

// CORRECT: Breaking change handled with a new version
// v1 DTO — unchanged (existing clients continue to work)
export class BookingV1ResponseDto {
  id: string;
  amount: number;  // ← unchanged
}

// v2 DTO — new version for the renamed field
export class BookingV2ResponseDto {
  id: string;
  totalAmount: number;  // ← renamed field in new version only
}

// v2 controller — new version alongside v1
@Controller({ path: 'orders', version: '2' })
export class BookingV2Controller {
  @Get()
  getBookings() { ... }
  // Exposes: GET /api/v2/orders — new contract ✓
}

// CORRECT: Test uses versioned URL
const response = await request(app)
  .get('/api/v1/orders')       // ← versioned ✓
  .set('Authorization', `Bearer ${token}`);

// CORRECT: Client SDK with versioned base URL
const apiClient = new ApiClient({
  baseURL: 'https://api.example.com/api/v1',  // ← version segment ✓
});

// CORRECT: Dual support during migration (temporary, with a removal plan)
@Controller({ path: 'sessions', version: ['1', VERSION_NEUTRAL] })
// ^ VERSION_NEUTRAL support has a defined removal date tracked in ISSUE-456
// Remove VERSION_NEUTRAL when all clients confirmed on v1 (target: 2026-06-01)
```

---

## Self-Review Heuristic

1. Every `@Controller(...)` declaration: does it have a `version` field? → If not, add it.
2. Every `VERSION_NEUTRAL` in a controller: is it temporary with a tracked removal date? → If no date exists, flag it.
3. Every test base URL: does it include the version segment? → If not, fix the test helper.
4. Every client base URL config: does it include the version segment? → If not, fix it.
5. Every DTO modification: is any existing field removed or renamed? → If yes, new version required.
