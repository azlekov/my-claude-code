---
description: Full guided Supabase integration for your project. Specify framework as argument.
allowed_arguments: nextjs, kotlin, ios, flutter
---

# Add Supabase to Project

Add Supabase to this $ARGUMENTS project with full authentication, database setup, and nanoid identifiers.

## Framework Detection

If no argument provided, detect the framework:
- `package.json` with `next` → Next.js
- `build.gradle.kts` with `supabase-kt` or Kotlin files → Kotlin
- `Package.swift` or `.xcodeproj` → iOS
- `pubspec.yaml` with Flutter → Flutter

## Common Setup (All Frameworks)

### 1. Verify/Create Supabase Project

Check if Supabase project exists:
```bash
# Check for existing config
ls supabase/config.toml 2>/dev/null || echo "No Supabase project found"
```

If no project exists:
```bash
# Initialize Supabase
npx supabase init

# Link to existing project (if user has one)
npx supabase link --project-ref <project-ref>
```

### 2. Set Up nanoid Function

Create migration for nanoid:
```bash
npx supabase migration new add_nanoid_function
```

Add to migration file:
```sql
-- Enable pgcrypto
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Helper function
CREATE OR REPLACE FUNCTION nanoid_optimized(
  size int, alphabet text, mask int, step int
) RETURNS text LANGUAGE plpgsql VOLATILE PARALLEL SAFE AS $$
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
        IF length(idBuilder) = size THEN RETURN idBuilder; END IF;
      END IF;
    END LOOP;
  END LOOP;
END $$;

-- Main nanoid function
CREATE OR REPLACE FUNCTION nanoid(
  prefix text DEFAULT '',
  size int DEFAULT 21,
  alphabet text DEFAULT '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ',
  additionalBytesFactor float DEFAULT 1.02
) RETURNS text LANGUAGE plpgsql VOLATILE PARALLEL SAFE AS $$
DECLARE
  alphabetLength int := length(alphabet);
  mask int := (2 << cast(floor(log(alphabetLength - 1) / log(2)) as int)) - 1;
  step int := cast(ceil(additionalBytesFactor * mask * (size - length(prefix)) / alphabetLength) as int);
BEGIN
  RETURN prefix || nanoid_optimized(size - length(prefix), alphabet, mask, step);
END $$;
```

### 3. Create Example profiles Table

```bash
npx supabase migration new create_profiles_table
```

```sql
-- Profiles table with nanoid
CREATE TABLE public.profiles (
  id TEXT NOT NULL DEFAULT nanoid('usr_') PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT profiles_id_format CHECK (id ~ '^usr_[0-9a-zA-Z]{17}$'),
  CONSTRAINT profiles_user_id_unique UNIQUE (user_id)
);

-- Enable RLS
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can view own profile"
  ON public.profiles FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.profiles (user_id, display_name)
  VALUES (new.id, new.raw_user_meta_data->>'full_name');
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- Index for user lookups
CREATE INDEX profiles_user_id_idx ON public.profiles(user_id);
```

---

## Next.js Setup

### 1. Install Dependencies

```bash
npm install @supabase/supabase-js @supabase/ssr
```

### 2. Environment Variables

Create `.env.local`:
```env
NEXT_PUBLIC_SUPABASE_URL=your-project-url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

### 3. Create Supabase Clients

**`lib/supabase/client.ts`** (Browser):
```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

**`lib/supabase/server.ts`** (Server):
```typescript
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // Called from Server Component
          }
        },
      },
    }
  )
}
```

**`lib/supabase/middleware.ts`**:
```typescript
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          )
          supabaseResponse = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  await supabase.auth.getUser()

  return supabaseResponse
}
```

### 4. Update Middleware

**`middleware.ts`**:
```typescript
import { type NextRequest } from 'next/server'
import { updateSession } from '@/lib/supabase/middleware'

export async function middleware(request: NextRequest) {
  return await updateSession(request)
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
}
```

### 5. Generate Types

```bash
npx supabase gen types typescript --local > src/types/database.ts
```

---

## Kotlin Setup

### 1. Add Dependencies

**`build.gradle.kts`**:
```kotlin
dependencies {
    implementation(platform("io.github.jan-tennert.supabase:bom:3.1.1"))
    implementation("io.github.jan-tennert.supabase:postgrest-kt")
    implementation("io.github.jan-tennert.supabase:auth-kt")
    implementation("io.github.jan-tennert.supabase:storage-kt")
    implementation("io.ktor:ktor-client-cio:3.1.1")
}
```

### 2. Create Supabase Client

```kotlin
import io.github.jan.supabase.createSupabaseClient
import io.github.jan.supabase.auth.Auth
import io.github.jan.supabase.postgrest.Postgrest
import io.github.jan.supabase.storage.Storage

object SupabaseClient {
    val client = createSupabaseClient(
        supabaseUrl = BuildConfig.SUPABASE_URL,
        supabaseKey = BuildConfig.SUPABASE_ANON_KEY
    ) {
        install(Auth)
        install(Postgrest)
        install(Storage)
    }
}
```

### 3. Repository Pattern

```kotlin
class ProfileRepository(private val supabase: SupabaseClient) {
    suspend fun getProfile(userId: String): Profile? {
        return supabase.client.postgrest
            .from("profiles")
            .select()
            .eq("user_id", userId)
            .decodeSingleOrNull<Profile>()
    }
}
```

---

## iOS (Swift) Setup

### 1. Add Package

In Xcode: File → Add Package Dependencies
URL: `https://github.com/supabase/supabase-swift`

### 2. Create Client

```swift
import Supabase

let supabase = SupabaseClient(
    supabaseURL: URL(string: "YOUR_SUPABASE_URL")!,
    supabaseKey: "YOUR_SUPABASE_ANON_KEY"
)
```

### 3. Auth State Management

```swift
@MainActor
class AuthManager: ObservableObject {
    @Published var session: Session?

    init() {
        Task {
            for await state in supabase.auth.authStateChanges {
                self.session = state.session
            }
        }
    }
}
```

---

## Flutter Setup

### 1. Add Dependency

**`pubspec.yaml`**:
```yaml
dependencies:
  supabase_flutter: ^2.8.3
```

### 2. Initialize in main.dart

```dart
import 'package:supabase_flutter/supabase_flutter.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Supabase.initialize(
    url: 'YOUR_SUPABASE_URL',
    anonKey: 'YOUR_SUPABASE_ANON_KEY',
  );

  runApp(const MyApp());
}

final supabase = Supabase.instance.client;
```

### 3. Deep Link Configuration

**Android** (`android/app/src/main/AndroidManifest.xml`):
```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="your.app.scheme" android:host="login-callback" />
</intent-filter>
```

**iOS** (`ios/Runner/Info.plist`):
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>your.app.scheme</string>
        </array>
    </dict>
</array>
```

---

## Verification Steps

After setup, verify:

1. **Database connection**: Run `npx supabase db push` to apply migrations
2. **nanoid function**: Test with `SELECT nanoid('test_');` in SQL Editor
3. **Auth flow**: Test sign up/sign in
4. **RLS policies**: Verify data access is properly restricted
5. **Types generated**: Check TypeScript types are up to date

## Next Steps

- Add more tables with nanoid pattern
- Configure additional auth providers
- Set up storage buckets with RLS
- Add Edge Functions for complex logic
