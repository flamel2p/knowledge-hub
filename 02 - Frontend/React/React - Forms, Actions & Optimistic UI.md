---
title: React - Forms, Actions & Optimistic UI
aliases: [React forms, form actions, Server Actions forms, useActionState, useOptimistic, useFormStatus, React Hook Form, progressive enhancement]
type: deep-dive
domain: frontend
tags: [domain/frontend, type/deep-dive, topic/react, topic/forms, topic/server-actions, lang/typescript]
status: draft
created: 2026-10-30
updated: 2026-10-30
version_checked: "React 19.3 · Next.js 16 Server Actions (updateTag / revalidatePath) — 2026-10"
parent: "[[React]]"
related: ["[[React - Hooks]]", "[[React - Server Components]]", "[[TypeScript - Runtime Validation with Zod & Type-Safe APIs]]", "[[Next.js - Caching]]", "[[REST API - Webhooks & Idempotency]]", "[[Cloudflare - R2 Storage & Presigned Uploads]]"]
---

# React - Forms, Actions & Optimistic UI

> [!info] Deep dive of [[React]]

> [!abstract] TL;DR
> In React 19 + Next.js, a form is `<form action={serverAction}>`:
> - **`useActionState`** holds the result (field errors, message);
> - **`useFormStatus`** drives the submit button;
> - **`useOptimistic`** shows the change instantly and rolls back automatically if the action fails.
>
> Validate with the **same Zod schema on the server** (the source of truth) and return structured field errors. Revalidate cached data after success, and make actions **idempotent**, because users double-click and networks retry.
> Reach for React Hook Form only for large, highly interactive client forms.

## Concept
| Piece | Runs | Role |
|---|---|---|
| `<form action={fn}>` | Client → server (Server Action) or client function | Submits `FormData`. Works without JS when `fn` is a Server Action (progressive enhancement) |
| Server Action (`'use server'`) | Server | Auth, validation, mutation, revalidation, redirect |
| `useActionState(action, initial)` | Client | `[state, formAction, isPending]`. State = last action result |
| `useFormStatus()` | Client (inside the form) | `{ pending, data, method }` for the submit button/fields |
| `useOptimistic(state, reducer)` | Client | Temporary UI state during the transition, discarded after |
| `requestFormReset` / automatic reset | Client | Uncontrolled forms reset after a successful action |

**Choosing a form approach**:

| Approach | Pick it when… |
|---|---|
| Native form + Server Action + Zod | CRUD forms, settings, checkout steps (default) |
| React Hook Form + Zod resolver (+ action on submit) | Dozens of fields, dynamic arrays, heavy client-side interactivity |
| Controlled inputs everywhere | Rarely: search-as-you-type, live previews |

## How It Works
```mermaid
sequenceDiagram
  participant U as User
  participant F as Form (client)
  participant A as Server Action
  participant DB as Postgres (RLS)
  U->>F: submit
  F->>F: useOptimistic → show pending item; useFormStatus pending=true
  F->>A: FormData (+ idempotency key hidden field)
  A->>A: requireUser → Zod safeParse
  alt invalid
    A-->>F: { ok: false, fieldErrors } → optimistic state discarded
  else valid
    A->>DB: insert/update (tenant-scoped)
    A->>A: updateTag('menu:t42') / revalidatePath
    A-->>F: { ok: true } → fresh server data replaces optimistic item
  end
```

