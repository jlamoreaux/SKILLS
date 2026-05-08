# Code Quality Examples

## Names reveal intent

❌ Bad — what is `d`? What does `proc` do?
```ts
const proc = async (d: any) => {
  const r = await fetch(d.url)
  return r.json()
}
```

✅ Good — readable at a glance, typed, handles failure
```ts
type UserProfile = { id: string; name: string; email: string }

const fetchUserProfile = async (userId: string): Promise<UserProfile> => {
  const response = await fetch(`/api/users/${userId}`)
  if (!response.ok) {
    throw new Error(`Failed to fetch user profile for ${userId}: ${response.status}`)
  }
  return response.json() as Promise<UserProfile>
}
```

---

## No swallowed errors

❌ Bad — error silently disappears
```ts
try {
  await saveRecord(data)
} catch (e) {}
```

❌ Also bad — logged but execution continues as if nothing happened
```ts
try {
  await saveRecord(data)
} catch (e) {
  console.error(e)
}
```

✅ Good — propagate with context, or handle deliberately
```ts
try {
  await saveRecord(data)
} catch (error) {
  throw new Error(
    `Failed to save record ${data.id}: ${error instanceof Error ? error.message : String(error)}`
  )
}
```

---

## No floating promises

❌ Bad — if `sendWelcomeEmail` throws, it's an unhandled rejection
```ts
function onUserCreated(user: User): void {
  sendWelcomeEmail(user.email) // floating promise
  redirect('/dashboard')
}
```

✅ Good — await in an async function, or explicitly handle
```ts
async function onUserCreated(user: User): Promise<void> {
  await sendWelcomeEmail(user.email)
  redirect('/dashboard')
}
```

---

## No magic values

❌ Bad — what does 3 mean? Why 'admin'?
```ts
if (retries > 3) throw new Error('Too many retries')
if (user.role === 'admin') allowAccess()
```

✅ Good — constants are self-documenting
```ts
const MAX_RETRIES = 3
const ADMIN_ROLE = 'admin' as const

if (retries > MAX_RETRIES) throw new Error(`Exceeded ${MAX_RETRIES} retry attempts`)
if (user.role === ADMIN_ROLE) allowAccess()
```

---

## Small, focused functions

❌ Bad — one function handling form parsing, validation, API call, and UI update
```ts
async function handleSubmit(e: Event): Promise<void> {
  e.preventDefault()
  const form = e.target as HTMLFormElement
  const name = (form.querySelector('#name') as HTMLInputElement).value
  const email = (form.querySelector('#email') as HTMLInputElement).value
  if (!name || !email || !email.includes('@')) {
    document.getElementById('error')!.textContent = 'Invalid input'
    return
  }
  const res = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name, email }),
  })
  if (!res.ok) {
    document.getElementById('error')!.textContent = 'Server error'
    return
  }
  window.location.href = '/dashboard'
}
```

✅ Good — each concern is named and testable independently
```ts
function parseUserForm(form: HTMLFormElement): { name: string; email: string } {
  return {
    name: (form.querySelector('#name') as HTMLInputElement).value,
    email: (form.querySelector('#email') as HTMLInputElement).value,
  }
}

function validateUserInput(input: { name: string; email: string }): string | null {
  if (!input.name) return 'Name is required'
  if (!input.email.includes('@')) return 'Valid email is required'
  return null
}

async function createUser(data: { name: string; email: string }): Promise<void> {
  const res = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  })
  if (!res.ok) throw new Error(`Create user failed: ${res.status}`)
}

async function handleSubmit(e: SubmitEvent): Promise<void> {
  e.preventDefault()
  const data = parseUserForm(e.target as HTMLFormElement)
  const validationError = validateUserInput(data)
  if (validationError) {
    showError(validationError)
    return
  }
  await createUser(data)
  window.location.href = '/dashboard'
}
```

---

## TypeScript: no `any`, use `unknown`

❌ Bad — `any` disables all type checking
```ts
function parseConfig(raw: any) {
  return raw.database.host // no safety net
}
```

✅ Good — `unknown` forces validation before use
```ts
function parseConfig(raw: unknown): { database: { host: string } } {
  if (
    typeof raw !== 'object' ||
    raw === null ||
    !('database' in raw) ||
    typeof (raw as { database: unknown }).database !== 'object'
  ) {
    throw new Error('Invalid config shape')
  }
  return raw as { database: { host: string } }
}
```

For complex shapes, prefer a schema library:
```ts
import { z } from 'zod'

const ConfigSchema = z.object({ database: z.object({ host: z.string() }) })

function parseConfig(raw: unknown) {
  return ConfigSchema.parse(raw) // throws with a clear message if invalid
}
```

---

## Comments: WHY not WHAT

❌ Bad — the code already says this
```ts
// Loop through users and send email
for (const user of users) {
  await sendEmail(user)
}
```

✅ Good — explains a non-obvious constraint
```ts
// Sequential: email provider rate-limits to 1 req/s per account
for (const user of users) {
  await sendEmail(user)
  await delay(1000)
}
```
