# Pattern: Auth Boundary Translation

**Stack:** NestJS (TypeScript) with external authentication provider (e.g., Cognito, Auth0, Firebase Auth)
**ARD Source:** `auth-boundaries.md`

---

## Overview

This pattern implements the translation boundary between the external authentication provider and the internal identity model. It is the canonical structure for isolating the provider from all business code.

---

## Layer Structure

```
External Provider                      Your System
─────────────────                      ─────────────
Token (JWT with provider claims)  →    Provider SDK Wrapper (isolated)
                                  →    userId Translation (in user module)
                                  →    RequestContext (userId, tenantId, role)
                                  →    Business Logic (only sees userId)
```

---

## Layer 1: External Provider SDK Wrapper

```typescript
// [PATTERN] Provider SDK Wrapper — the ONLY file that imports the provider SDK
// [CONSTRAINT] All provider SDK imports are confined to this file and this layer
// [LOCATION] common/external-services/auth-provider/ (NOT in any business module)

// auth-provider.wrapper.ts
import { ProviderSdk } from 'provider-sdk';  // [ONLY HERE]

export interface ProviderTokenClaims {
  providerUserId: string;  // internal name — maps provider's user ID concept
  email?: string;
  tokenIssuedAt: number;
  tokenExpiresAt: number;
}

@Injectable()
export class AuthProviderWrapper {
  private readonly client: ProviderSdk;
  
  constructor(private readonly config: AuthConfig) {
    // SDK initialized once in the wrapper — no other class instantiates it
    this.client = new ProviderSdk(config.providerConfig);
  }

  // [ONLY PUBLIC METHOD NEEDED BY BUSINESS CODE]
  async verifyToken(token: string): Promise<ProviderTokenClaims> {
    try {
      // Provider-specific verification logic — contained here
      const decoded = await this.client.verifyJwt(token);
      
      // [TRANSLATION] Map provider-specific field names to our stable interface
      // When the provider changes, only this mapping changes
      return {
        providerUserId: decoded.sub,         // provider's subject claim
        email: decoded.email,
        tokenIssuedAt: decoded.iat,
        tokenExpiresAt: decoded.exp,
      };
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
  
  // Additional provider operations (create user, delete user) also live here
  async createProviderUser(email: string, temporaryPassword: string): Promise<string> {
    const result = await this.client.createUser({ email, temporaryPassword });
    return result.providerUserId;  // returned only for immediate translation
  }
  
  async deleteProviderUser(providerUserId: string): Promise<void> {
    await this.client.deleteUser(providerUserId);
  }
}
```

---

## Layer 2: Provider ID ↔ userId Translation (User Module)

```typescript
// [PATTERN] Translation table — maps provider IDs to internal userIds
// [CONSTRAINT] This table is the ONLY place where provider IDs are stored
// [LOCATION] user module's database — NOT in business tables

// user-auth-mapping.entity.ts (in user module)
@Entity({ name: 'user_auth_mappings' })
export class UserAuthMappingEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  // [CONSTRAINT] provider_user_id stored ONLY in this table
  @Column({ name: 'provider_user_id', unique: true })
  providerUserId!: string;

  // [KEY CONSTRAINT] This is the translation: provider ID → internal userId
  @Column({ name: 'user_id', nullable: false })
  userId!: string;  // your internal UUID

  @Column({ name: 'created_at', type: 'timestamptz', default: () => 'CURRENT_TIMESTAMP' })
  createdAt!: Date;
}

// [PATTERN] User repository method: translate provider ID → internal userId
// [LOCATION] user module repository — NOT a general-purpose service

async findUserIdByProviderUserId(providerUserId: string): Promise<string | null> {
  const mapping = await this.authMappingRepo.findOne({
    where: { providerUserId },
  });
  return mapping?.userId ?? null;
}
```

---

## Layer 3: JWT Guard (Uses Wrapper + Translation)

```typescript
// [PATTERN] JWT Guard — uses wrapper and translation, produces internal userId
// [CONSTRAINT] No business module imports this guard directly — it's in common/guards/

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly authProviderWrapper: AuthProviderWrapper,  // from wrapper layer
    private readonly userRepository: UserRepository,             // user module only
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    
    const token = this.extractBearerToken(request);
    if (!token) throw new UnauthorizedException('No token provided');
    
    // [STEP 1] Verify with provider wrapper — provider SDK NOT called directly
    const claims = await this.authProviderWrapper.verifyToken(token);
    
    // [STEP 2] Translate: provider ID → internal userId
    // [CONSTRAINT] providerUserId used ONLY here — never passed further
    const userId = await this.userRepository.findUserIdByProviderUserId(
      claims.providerUserId,
    );
    if (!userId) throw new UnauthorizedException('User not found');
    
    // [STEP 3] Load full user from DB using internal userId
    const user = await this.userRepository.findById(userId);
    if (!user || !user.isActive) throw new UnauthorizedException('User inactive');
    
    // [STEP 4] Attach to request — downstream only sees user.id (internal)
    request.user = user;
    // [CONSTRAINT] claims.providerUserId is dropped here — never attached to request
    
    return true;
  }
  
  private extractBearerToken(request: Request): string | null {
    const authHeader = request.headers['authorization'];
    if (!authHeader?.startsWith('Bearer ')) return null;
    return authHeader.slice(7);
  }
}
```

