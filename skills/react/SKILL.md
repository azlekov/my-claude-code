---
name: react
description: This skill should be used when the user asks to "create a React component", "use React hooks", "handle state", "implement forms", "use useOptimistic", "use useActionState", "create Server Components", "add interactivity", or discusses React patterns, component architecture, or state management. Always use the latest React version and modern patterns.
version: 1.0.0
---

# React Development

This skill provides guidance for building applications with React, focusing on **always using the latest version** and modern patterns.

> **Philosophy:** Prefer Server Components by default. Use modern hooks (useActionState, useOptimistic). Leverage the React Compiler for automatic optimization.

## Quick Reference

| Feature | Modern Approach | Legacy (Avoid) |
|---------|----------------|----------------|
| Form State | `useActionState` | `useFormState` (deprecated) |
| Optimistic UI | `useOptimistic` | Manual state management |
| Promises in Render | `use()` hook | useEffect + useState |
| Context | `use(Context)` | `useContext(Context)` |
| Memoization | React Compiler | Manual `useMemo`, `useCallback` |
| Refs | `ref` prop on functions | `forwardRef` wrapper |

## Component Types

### Server Components (Default in App Router)

```tsx
// No directive needed - server by default
async function UserProfile({ userId }: { userId: string }) {
  const user = await db.users.find(userId)

  return (
    <div className="profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

Server Components can:
- Use `async/await`
- Access databases directly
- Read files from filesystem
- Keep secrets server-side

Server Components cannot:
- Use `useState`, `useEffect`
- Add event handlers
- Access browser APIs

### Client Components

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

Use Client Components when you need:
- Interactivity (onClick, onChange)
- Browser APIs (localStorage, window)
- React hooks (useState, useEffect)

## Modern Hooks

### useActionState

Handle form actions with loading and error states:

```tsx
'use client'

import { useActionState } from 'react'

interface FormState {
  error: string | null
  success: boolean
}

async function submitForm(prevState: FormState, formData: FormData): Promise<FormState> {
  const email = formData.get('email') as string

  if (!email.includes('@')) {
    return { error: 'Invalid email', success: false }
  }

  await saveEmail(email)
  return { error: null, success: true }
}

export function EmailForm() {
  const [state, formAction, isPending] = useActionState(submitForm, {
    error: null,
    success: false
  })

  return (
    <form action={formAction}>
      <input
        name="email"
        type="email"
        disabled={isPending}
        placeholder="Enter email"
      />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>

      {state.error && <p className="text-red-500">{state.error}</p>}
      {state.success && <p className="text-green-500">Success!</p>}
    </form>
  )
}
```

### useOptimistic

Provide instant UI feedback while async operations complete:

```tsx
'use client'

import { useOptimistic, useTransition } from 'react'

interface Message {
  id: string
  text: string
  sending?: boolean
}

export function MessageList({
  messages,
  sendMessage
}: {
  messages: Message[]
  sendMessage: (text: string) => Promise<void>
}) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage: Message) => [...state, { ...newMessage, sending: true }]
  )

  const [, startTransition] = useTransition()

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string

    // Instantly show the message
    addOptimisticMessage({ id: `temp-${Date.now()}`, text })

    // Then actually send it
    startTransition(async () => {
      await sendMessage(text)
    })
  }

  return (
    <div>
      <ul>
        {optimisticMessages.map(msg => (
          <li
            key={msg.id}
            className={msg.sending ? 'opacity-50' : ''}
          >
            {msg.text}
            {msg.sending && <span className="ml-2">Sending...</span>}
          </li>
        ))}
      </ul>

      <form action={handleSubmit}>
        <input name="text" placeholder="Type a message" />
        <button type="submit">Send</button>
      </form>
    </div>
  )
}
```

### use() Hook

Read promises and context directly in render:

```tsx
import { use, Suspense } from 'react'

// Reading a promise
function UserName({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise)
  return <h1>{user.name}</h1>
}

export function UserProfile({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId) // Start fetching

  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserName userPromise={userPromise} />
    </Suspense>
  )
}
```

```tsx
// Reading context (replaces useContext)
import { ThemeContext } from './theme'

function ThemedButton() {
  const theme = use(ThemeContext)
  return <button className={theme.buttonClass}>Click me</button>
}
```

## React Compiler

Enable automatic memoization without manual `useMemo`/`useCallback`:

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    reactCompiler: true
  }
}
```

