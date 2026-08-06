# Data security — who can read this, and where is that enforced

Sources: [`/guides/data-security`](https://nextjs.org/docs/app/guides/data-security) ·
[`/guides/authentication`](https://nextjs.org/docs/app/guides/authentication) ·
[`/guides/backend-for-frontend`](https://nextjs.org/docs/app/guides/backend-for-frontend)
Next.js `16.3.0`, read `2026-08-06`.

## Why Server Components changed the threat model

The docs put it plainly: Server Components "shift where and how data is accessed, changing
some of the traditional security assumptions for handling data in frontend apps."

The old assumption was that the frontend had no database access, so a frontend bug could not
leak a column. Now a component can query directly — so **an authorization mistake and a
rendering mistake are now the same kind of mistake.**

## Pick ONE of three architectures — the docs say do not mix

| Approach | For |
|---|---|
| **External HTTP APIs** | Existing large applications and organizations |
| **Data Access Layer (DAL)** | **New projects** |
| **Component-level data access** | Prototypes and learning |

> "We recommend choosing one data fetching approach and avoiding mixing them. This makes it
> clear for both developers working in your code base and security auditors what to expect."

The reason is auditability, not elegance. In a mixed codebase, "is every read authorized?" has
no answer short of reading every component. **If the project has a DAL, a component that
queries directly is a defect even when it returns the right data.**

For an existing app adopting Server Components, follow **Zero Trust**: keep calling your
existing APIs with the caller's credentials, exactly as a Client Component would.

## The Data Access Layer, which is the recommendation for new projects

Per the docs, a DAL should:

- **Only run on the server.**
- **Perform authorization checks.**
- **Return safe, minimal Data Transfer Objects (DTOs).**

```ts
// data/auth.ts — the documented shape
import { cache } from 'react'
import { cookies } from 'next/headers'

export const getCurrentUser = cache(async () => {
  const cookieStore = await cookies()
  const token = cookieStore.get('AUTH_TOKEN')
  const decodedToken = await decryptAndValidate(token)
  // Do NOT include secret tokens or private information as public fields.
  // Use classes to avoid accidentally exposing fields to the client.
  return new User(decodedToken.id)
})
```

Two details in that snippet carry most of the value:

- **`cache()` from React**, so the same value is available in many places without threading it
  through props. The docs give the security reason, not a performance one: it *"discourages
  passing it from Server Component to Server Component which minimizes risk of passing it to a
  Client Component."* Deduping is a bonus; not passing secrets around is the point.
- **Authorization lives *inside* the DAL**, next to the query. A check in the page that renders
  the data is a check the next caller forgets.

## DTOs — return the minimum, not the row

A whole database row passed to a Client Component is shipped to the browser and readable:
`password_hash`, `stripe_customer_id`, internal flags, the other tenant's id. Serialising a row
because it is convenient is the single most common leak in this architecture, and it is
invisible in the rendered output.

- **Map explicitly** — `{ id, name }`, not `...user`.
- **Never spread a row across the boundary.** `{...user}` today is a leak the first time
  somebody adds a column.
- **Prefer a class over a bare object** for the user model, as the docs suggest, so fields are
  not accidentally enumerable across the boundary.

## Where authorization goes

| Layer | Role |
|---|---|
| **Proxy (`proxy.ts`)** | *Optimistic* checks only — cheap redirects. Explicitly **not** a full authorization solution |
| **DAL / data functions** | **The enforcement point.** Every read and write authorizes here |
| **Server Actions** | Re-authorize from the session; the action is a public endpoint |
| **UI** | Presentation. Hiding a control is not a control |

**Enforce at the point of data access**, because that is the only layer every path goes
through. A check in a layout is bypassed by a Server Action; a check in an action is bypassed
by a Route Handler; a check in the DAL is bypassed by nothing.

## Practices

- **Re-derive identity from the session inside every action and data function.** Never trust a
  `userId` argument — see [`server-actions.md`](server-actions.md).
- **Keep server-only modules server-only.** Use the `server-only` package so an accidental
  client import fails at build rather than shipping your query layer to the browser.
- **Environment variables without a public prefix stay on the server** — but a value read into
  a prop crosses anyway. The prefix protects the variable, not your use of it.
- **Test the refusal, not just the success.** A test that asserts the right user sees their data
  passes identically whether or not the wrong user is blocked.
