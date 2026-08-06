# Forms

Sources: [`/guides/forms`](https://nextjs.org/docs/app/guides/forms) ·
[`/getting-started/mutating-data`](https://nextjs.org/docs/app/getting-started/mutating-data)
Next.js `16.3.0`, read `2026-08-06`.

## Start from the platform

A `<form>` posting to a Server Action works before hydration, without JavaScript, and needs no
client state. Use `useActionState` for the pending/result plumbing and `useFormStatus` for
nested submit buttons.

**Prefer this to a hand-rolled `onSubmit` + `fetch`.** The bespoke version re-implements pending
state, error handling and reset, and loses progressive enhancement — usually without anyone
deciding to give that up.

## Validate twice, and mean it

- **Client-side** for feedback: fast, and improves the experience.
- **Server-side for correctness**: the only one that counts. Client validation is a courtesy
  and is trivially bypassed — the action is a public endpoint.

**Share the schema** between both so they cannot disagree. Two hand-written validations drift,
and the server one is the one nobody re-reads.

## Return field-level errors

Return a shape the UI can attach to inputs:

```ts
{ ok: false, kind: 'invalid', errors: { email: 'Already registered.' } }
```

A single banner for a five-field form makes the user hunt. **Name the field.** And render the
server's message verbatim rather than mapping it to your own copy — the server may have worded
it deliberately (some wording is vague on purpose so it cannot be used to enumerate accounts).

## Practices

- **Disable the submit button while pending**, and make double submission harmless anyway. See
  the idempotency note in [`server-actions.md`](server-actions.md) — the button is UX, not a
  guard.
- **Preserve what the user typed on a failed submit.** Losing a long form to a validation error
  is the most annoying possible failure.
- **Redirect after a successful mutation that changes location**, so back-navigation does not
  resubmit.
- **Do not put a secret in a hidden field.** It is in the page. Derive it on the server.
- **`revalidatePath`/`revalidateTag` after the write**, or the list the user returns to is
  stale. See [`revalidation.md`](revalidation.md).
- **Label every input and associate errors with `aria-describedby`.** For the fuller UI review,
  use the official `web-design-guidelines` skill.
