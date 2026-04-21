# Agent Guide: Engineering Practices

**ARD Source:** `engineering-practices-boundaries.md`
**Scope:** All stacks — all contributors (human and AI)

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Building features on top of boundary violations.** New code added to a module that already violates a core boundary (e.g., circular dependency, unscoped tenant query) inherits the violation. The debt grows. When the underlying issue is eventually fixed, all the new code must be re-examined to ensure it doesn't depend on the broken state.

2. **Root symptom fixed, underlying cause remains.** A null pointer exception is fixed by adding a null check. The root cause — a service that can return null when it should always return a value — is not addressed. The null check propagates. Future callers add their own null checks. The system becomes defensive against its own internal contracts.

3. **Business logic duplicated across client and server.** A validation rule that exists in the mobile app is not enforced in the backend. A user who bypasses the app (e.g., via a direct API call) can violate the rule. When the rule changes, two places must be updated. They inevitably drift.

4. **Large PRs that no reviewer can meaningfully review.** A 2,000-line PR containing a new feature, a refactor, and a dependency upgrade is reviewed by checking that it "looks fine." Architecture violations, logic errors, and security issues hide in the volume. The review is theater.

5. **Speculative complexity added for hypothetical requirements.** A service is designed with a plugin architecture because "we might need to swap implementations in the future." The plugin architecture is never used. It adds 300 lines of code and two abstraction layers to every future change. The original requirement was simpler than the code.

---

## The Mental Model

These seven practices are not stylistic preferences — they are **structural contracts** that keep the codebase navigable, testable, and safe to change at any scale.

1. **Architecture first:** Check boundaries before implementing.
2. **Separation of concerns:** Backend owns business logic; clients own presentation.
3. **Root cause:** Fix the cause, not the symptom.
4. **Planning and reviewability:** Small, focused, reviewable units of change.
5. **Code quality:** Clear names, small units, no duplication, KISS/YAGNI.
6. **Documentation and traceability:** Non-obvious decisions documented; commits explain why.
7. **Testing:** Behavior that matters is tested; tests are first-class.

---

## Invariants

**1. Check architectural boundaries before implementing — do not build on degradation.**

- **Violation:** A new feature is added to a module that has an existing circular dependency, without first resolving or escalating the dependency issue.
- **Cost:** The new feature inherits the circular dependency. When the dependency is eventually resolved, the new feature must be re-examined. The refactor scope grows with every deferred fix.

**2. Business logic lives in the backend. Clients handle presentation only.**

- **Violation:** A mobile app applies a discount calculation rule that the backend does not validate.
- **Cost:** Any request that bypasses the client (direct API call, compromised device, API client tool) bypasses the business rule. The backend is not the source of truth. Rules must be updated in two places.

**3. Identify root cause before fixing. No workarounds without approval.**

- **Violation:** An error is suppressed with `try { ... } catch { /* ignore */ }` because investigating the cause would take too long.
- **Cost:** The error continues silently. It surfaces later as a data inconsistency or a production incident. The original investigation cost is dwarfed by the incident resolution cost.

**4. Non-trivial changes share a plan early. Changes are small and reviewable.**

- **Violation:** A significant architectural change is submitted as a 2,000-line PR with no prior discussion.
- **Cost:** The reviewer cannot meaningfully review the PR. Structural issues are missed. The feedback cycle is long. The PR either blocks the team or is merged with inadequate review.

**5. Simplest solution that meets the requirement — no speculative complexity (YAGNI).**

- **Violation:** A repository is abstracted behind two interfaces and a factory because "we might need to swap database implementations."
- **Cost:** Every new developer must understand three layers of indirection to make a simple change. The abstraction is never used for its stated purpose. The codebase complexity grows permanently.

**6. No duplication — single source of truth for logic, constants, and configuration.**

- **Violation:** A status enum is defined in both the backend module and the mobile app's type system. They drift when one is updated.
- **Cost:** Inconsistencies between the two copies cause hard-to-debug runtime errors. Finding and updating all copies of duplicated logic is the most common source of regressions.

**7. Test behavior that matters. Leave tests better than you found them.**

- **Violation:** A new service method has no tests because "it's simple."
- **Cost:** A future refactor changes the method's behavior. No test catches it. The regression surfaces in production.

---

## Decision Protocol

**IF you are about to implement a new feature or change:**
→ THEN first read the relevant ARD files and compare the approach against every boundary.
→ CHECK: Does the target area have existing boundary violations? If yes, do not add new code on top — escalate to CTO with options and a recommendation.

**IF you encounter a bug:**
→ THEN identify the root cause before writing a fix. Document the root cause in the commit message.
→ CHECK: Is the fix addressing the cause or the symptom? If it is adding a null check, guard, or catch clause around someone else's code, it is probably a symptom fix.

