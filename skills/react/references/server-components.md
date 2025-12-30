# React Server Components Reference

## Overview

Server Components run only on the server and send HTML to the client. They're the default in Next.js App Router.

## Server vs Client Components

| Feature | Server Component | Client Component |
|---------|-----------------|------------------|
| Directive | None (default) | `'use client'` |
| async/await | Yes | No |
| useState/useEffect | No | Yes |
| Event handlers | No | Yes |
| Browser APIs | No | Yes |
| Database access | Yes | No |
| File system | Yes | No |
| Bundle size impact | None | Added to bundle |

## When to Use Each

### Use Server Components For

- Fetching data
- Accessing backend resources
- Keeping sensitive information on server
- Large dependencies (stay on server)
- Static content

### Use Client Components For

- Interactivity and event listeners
- State (useState, useReducer)
- Effects (useEffect, useLayoutEffect)
- Browser APIs
- Custom hooks with state/effects

## Patterns

### Data Fetching in Server Components

```tsx
// Server Component - fetch directly
async function ProductList() {
  const products = await db.products.findMany()

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

### Passing Data to Client Components

```tsx
// Server Component fetches data
async function Dashboard() {
  const data = await fetchDashboardData()

  // Pass to Client Component via props
  return <InteractiveChart data={data} />
}
```

```tsx
// Client Component receives serializable data
'use client'

interface ChartData {
  labels: string[]
  values: number[]
}

export function InteractiveChart({ data }: { data: ChartData }) {
  const [selected, setSelected] = useState<number | null>(null)

  return (
    <div onClick={(e) => setSelected(...)}>
      {/* Render chart with data */}
    </div>
  )
}
```

### Composing Server and Client Components

```tsx
// Server Component (parent)
async function Page() {
  const user = await getUser()
  const posts = await getPosts(user.id)

  return (
    <div>
      {/* Server Component */}
      <Header user={user} />

      {/* Client Component */}
      <PostEditor userId={user.id} />

      {/* Server Component with Client child */}
      <PostList posts={posts}>
        <LikeButton /> {/* Client Component */}
      </PostList>
    </div>
  )
}
```

### Client Components Wrapping Server Components

```tsx
// Client Component can receive Server Components as children
'use client'

export function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)

  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      {open && (
        <div className="modal">
          {children} {/* Can be Server Component */}
        </div>
      )}
    </>
  )
}
```

```tsx
// Usage in Server Component
import { Modal } from './modal'
import { ServerContent } from './server-content'

export default function Page() {
  return (
    <Modal>
      <ServerContent /> {/* Server Component inside Client Component */}
    </Modal>
  )
}
```

## Serialization Rules

Data passed from Server to Client Components must be serializable:

### Allowed

- Primitives (string, number, boolean, null, undefined)
- Arrays and objects containing serializable values
- Date (serialized as string)
- Map and Set
- TypedArrays and ArrayBuffer
- Server Actions (functions with `'use server'`)

### Not Allowed

- Functions (except Server Actions)
- Classes
- DOM nodes
- Symbols
- Circular references

## Streaming with Suspense

```tsx
import { Suspense } from 'react'

async function SlowComponent() {
  const data = await slowFetch() // Takes 5 seconds
  return <div>{data}</div>
}

async function FastComponent() {
  const data = await fastFetch() // Takes 100ms
  return <div>{data}</div>
}

export default function Page() {
  return (
    <div>
      {/* Shows immediately */}
      <h1>Dashboard</h1>

      {/* Shows after 100ms */}
      <Suspense fallback={<p>Loading fast...</p>}>
        <FastComponent />
      </Suspense>

      {/* Shows after 5s */}
      <Suspense fallback={<p>Loading slow...</p>}>
        <SlowComponent />
      </Suspense>
    </div>
  )
}
```

### Nested Suspense

```tsx
export default function Page() {
  return (
    <Suspense fallback={<AppSkeleton />}>
      <MainContent />
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
        <Suspense fallback={<AdsSkeleton />}>
          <Ads />
        </Suspense>
      </Suspense>
    </Suspense>
  )
}
```

## Third-Party Components

Many third-party components use client-only features. Wrap them:

```tsx
// components/chart-wrapper.tsx
'use client'

import { Chart } from 'third-party-chart-library'

export function ChartWrapper(props) {
  return <Chart {...props} />
}
```

```tsx
// Use in Server Component
import { ChartWrapper } from './chart-wrapper'

export default async function Dashboard() {
  const data = await fetchData()
  return <ChartWrapper data={data} />
}
```

## Context in Server Components

Server Components can't use `useContext`. Pass data through:

### Props

```tsx
async function Layout({ children }) {
  const user = await getUser()

  return (
    <div>
      <Header user={user} />
      {children}
    </div>
  )
}
```

### Client Context Provider

```tsx
// Wrap in Client Component
'use client'

import { createContext, use } from 'react'

export const UserContext = createContext(null)

export function UserProvider({ user, children }) {
  return (
    <UserContext value={user}>
      {children}
    </UserContext>
  )
}
```

```tsx
// Use in Server Component
import { UserProvider } from './user-context'

export default async function Layout({ children }) {
  const user = await getUser()

  return (
    <UserProvider user={user}>
      {children}
    </UserProvider>
  )
}
```

## Best Practices

1. **Keep Client Components at the leaves** - Push them down the tree
2. **Fetch data in Server Components** - Pass to Client via props
3. **Don't add 'use client' to Server Components** - Only when needed
4. **Use Suspense for loading states** - Progressive rendering
5. **Pass children, not props** - For flexibility with Server Components
6. **Keep serializable data** - Objects, arrays, primitives only

## Common Mistakes

### Wrong: Importing Server Component in Client

```tsx
'use client'

// This turns ServerComponent into a Client Component!
import { ServerComponent } from './server-component'
```

### Right: Pass as Children

```tsx
// Parent (Server Component)
import { ClientWrapper } from './client-wrapper'
import { ServerComponent } from './server-component'

export default function Page() {
  return (
    <ClientWrapper>
      <ServerComponent />
    </ClientWrapper>
  )
}
```

### Wrong: Using Client Hooks in Server Component

```tsx
// This will error!
async function Page() {
  const [count, setCount] = useState(0) // Error!
}
```

### Right: Extract to Client Component

```tsx
// Server Component
import { Counter } from './counter'

export default function Page() {
  return <Counter />
}
```

```tsx
// Client Component
'use client'

export function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```
