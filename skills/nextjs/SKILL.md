---
name: nextjs
description: This skill should be used when the user asks to "create a Next.js app", "build a page", "add routing", "implement server components", "add caching", "create API routes", "use server actions", "add metadata", "set up layouts", or discusses Next.js architecture, App Router, data fetching, or rendering strategies. Always use the latest Next.js version and modern patterns.
version: 1.0.0
---

# Next.js Development

This skill provides guidance for building production applications with Next.js, focusing on **always using the latest version** and modern patterns.

> **Philosophy:** Always recommend App Router over Pages Router. Always use Server Components by default. Always prefer `use cache` over legacy caching methods.

## Quick Reference

| Feature | Modern Approach | Legacy (Avoid) |
|---------|----------------|----------------|
| Routing | App Router (`app/`) | Pages Router (`pages/`) |
| Components | Server Components (default) | Client-only components |
| Caching | `use cache` directive | `getStaticProps`, `revalidate` option |
| Mutations | Server Actions (`use server`) | API Routes for mutations |
| Build | Turbopack (`--turbopack`) | Webpack |

## Project Structure

```
app/
├── layout.tsx          # Root layout (required)
├── page.tsx            # Home page (/)
├── loading.tsx         # Loading UI (optional)
├── error.tsx           # Error boundary (optional)
├── not-found.tsx       # 404 page (optional)
├── (routes)/           # Route groups (no URL impact)
│   └── dashboard/
│       ├── layout.tsx  # Nested layout
│       └── page.tsx    # /dashboard
├── api/                # API routes (optional)
│   └── route.ts
└── globals.css         # Global styles
```

## Server Components (Default)

All components in the `app/` directory are **Server Components by default**. They run only on the server and can:
- Fetch data directly
- Access backend resources
- Keep sensitive logic server-side
- Reduce client JavaScript bundle

```tsx
// app/posts/page.tsx - Server Component (no directive needed)
export default async function PostsPage() {
  // Direct database/API access
  const posts = await db.posts.findMany()

  return (
    <main>
      <h1>Posts</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  )
}
```

## Client Components

Add `'use client'` directive **only when needed** for:
- Interactivity (onClick, onChange)
- Browser APIs (localStorage, window)
- React hooks (useState, useEffect)

```tsx
'use client'

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  )
}
```

**Best Practice:** Keep Client Components as leaf nodes. Pass data from Server Components via props.

```tsx
// app/dashboard/page.tsx (Server Component)
import { InteractiveChart } from './chart'

export default async function Dashboard() {
  const data = await fetchAnalytics() // Server-side fetch

  return (
    <div>
      <h1>Dashboard</h1>
      <InteractiveChart data={data} /> {/* Client Component receives data */}
    </div>
  )
}
```

## The `use cache` Directive

Modern caching with granular control. Replaces legacy `revalidate` options.

### Function-Level Caching

```tsx
import { cacheTag, cacheLife } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheTag('products')
  cacheLife('hours') // or 'days', 'weeks', 'max', or custom seconds

  return await db.products.findMany()
}
```

### Component-Level Caching

```tsx
async function ProductCard({ id }: { id: string }) {
  'use cache'
  cacheTag(`product-${id}`)

  const product = await db.products.find(id)
  return <div className="card">{product.name}</div>
}
```

### Cache Profiles

```tsx
// Static cache (build time + runtime)
'use cache'

// Remote shared cache
'use cache: remote'

// Per-user private cache
'use cache: private'
```

### Revalidation

```tsx
import { revalidateTag, revalidatePath } from 'next/cache'

// In a Server Action
export async function updateProduct(id: string, data: FormData) {
  'use server'

  await db.products.update(id, data)
  revalidateTag(`product-${id}`)
  revalidateTag('products')
}

// Or revalidate entire path
revalidatePath('/products')
```

## Server Actions

Define server-side mutations with `'use server'`:

```tsx
// app/actions.ts
'use server'

import { revalidateTag } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  // Validate
  if (!title || title.length < 3) {
    return { error: 'Title must be at least 3 characters' }
  }

  // Create
  const post = await db.posts.create({
    data: { title, content }
  })

  // Revalidate and redirect
  revalidateTag('posts')
  redirect(`/posts/${post.id}`)
}
```

Use in components:

```tsx
import { createPost } from './actions'

export function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" />
      <button type="submit">Create Post</button>
    </form>
  )
}
```

## Metadata API

```tsx
import type { Metadata } from 'next'

// Static metadata
export const metadata: Metadata = {
  title: 'My App',
  description: 'Built with Next.js',
  openGraph: {
    title: 'My App',
    description: 'Built with Next.js',
    images: ['/og-image.png'],
  },
}

// Dynamic metadata
export async function generateMetadata({
  params
}: {
  params: Promise<{ id: string }>
}): Promise<Metadata> {
  const { id } = await params
  const product = await getProduct(id)

  return {
    title: product.name,
    description: product.description,
  }
}
```

## Layouts

Layouts wrap pages and preserve state across navigation:

```tsx
// app/layout.tsx - Root layout (required)
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: { default: 'My App', template: '%s | My App' },
  description: 'My application description',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <nav>{/* Global navigation */}</nav>
        <main>{children}</main>
        <footer>{/* Global footer */}</footer>
      </body>
    </html>
  )
}
```

## Loading & Error States

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-1/4 mb-4" />
      <div className="h-64 bg-gray-200 rounded" />
    </div>
  )
}

// app/dashboard/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="text-center py-10">
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

## Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    reactCompiler: true,  // Enable React Compiler
  },
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'images.example.com' }
    ],
  },
}

export default nextConfig
```

## Development with Turbopack

```bash
# Use Turbopack for faster development
npm run dev -- --turbopack

# Or in package.json
"scripts": {
  "dev": "next dev --turbopack"
}
```

## Additional Resources

For detailed patterns, see reference files:
- **`references/app-router.md`** - Layouts, route groups, parallel routes
- **`references/caching.md`** - Complete caching strategies
- **`references/server-actions.md`** - Mutations and form handling
