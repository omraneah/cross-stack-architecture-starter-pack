# Anti-Pattern: Controller Tenant Logic

**ARD Source:** `multi-tenancy-boundaries.md`
**Detection heuristic:** Any controller method that reads `tenantId` from `@Query()`, `@Body()`, `@Param()`, or contains any conditional logic based on tenant context.

---

## What It Looks Like

```typescript
// VIOLATION 1: Controller accepts tenantId from query params
@Get()
async getResources(
  @Query('tenantId') tenantId: string,  // ← VIOLATION
  @Req() req: IRequestWithContext,
) {
  return this.resourceService.getResources(tenantId);
}

// VIOLATION 2: Controller accepts tenantId from request body
@Post()
async createResource(
  @Body() dto: CreateResourceDto,  // where CreateResourceDto has tenantId field
  @Req() req: IRequestWithContext,
) {
  return this.resourceService.createResource(dto.tenantId, dto);  // ← VIOLATION
}

// VIOLATION 3: Controller derives tenant scope itself
@Get()
async getResources(@Req() req: IRequestWithContext) {
  const tenantId = req.requestContext.tenantId;
  
  // Controller applying tenant logic — VIOLATION
  if (req.requestContext.role === Role.ADMIN) {
    return this.resourceService.getAllResources();        // ← unscoped for admin
  } else {
    return this.resourceService.getResources(tenantId);  // ← scoped for others
  }
}

// VIOLATION 4: Controller validates tenant membership
@Get(':id')
async getResource(@Param('id') id: string, @Req() req: IRequestWithContext) {
  const resource = await this.resourceService.findById(id);
  
  // Tenant check in the controller — VIOLATION
  if (resource.tenantId !== req.requestContext.tenantId) {
    throw new ForbiddenException();
  }
  
  return resource;
}
```

---

## Why an Agent Gravitates Toward It

1. **Feels like "security-conscious" code.** Adding a check before returning data looks like security. The agent doesn't realize the check belongs in the access policy layer, not the controller.

2. **The tenantId is available on the request.** Once `requestContext.tenantId` is present, it's easy to use it anywhere. The discipline is knowing *where* that use belongs.

3. **Simplest-looking path for admin/non-admin branching.** A role-based if/else in a controller seems straightforward. The fact that it violates the responsibility boundary is non-obvious.

4. **Copying from unversioned legacy code.** The codebase may contain older controllers that accept tenant parameters. An agent copying patterns from legacy code extends the violation.

---

## What It Breaks

**Client can spoof tenant context.** If `tenantId` comes from a request parameter, any authenticated user can set it to any value. There is no validation that the client-supplied `tenantId` matches their actual tenant — or is even a valid tenant.

**Tenant logic becomes unauditable.** When tenant scoping lives in 30 different controllers, auditing what data each role can access requires reading every controller. With access policies, it's in one place.

**Logic drifts between controllers.** Two controllers handling similar resources implement different tenant-scoping rules because each was written independently. Users in the same role get different access depending on which endpoint they call.

**Tests cannot verify the scoping contract centrally.** Each controller needs its own tenant isolation test. Tests are written (or not written) independently. Coverage gaps are invisible.

---

## Compliant Alternative

```typescript
// CORRECT: Controller is a pure adapter

@Controller({ path: 'resources', version: '1' })
@UseGuards(RolesGuard)
export class ResourceController {
  constructor(private readonly resourceService: ResourceService) {}

  @Get()
  @Roles(Role.ADMIN, Role.MANAGER, Role.USER)
  async getResources(
    @Req() req: IRequestWithContext,
    // ← NO @Query('tenantId'), NO tenant parameter at all
  ) {
    // Controller passes the whole context — service and access policy do the work
    return this.resourceService.getResources(req.requestContext);
  }

  @Get(':id')
  @Roles(Role.ADMIN, Role.MANAGER, Role.USER)
  async getResource(
    @Param('id') id: string,
    @Req() req: IRequestWithContext,
  ) {
    // Repository-level scoping ensures this can only return data in the user's tenant
    return this.resourceService.findById(id, req.requestContext);
  }
}

// Access policy (single source of truth for tenant logic):
@Injectable()
export class ResourceAccessPolicy {
  buildFilter(context: RequestContext): ResourceFilter {
    if (context.role === Role.PLATFORM_ADMIN) return {};           // explicit exception
    return { tenantId: context.tenantId };                         // all other roles
  }
}

// Service delegates to access policy:
async getResources(context: RequestContext) {
  const filter = this.accessPolicy.buildFilter(context);           // scoping here
  return this.repository.findMany(filter);
}
```

---

## Self-Review Heuristic

In any controller file, search for:
- `@Query('tenantId')` or `@Query('tenant_id')`
- `@Body()` with a DTO that has a `tenantId` or `tenant_id` field
- `@Param('tenantId')`
- Any `if` or `switch` statement that branches on tenant context
- Any direct call to `this.repository.*` (skipping the service layer)

**Any match = violation. Move the logic to the access policy.**
