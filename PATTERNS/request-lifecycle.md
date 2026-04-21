# Pattern: Request Lifecycle

**Stack:** NestJS (TypeScript) backend
**ARD Sources:** `auth-boundaries.md`, `multi-tenancy-boundaries.md`, `api-boundaries.md`

---

## Overview

This pattern describes the complete lifecycle of an authenticated, tenant-scoped API request from HTTP ingress to response. Every business API request follows this path. Deviating from it creates auth or tenancy violations.

---

## Lifecycle Stages

```
1. HTTP Request arrives
2. JWT Guard verifies the token (auth provider SDK called here — nowhere else)
3. JWT Guard loads the user from the database using the translated userId
4. JWT Guard attaches the user entity to req.user
5. UserInfo Interceptor builds the request context (userId, tenantId, role)
6. Role Guard checks that the user's role is in the allowed set for this route
7. Controller extracts context from the request (passes to service — does not transform)
8. Access Policy produces tenant-scoped filters from the context
9. Service orchestrates the business operation using the filters
10. Repository applies filters to all queries (tenant isolation enforced here)
11. Response is returned with camelCase payload (versioned routes)
```

---

## Stage 2–4: JWT Guard (Auth Boundary)

```typescript
// [PATTERN] JWT guard — the ONLY place the auth provider SDK is called

@Injectable()
class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly providerWrapper: ExternalProviderWrapper, // SDK isolated here
    private readonly userRepository: UserRepository,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    
    // 1. Extract token from header
    const token = this.extractToken(request);
    if (!token) throw new UnauthorizedException();
    
    // 2. Verify with auth provider (ONLY here — never in business code)
    const providerClaims = await this.providerWrapper.verifyToken(token);
    
    // 3. Translate provider ID → internal userId (ONLY here)
    const userId = await this.userRepository.findIdByProviderUserId(
      providerClaims.providerUserId  // provider identifier used ONLY here
    );
    
    // 4. Load the full user from DB using internal userId
    const user = await this.userRepository.findById(userId);
    if (!user) throw new UnauthorizedException();
    
    // 5. Attach to request — from here on, business code only sees userId
    request.user = user;
    return true;
  }
}
```

**Constraint annotations:**
- `providerWrapper` = the ONLY file that imports the auth provider SDK
- `providerClaims.providerUserId` = used ONLY in this translation step
- After this guard, `req.user.id` is your internal `userId` — never a provider ID

---

## Stage 5: UserInfo Interceptor (Context Building)

```typescript
// [PATTERN] Interceptor builds request context from DB-loaded user
// Role derived from DB user entity, NOT from JWT claims

@Injectable()
class UserInfoInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler) {
    const req = context.switchToHttp().getRequest();
    
    // req.user is the full user entity loaded from DB by JwtAuthGuard
    const user = req.user;
    
    // Role derived from DB profile type — NOT from JWT
    // [CONSTRAINT] Role source is ALWAYS the database
    const role = this.deriveRoleFromProfile(user.profileType);
    
    // tenantId from the DB-loaded user entity
    // [CONSTRAINT] tenantId NEVER comes from the request body/params
    const tenantId = user.tenantId;
    
    req.requestContext = {
      userId: user.id,         // internal userId from your DB
      tenantId: tenantId,      // from DB user entity
      role: role,              // derived from DB profile, not JWT
    };
    
    return next.handle();
  }
  
  private deriveRoleFromProfile(profileType: ProfileType): Role {
    // [CONSTRAINT] Simple mapping — no provider claims, no ABAC
    const mapping: Record<ProfileType, Role> = {
      [ProfileType.RIDER]: Role.RIDER,
      [ProfileType.DRIVER]: Role.DRIVER,
      [ProfileType.ADMIN_OWNER]: Role.ADMIN,
      // ... other profile types
    };
    return mapping[profileType];
  }
}
```

---

## Stage 6: Role Guard (Authorization)

```typescript
// [PATTERN] Role guard reads from request context — NOT from JWT

@Injectable()
class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<Role[]>('roles', context.getHandler());
    if (!requiredRoles) return true; // no role requirement = open to all authenticated users
    
    const req = context.switchToHttp().getRequest();
    
    // [CONSTRAINT] Role read from requestContext (DB-derived), never from JWT
    const userRole = req.requestContext.role;
    
    if (!userRole) throw new ForbiddenException();
    
    return requiredRoles.includes(userRole);
  }
}
```

