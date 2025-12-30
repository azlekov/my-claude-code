# Next.js Caching Reference

## The `use cache` Directive

The modern approach to caching in Next.js. Replaces legacy patterns.

## Basic Usage

### Function Caching

```tsx
async function getProducts() {
  'use cache'
  return await db.products.findMany()
}
```

### Component Caching

```tsx
async function ProductList() {
  'use cache'
  const products = await db.products.findMany()
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  )
}
```

### Page Caching

```tsx
// app/products/page.tsx
export default async function ProductsPage() {
  'use cache'
  const products = await getProducts()
  return <ProductGrid products={products} />
}
```

## Cache Tags

Tag cached data for targeted revalidation:

```tsx
import { cacheTag } from 'next/cache'

async function getProduct(id: string) {
  'use cache'
  cacheTag('products')
  cacheTag(`product-${id}`)

  return await db.products.find(id)
}

async function getProductsByCategory(category: string) {
  'use cache'
  cacheTag('products')
  cacheTag(`category-${category}`)

  return await db.products.findMany({ where: { category } })
}
```

## Cache Life

Control cache duration:

```tsx
import { cacheLife } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheLife('hours') // Cache for 1 hour

  return await db.products.findMany()
}
```

### Built-in Profiles

| Profile | `stale` | `revalidate` | `expire` |
|---------|---------|--------------|----------|
| `'seconds'` | undefined | 1 | 60 |
| `'minutes'` | 300 | 60 | 3600 |
| `'hours'` | 300 | 3600 | 86400 |
| `'days'` | 300 | 86400 | 604800 |
| `'weeks'` | 300 | 604800 | 2592000 |
| `'max'` | 300 | 2592000 | Infinity |

### Custom Duration

```tsx
import { cacheLife } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheLife({
    stale: 60,        // Serve stale for 60s while revalidating
    revalidate: 300,  // Revalidate every 5 minutes
    expire: 3600      // Remove from cache after 1 hour
  })

  return await db.products.findMany()
}
```

## Cache Scopes

### Static Cache (Default)

Cache shared across all users:

```tsx
async function getProducts() {
  'use cache'
  return await db.products.findMany()
}
```

### Remote Cache

Distributed cache for multi-instance deployments:

```tsx
async function getProducts() {
  'use cache: remote'
  return await db.products.findMany()
}
```

### Private Cache

Per-user cache (uses cookies/headers):

```tsx
async function getUserCart() {
  'use cache: private'
  const user = await getCurrentUser()
  return await db.carts.find({ userId: user.id })
}
```

## Revalidation

### Tag-Based Revalidation

```tsx
'use server'

import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: FormData) {
  await db.products.update(id, Object.fromEntries(data))

  // Revalidate specific product
  revalidateTag(`product-${id}`)

  // Revalidate all products
  revalidateTag('products')
}

export async function deleteProduct(id: string) {
  await db.products.delete(id)

  revalidateTag(`product-${id}`)
  revalidateTag('products')
}
```

### Path-Based Revalidation

```tsx
'use server'

import { revalidatePath } from 'next/cache'

export async function updateProduct(id: string, data: FormData) {
  await db.products.update(id, Object.fromEntries(data))

  // Revalidate specific page
  revalidatePath(`/products/${id}`)

  // Revalidate all products pages
  revalidatePath('/products')

  // Revalidate layout (and all nested pages)
  revalidatePath('/products', 'layout')
}
```

### On-Demand Revalidation via API

```tsx
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache'
import { NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  const { tag, secret } = await request.json()

  // Validate secret
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 })
  }

  revalidateTag(tag)
  return NextResponse.json({ revalidated: true })
}
```

## Combining Tags and Life

```tsx
import { cacheTag, cacheLife } from 'next/cache'

async function getProduct(id: string) {
  'use cache'
  cacheTag('products', `product-${id}`)
  cacheLife('hours')

  return await db.products.find(id)
}
```

## Opting Out of Caching

### Dynamic Functions

Using these opts out of static caching:

```tsx
import { cookies, headers } from 'next/headers'

async function getUser() {
  const cookieStore = await cookies()
  const token = cookieStore.get('token')
  // This function is now dynamic
}
```

### Dynamic Route Segments

```tsx
// Force dynamic rendering
export const dynamic = 'force-dynamic'

// Force static rendering (error if dynamic functions used)
export const dynamic = 'force-static'

// Default: auto-detect
export const dynamic = 'auto'
```

## Fetch Caching

When using fetch() in Server Components:

```tsx
// Cached by default
const data = await fetch('https://api.example.com/data')

// Revalidate every hour
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }
})

// No caching
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store'
})

// With tags
const data = await fetch('https://api.example.com/data', {
  next: { tags: ['data'] }
})
```

## Best Practices

1. **Use `use cache` over fetch options** - More explicit and composable
2. **Tag granularly** - Use both general (`'products'`) and specific (`'product-123'`) tags
3. **Choose appropriate cache life** - Balance freshness vs. performance
4. **Use private cache for user data** - Never cache user-specific data publicly
5. **Revalidate on mutations** - Always revalidate after data changes