Before (manual memoization):

```tsx
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  const processedData = useMemo(() => expensiveProcess(data), [data])
  const handleClick = useCallback(() => doSomething(data), [data])

  return <div onClick={handleClick}>{processedData}</div>
})
```

After (React Compiler handles it):

```tsx
function ExpensiveComponent({ data }) {
  const processedData = expensiveProcess(data)
  const handleClick = () => doSomething(data)

  return <div onClick={handleClick}>{processedData}</div>
}
```

## Component Patterns

### Composition Over Props

```tsx
// Instead of prop drilling
function Card({ title, subtitle, children, footer }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <p>{subtitle}</p>
      {children}
      <div>{footer}</div>
    </div>
  )
}

// Use composition
function Card({ children }) {
  return <div className="card">{children}</div>
}

Card.Header = function Header({ children }) {
  return <div className="card-header">{children}</div>
}

Card.Body = function Body({ children }) {
  return <div className="card-body">{children}</div>
}

Card.Footer = function Footer({ children }) {
  return <div className="card-footer">{children}</div>
}

// Usage
<Card>
  <Card.Header>
    <h2>Title</h2>
  </Card.Header>
  <Card.Body>Content here</Card.Body>
  <Card.Footer>
    <button>Action</button>
  </Card.Footer>
</Card>
```

### Render Props

```tsx
interface MousePosition {
  x: number
  y: number
}

function MouseTracker({
  children
}: {
  children: (position: MousePosition) => React.ReactNode
}) {
  const [position, setPosition] = useState({ x: 0, y: 0 })

  useEffect(() => {
    const handleMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY })
    }
    window.addEventListener('mousemove', handleMove)
    return () => window.removeEventListener('mousemove', handleMove)
  }, [])

  return <>{children(position)}</>
}

// Usage
<MouseTracker>
  {({ x, y }) => <div>Mouse: {x}, {y}</div>}
</MouseTracker>
```

### Server/Client Boundary

```tsx
// Server Component (fetches data)
async function Dashboard() {
  const stats = await fetchStats()
  const activities = await fetchActivities()

  return (
    <div>
      <StatsCards stats={stats} />           {/* Server */}
      <InteractiveChart data={stats} />      {/* Client */}
      <ActivityFeed activities={activities} /> {/* Server */}
      <FilterPanel />                         {/* Client */}
    </div>
  )
}

// Client Component (interactive)
'use client'
function InteractiveChart({ data }) {
  const [range, setRange] = useState('7d')
  // Chart logic...
}
```

## Refs in React 19

### Direct Ref Prop (No More forwardRef)

```tsx
// React 19: Direct ref prop
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />
}

// Usage
function Form() {
  const inputRef = useRef<HTMLInputElement>(null)

  return <Input ref={inputRef} placeholder="Name" />
}
```

### Cleanup Functions

```tsx
function Component() {
  const ref = useCallback((node: HTMLDivElement | null) => {
    if (node) {
      // Setup
      const observer = new IntersectionObserver(...)
      observer.observe(node)

      // Cleanup (new in React 19)
      return () => observer.disconnect()
    }
  }, [])

  return <div ref={ref}>Observed</div>
}
```

## Context

### Creating Context

```tsx
import { createContext, use } from 'react'

interface User {
  id: string
  name: string
}

const UserContext = createContext<User | null>(null)

function UserProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null)

  return (
    <UserContext value={user}>
      {children}
    </UserContext>
  )
}
```

### Consuming Context (Modern)

```tsx
function UserProfile() {
  const user = use(UserContext)

  if (!user) return <div>Please log in</div>

  return <div>Hello, {user.name}</div>
}
```

## Error Boundaries

```tsx
'use client'

import { Component, type ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback: ReactNode
}

interface State {
  hasError: boolean
}

class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback
    }
    return this.props.children
  }
}

// Usage
<ErrorBoundary fallback={<div>Something went wrong</div>}>
  <RiskyComponent />
</ErrorBoundary>
```

## Additional Resources

For detailed patterns, see reference files:
- **`references/hooks.md`** - Complete hooks reference
- **`references/server-components.md`** - RSC patterns and best practices