---

## Stage 7: Controller (Adapter Only)

```typescript
// [PATTERN] Controller is an adapter — extracts context, calls service, returns result
// [CONSTRAINT] NO business logic, NO tenant derivation, NO access policy calls

@Controller({ path: 'orders', version: '1' })
@UseGuards(RolesGuard)
@UseInterceptors(UserInfoInterceptor)
export class BookingController {
  constructor(private readonly bookingService: BookingService) {}

  @Get()
  @Roles(Role.RIDER, Role.ADMIN)
  async getBookings(
    @Req() req: IRequestWithContext,
    // [CONSTRAINT] NO @Query('tenantId'), NO @Body() with tenantId
  ) {
    // Controller only forwards context — no reasoning about tenancy
    return this.bookingService.getBookings(req.requestContext);
  }
}
```

---

## Stage 8–10: Service + Access Policy + Repository

```typescript
// [PATTERN] Service orchestrates: access policy → repository → response

@Injectable()
class BookingService {
  constructor(
    private readonly bookingRepository: BookingRepository,
    private readonly bookingAccessPolicy: BookingAccessPolicy,
  ) {}

  async getBookings(context: RequestContext): Promise<BookingDto[]> {
    // [CONSTRAINT] Access policy called before ANY repository operation
    const filter = this.bookingAccessPolicy.buildFilter(context);
    
    // [CONSTRAINT] Filter passed to repository — repository enforces it
    const orders = await this.bookingRepository.findByFilter(filter);
    
    return orders.map(toBookingDto);
  }
}

// [PATTERN] Access policy — single source of truth for tenant visibility

@Injectable()
class BookingAccessPolicy {
  buildFilter(context: RequestContext): BookingFilter {
    // [CONSTRAINT] tenantId comes from context — never from parameters
    const { tenantId, userId, role } = context;
    
    if (role === Role.ADMIN_PLATFORM) {
      // Platform admin sees all tenants — explicit case only
      return {};
    }
    
    // All other roles: always scoped to their tenant
    return { tenantId };
  }
}

// [PATTERN] Repository — enforces tenant filter on every query

@Injectable()
class BookingRepository {
  constructor(@InjectRepository(BookingEntity) private readonly repo: Repository<BookingEntity>) {}

  async findByFilter(filter: BookingFilter): Promise<BookingEntity[]> {
    const query = this.repo.createQueryBuilder('order');
    
    // [CONSTRAINT] Tenant filter always applied if present
    if (filter.tenantId) {
      query.andWhere('order.tenantId = :tenantId', { tenantId: filter.tenantId });
    }
    
    return query.getMany();
  }
  
  // [ANTI-PATTERN] DO NOT expose this:
  // async findAll(): Promise<BookingEntity[]> { return this.repo.find(); }
}
```

---

## Stage 11: Response (Naming Convention)

```typescript
// [PATTERN] Response DTOs use camelCase for versioned routes
// [CONSTRAINT] No snake_case in API responses; no provider identifiers

export class BookingResponseDto {
  id: string;               // camelCase ✓
  userId: string;           // internal userId ✓ (NOT providerUserId)
  tenantId: string;         // [DEBATE: whether to expose tenantId — see CLARIFICATION-NEEDED.md]
  status: BookingStatus;    // enum value
  createdAt: Date;          // camelCase ✓ (NOT created_at)
}
```

---

## Key Invariants Summary

| Stage | Constraint |
|-------|-----------|
| JWT Guard | Only place auth provider SDK is called |
| JWT Guard | Provider ID used only in userId translation |
| UserInfo Interceptor | Role derived from DB, not JWT |
| UserInfo Interceptor | tenantId from DB user entity, not request |
| Role Guard | Role from requestContext, not JWT |
| Controller | No tenant logic, no business logic |
| Access Policy | Single source of tenant visibility rules |
| Repository | Applies tenant filter on every query |
| Response | camelCase, no provider identifiers |
