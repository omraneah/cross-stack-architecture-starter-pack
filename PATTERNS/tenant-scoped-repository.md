# Pattern: Tenant-Scoped Repository

**Stack:** NestJS + TypeORM (TypeScript)
**ARD Sources:** `multi-tenancy-boundaries.md`, `naming-conventions-boundaries.md`

---

## Overview

Every repository that manages tenant-owned data must enforce tenant scoping as a structural guarantee — not as optional behavior that callers remember to invoke. This pattern shows how to build a repository that cannot return unscoped data.

---

## Entity Definition (Naming Convention)

```typescript
// [PATTERN] Entity: camelCase TypeScript properties, snake_case DB columns
// [CONSTRAINT] Every tenant-owned entity has a tenantId column

import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, JoinColumn } from 'typeorm';

@Entity({ name: 'resource' })  // DB table name: snake_case
export class ResourceEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  // [CONSTRAINT] tenantId is non-nullable — all tenant-owned records must have it
  @Column({ name: 'tenant_id', nullable: false })
  tenantId!: string;               // camelCase property ↔ snake_case column

  @Column({ name: 'created_by_user_id', nullable: false })
  createdByUserId!: string;        // camelCase ↔ snake_case

  @Column({ name: 'resource_name' })
  resourceName!: string;

  @Column({ name: 'is_active', default: true })
  isActive!: boolean;

  @Column({ name: 'created_at', type: 'timestamptz', default: () => 'CURRENT_TIMESTAMP' })
  createdAt!: Date;                // camelCase ↔ snake_case

  @Column({ name: 'updated_at', type: 'timestamptz', default: () => 'CURRENT_TIMESTAMP' })
  updatedAt!: Date;
}
```

**Key constraint annotations:**
- `@Column({ name: 'snake_case_name' })` is required on every column where TypeScript property name differs from column name
- `tenantId` is `nullable: false` — no tenant-owned record can exist without a tenant

---

## Access Policy (Source of Truth for Visibility Rules)

```typescript
// [PATTERN] Access policy: single source for tenant visibility rules
// [CONSTRAINT] Reads ONLY from requestContext — never from request params

export interface ResourceFilter {
  tenantId?: string;          // undefined = platform admin (no scoping)
  userId?: string;            // further scoping when needed
}

@Injectable()
export class ResourceAccessPolicy {
  buildFilter(context: RequestContext): ResourceFilter {
    const { tenantId, userId, role } = context;
    
    switch (role) {
      case Role.PLATFORM_ADMIN:
        // Platform admin: explicit case — no tenant filter
        // [CONSTRAINT] This is the ONLY path that returns no tenantId filter
        return {};
        
      case Role.TENANT_ADMIN:
      case Role.MANAGER:
        // Tenant-scoped: see all resources within their tenant
        return { tenantId };
        
      case Role.USER:
        // User-scoped: see only their own resources within their tenant
        return { tenantId, userId };
        
      default:
        // Deny by returning an impossible filter if role is unrecognized
        return { tenantId: 'DENY_ALL' };
    }
  }
}
```

---

## Repository (Enforces Tenant Filter)

```typescript
// [PATTERN] Repository: tenant filter is required on every data-returning method
// [CONSTRAINT] No method returns data without applying the filter
// [ANTI-PATTERN] DO NOT expose findAll() with no filter

@Injectable()
export class ResourceRepository {
  constructor(
    @InjectRepository(ResourceEntity)
    private readonly repo: Repository<ResourceEntity>,
  ) {}

  // [PATTERN] All queries require a filter — tenant scoping is not optional
  async findMany(filter: ResourceFilter): Promise<ResourceEntity[]> {
    const query = this.repo.createQueryBuilder('resource');
    
    this.applyFilter(query, filter);
    
    return query.getMany();
  }

  async findById(id: string, filter: ResourceFilter): Promise<ResourceEntity | null> {
    const query = this.repo.createQueryBuilder('resource')
      .where('resource.id = :id', { id });
    
    // [CONSTRAINT] Filter applied even for single-record lookups
    // This prevents: GET /resources/:id from returning another tenant's resource
    this.applyFilter(query, filter);
    
    return query.getOne();
  }

  async create(data: Partial<ResourceEntity>, filter: ResourceFilter): Promise<ResourceEntity> {
    // [CONSTRAINT] tenantId on new record must come from the filter (from requestContext)
    // Never from client-supplied data
    const entity = this.repo.create({
      ...data,
      tenantId: filter.tenantId,  // from access policy filter, not from DTO
    });
    return this.repo.save(entity);
  }

  async update(
    id: string,
    data: Partial<ResourceEntity>,
    filter: ResourceFilter,
  ): Promise<ResourceEntity | null> {
    // [CONSTRAINT] Update scoped to tenant — cannot update another tenant's resource
    const existing = await this.findById(id, filter);
    if (!existing) return null;
    
    Object.assign(existing, data);
    return this.repo.save(existing);
  }

  async delete(id: string, filter: ResourceFilter): Promise<boolean> {
    // [CONSTRAINT] Delete scoped to tenant
    const existing = await this.findById(id, filter);
    if (!existing) return false;
    
    await this.repo.remove(existing);
    return true;
  }

  // [PRIVATE] Filter application — called by every public method
  private applyFilter(
    query: SelectQueryBuilder<ResourceEntity>,
    filter: ResourceFilter,
  ): void {
    if (filter.tenantId) {
      query.andWhere('resource.tenant_id = :tenantId', { tenantId: filter.tenantId });
    }
    if (filter.userId) {
      query.andWhere('resource.created_by_user_id = :userId', { userId: filter.userId });
    }
    // If filter is empty (platform admin case): no WHERE clause added
    // This is intentional and explicit — the access policy controls when this happens
  }
}
```

