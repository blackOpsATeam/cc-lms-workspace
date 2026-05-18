# Structure Reference

Full folder trees for each scaling tier, and what each requirement axis adds. Start
from the tier tree, then layer on the additions for every enabled axis.

## Table of contents

1. Minimal tier
2. Standard tier (the default)
3. Large tier
4. Per-axis folder additions
5. Inside a feature folder

---

## 1. Minimal tier

Landing pages, marketing sites, prototypes, small tools. No `features/` layer — it
would be ceremony. Code lives directly in `app/`, `components/`, and `lib/`.

```
my-app/
├── src/
│   ├── app/
│   │   ├── layout.tsx           # root layout + providers
│   │   ├── page.tsx
│   │   ├── globals.css
│   │   └── not-found.tsx
│   ├── components/
│   │   ├── ui/                  # shadcn primitives
│   │   ├── layout/              # Header, Footer
│   │   └── providers/           # ThemeProvider
│   ├── config/
│   │   └── site.ts
│   └── lib/
│       └── utils.ts             # cn()
├── public/
├── .env.example
├── components.json
└── (generated configs: next.config, tsconfig, package.json, etc.)
```

If a Minimal project grows past ~10 routes or gains real domains, promote it to
Standard by introducing `features/` and route groups — that is a clean, incremental
migration, not a rewrite.

---

## 2. Standard tier (the default)

Dashboards, SaaS, internal tools — anything with auth and several distinct areas.

```
my-app/
├── src/
│   ├── app/
│   │   ├── (auth)/                  # public; redirects out if logged in
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   └── layout.tsx           # centered, no app shell
│   │   ├── (protected)/             # private; middleware-guarded
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── settings/page.tsx
│   │   │   ├── layout.tsx           # app shell: sidebar + topbar
│   │   │   ├── loading.tsx
│   │   │   └── error.tsx
│   │   ├── api/                     # route handlers
│   │   ├── layout.tsx               # root: fonts, providers, metadata
│   │   ├── globals.css
│   │   └── not-found.tsx
│   ├── components/
│   │   ├── ui/                      # shadcn primitives
│   │   ├── layout/                  # AppShell, Sidebar, Topbar, ThemeToggle
│   │   └── providers/               # ThemeProvider, (QueryProvider…), composition
│   ├── features/                    # one folder per domain — see section 5
│   │   └── _example/
│   ├── config/
│   │   ├── site.ts
│   │   └── navigation.ts
│   ├── lib/
│   │   ├── utils.ts                 # cn()
│   │   ├── api-client.ts            # fetch wrapper
│   │   └── auth.ts                  # session helpers (if auth on)
│   ├── hooks/                       # globally shared hooks only
│   ├── types/                       # globally shared types only
│   └── middleware.ts                # (if auth on)
├── public/
├── .env.example
├── components.json
├── STRUCTURE.md
└── (generated configs)
```

`hooks/` and `types/` at the top level are for genuinely cross-feature code only. A
hook used by one feature belongs in that feature, not here.

---

## 3. Large tier

Big single-project codebases — many domains, a large team, a long-lived product. The
**folder tree is the Standard tier, unchanged**: one app, one repository, the same
`app/` + `features/` + `components/` + `config/` + `lib/` layout. The Large tier is not
a different shape; it is the Standard shape plus *enforced discipline*.

What Large adds on top of Standard:

- **Lint-enforced module boundaries.** Add an ESLint rule — `import/no-restricted-paths`
  (built into `eslint-plugin-import`) or `eslint-plugin-boundaries` — that forbids one
  feature from deep-importing another's internals. Cross-feature use must go through the
  target feature's public `index.ts`. In the Standard tier this is a convention; at
  Large scale it is enforced, because conventions quietly erode once enough people and
  domains are involved.
- **Every feature exposes a deliberate public surface.** The `index.ts` is treated as a
  real contract — only what is exported there may be used elsewhere. This is what lets a
  feature be refactored freely behind a stable surface even in a codebase with dozens of
  them.
- **Stricter `app/` thinness.** Route files do nothing but import a feature and render.
  No exceptions, because at this scale one leak invites a hundred.

This tier deliberately stays a single project. A large, well-structured single Next.js
app with enforced boundaries scales to a very large team and domain count without the
operational cost of multiple apps. Splitting into multiple apps is a separate decision
driven by genuinely separate deployables — it is out of scope for this skill, which
builds one fully scalable project.

---

## 4. Per-axis folder additions

Layer these onto the tier tree for every enabled requirement.

**Auth**
```
src/app/(auth)/ and src/app/(protected)/   # route groups
src/middleware.ts
src/features/auth/                          # login/register forms, hooks, schemas
src/lib/auth.ts
```

**RBAC** (requires auth)
```
src/config/roles.ts                         # role enum + path→roles access map
# middleware.ts extended with the role check
# config/navigation.ts items gain a `roles` field
```

**shadcn/ui**
```
src/components/ui/                           # generated primitives
components.json
src/lib/utils.ts                             # cn() — created by shadcn init
```

**Theming**
```
src/components/providers/theme-provider.tsx
src/components/layout/theme-toggle.tsx
```

**TanStack Query / SWR**
```
src/components/providers/query-provider.tsx
src/lib/query-client.ts                      # TanStack only
```

**Forms & validation**
```
# no new top-level folders; each feature gets:
src/features/<name>/schemas/                 # zod schemas
# deps: react-hook-form, zod, @hookform/resolvers
```

**State management**
```
src/features/<name>/store.ts                 # preferred: feature-scoped
src/stores/                                  # only for genuinely global state
```

**Database / ORM**
```
prisma/schema.prisma          # Prisma
# or
src/db/schema.ts + src/db/index.ts            # Drizzle
src/lib/db.ts                                 # singleton client export
# data access lives in features/<name>/api/, never in components
```

**API style**
```
src/app/api/<route>/route.ts                  # Route Handlers
src/features/<name>/actions/                  # Server Actions
src/server/ + src/lib/trpc.ts                 # tRPC
```

**i18n (next-intl)**
```
src/messages/                                 # en.json, bn.json, …
src/i18n/                                      # request config, routing
# middleware.ts updated for locale routing
# optional src/app/[locale]/ segment
```

**Testing**
```
vitest.config.ts                              # unit; *.test.ts(x) co-located in features
e2e/ + playwright.config.ts                   # end-to-end
```

---

## 5. Inside a feature folder

Every domain gets one folder under `features/`. The internal layout is consistent so
any feature is predictable to open. Not every subfolder is mandatory — create only
what the feature uses.

```
features/<name>/
├── components/        # UI specific to this feature
├── api/               # data fetching for this feature (server functions / queries)
├── actions/           # Server Actions (if the project uses them)
├── hooks/             # hooks used only by this feature
├── schemas/           # zod schemas (if the feature has forms)
├── store.ts           # feature-scoped state (if needed)
├── types.ts           # feature-local types
└── index.ts           # public surface — what other code may import
```

The `index.ts` is the feature's contract. Other parts of the app import from
`features/<name>` (the index), never deep-import `features/<name>/components/Foo`. In
the Large tier this is lint-enforced; in Standard it is a convention worth keeping
because it is what lets a feature be refactored freely behind a stable surface.

The `_example/` feature shipped in the scaffold demonstrates this layout and is meant
to be copied as the template for real features, then deleted.
