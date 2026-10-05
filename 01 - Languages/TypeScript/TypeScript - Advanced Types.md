---
title: TypeScript - Advanced Types
aliases: [Conditional Types, Mapped Types, Template Literal Types, Generics, Type-level programming]
type: deep-dive
domain: languages
tags: [domain/languages, type/deep-dive, topic/typescript, lang/typescript]
status: draft
created: 2026-10-05
updated: 2026-10-05
version_checked: "TypeScript 7.0 — 2026-10"
parent: "[[TypeScript]]"
related: ["[[JavaScript]]", "[[React]]", "[[Next.js]]", "[[Supabase]]"]
---

# TypeScript - Advanced Types

> [!info] Deep dive of [[TypeScript]]

> [!abstract] TL;DR
> TypeScript's type system is a small functional language: **generics** are parameters, **conditional types** are `if`, **`infer`** is pattern matching, **mapped types** are `map` over keys, **template literal types** are string manipulation, and recursion gives you loops. Use these to make invalid states unrepresentable (discriminated unions, branded IDs) and to derive types from a single source of truth (schemas, DB types, route maps). Remember: **every clever type has a compile-time and readability cost**. Stop when a simpler type gives 95% of the safety.

## Concept
| Tool | Syntax | Mental model |
|---|---|---|
| Generic | `function f<T>(x: T): T` | Type parameter |
| Constraint | `<T extends { id: string }>` | Parameter bound |
| Indexed access | `T["user"]`, `T[number]` | Property lookup on types |
| `keyof` / `typeof` | `keyof User`, `typeof config` | Keys union / type of a value |
| Conditional | `T extends U ? X : Y` | Type-level `if` |
| `infer` | `T extends Promise<infer R> ? R : T` | Pattern-match + capture |
| Distributive conditional | Over naked `T` in unions | `map` over union members |
| Mapped | `{ [K in keyof T]: … }` | `map` over keys, with `as` for renaming |
| Template literal | `` `${A}-${B}` `` | String concatenation on types |
| Recursive types | `type Json = … \| Json[]` | Loops (depth-limited) |
| `satisfies` | `x satisfies T` | Check without widening |
| `const` type params | `<const T>` | Infer literal types |

## How It Works
- **Structural assignability**: `A` is assignable to `B` if A has at least B's members with compatible types. Function parameters are checked contravariantly under `strictFunctionTypes` (method syntax stays bivariant).
- **Distribution**: `T extends U ? X : Y` with `T = A | B` evaluates to `(A extends U ? X : Y) | (B extends U ? X : Y)`. Wrap it in brackets (`[T] extends [U]`) to stop distribution.
- **`never`** is the empty union. Conditional types return `never` to filter union members (`Exclude`).
- **Inference sites**: TS infers type arguments from call arguments, left to right. `NoInfer<T>` (5.4) blocks a site from contributing.
- **Limits**: instantiation depth (~50 nested, 1000 for tail-recursive conditional types), union size caps (100k), and "Type instantiation is excessively deep" errors. TS 7's native compiler is faster but keeps the same semantics.

## Practical Usage

### Discriminated unions: make invalid states unrepresentable
```ts
type Payment =
  | { status: "pending"; createdAt: Date }
  | { status: "paid"; paidAt: Date; receiptUrl: string }
  | { status: "failed"; error: string };

function label(p: Payment) {
  switch (p.status) {
    case "paid": return `Paid ${p.paidAt.toISOString()}`;   // receiptUrl available here only
    case "failed": return `Failed: ${p.error}`;
    case "pending": return "Pending";
    default: return assertNever(p);
  }
}
const assertNever = (x: never): never => { throw new Error(`Unhandled: ${JSON.stringify(x)}`); };
```

### Branded (nominal) IDs
```ts
type Brand<T, B extends string> = T & { readonly __brand: B };
type TenantId = Brand<string, "TenantId">;
type OrderId = Brand<string, "OrderId">;
const asOrderId = (s: string) => s as OrderId;            // only at validated boundaries
function getOrder(tenant: TenantId, id: OrderId) { /* … */ }
// getOrder(orderId, tenantId) → compile error (args swapped)
```

### Conditional types + infer
```ts
type Awaited2<T> = T extends PromiseLike<infer U> ? Awaited2<U> : T;
type ElementOf<T> = T extends readonly (infer E)[] ? E : never;
type FnReturn<F> = F extends (...a: any[]) => infer R ? R : never;
type ActionPayload<A, K> = A extends { type: K; payload: infer P } ? P : never;
```

### Mapped types with key remapping
```ts
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type Nullable<T> = { [K in keyof T]: T[K] | null };
type PickByValue<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
```

