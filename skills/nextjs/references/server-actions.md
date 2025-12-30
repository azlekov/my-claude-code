# Next.js Server Actions Reference

## Basic Server Actions

### Inline Actions

```tsx
// In a Server Component
export default function Page() {
  async function handleSubmit(formData: FormData) {
    'use server'
    const name = formData.get('name')
    await db.users.create({ data: { name } })
  }

  return (
    <form action={handleSubmit}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  )
}
```

### Separate Action Files

```tsx
// app/actions.ts
'use server'

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string
  const email = formData.get('email') as string

  await db.users.create({
    data: { name, email }
  })
}

export async function deleteUser(id: string) {
  await db.users.delete({ where: { id } })
}
```

## Form Handling

### With Validation

```tsx
'use server'

import { z } from 'zod'
import { revalidatePath } from 'next/cache'

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export async function register(formData: FormData) {
  const rawData = {
    email: formData.get('email'),
    password: formData.get('password'),
  }

  const result = schema.safeParse(rawData)

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors
    }
  }

  await db.users.create({ data: result.data })
  revalidatePath('/users')

  return { success: true }
}
```

### With useActionState (React 19)

```tsx
'use client'

import { useActionState } from 'react'
import { register } from './actions'

const initialState = {
  success: false,
  errors: null as Record<string, string[]> | null
}

export function RegisterForm() {
  const [state, formAction, isPending] = useActionState(register, initialState)

  return (
    <form action={formAction}>
      <div>
        <input name="email" type="email" disabled={isPending} />
        {state.errors?.email && (
          <span className="text-red-500">{state.errors.email[0]}</span>
        )}
      </div>

      <div>
        <input name="password" type="password" disabled={isPending} />
        {state.errors?.password && (
          <span className="text-red-500">{state.errors.password[0]}</span>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Registering...' : 'Register'}
      </button>

      {state.success && (
        <p className="text-green-500">Registration successful!</p>
      )}
    </form>
  )
}
```

## Optimistic Updates

### With useOptimistic

```tsx
'use client'

import { useOptimistic } from 'react'
import { addTodo } from './actions'

interface Todo {
  id: string
  text: string
  pending?: boolean
}

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo: Todo) => [...state, { ...newTodo, pending: true }]
  )

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string

    // Optimistically add the todo
    addOptimisticTodo({
      id: `temp-${Date.now()}`,
      text,
    })

    // Actually create it
    await addTodo(formData)
  }

  return (
    <div>
      <form action={handleSubmit}>
        <input name="text" placeholder="New todo" />
        <button type="submit">Add</button>
      </form>

      <ul>
        {optimisticTodos.map(todo => (
          <li
            key={todo.id}
            className={todo.pending ? 'opacity-50' : ''}
          >
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## Redirects and Navigation

```tsx
'use server'

import { redirect } from 'next/navigation'
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  const post = await db.posts.create({
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string,
    }
  })

  revalidatePath('/posts')
  redirect(`/posts/${post.id}`)
}
```

## Error Handling

### Returning Errors

```tsx
'use server'

export async function updateProfile(formData: FormData) {
  try {
    await db.profiles.update({
      where: { userId: getCurrentUserId() },
      data: Object.fromEntries(formData)
    })

    return { success: true }
  } catch (error) {
    // Log server-side
    console.error('Update failed:', error)

    // Return user-friendly error
    return {
      success: false,
      error: 'Failed to update profile. Please try again.'
    }
  }
}
```

### With Error Boundaries

```tsx
'use server'

export async function riskyAction() {
  const result = await externalApi.call()

  if (!result.ok) {
    // This will be caught by error.tsx
    throw new Error('External API failed')
  }

  return result.data
}
```

## File Uploads

```tsx
'use server'

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File

  if (!file || file.size === 0) {
    return { error: 'No file provided' }
  }

  // Validate file type
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp']
  if (!allowedTypes.includes(file.type)) {
    return { error: 'Invalid file type' }
  }

  // Validate file size (5MB)
  if (file.size > 5 * 1024 * 1024) {
    return { error: 'File too large' }
  }

  // Convert to buffer and upload
  const bytes = await file.arrayBuffer()
  const buffer = Buffer.from(bytes)

  // Upload to storage (S3, Cloudinary, etc.)
  const url = await uploadToStorage(buffer, file.name)

  return { success: true, url }
}
```

## Binding Arguments

Pass additional data to actions:

```tsx
'use server'

export async function updateItem(id: string, formData: FormData) {
  await db.items.update({
    where: { id },
    data: { name: formData.get('name') as string }
  })
}
```

```tsx
// In component
import { updateItem } from './actions'

export function ItemForm({ itemId }: { itemId: string }) {
  const updateItemWithId = updateItem.bind(null, itemId)

  return (
    <form action={updateItemWithId}>
      <input name="name" />
      <button type="submit">Update</button>
    </form>
  )
}
```

## Calling Actions Programmatically

```tsx
'use client'

import { deleteItem } from './actions'

export function DeleteButton({ id }: { id: string }) {
  async function handleDelete() {
    const confirmed = confirm('Delete this item?')
    if (confirmed) {
      await deleteItem(id)
    }
  }

  return (
    <button onClick={handleDelete}>
      Delete
    </button>
  )
}
```

## Non-Form Actions

Actions don't have to be form-based:

```tsx
'use server'

export async function incrementCounter() {
  const count = await db.counter.increment()
  revalidatePath('/counter')
  return count
}
```

```tsx
'use client'

import { incrementCounter } from './actions'
import { useTransition } from 'react'

export function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial)
  const [isPending, startTransition] = useTransition()

  async function handleClick() {
    startTransition(async () => {
      const newCount = await incrementCounter()
      setCount(newCount)
    })
  }

  return (
    <button onClick={handleClick} disabled={isPending}>
      Count: {count}
    </button>
  )
}
```

## Best Practices

1. **Always validate input** - Never trust client data
2. **Use Zod for validation** - Type-safe schema validation
3. **Return structured responses** - `{ success: boolean, data?, error? }`
4. **Revalidate after mutations** - Use `revalidatePath` or `revalidateTag`
5. **Handle errors gracefully** - Catch and return user-friendly messages
6. **Use `useActionState`** - For form state and pending UI
7. **Use `useOptimistic`** - For instant feedback
8. **Bind arguments** - Pass IDs and context to actions