**IF you are designing a new abstraction:**
→ THEN ask: is this needed for a current, concrete requirement? If the answer is "it might be useful later," do not add it.
→ CHECK: How many concrete cases currently require this abstraction? If it is one or zero, the abstraction is premature.

**IF you are writing a change larger than ~300 lines:**
→ THEN split it. A change that refactors a module's structure is separate from the feature that uses the new structure. Reviewers need to be able to understand each unit independently.
→ CHECK: Can a reviewer understand and evaluate the correctness of this PR in under 30 minutes? If not, split it.

**IF you are adding validation logic to a client application:**
→ THEN also enforce it in the backend service. Client validation is UX; backend validation is the contract.
→ CHECK: If someone calls the API directly without the client, does the backend still enforce the rule? If not, add the backend enforcement.

---

## Generation Rules

**Always include:**
- Boundary checks before generating implementation code (read relevant ARDs)
- Tests for behavior that matters (not trivial getters/setters)
- Single source of truth: no duplication of constants, enums, or validation logic across layers
- Meaningful names: functions and variables that reveal intent
- Explicit types: no `any`, no untyped parameters in typed languages
- Backend enforcement for all business rules (client enforcement is additive, never primary)

**Never generate:**
- Workarounds, empty catch blocks, or error suppressions without explanation
- Speculative abstractions (interfaces, factories, plugins) for requirements that do not currently exist
- Business logic in client applications that is not also enforced in the backend
- PRs combining unrelated concerns (feature + refactor + dependency update)
- Duplicated constants, enums, or validation logic across modules or stacks
- Functions or classes that mix multiple responsibilities (single responsibility principle)

**Verify before outputting:**
- Is the code being added to an area with existing boundary violations? → Flag and escalate.
- Is every business rule enforced in the backend? → Add backend enforcement if client-only.
- Is there duplicated logic between this code and existing code? → Consolidate.
- Are there tests for the behavior this code implements? → Add if not.
- Is the abstraction level appropriate for the current requirements? → Remove speculative layers.

---

## Self-Check Before Submitting

1. Have the relevant ARD files been checked before implementation? → If not, check them now.
2. Is any business logic implemented only in a client application (not enforced in the backend)? → If yes, add backend enforcement.
3. Is there duplicated logic, constants, or validation across modules or stacks? → If yes, consolidate.
4. Are there tests for the behavior this code introduces? → If not, add them.
5. Is the change focused on a single concern, reviewable in isolation? → If it mixes concerns, split it.
6. Is any workaround, empty catch, or suppression without documented rationale? → If yes, document the rationale or fix the root cause.
7. Does the code introduce speculative abstractions for hypothetical future requirements? → If yes, remove them.

---

## Common Violations and Corrections

**Violation 1:** New feature added on top of existing circular dependency.
**Correction:** Before adding the feature, identify the circular dependency. Either break it (convert to events) or escalate to CTO with a documented plan. Do not add new code on top of the structural issue.

---

**Violation 2:** Validation logic in client only.
```typescript
// WRONG — mobile app validates email format, backend does not
if (!isValidEmail(email)) { showError('Invalid email'); return; }
// API call made — backend accepts any email
```
**Correction:** Add the same validation in the backend service. The client validation improves UX. The backend validation enforces the contract. Both must exist.

---

**Violation 3:** Speculative interface for a single-implementation service.
```typescript
// WRONG — factory pattern for one current implementation
interface INotificationSender { send(msg: Message): Promise<void>; }
class EmailNotificationSender implements INotificationSender { ... }
class NotificationSenderFactory {
  create(type: string): INotificationSender { ... }
}
```
**Correction:** Inject `EmailNotificationSender` directly. When a second implementation is needed, introduce the interface at that time with a concrete reason.

---

**Violation 4:** Empty catch block suppressing errors.
```typescript
// WRONG
try {
  await this.externalService.notify(userId);
} catch {
  // ignore — notification is best-effort
}
```
**Correction:** Either handle the error explicitly (log it, emit a monitoring event, use a dead-letter queue) or document the rationale clearly. Silent suppression hides failures that should be investigated.

---

**Violation 5:** Duplicated enum across backend and mobile app.
```typescript
// WRONG — BookingStatus defined in backend AND mobile app types
// Backend: enum BookingStatus { PENDING, CONFIRMED, CANCELLED }
// Mobile:  enum BookingStatus { pending, confirmed, cancelled }
```
**Correction:** The backend is the source of truth. Mobile app types are derived from the API contract (e.g., generated from OpenAPI schema). One definition, one place.