### Template literal types
```ts
type Locale = "en" | "ms" | "zh";
type EventName = `order.${"created" | "paid" | "refunded"}`;
type Route = "/orders/:id" | "/customers/:cid/orders/:oid";
type Params<R extends string> =
  R extends `${string}:${infer P}/${infer Rest}` ? { [K in P]: string } & Params<`/${Rest}`>
  : R extends `${string}:${infer P}` ? { [K in P]: string } : {};
type P = Params<"/customers/:cid/orders/:oid">;   // { cid: string } & { oid: string }
```

### Derive types from a single source of truth
```ts
// Runtime schema → static type (Zod)
const Order = z.object({ id: z.string().uuid(), total: z.number().int(), status: z.enum(["pending", "paid"]) });
type Order = z.infer<typeof Order>;

// Const object → union
const ROLES = ["owner", "admin", "staff"] as const;
type Role = (typeof ROLES)[number];                 // "owner" | "admin" | "staff"

// Supabase generated DB types
type OrderRow = Database["public"]["Tables"]["orders"]["Row"];
```

### `satisfies` + `const` type parameters
```ts
const config = { retries: 3, region: "ap-southeast-1" } satisfies Partial<Settings>;  // keeps literal types
function tuple<const T extends readonly unknown[]>(...xs: T) { return xs; }          // tuple(1, "a") → readonly [1, "a"]
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Discriminated unions + exhaustive `never` check | State machines, API responses | Optional-field soup (`{ paidAt?: Date; error?: string }`) |
| Schema-derived types (Zod/Valibot) | External input | Hand-written interfaces that drift from validation |
| Branded IDs | Many string IDs of different kinds | Plain `string` everywhere |
| `satisfies` for config/maps | Keep literals + check shape | `as Type` assertions that hide errors |
| Small utility types, named and documented | Reuse across codebase | 40-line conditional types nobody can debug |
| `unknown` + narrowing at boundaries | Parsing JSON, catch blocks | `any` (spreads silently) |

## Performance & Trade-offs
- Deeply recursive conditional types and big unions slow type-checking and editor responsiveness. Profile with `tsc --extendedDiagnostics` / `--generateTrace`.
- Interfaces (`extends`) are cached better than large intersections (`&`). Prefer interfaces for object shapes that get extended.
- Annotate exported function return types. It reduces inference work and is required by `isolatedDeclarations`.
- Library authors: complex public types make error messages unreadable for consumers. Expose simple surface types.

## Tips & Reminders
> [!tip]
> - Debug a type by hovering it or using `type _ = Expand<T>` (`type Expand<T> = T extends infer O ? { [K in keyof O]: O[K] } : never`).
> - Test types with `expectTypeOf` (Vitest) or `// @ts-expect-error` lines in a `*.test-d.ts` file.
> - Prefer `as const` objects over `enum` (works with `erasableSyntaxOnly`/Node type stripping).
> - **In ZP's stack**: derive everything from **Supabase generated types + Zod schemas**. Share them between Next.js server actions, n8n webhook contracts (validate payloads) and API clients.

## Version Notes
| Version | Change |
|---|---|
| 2.8 | Conditional types, `infer` |
| 4.1 | Template literal types, key remapping in mapped types |
| 4.9 | `satisfies` |
| 5.0 | `const` type parameters |
| 5.4 | `NoInfer<T>` |
| 5.5 | Inferred type predicates, `isolatedDeclarations` |
| 6.0 / 7.0 | `strict` on by default. 7.0 native compiler: same type semantics, much faster checking |

## Critical Issues & Gotchas
> [!danger] Types are erased: advanced types don't validate data
> A perfect type for `Order` means nothing if `JSON.parse` output is cast with `as Order`. Validate at runtime at every boundary (HTTP, webhooks, LLM output, `localStorage`).

> [!warning] Gotchas
> - Distributive surprises: `type IsString<T> = T extends string ? true : false` gives `boolean` for `string | number`. Use `[T] extends [string]`.
> - `keyof` on a union gives only the **common** keys.
> - Excess property checks only apply to fresh object literals.
> - Optional vs `undefined`: `{ a?: string }` ≠ `{ a: string | undefined }` under `exactOptionalPropertyTypes`.
> - Index signatures return `T` not `T | undefined` without `noUncheckedIndexedAccess`.

## Related
- [[TypeScript]]
- [[JavaScript]] — runtime semantics
- [[React]] — typed props, generics in components
- [[Next.js]] — typed routes, `PageProps`
- [[Supabase]] — generated database types

## References
- Handbook, Type Manipulation: https://www.typescriptlang.org/docs/handbook/2/types-from-types.html
- Conditional types: https://www.typescriptlang.org/docs/handbook/2/conditional-types.html
- Type challenges: https://github.com/type-challenges/type-challenges
- Total TypeScript (Matt Pocock): https://www.totaltypescript.com/