---

## Service (Orchestrates Policy + Repository)

```typescript
// [PATTERN] Service: calls access policy before repository
// [CONSTRAINT] Access policy call is mandatory before any repository operation on tenant data

@Injectable()
export class ResourceService {
  constructor(
    private readonly resourceRepository: ResourceRepository,
    private readonly resourceAccessPolicy: ResourceAccessPolicy,
  ) {}

  async getResources(context: RequestContext): Promise<ResourceResponseDto[]> {
    // [STEP 1] Build tenant-scoped filter from context
    const filter = this.resourceAccessPolicy.buildFilter(context);
    
    // [STEP 2] Pass filter to repository
    const resources = await this.resourceRepository.findMany(filter);
    
    // [STEP 3] Map to response DTO (camelCase, no internal implementation details)
    return resources.map(this.toResponseDto);
  }

  async createResource(
    context: RequestContext,
    dto: CreateResourceDto,
  ): Promise<ResourceResponseDto> {
    // [STEP 1] Build filter (contains tenantId from context)
    const filter = this.resourceAccessPolicy.buildFilter(context);
    
    // [STEP 2] Pass to repository — tenantId comes from filter, not from dto
    const resource = await this.resourceRepository.create(
      { resourceName: dto.resourceName },
      filter,
    );
    
    return this.toResponseDto(resource);
  }

  private toResponseDto(entity: ResourceEntity): ResourceResponseDto {
    return {
      id: entity.id,
      resourceName: entity.resourceName,
      isActive: entity.isActive,
      createdAt: entity.createdAt,
      // [CONSTRAINT] tenantId may or may not be exposed — depends on the API contract
      // See CLARIFICATION-NEEDED.md for the tenantId exposure question
    };
  }
}
```

---

## Migration (Schema Definition)

```typescript
// [PATTERN] Migration: snake_case column names, complete down()

export class CreateResourceTable<timestamp> implements MigrationInterface {
  name = 'CreateResourceTable<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE IF NOT EXISTS "resource" (
        "id"                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        "tenant_id"            UUID NOT NULL,   -- snake_case ✓, NOT NULL required
        "resource_name"        VARCHAR(255) NOT NULL,
        "created_by_user_id"   UUID NOT NULL,
        "is_active"            BOOLEAN NOT NULL DEFAULT TRUE,
        "created_at"           TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
        "updated_at"           TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    // Add index on tenant_id for query performance
    await queryRunner.query(`
      CREATE INDEX IF NOT EXISTS "idx_resource_tenant_id" ON "resource" ("tenant_id")
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX IF EXISTS "idx_resource_tenant_id"`);
    await queryRunner.query(`DROP TABLE IF EXISTS "resource"`);
  }
}
```

---

## Summary of Constraints

| Layer | Key Constraint |
|-------|---------------|
| Entity | `tenantId` column is `NOT NULL`; all columns snake_case with explicit `name:` |
| Access Policy | Reads only from `requestContext`; returns empty filter only for platform admin |
| Repository | Every public method accepts and applies a `filter` parameter |
| Repository | No `findAll()` without filter |
| Service | Access policy called before every repository operation on tenant data |
| Service | `tenantId` on created records comes from filter (requestContext), not from DTO |
| Migration | Column names snake_case; `IF NOT EXISTS` for idempotency; complete `down()` |