## Practical Usage
**Server Action with structured results**:
```ts
// app/(app)/menu/actions.ts
'use server';
import { z } from 'zod';
import { updateTag } from 'next/cache';

const AddItem = z.strictObject({
  name: z.string().trim().min(2).max(80),
  priceSen: z.coerce.number().int().positive().max(1_000_000),
  idempotencyKey: z.uuid(),
});
export type AddItemState = { ok: boolean; message?: string; fieldErrors?: Record<string, string[]> };

export async function addMenuItem(_prev: AddItemState, fd: FormData): Promise<AddItemState> {
  const { db, tenantId } = await requireTenantUser();
  const parsed = AddItem.safeParse(Object.fromEntries(fd));
  if (!parsed.success) return { ok: false, fieldErrors: z.flattenError(parsed.error).fieldErrors };

  const { error } = await db.from('menu_items')
    .upsert({ tenant_id: tenantId, name: parsed.data.name, price_sen: parsed.data.priceSen,
              idempotency_key: parsed.data.idempotencyKey }, { onConflict: 'idempotency_key', ignoreDuplicates: true });
  if (error) return { ok: false, message: 'Could not save. Please retry.' };

  updateTag(`menu:${tenantId}`);                       // read-your-writes for cached menu data (Next 16)
  return { ok: true, message: 'Saved' };
}
```
**Client form with optimistic list + accessible errors**:
```tsx
'use client';
import { useActionState, useOptimistic, useRef } from 'react';
import { useFormStatus } from 'react-dom';
import { addMenuItem, type AddItemState } from './actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending} aria-busy={pending}>{pending ? 'Saving…' : 'Add item'}</button>;
}

export function MenuEditor({ items }: { items: MenuItem[] }) {
  const [optimistic, addOptimistic] = useOptimistic(items, (list, item: MenuItem) => [...list, item]);
  const keyRef = useRef(crypto.randomUUID());
  const [state, formAction] = useActionState(async (prev: AddItemState, fd: FormData) => {
    addOptimistic({ id: 'tmp', name: String(fd.get('name')), priceSen: Number(fd.get('priceSen')), pending: true });
    const res = await addMenuItem(prev, fd);
    if (res.ok) keyRef.current = crypto.randomUUID();  // new key for the next submission
    return res;
  }, { ok: false });

  const err = state.fieldErrors ?? {};
  return (
    <>
      <ul>{optimistic.map((i) => <li key={i.id} aria-busy={i.pending}>{i.name}</li>)}</ul>
      <form action={formAction} noValidate>
        <input type="hidden" name="idempotencyKey" value={keyRef.current} />
        <label htmlFor="name">Name</label>
        <input id="name" name="name" aria-invalid={!!err.name} aria-describedby="name-err" required />
        {err.name && <p id="name-err" role="alert">{err.name[0]}</p>}
        <label htmlFor="price">Price (sen)</label>
        <input id="price" name="priceSen" inputMode="numeric" aria-invalid={!!err.priceSen} />
        <SubmitButton />
        {state.message && <p role="status">{state.message}</p>}
      </form>
    </>
  );
}
```
**Uploads**: don't stream files through Server Actions. Get a presigned URL in an action, upload directly to R2, then submit the key ([[Cloudflare - R2 Storage & Presigned Uploads]]).

