# PostgreSQL Nanoid Installation

Complete setup guide for nanoid functions in PostgreSQL/Supabase.

## Prerequisites

PostgreSQL v12+ with pgcrypto extension.

## Step 1: Enable pgcrypto

```sql
-- Run in Supabase SQL Editor or migration
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

## Step 2: Create Helper Function

```sql
CREATE OR REPLACE FUNCTION nanoid_optimized(
  size int,
  alphabet text,
  mask int,
  step int
)
RETURNS text
LANGUAGE plpgsql
VOLATILE
PARALLEL SAFE
AS $$
DECLARE
  idBuilder text := '';
  counter int := 0;
  bytes bytea;
  alphabetIndex int;
  alphabetArray text[];
  alphabetLength int := length(alphabet);
BEGIN
  alphabetArray := regexp_split_to_array(alphabet, '');

  LOOP
    bytes := gen_random_bytes(step);
    FOR counter IN 0..step - 1 LOOP
      alphabetIndex := (get_byte(bytes, counter) & mask) + 1;
      IF alphabetIndex <= alphabetLength THEN
        idBuilder := idBuilder || alphabetArray[alphabetIndex];
        IF length(idBuilder) = size THEN
          RETURN idBuilder;
        END IF;
      END IF;
    END LOOP;
  END LOOP;
END
$$;
```

## Step 3: Create Main nanoid Function

```sql
CREATE OR REPLACE FUNCTION nanoid(
  prefix text DEFAULT '',
  size int DEFAULT 21,
  alphabet text DEFAULT '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ',
  additionalBytesFactor float DEFAULT 1.02
)
RETURNS text
LANGUAGE plpgsql
VOLATILE
PARALLEL SAFE
AS $$
DECLARE
  alphabetLength int := length(alphabet);
  mask int := (2 << cast(floor(log(alphabetLength - 1) / log(2)) as int)) - 1;
  step int := cast(ceil(additionalBytesFactor * mask * (size - length(prefix)) / alphabetLength) as int);
  randomPart text;
BEGIN
  -- Generate the random part (size minus prefix length)
  randomPart := nanoid_optimized(size - length(prefix), alphabet, mask, step);

  -- Return prefix + random part
  RETURN prefix || randomPart;
END
$$;
```

## Complete Migration File

Create a migration file (e.g., `20240101000000_add_nanoid.sql`):

```sql
-- Migration: Add nanoid function to database
-- This provides URL-safe, collision-resistant identifiers with optional prefixes

-- Enable pgcrypto for secure random bytes
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Helper function for optimized generation
CREATE OR REPLACE FUNCTION nanoid_optimized(
  size int,
  alphabet text,
  mask int,
  step int
)
RETURNS text
LANGUAGE plpgsql
VOLATILE
PARALLEL SAFE
AS $$
DECLARE
  idBuilder text := '';
  counter int := 0;
  bytes bytea;
  alphabetIndex int;
  alphabetArray text[];
  alphabetLength int := length(alphabet);
BEGIN
  alphabetArray := regexp_split_to_array(alphabet, '');

  LOOP
    bytes := gen_random_bytes(step);
    FOR counter IN 0..step - 1 LOOP
      alphabetIndex := (get_byte(bytes, counter) & mask) + 1;
      IF alphabetIndex <= alphabetLength THEN
        idBuilder := idBuilder || alphabetArray[alphabetIndex];
        IF length(idBuilder) = size THEN
          RETURN idBuilder;
        END IF;
      END IF;
    END LOOP;
  END LOOP;
END
$$;

-- Main nanoid function with prefix support
CREATE OR REPLACE FUNCTION nanoid(
  prefix text DEFAULT '',
  size int DEFAULT 21,
  alphabet text DEFAULT '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ',
  additionalBytesFactor float DEFAULT 1.02
)
RETURNS text
LANGUAGE plpgsql
VOLATILE
PARALLEL SAFE
AS $$
DECLARE
  alphabetLength int := length(alphabet);
  mask int := (2 << cast(floor(log(alphabetLength - 1) / log(2)) as int)) - 1;
  step int := cast(ceil(additionalBytesFactor * mask * (size - length(prefix)) / alphabetLength) as int);
  randomPart text;
BEGIN
  randomPart := nanoid_optimized(size - length(prefix), alphabet, mask, step);
  RETURN prefix || randomPart;
END
$$;

-- Add comment for documentation
COMMENT ON FUNCTION nanoid IS 'Generate URL-safe nanoid with optional prefix. Default size is 21 chars.';
```

## Usage Examples

### Basic Generation

```sql
-- Generate with prefix (recommended)
SELECT nanoid('usr_');    -- usr_V1StGXR8_Z5jdHi6B

-- Generate without prefix
SELECT nanoid();          -- V1StGXR8_Z5jdHi6B-2Sg

-- Custom size (total including prefix)
SELECT nanoid('ord_', 25); -- ord_xYz7aBcDeF2gH5iJ8kL
```

### Table Column Default

```sql
CREATE TABLE public.orders (
  id TEXT NOT NULL DEFAULT nanoid('ord_') PRIMARY KEY,
  customer_id TEXT NOT NULL,
  total_amount DECIMAL(10,2),
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### Batch Generation

```sql
-- Generate 100 IDs efficiently
SELECT nanoid('inv_')
FROM generate_series(1, 100);
```

### Custom Alphabet

```sql
-- Hex-only IDs
SELECT nanoid('tx_', 24, '0123456789abcdef');
-- tx_a1b2c3d4e5f6a7b8c9d0

-- Lowercase only
SELECT nanoid('key_', 21, 'abcdefghijklmnopqrstuvwxyz');
-- key_abcdefghijklmnopqr
```

## Verification

Test your installation:

```sql
-- Should return a 21-char string starting with 'test_'
SELECT nanoid('test_');

-- Verify format
SELECT nanoid('usr_') ~ '^usr_[0-9a-zA-Z]{17}$' AS valid_format;
-- Should return: true

-- Performance test (should complete in < 1 second)
SELECT count(*) FROM (
  SELECT nanoid('perf_')
  FROM generate_series(1, 10000)
) t;
-- Should return: 10000
```

## Supabase Dashboard Setup

1. Go to **SQL Editor** in Supabase Dashboard
2. Create a new query
3. Paste the complete migration file
4. Click **Run**
5. Verify with: `SELECT nanoid('test_');`

## Troubleshooting

### Error: function gen_random_bytes does not exist

```sql
-- pgcrypto not enabled
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### Error: permission denied

```sql
-- Grant execute permission (if needed)
GRANT EXECUTE ON FUNCTION nanoid TO authenticated;
GRANT EXECUTE ON FUNCTION nanoid TO service_role;
```

### Slow generation

- Ensure `PARALLEL SAFE` is set
- Check PostgreSQL version (v12+ recommended)
- Consider connection pooling for high-volume generation
