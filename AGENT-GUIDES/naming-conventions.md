# Agent Guide: Naming Conventions

**ARD Source:** `naming-conventions-boundaries.md`
**Scope:** Backend, admin application, mobile clients, database schema

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Transformation layer sprawl.** When different layers use different naming conventions, every boundary crossing (API → client, application → database) requires a conversion function. Each converter is a bug surface. When a converter is missing or silently fails, data arrives with the wrong keys and the client breaks with a confusing runtime error.

2. **Legacy naming becoming the new default.** If a codebase has historical snake_case in application types (tech debt), a new engineer copies the pattern they see and extends it. Within months, new code has mixed conventions. The cost to normalize doubles with each month of drift.

3. **API contract confusion for client developers.** If some API endpoints return camelCase and others return snake_case (even for legitimate reasons), client developers must check which convention each endpoint uses. Every integration involves guessing, testing, and documenting exceptions. This overhead compounds across every client stack.

4. **ORM mapping errors at scale.** When an entity's TypeScript property names do not match a predictable transformation of their database column names, ORM-level auto-mapping fails silently or requires extensive per-field annotation. Missing annotations surface as null fields at runtime, not compile time.

---

## The Mental Model

There are exactly **three contexts** and each has exactly **one convention**:

| Context | Convention | Why |
|---------|-----------|-----|
| Application code (variables, properties, API payloads) | camelCase | Language standard for TypeScript/JavaScript/Dart |
| Types, classes, enums | PascalCase | Language standard |
| Database columns and schema | snake_case | SQL convention; ORM maps this predictably |

The mapping between application code and database is **localised to the entity/repository layer**. Above that layer, everything is camelCase. Below that layer (in SQL), everything is snake_case. The mapping is explicit and in one place.

---

## Invariants

**1. New application code uses camelCase for all properties, variables, and parameters.**

- **Violation:** A new DTO has fields like `user_id`, `created_at`, `first_name`.
- **Cost:** Mixed convention in the same codebase. Any client that consumes this endpoint receives snake_case keys unexpectedly. Conversion layers must be added. The inconsistency propagates to generated types.

**2. New API payloads (request and response) are camelCase on versioned routes.**

- **Violation:** A new v1 endpoint returns `{ user_id: '...', first_name: '...', created_at: '...' }`.
- **Cost:** Clients must handle two conventions. Integration tests for versioned routes pass but test the wrong contract. If clients are auto-generated from OpenAPI schemas, they receive snake_case models.

**3. Database columns use snake_case.**

- **Violation:** A migration adds a column named `createdAt` or `userId` to a table.
- **Cost:** ORM auto-mapping breaks. The column must be manually annotated. If the annotation is added inconsistently across entities, some fields are null at runtime.

**4. The application/API ↔ database mapping is explicit and localised.**

- **Violation:** A service method manually renames fields when reading from the repository to compensate for inconsistent column naming.
- **Cost:** The conversion responsibility drifts into services. Each service that touches the database layer must also handle naming. Business logic and naming logic are mixed.

**5. Legacy naming divergence (snake_case in application code, conversion interceptors for unversioned routes) is not extended.**

- **Violation:** A new type is added that follows the legacy snake_case pattern because it was copied from an existing legacy type.
- **Cost:** Tech debt grows. The target state (all application code in camelCase) becomes further away. Each new instance of legacy naming delays migration and increases the cost of eventual normalization.

**6. File naming follows the project language convention: kebab-case for TypeScript, snake_case for Dart.**

- **Violation:** A new TypeScript file is named `UserService.ts` or `user_service.ts`.
- **Cost:** Inconsistent file lookup. Import paths become unpredictable. IDE auto-imports suggest wrong paths.

---

## Decision Protocol

**IF you are creating a new DTO, type, or interface:**
→ THEN use camelCase for all property names.
→ CHECK: Does the type look like it was copied from a database entity? If it has snake_case properties, rename them.

**IF you are adding a new column to a database migration:**
→ THEN name the column in snake_case (e.g., `tenant_id`, `created_at`, `first_name`).
→ CHECK: Does the entity mapping for this column have the application-side property in camelCase? If not, add the explicit column name annotation.

