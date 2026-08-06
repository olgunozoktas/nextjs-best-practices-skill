# Error handling — boundaries, and what an error tells an attacker

Sources: [`/getting-started/error-handling`](https://nextjs.org/docs/app/getting-started/error-handling) ·
[`/api-reference/file-conventions/error`](https://nextjs.org/docs/app/api-reference/file-conventions/error) ·
[`/api-reference/functions/not-found`](https://nextjs.org/docs/app/api-reference/functions/not-found)
Next.js `16.3.0`, read `2026-08-06`.

## The conventions, and what each one catches

| File | Catches |
|---|---|
| `error.tsx` | Errors in that segment's subtree. A Client Component, with `reset()` |
| `global-error.tsx` | Errors in the root layout — must render its own `<html>`/`<body>` |
| `not-found.tsx` | `notFound()` calls, and unmatched routes |
| `forbidden` / `unauthorized` (where enabled) | Authorization failures as first-class states |

**`error.tsx` does not catch errors in the layout above it.** That is what `global-error` is
for, and a root layout that can throw with no `global-error` produces a blank page. Add one.

## Distinguish "not found" from "not yours" — deliberately

`notFound()` for a record that does not exist is obvious. The interesting case is a record that
exists and the viewer may not see: returning 404 rather than 403 hides existence, which is
usually what you want for anything id-addressable.

**Pick one and apply it consistently**, because the inconsistency is the leak: 403 on some paths
and 404 on others turns your app into an existence oracle for anyone comparing.

## What an error message may say

Production error messages are sanitised, and that is a feature — a stack trace or a database
message tells an attacker about your schema and your dependencies. Two consequences:

- **Do not put anything a user needs into a thrown error's message.** They will not see it in
  production. Return a typed result instead — see [`server-actions.md`](server-actions.md).
- **Log the detail server-side with a correlation id and show the id.** Then a support
  conversation can find the actual error without exposing it.

## Expected refusals are not errors

A validation failure, an insufficient permission, a stale row — these are outcomes, not
exceptions. Return them as data so the UI can render a specific message next to the specific
field. Throwing turns a precise refusal into a generic error page, which is a worse product and
a worse debugging experience.

Reserve `error.tsx` for the genuinely unexpected.

## Practices

- **Every error boundary should offer a way forward** — `reset()`, or a link somewhere real. A
  boundary that only apologises is a dead end.
- **Never render a real `0` for "not measured".** An empty state and a failed load lead a user to
  different actions; conflating them is the most common honest-UI mistake.
- **`redirect()` throws** — do not swallow it in a `try/catch` around a mutation.
- **Test the boundary.** An error boundary nobody has triggered is a hypothesis.
