# Server Actions — every export is a public endpoint

Sources: [`/guides/server-actions`](https://nextjs.org/docs/app/guides/server-actions) ·
[`/getting-started/mutating-data`](https://nextjs.org/docs/app/getting-started/mutating-data) ·
[`/guides/data-security`](https://nextjs.org/docs/app/guides/data-security) ·
[`/guides/forms`](https://nextjs.org/docs/app/guides/forms)
Next.js `16.3.0`, read `2026-08-06`.

## The one fact that governs everything else

**Treat every export of a `'use server'` module as a callable HTTP endpoint.** Anyone who can
reach your app can invoke a mounted action with arguments of their choosing, in any order, at any
time — regardless of which component imported it or whether the UI that calls it is rendered.

*Precision added 2026-08-06:* the mechanism is not quite absolute — the docs note that "unused
Server Functions are stripped from client bundles so they have no public endpoint." That does not
weaken the practice, and the docs' own posture is the same as this rule's: **"Treat every action as
an untrusted entry point."** Rely on the posture, not on tree-shaking, because whether an action is
"unused" changes with every UI edit.

Everything below follows from that.

## Consequences to design for

**1. Authorize inside the action, every time.**

```ts
'use server'
export async function deleteProject(input: { projectId: string }) {
  const user = await getCurrentUser()          // from the SESSION
  if (!user) return { ok: false, kind: 'unauthenticated' }
  await assertCanDelete(user, input.projectId) // authorize the OBJECT, not just the verb
  // …
}
```

The two mistakes this prevents: no check at all (the UI hid the button, so it felt covered),
and a permission check without an object check (`can('projects.delete')` is true, but not for
*that* project).

**2. Never accept the CALLER's identity as an argument.** A `userId` parameter meaning "who I am"
is an invitation to pass somebody else's — derive that from the session. This does **not** forbid
object identifiers: `{ organizationId }`, `{ targetUserId }` on an admin action are legitimate and
necessary. The rule is that you authorize them, which is what point 1 is for. (Clarified
2026-08-06; the earlier phrasing read as banning both.)

**3. Never export the service with its dependency bundle.** If a service takes injected
`deps` — a session resolver, a repository, a decision function — publishing it lets a caller
supply their own. Every guard then runs faithfully against attacker-chosen collaborators. Wrap
it: the action forwards the minimum and the service resolves its own dependencies.

**4. Do not export a read as an action *just* to reach it from a Server Component.** Import it
directly instead; exporting publishes "list every X" as an RPC endpoint. But this is not a ban —
client-driven pagination, autocomplete and infinite scroll legitimately need an exported read, and
the docs bless calling server functions from Client Components. The real rule is **authorize and
scope every exported read** exactly as you would a mutation. (Narrowed 2026-08-06; the earlier
wording forbade a supported pattern.)

**5. Validate the payload.** Arguments arrive over the wire and are not what your types claim.
Parse with a schema at the top of the action; TypeScript is erased.

## Return values, not exceptions, for expected refusals

A refusal a user can cause — invalid input, insufficient permission, a stale row — is a normal
outcome. Return a typed result so the UI can render the message. Reserve throwing for genuinely
unexpected states, and remember that a thrown error's message may be sanitised in production
(see [`error-handling.md`](error-handling.md)) so it is a bad channel for anything actionable.

## After the write

- **`revalidatePath` / `revalidateTag`** so the affected surfaces are not stale. See
  [`revalidation.md`](revalidation.md).
- **`redirect()` after a mutation that changes where the user should be.** Note it works by
  throwing, so do not wrap it in a `try` that swallows.
- **If the action invalidated the caller's own session** (sign-out, self-deletion, password
  change that revokes sessions), the caller must redirect, not revalidate — revalidating with a
  dead session lands the user on a signed-out page with no explanation.

## Idempotency

Actions can run twice — a retried submission, a double click, a resubmitted form. **If running
twice would double-charge, double-send, or double-invite, key the effect on something stable**
and make the second run a no-op. This is not a rare edge; it is the normal behaviour of
networks and users.

## Testing

Test the **contract**: what the action returns for each outcome, and that the refusals refuse.
A test proving the happy path passes identically whether or not authorization exists — so the
valuable assertions are the negative ones, plus one that the mutation was *not* called when the
guard refused.
