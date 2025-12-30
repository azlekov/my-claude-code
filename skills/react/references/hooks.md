# React Hooks Reference

## State Hooks

### useState

```tsx
const [state, setState] = useState(initialValue)

// With lazy initialization
const [state, setState] = useState(() => expensiveComputation())

// Updater function
setState(prev => prev + 1)

// Reset to initial
setState(initialValue)
```

### useReducer

```tsx
interface State {
  count: number
  error: string | null
}

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setError'; error: string }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 }
    case 'decrement':
      return { ...state, count: state.count - 1 }
    case 'setError':
      return { ...state, error: action.error }
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, error: null })

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </div>
  )
}
```

## Effect Hooks

### useEffect

```tsx
// Run on every render
useEffect(() => {
  console.log('Rendered')
})

// Run on mount only
useEffect(() => {
  console.log('Mounted')
}, [])

// Run when dependencies change
useEffect(() => {
  console.log('Value changed:', value)
}, [value])

// With cleanup
useEffect(() => {
  const handler = () => console.log('scroll')
  window.addEventListener('scroll', handler)

  return () => {
    window.removeEventListener('scroll', handler)
  }
}, [])
```

### useLayoutEffect

Runs synchronously after DOM mutations, before paint:

```tsx
useLayoutEffect(() => {
  // Measure DOM elements
  const { height } = ref.current.getBoundingClientRect()
  setHeight(height)
}, [])
```

### useInsertionEffect

For CSS-in-JS libraries to inject styles:

```tsx
useInsertionEffect(() => {
  const style = document.createElement('style')
  style.textContent = '.my-class { color: red; }'
  document.head.appendChild(style)

  return () => style.remove()
}, [])
```

## Ref Hooks

### useRef

```tsx
// DOM reference
function TextInput() {
  const inputRef = useRef<HTMLInputElement>(null)

  function focus() {
    inputRef.current?.focus()
  }

  return <input ref={inputRef} />
}

// Mutable value (doesn't trigger re-render)
function Timer() {
  const intervalRef = useRef<number>()

  useEffect(() => {
    intervalRef.current = window.setInterval(() => {
      console.log('tick')
    }, 1000)

    return () => clearInterval(intervalRef.current)
  }, [])
}
```

### useImperativeHandle

Customize the ref exposed to parent:

```tsx
interface InputHandle {
  focus: () => void
  clear: () => void
}

function FancyInput({ ref }: { ref: React.Ref<InputHandle> }) {
  const inputRef = useRef<HTMLInputElement>(null)

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current?.focus()
    },
    clear() {
      if (inputRef.current) {
        inputRef.current.value = ''
      }
    }
  }))

  return <input ref={inputRef} />
}
```

## Context Hooks

### use (Modern - React 19)

```tsx
import { use } from 'react'

function Component() {
  const theme = use(ThemeContext)
  const user = use(UserContext)

  return <div className={theme.class}>{user.name}</div>
}
```

### useContext (Legacy)

```tsx
import { useContext } from 'react'

function Component() {
  const theme = useContext(ThemeContext)
  return <div className={theme.class}>...</div>
}
```

## Transition Hooks

### useTransition

Mark state updates as non-urgent:

```tsx
function FilterList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('')
  const [isPending, startTransition] = useTransition()

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    // Urgent: update input
    setQuery(e.target.value)

    // Non-urgent: filter list
    startTransition(() => {
      setFilteredItems(items.filter(item =>
        item.name.includes(e.target.value)
      ))
    })
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <span>Filtering...</span>}
      <ItemList items={filteredItems} />
    </div>
  )
}
```

### useDeferredValue

Defer a value during re-renders:

```tsx
function SearchResults({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query)
  const isStale = query !== deferredQuery

  const results = useMemo(
    () => searchItems(deferredQuery),
    [deferredQuery]
  )

  return (
    <div className={isStale ? 'opacity-50' : ''}>
      {results.map(item => <Result key={item.id} item={item} />)}
    </div>
  )
}
```

## Form Hooks (React 19)

### useActionState

```tsx
interface State {
  error: string | null
  data: Data | null
}

async function action(prevState: State, formData: FormData): Promise<State> {
  try {
    const data = await submitForm(formData)
    return { error: null, data }
  } catch (e) {
    return { error: 'Failed to submit', data: null }
  }
}

function Form() {
  const [state, formAction, isPending] = useActionState(action, {
    error: null,
    data: null
  })

  return (
    <form action={formAction}>
      <input name="email" disabled={isPending} />
      <button disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit'}
      </button>
      {state.error && <p>{state.error}</p>}
    </form>
  )
}
```

### useFormStatus

Get status of parent form:

```tsx
import { useFormStatus } from 'react-dom'

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus()

  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  )
}
```

### useOptimistic

```tsx
const [optimisticState, addOptimistic] = useOptimistic(
  actualState,
  (currentState, optimisticValue) => {
    // Return new state with optimistic update
    return [...currentState, { ...optimisticValue, pending: true }]
  }
)
```

## Performance Hooks

### useMemo

Cache expensive calculations:

```tsx
const sortedItems = useMemo(
  () => items.sort((a, b) => a.name.localeCompare(b.name)),
  [items]
)
```

### useCallback

Cache function references:

```tsx
const handleClick = useCallback(
  (id: string) => {
    setItems(items.filter(item => item.id !== id))
  },
  [items]
)
```

> **Note:** With React Compiler enabled, these are often unnecessary.

## Other Hooks

### useId

Generate unique IDs for accessibility:

```tsx
function FormField({ label }: { label: string }) {
  const id = useId()

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </div>
  )
}
```

### useSyncExternalStore

Subscribe to external stores:

```tsx
function useWindowWidth() {
  return useSyncExternalStore(
    // Subscribe
    (callback) => {
      window.addEventListener('resize', callback)
      return () => window.removeEventListener('resize', callback)
    },
    // Get snapshot (client)
    () => window.innerWidth,
    // Get server snapshot
    () => 1024
  )
}
```

### useDebugValue

Add labels to custom hooks in DevTools:

```tsx
function useOnlineStatus() {
  const isOnline = useSyncExternalStore(...)

  useDebugValue(isOnline ? 'Online' : 'Offline')

  return isOnline
}
```

## Custom Hooks

### Pattern

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue

    try {
      const item = localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch {
      return initialValue
    }
  })

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))
  }, [key, value])

  return [value, setValue] as const
}
```

### Example: useDebounce

```tsx
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}
```

### Example: useFetch

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null)
  const [error, setError] = useState<Error | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const controller = new AbortController()

    fetch(url, { signal: controller.signal })
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false))

    return () => controller.abort()
  }, [url])

  return { data, error, loading }
}
```