---

## Layer 4: Role Derivation (Database-Based)

```typescript
// [PATTERN] Role derivation from database profile type — NOT from provider claims
// [CONSTRAINT] Role is NEVER read from JWT claims or provider-side groups

// role-derivation.service.ts (in auth or user module)
@Injectable()
export class RoleDerivationService {
  // [PATTERN] Simple mapping: profile type → role
  // [CONSTRAINT] No external calls, no provider involvement
  deriveRole(user: UserEntity): Role {
    // Role comes from the DB-loaded user entity's profile type
    // [ANTI-PATTERN PREVENTED] NOT: decodeToken(token).role or provider.getUserGroups(userId)
    
    switch (user.profileType) {
      case ProfileType.ADMIN:
        return Role.ADMIN;
      case ProfileType.DRIVER:
        return Role.DRIVER;
      case ProfileType.RIDER:
        return Role.RIDER;
      default:
        throw new Error(`Unknown profile type: ${user.profileType}`);
    }
  }
}
```

---

## User Creation Flow (Keeping Boundary During User Onboarding)

```typescript
// [PATTERN] Creating a user while maintaining the auth boundary

@Injectable()
export class UserCreationService {
  constructor(
    private readonly authProviderWrapper: AuthProviderWrapper,
    private readonly userRepository: UserRepository,
    private readonly authMappingRepository: UserAuthMappingRepository,
  ) {}

  async createUser(dto: CreateUserDto, context: RequestContext): Promise<UserResponseDto> {
    // [STEP 1] Create in auth provider — returns a provider ID
    const providerUserId = await this.authProviderWrapper.createProviderUser(
      dto.email,
      dto.temporaryPassword,
    );
    
    // [STEP 2] Create internal user — your internal UUID
    const user = await this.userRepository.create({
      tenantId: context.tenantId,  // from requestContext, not from DTO
      role: dto.role,
      profileType: this.toProfileType(dto.role),
      // [CONSTRAINT] providerUserId is NOT stored here — it goes to the mapping table
    });
    
    // [STEP 3] Store the provider ID → internal userId mapping
    // [CONSTRAINT] This is the ONLY table that stores providerUserId
    await this.authMappingRepository.create({
      providerUserId: providerUserId,
      userId: user.id,
    });
    
    // [STEP 4] Response uses internal userId only — providerUserId is dropped
    return this.toResponseDto(user);
    // [CONSTRAINT] Response does NOT include providerUserId
  }
}
```

---

## Delete & Recreate Flow (User Movement)

```typescript
// [PATTERN] User movement via Delete & Recreate — maintains auth boundary

async moveUserToNewTenant(
  userId: string,
  newTenantId: string,
  context: RequestContext,
): Promise<void> {
  // [STEP 1] Load current user and their auth mapping
  const user = await this.userRepository.findById(userId);
  const mapping = await this.authMappingRepository.findByUserId(userId);
  
  // [STEP 2] Delete from auth provider
  await this.authProviderWrapper.deleteProviderUser(mapping.providerUserId);
  
  // [STEP 3] Anonymize the existing user record
  await this.userRepository.anonymize(userId, {
    firstName: `ANONYMIZED`,
    lastName: `ANONYMIZED`,
    email: `anonymized_${userId}@domain.com`,
    // [CONSTRAINT] Retain userId, tenantId for analytics/audit traceability
    // [CONSTRAINT] Do NOT update tenantId — it remains as the old tenant for history
  });
  
  // [STEP 4] Delete the auth mapping (it referenced the old provider identity)
  await this.authMappingRepository.delete(mapping.id);
  
  // [STEP 5] Create a fresh user in the new tenant (triggers full creation flow)
  // This creates a new provider identity, new userId, new mapping
  await this.createUser(
    { ...newUserDetails, tenantId: newTenantId },
    context,
  );
}
```

---

## What to NEVER Generate

```typescript
// [ANTI-PATTERN 1] Provider SDK in a business module
import { ProviderSdk } from 'provider-sdk'; // in booking.service.ts
// → VIOLATION: provider must be isolated to the wrapper

// [ANTI-PATTERN 2] providerUserId in a business entity
@Column({ name: 'cognito_sub' })
cognitoSub: string; // in booking.entity.ts
// → VIOLATION: provider ID must live only in the auth mapping table

// [ANTI-PATTERN 3] Role from JWT claims
const role = decodedToken.groups[0]; // in role.guard.ts
// → VIOLATION: role must come from DB profile type

// [ANTI-PATTERN 4] providerUserId in an API response
return { userId: user.id, providerUserId: mapping.providerUserId }; // in a controller
// → VIOLATION: clients must never see provider identifiers
```
