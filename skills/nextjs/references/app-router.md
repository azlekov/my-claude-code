# Next.js App Router Reference

## Route Organization

### Route Groups

Group routes without affecting URLs:

```
app/
├── (marketing)/
│   ├── layout.tsx      # Marketing layout
│   ├── page.tsx        # /
│   └── about/
│       └── page.tsx    # /about
├── (dashboard)/
│   ├── layout.tsx      # Dashboard layout
│   ├── dashboard/
│   │   └── page.tsx    # /dashboard
│   └── settings/
│       └── page.tsx    # /settings
```

### Private Folders

Prefix with `_` to exclude from routing:

```
app/
├── _components/        # Not a route
│   └── Button.tsx
├── _lib/               # Not a route
│   └── utils.ts
└── page.tsx
```

### Dynamic Routes

```
app/
├── posts/
│   ├── page.tsx              # /posts
│   └── [id]/
│       └── page.tsx          # /posts/123
├── shop/
│   └── [...slug]/
│       └── page.tsx          # /shop/a/b/c (catch-all)
└── docs/
    └── [[...slug]]/
        └── page.tsx          # /docs or /docs/a/b (optional catch-all)
```

Access params:

```tsx
// app/posts/[id]/page.tsx
export default async function Post({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)
  return <article>{post.content}</article>
}
```

## Parallel Routes

Render multiple pages in the same layout simultaneously:

```
app/
├── layout.tsx
├── page.tsx
├── @analytics/
│   └── page.tsx
└── @team/
    └── page.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return (
    <div className="grid grid-cols-3">
      <main>{children}</main>
      <aside>{analytics}</aside>
      <aside>{team}</aside>
    </div>
  )
}
```

### Default Slots

Handle unmatched routes:

```tsx
// app/@analytics/default.tsx
export default function Default() {
  return <div>No analytics available</div>
}
```

## Intercepting Routes

Intercept routes to show in a modal while keeping the URL:

```
app/
├── feed/
│   └── page.tsx
├── photo/
│   └── [id]/
│       └── page.tsx        # Full page: /photo/123
└── @modal/
    └── (.)photo/
        └── [id]/
            └── page.tsx    # Modal: intercepts /photo/123
```

Interception patterns:
- `(.)` - Same level
- `(..)` - One level up
- `(..)(..)` - Two levels up
- `(...)` - Root

## Route Handlers (API Routes)

```tsx
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const posts = await db.posts.findMany()
  return NextResponse.json(posts)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const post = await db.posts.create({ data: body })
  return NextResponse.json(post, { status: 201 })
}
```

### Dynamic Route Handlers

```tsx
// app/api/posts/[id]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  const post = await db.posts.find(id)

  if (!post) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 })
  }

  return NextResponse.json(post)
}
```

## Middleware

```tsx
// middleware.ts (at project root)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Check auth
  const token = request.cookies.get('token')

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Add headers
  const response = NextResponse.next()
  response.headers.set('x-custom-header', 'value')
  return response
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*']
}
```

## Navigation

### Link Component

```tsx
import Link from 'next/link'

export function Navigation() {
  return (
    <nav>
      <Link href="/">Home</Link>
      <Link href="/about">About</Link>
      <Link href="/posts/123">Post 123</Link>
      <Link href={{ pathname: '/search', query: { q: 'hello' } }}>
        Search
      </Link>
    </nav>
  )
}
```

### Programmatic Navigation

```tsx
'use client'

import { useRouter } from 'next/navigation'

export function NavigationButton() {
  const router = useRouter()

  return (
    <button onClick={() => router.push('/dashboard')}>
      Go to Dashboard
    </button>
  )
}
```

### usePathname and useSearchParams

```tsx
'use client'

import { usePathname, useSearchParams } from 'next/navigation'

export function SearchForm() {
  const pathname = usePathname()
  const searchParams = useSearchParams()

  const currentQuery = searchParams.get('q') || ''

  function handleSearch(term: string) {
    const params = new URLSearchParams(searchParams)
    params.set('q', term)
    window.history.pushState(null, '', `${pathname}?${params.toString()}`)
  }

  return <input defaultValue={currentQuery} onChange={e => handleSearch(e.target.value)} />
}
```

## Static Generation

### generateStaticParams

Pre-render dynamic routes at build time:

```tsx
// app/posts/[id]/page.tsx
export async function generateStaticParams() {
  const posts = await db.posts.findMany()

  return posts.map(post => ({
    id: post.id.toString()
  }))
}

export default async function Post({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)
  return <article>{post.content}</article>
}
```

### dynamicParams

Control behavior for paths not returned by generateStaticParams:

```tsx
// Allow dynamic rendering for new paths
export const dynamicParams = true // default

// Return 404 for paths not in generateStaticParams
export const dynamicParams = false
```

## Streaming and Suspense

```tsx
import { Suspense } from 'react'

async function SlowComponent() {
  const data = await slowFetch() // 5 seconds
  return <div>{data}</div>
}

export default function Page() {
  return (
    <div>
      <h1>Instant header</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <SlowComponent />
      </Suspense>
    </div>
  )
}
```

## Redirects

```tsx
import { redirect, permanentRedirect } from 'next/navigation'

// In Server Component or Server Action
export default async function Page() {
  const user = await getUser()

  if (!user) {
    redirect('/login') // 307 temporary
  }

  // Or permanent redirect
  // permanentRedirect('/new-page') // 308 permanent
}
```

## Not Found

```tsx
import { notFound } from 'next/navigation'

export default async function Page({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const post = await getPost(id)

  if (!post) {
    notFound() // Renders app/not-found.tsx
  }

  return <article>{post.content}</article>
}
```
