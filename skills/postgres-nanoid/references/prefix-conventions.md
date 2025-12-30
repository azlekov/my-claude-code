# Nanoid Prefix Conventions

Complete guide for choosing and implementing contextual prefixes.

## Standard Prefix Table

### User & Account Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| User Profile | `usr_` | 21 | Public user identifier |
| Account | `acc_` | 21 | Account/billing entity |
| Customer | `cus_` | 21 | Customer record |
| Member | `mem_` | 21 | Membership record |
| Contact | `cnt_` | 21 | Contact/lead |

### Organization Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Organization | `org_` | 21 | Company/organization |
| Team | `team_` | 22 | Team within org |
| Workspace | `ws_` | 20 | Workspace/environment |
| Project | `proj_` | 22 | Project container |
| Department | `dept_` | 22 | Department unit |

### Commerce Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Product | `prd_` | 21 | Product catalog item |
| Order | `ord_` | 21 | Purchase order |
| Invoice | `inv_` | 21 | Invoice document |
| Payment | `pay_` | 21 | Payment record |
| Subscription | `sub_` | 21 | Recurring subscription |
| Cart | `cart_` | 22 | Shopping cart |
| Discount | `disc_` | 22 | Discount code |
| Refund | `ref_` | 21 | Refund record |

### Financial Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Transaction | `txn_` | 21 | Financial transaction |
| Transfer | `xfr_` | 21 | Fund transfer |
| Payout | `po_` | 20 | Payout to recipient |
| Balance | `bal_` | 21 | Balance record |

### Content Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Post | `post_` | 22 | Blog/social post |
| Comment | `cmt_` | 21 | Comment on content |
| Article | `art_` | 21 | Article/document |
| Page | `pg_` | 20 | Page content |
| Media | `med_` | 21 | Media asset |
| File | `file_` | 22 | File upload |
| Folder | `fldr_` | 22 | Folder/directory |

### Communication Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Message | `msg_` | 21 | Chat/email message |
| Thread | `thd_` | 21 | Conversation thread |
| Notification | `ntf_` | 21 | Notification record |
| Email | `eml_` | 21 | Email record |
| Channel | `ch_` | 20 | Communication channel |

### System Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Session | `ses_` | 21 | User session |
| API Key | `key_` | 21 | API key/token |
| Webhook | `whk_` | 21 | Webhook endpoint |
| Event | `evt_` | 21 | System event |
| Job | `job_` | 21 | Background job |
| Task | `tsk_` | 21 | Task/action item |
| Log | `log_` | 21 | Log entry |
| Audit | `aud_` | 21 | Audit record |

### Integration Entities

| Entity | Prefix | Total Length | Description |
|--------|--------|--------------|-------------|
| Connection | `con_` | 21 | External connection |
| Integration | `int_` | 21 | Integration config |
| Sync | `sync_` | 22 | Sync operation |
| Import | `imp_` | 21 | Import job |
| Export | `exp_` | 21 | Export job |

## Prefix Design Guidelines

### 1. Length Rules

```
Standard format: {prefix}_{random}
                 └─3-4 chars─┘ └─17 chars─┘

Total length = prefix length + 1 (underscore) + 17 = 21-22 chars
```

### 2. Naming Conventions

**Do:**
- Use lowercase letters only
- Keep prefix 2-4 characters
- Use common abbreviations
- End with underscore separator

**Don't:**
- Use numbers in prefix
- Use special characters (except underscore separator)
- Make prefix too long (>5 chars)
- Use ambiguous abbreviations

### 3. Creating New Prefixes

```sql
-- Step 1: Choose prefix (check for conflicts)
-- New entity: "Campaign" -> "cmp_"

-- Step 2: Define with CHECK constraint
CREATE TABLE public.campaigns (
  id TEXT NOT NULL DEFAULT nanoid('cmp_') PRIMARY KEY,
  name TEXT NOT NULL,
  CONSTRAINT campaigns_id_format CHECK (id ~ '^cmp_[0-9a-zA-Z]{17}$')
);

-- Step 3: Document in this file
```

## Validation Patterns

### PostgreSQL CHECK Constraints

```sql
-- Generic pattern (adjust prefix)
CHECK (column_name ~ '^{prefix}_[0-9a-zA-Z]{17}$')

-- Examples
CHECK (id ~ '^usr_[0-9a-zA-Z]{17}$')   -- User
CHECK (id ~ '^org_[0-9a-zA-Z]{17}$')   -- Organization
CHECK (id ~ '^team_[0-9a-zA-Z]{17}$')  -- Team (note: 17 random chars)
```

### TypeScript Validation

```typescript
// Type definitions
type UserId = `usr_${string}`
type OrgId = `org_${string}`
type TeamId = `team_${string}`

// Validation functions
const ID_PATTERNS = {
  usr: /^usr_[0-9a-zA-Z]{17}$/,
  org: /^org_[0-9a-zA-Z]{17}$/,
  team: /^team_[0-9a-zA-Z]{17}$/,
  ord: /^ord_[0-9a-zA-Z]{17}$/,
} as const

function validateId<T extends keyof typeof ID_PATTERNS>(
  prefix: T,
  id: string
): boolean {
  return ID_PATTERNS[prefix].test(id)
}

// Usage
validateId('usr', 'usr_V1StGXR8_Z5jdHi')  // true
validateId('usr', 'invalid')               // false
```

### Zod Schema

```typescript
import { z } from 'zod'

const userIdSchema = z.string().regex(
  /^usr_[0-9a-zA-Z]{17}$/,
  'Invalid user ID format'
)

const orgIdSchema = z.string().regex(
  /^org_[0-9a-zA-Z]{17}$/,
  'Invalid organization ID format'
)

// Combined schema
const entityIdSchema = z.union([
  z.string().regex(/^usr_[0-9a-zA-Z]{17}$/),
  z.string().regex(/^org_[0-9a-zA-Z]{17}$/),
  z.string().regex(/^ord_[0-9a-zA-Z]{17}$/),
])
```

## Cross-Reference Matrix

When entities reference each other, maintain clear foreign key patterns:

```sql
-- Orders reference users and organizations
CREATE TABLE public.orders (
  id TEXT NOT NULL DEFAULT nanoid('ord_') PRIMARY KEY,

  -- References (nanoid to nanoid)
  customer_id TEXT NOT NULL REFERENCES public.customers(id),
  organization_id TEXT REFERENCES public.organizations(id),

  -- Auth reference (nanoid table -> UUID auth)
  created_by UUID NOT NULL REFERENCES auth.users(id),

  -- Constraints
  CONSTRAINT orders_id_format CHECK (id ~ '^ord_[0-9a-zA-Z]{17}$')
);
```

## Migration from UUID

When migrating existing UUID-based tables:

```sql
-- Add public_id while keeping UUID primary key
ALTER TABLE public.legacy_table
ADD COLUMN public_id TEXT UNIQUE DEFAULT nanoid('leg_');

-- Backfill existing rows
UPDATE public.legacy_table
SET public_id = nanoid('leg_')
WHERE public_id IS NULL;

-- Make non-nullable
ALTER TABLE public.legacy_table
ALTER COLUMN public_id SET NOT NULL;

-- Use public_id in APIs, keep UUID internally
```

## Best Practices Summary

1. **Consistency**: Use same prefix across all environments (dev, staging, prod)
2. **Documentation**: Document all prefixes in a central location
3. **Validation**: Always add CHECK constraints in database
4. **API Design**: Only expose nanoid in public APIs
5. **Logging**: Include full nanoid (with prefix) in logs
6. **Search**: Index public_id columns for efficient lookups