**Wrong way**:
```tsx
const [loading, setLoading] = useState(false); const [error, setError] = useState('');   // ❌ hand-rolled flags per form
async function onSubmit(e) { e.preventDefault(); setLoading(true); await fetch('/api/x', …) }   // ❌ no JS = broken form; no shared validation
export async function deleteItem(id: string) { 'use server'; await db.delete(id); }           // ❌ no auth/tenant check: Server Actions are public endpoints
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| One Zod schema on the server (optionally reused client-side for hints) | All forms | Client-only validation |
| Return `{ ok, fieldErrors, message }`; throw only for unexpected errors | Expected failures | Throwing for validation errors (error boundary instead of inline errors) |
| Idempotency key per submission | Payments, orders, any create | Double-submit creating duplicates |
| `updateTag`/`revalidatePath` after success | Cached reads | Stale lists after save |
| `redirect()` after create flows (PRG) | Create → detail page | Re-submission on browser refresh |
| Focus the first invalid field, `aria-invalid` + `role="alert"` | Accessibility | Red borders only |

## Performance & Trade-offs
- **Round trips**: a Server Action is one POST that can return the re-rendered RSC payload, often fewer requests than a REST call plus a refetch.
- **Optimistic UI** hides latency (noticeable on Malaysian mobile networks), but every optimistic change must be safely reversible. Don't fake payments or stock decrements.
- **Uncontrolled inputs** re-render far less than controlled ones. Keep large forms uncontrolled with `FormData`, and add controlled state only where live UI needs it.
- **React Hook Form** adds client JS (~10 KB) but gives fine-grained field subscriptions and dynamic field arrays. Use it when the form genuinely needs them.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Form stack**: Next.js Server Actions + Zod (`packages/schemas`) + Supabase with RLS for staff dashboards (menus, orders, settings).
> - **Same schemas in Flutter**: export them via JSON Schema so the mobile app validates identically ([[TypeScript - Runtime Validation with Zod & Type-Safe APIs]]).
> - **Money**: integer sen fields, formatted for display only (`Intl.NumberFormat('ms-MY', { style: 'currency', currency: 'MYR' })`).

- Every Server Action is a public POST endpoint. Authenticate, authorize the tenant and validate in **each** action, regardless of UI visibility ([[Next.js - Middleware, Auth & Multi-Tenant Routing]]).
- Use `useFormStatus` only in components rendered **inside** the `<form>`. It reads the nearest parent form.
- Keep error messages user-facing and localized (BM/EN), and log technical details server-side with the request ID.
- Test forms with JS disabled once. Progressive enhancement catches missing `name` attributes and wrong action wiring.

## Version Notes
| Version | Change relevant here |
|---|---|
| React 19.0 (2024-12) | Actions, `useActionState` (replaces `useFormState`), `useOptimistic`, `useFormStatus`, `<form action>`, automatic form reset |
| Next.js 14–15 | Server Actions stable. Async request APIs (15) |
| Next.js 16 (2025-10) | `updateTag` (read-your-writes) / `revalidateTag(tag, profile)` with Cache Components |
| React 19.2–19.3 (2025–26) | Refinements. Actions APIs stable |

## Critical Issues & Gotchas
> [!danger] Server Actions are public endpoints
> Any exported Server Action can be invoked directly with a crafted POST, not only from your UI. Missing auth or tenant checks inside the action is IDOR/BOLA ([[OWASP Top 10 - API Security Top 10 in Practice]]). Treat every action like an API route: authenticate, authorize, validate, rate-limit.

> [!danger] Optimistic UI for irreversible effects
> Showing "Paid ✓" or decrementing stock optimistically, then failing server-side, misleads users and staff. Use optimistic updates only for reversible, low-stakes changes. Show explicit pending states for money and inventory.

- **`useActionState` signature**: the action receives `(previousState, formData)`. Forgetting the first parameter shifts `FormData` into `prevState`.
- **Serialization limits**: action arguments and return values must be serializable (no class instances, functions or Dates without conversion). Return plain objects.
- **Closures capture secrets**: inline Server Actions defined inside Server Components close over variables, and React encrypts those. Still, never close over secrets you wouldn't want sent anywhere.
- **Body size limits**: Next.js caps Server Action payloads (default ~1 MB). Large inputs or files must go elsewhere.

## Related
- [[React]]
- [[React - Hooks]] — the Actions hooks in context
- [[React - Server Components]] — where Server Actions live
- [[TypeScript - Runtime Validation with Zod & Type-Safe APIs]] — shared schemas and field errors
- [[Next.js - Caching]] — `updateTag` / `revalidatePath` after mutations
- [[REST API - Webhooks & Idempotency]] — idempotency keys

## References
- https://react.dev/reference/react/useActionState
- https://react.dev/reference/react/useOptimistic
- https://react.dev/reference/react-dom/hooks/useFormStatus
- https://nextjs.org/docs/app/getting-started/updating-data
- https://react.dev/blog/2024/12/05/react-19