**IF you are writing a new API endpoint:**
→ THEN return camelCase keys in the response body.
→ CHECK: Does the response DTO have any snake_case properties? If yes, rename them.

**IF you are copying an existing type or entity as a starting point:**
→ THEN inspect the source for legacy naming (snake_case in application types). Do not carry forward the violation.
→ CHECK: After copying, scan all property names. Any snake_case in the application layer must be converted before the new type is used.

**IF you see a transformation interceptor being applied to a new route:**
→ THEN question whether the interceptor is needed. Versioned routes should not need a conversion interceptor — they should return camelCase natively.
→ CHECK: Is this interceptor on a versioned route? If yes, it is a sign the response DTO has the wrong convention. Fix the DTO instead.

---

## Generation Rules

**Always include:**
- camelCase property names in all application types, DTOs, interfaces, and API responses
- PascalCase for class, type, enum, and interface names
- snake_case in database column name annotations in entity definitions
- An explicit `name: 'snake_case_column'` annotation on entity columns where the property name is camelCase
- kebab-case file names in TypeScript projects; snake_case in Dart projects

**Never generate:**
- snake_case property names in DTOs, response types, request types, or domain models
- camelCase database column names in migrations or entity definitions
- New transformation interceptors for versioned routes (the DTO should be correct natively)
- File names in PascalCase or with inconsistent casing for the language

**Verify before outputting:**
- Every DTO property name: is it camelCase? → Fix if snake_case.
- Every new migration column: is it snake_case? → Fix if camelCase.
- Every entity property: does it have an explicit column `name` annotation mapping to snake_case? → Add if missing.
- Every new file name: does it follow kebab-case (TypeScript) or snake_case (Dart)? → Rename if not.

---

## Self-Check Before Submitting

1. Do all new DTOs and response types use camelCase properties? → If not, rename before submitting.
2. Do all new database columns in migrations use snake_case? → If not, fix the migration.
3. Does every entity property that maps to a database column have an explicit `name` annotation? → Add if missing.
4. Are any new types copying the snake_case pattern from legacy code? → If yes, break the pattern and use camelCase.
5. Is a new transformation interceptor being added for a versioned route? → If yes, fix the DTO instead of adding an interceptor.
6. Do all new TypeScript files use kebab-case names? → If not, rename.
7. Is the mapping between camelCase application properties and snake_case columns localised to the entity definition? → If the mapping happens in services or controllers, move it.

---

## Common Violations and Corrections

**Violation 1:** New DTO with snake_case properties.
```typescript
// WRONG
export class CreateUserDto {
  first_name: string;
  last_name: string;
  phone_number: string;
}
```
**Correction:**
```typescript
// CORRECT
export class CreateUserDto {
  firstName: string;
  lastName: string;
  phoneNumber: string;
}
```

---

**Violation 2:** Database migration adds camelCase column.
```sql
-- WRONG
ALTER TABLE users ADD COLUMN firstName VARCHAR(255);
```
**Correction:**
```sql
-- CORRECT
ALTER TABLE users ADD COLUMN first_name VARCHAR(255);
```
And the entity property is annotated: `@Column({ name: 'first_name' }) firstName: string;`

---

**Violation 3:** Entity property lacks explicit column name annotation.
```typescript
// WRONG — ORM will try to find a column named 'firstName' (camelCase)
@Column()
firstName: string;
```
**Correction:**
```typescript
// CORRECT — ORM maps 'firstName' property to 'first_name' column
@Column({ name: 'first_name' })
firstName: string;
```

---

**Violation 4:** Response interceptor added to convert camelCase → snake_case on a versioned route.
**Correction:** The versioned route should return camelCase natively. Remove the interceptor from the versioned route and fix the response DTO to use camelCase properties.

---

**Violation 5:** New TypeScript file named in PascalCase.
```
// WRONG
UserManagementService.ts
UserManagementService.spec.ts
```
**Correction:**
```
// CORRECT
user-management.service.ts
user-management.service.spec.ts
```
