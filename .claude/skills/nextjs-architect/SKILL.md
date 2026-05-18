---
name: nextjs-architect
description: Scaffolds a professional, scalable Next.js project structure tailored to stated requirements. Given a description of what a project needs (authentication, role-based access, dark mode, database/ORM, i18n, testing, state management, etc.) and what it does not, this skill produces the right folder structure, configuration, and wired-up base files. Use this skill whenever the user wants to start a new Next.js project, scaffold or bootstrap a Next.js app, set up a Next.js codebase, decide on a Next.js folder structure, create a Next.js boilerplate or starter template, or reorganize an existing Next.js project — even when they do not say the word "structure" explicitly. Trigger on phrasings like "new Next.js project", "set up Next.js", "Next.js boilerplate", "how should I structure my Next.js app", "best folder structure for Next.js", or a list of features the user wants in a Next.js build.
---

# Next.js Architect

This skill turns a set of requirements into a professional Next.js project: the folder
structure, the configuration, and the wired-up base files (routing, route protection,
theming, providers, navigation). The output is consistent and scalable regardless of
project size — the *structure* adapts to the project, the *principles* never change.

The deliverable is a working scaffold the user can `npm run dev` immediately, plus a
short `STRUCTURE.md` in the project root explaining the layout. If the user only wants
a plan (no files), produce the folder tree and the decision summary instead.

Work through four steps in order: **gather requirements → pick a scaling tier →
resolve the structure from the decision matrix → scaffold**.

---

## Adapt to whoever is using this skill

People invoke this skill across the full range of experience — from a senior engineer
who hands over a precise spec, to someone building their first project who is not sure
what "auth" or "ORM" even means. The skill must deliver a professional result for
both, and the user should never need to know the optimal way to ask. Read the request
for cues and adjust posture:

- **Detailed, confident request** — names a stack, lists requirements, uses terms like
  "RBAC", "Server Actions", "Drizzle". This user knows what they want. Skip the
  teaching, confirm the choices back in one line, and build.
- **Vague or unsure request** — "I want to start a Next.js project", "make me
  something professional", a hesitant question, or no requirements at all. Do not dump
  fourteen questions and do not assume any knowledge. Run the guided interview: ask one
  small group of questions at a time, and for every option give a one-line
  plain-language explanation and a clear recommendation, so the user can answer by
  *choosing*, not by already knowing.

When unsure which posture applies, lean toward guiding. A user who already knows the
answer loses nothing from a one-line explanation; a user who does not is rescued by it.
Making a good project should never require the user to understand the jargon — that is
the skill's job, not theirs.

Rules for the guided path:

- **Define each term the first time it appears**, briefly and in parentheses — "auth
  (login and protected pages)", "ORM (the layer your code uses to talk to a database)",
  "RBAC (different permissions for different user roles)".
- **Always pair a question with a recommendation** — "Do you need login? Most apps do;
  I'd include it unless this is a purely public site."
- **Offer the escape hatch** — tell the user they can simply say "use good defaults"
  and the skill will make every reasonable choice for them and explain it afterwards.
- **Close the loop** — after building, briefly explain what was chosen and why, so the
  user finishes knowing a little more than they started. The two always-ask questions
  (language and styling) still apply even on the guided path.

---

## Step 1 — Gather requirements

The whole point of this skill is that the structure follows the requirements. So the
first job is to know them. Often the user has already said what they want earlier in
the conversation — extract from there first. Only ask about what is genuinely unknown,
and ask concisely, grouped into one message. If the user says "use sensible defaults"
or does not care, apply the **default** for every axis below and move on — never block
on questions the user does not want to answer.

The requirement axes, with their defaults:

| Axis | Default | Options |
|---|---|---|
| Language | TypeScript (recommended) | **Always ask** — TypeScript / JavaScript |
| Router | App Router | App Router only — Pages Router is legacy, do not offer it for new projects |
| Auth & route protection | Yes | None / cookie-session / Auth.js (NextAuth) / Clerk / custom JWT |
| Role-based access (RBAC) | No | Yes / No — only relevant if auth is on |
| Styling & UI | Tailwind + shadcn/ui (recommended) | **Always ask** — Tailwind + shadcn/ui / Tailwind only / Tailwind + a component library (Mantine, MUI, Chakra, HeroUI) / non-Tailwind (CSS Modules, styled-components, Panda CSS) |
| Dark/light theming | Yes | Yes / No |
| Data fetching | Server Components + native fetch | + TanStack Query / + SWR |
| Forms & validation | react-hook-form + zod | Include / omit |
| State management | None | None / Zustand / Redux Toolkit / Jotai |
| Database / ORM | None | None / Prisma / Drizzle |
| API style | Route Handlers | Route Handlers / Server Actions / external API / tRPC |
| Internationalization | None | None / next-intl |
| Testing | None | None / Vitest (unit) / Playwright (e2e) / both |

The defaults describe a typical production web app with login. They are deliberately
lean — every "Yes" adds folders, files, and dependencies, so absent a reason, prefer
the default. The user saying "I don't need X" is a first-class input: actively *remove*
the corresponding folders and deps, do not just leave them empty.

**Two choices must always be asked explicitly and are never silently defaulted —
language and styling.** They are the most foundational decisions in the project: each
one shapes every file and the `create-next-app` flags, and users frequently have a
firm preference they will not state unprompted. Make them the first two questions of
the interview, even when in a hurry and even if the answer seems obvious.

- **Language — TypeScript or JavaScript.** Recommend TypeScript in one line (type
  safety pays off as the project grows; the base templates are typed), but honor
  JavaScript fully if chosen — adapt the templates (drop type annotations, use
  `.js`/`.jsx`, omit `tsconfig.json` steps, drop the `--typescript` flag).
- **Styling & UI — how the project is styled and where components come from.** The
  options: *Tailwind + shadcn/ui* (recommended — Tailwind for styling, shadcn for
  components owned in-repo); *Tailwind only* (utility classes, build components
  yourself); *Tailwind + an existing component library* (Mantine, MUI, Chakra,
  HeroUI…); or a *non-Tailwind* approach (CSS Modules, styled-components, Panda CSS).
  Recommend Tailwind + shadcn and say why in one line, but let the user decide. Adapt
  to the answer: for *Tailwind only*, skip `shadcn init` and `components.json` and
  build primitives by hand; for a *component library*, install it and skip shadcn; for
  *non-Tailwind*, drop the `--tailwind` flag, skip shadcn entirely, and adjust the
  theming templates (they use Tailwind's `dark:` class) to the chosen system.

Do not interview past what matters otherwise. If the user said "a simple marketing
site", they do not need auth, RBAC, or a database — infer it and confirm in one line
rather than asking fourteen questions. Language and styling are the two exceptions:
always ask them.

---

## Step 2 — Pick the scaling tier

Project size determines how much structure is appropriate. Too little and a growing app
turns to mud; too much and a small app drowns in ceremony. Pick one tier:

**Minimal** — landing pages, marketing sites, prototypes, small tools (roughly < 10
routes, one developer, short-lived or simple).
The `features/` layer is *omitted*. Code lives in `app/`, `components/`, and `lib/`.
Adding a `features/` folder here is over-engineering.

**Standard** — the default for almost everything: dashboards, SaaS apps, internal
tools, anything with auth and several distinct areas. Feature-based, route groups,
the full structure below. When unsure, choose this.

**Large** — big single-project codebases: many domains, a large team, a long-lived
product. The Standard structure plus lint-enforced module boundaries, so features
cannot reach into each other's internals and every feature is used only through its
public `index.ts`. It is still **one app in one repository** — the structure scales
all the way up without ever splitting into multiple apps.

Tell the user which tier was chosen and why in one sentence. If they push back, switch
— the tiers are a starting point, not a verdict.

For the exact folder tree of each tier, and what each requirement axis *adds* to the
tree, read `references/structure-reference.md`.

---

## Step 3 — Resolve the structure (decision matrix)

The structure is the base tree for the chosen tier, plus the additions triggered by
each enabled requirement. The governing rule across all of it:

> **`app/` is routing only.** Page and layout files stay thin — they import from
> `features/` and render. Business logic, data fetching, and complex components live
> in feature folders, never in `app/`.

This single rule is what keeps a Next.js project from rotting. The framework will not
enforce it; the structure does.

How each axis changes the structure:

- **Auth on** → `(auth)` and `(protected)` route groups; `src/middleware.ts`;
  `features/auth/`; `lib/auth.ts`; `.env` entries. Route groups cost nothing (no URL
  change) but give public and private pages separate layouts and give middleware a
  clean target.
- **RBAC on** → `config/roles.ts` holding the role enum and a `path → roles` access
  map; middleware extended to check it; navigation config gains a `roles` field per
  item. Keep the access rules as *data* in one file — never scatter `if (role === …)`
  through pages.
- **shadcn/ui** → `components/ui/` (generated, owned in-repo), `components.json`,
  `lib/utils.ts` with the `cn()` helper.
- **Theming on** → `components/providers/theme-provider.tsx` using `next-themes`;
  the provider wraps the root layout. `next-themes` injects a blocking script so there
  is no flash of the wrong theme — do not hand-roll this.
- **TanStack Query / SWR** → a query provider in `components/providers/`; `lib/`
  gains the client setup. Native fetch in Server Components needs none of this.
- **Forms** → `react-hook-form` + `zod` + `@hookform/resolvers`; each feature keeps
  its form schemas in `features/<name>/schemas/`.
- **State management** → a `stores/` folder (global) or `features/<name>/store.ts`
  (feature-scoped). Prefer feature-scoped; reach for global state only for things that
  are genuinely cross-cutting.
- **Database / ORM** → `prisma/` or `src/db/` (Drizzle); `lib/db.ts` exporting a
  singleton client; `.env` `DATABASE_URL`. Data access lives in `features/<name>/api/`,
  not in components.
- **API style** → Route Handlers live in `app/api/`; Server Actions live in
  `features/<name>/actions/`; tRPC adds `server/` and a typed client in `lib/`.
- **i18n** → `next-intl` with `messages/` (locale JSON), middleware updated for locale
  routing, an optional `[locale]` segment.
- **Testing** → `vitest.config.ts` + co-located `*.test.ts(x)` for unit; `e2e/` +
  `playwright.config.ts` for end-to-end.
- **Large tier** → add an ESLint boundary rule (`import/no-restricted-paths` or
  `eslint-plugin-boundaries`) so features cannot deep-import each other — cross-feature
  use must go through a feature's public `index.ts`. No new folders; it is enforcement
  layered on the Standard tree.

Regardless of which axes are enabled, every project always gets `config/` (site
metadata and navigation, kept as data) and `lib/` (shared helpers) — these never vary.
The *why* behind every structural choice above is in **Core principles** below; apply
that reasoning, don't re-derive it per project.

---

## Step 4 — Scaffold

Let the official generators do the parts they own; this skill adds structure on top.

1. **Generate the base app.** Run `create-next-app` with the user's choices, e.g.
   `npx create-next-app@latest <name> --typescript --tailwind --app --src-dir --eslint
   --import-alias "@/*"`. This decides the Tailwind major version and creates
   `globals.css` / Tailwind config correctly for that version — do not hand-write
   Tailwind config that fights the generator.
2. **Add shadcn/ui** if selected: `npx shadcn@latest init`, then add the handful of
   primitives the project needs (`button card input dialog dropdown-menu sonner
   skeleton` is a sensible starting set). Keep `cssVariables: true`.
3. **Install the dependencies** implied by the enabled axes (auth lib, `next-themes`,
   `zod` + `react-hook-form`, query lib, ORM, `next-intl`, test runners…). Install only
   what the requirements call for.
4. **Create the structure** — the folders for the chosen tier plus the axis additions
   from Step 3. Add a `.gitkeep` to otherwise-empty folders so the structure survives
   in version control.
5. **Write the base files.** Their exact, copy-ready contents are in
   `references/file-templates.md` — middleware, providers, `config/` files, `lib/`
   helpers, the example feature folder, and the app-shell components. Read that file
   and adapt each template to the requirements (e.g. include the role check in
   middleware only if RBAC is on). Do not invent these from scratch each time; the
   templates encode the correct, current patterns.
6. **Write `STRUCTURE.md`** in the project root: the final folder tree, one line per
   top-level folder explaining its job, the requirement choices that were made, and
   the "how to add a feature" workflow. This is what makes the scaffold *teachable* to
   whoever opens it next.
7. **Present** the project and `STRUCTURE.md`, and state the tier and key choices.

---

## Core principles (the "why")

Carry these into every decision; they are the reason the output is professional and
not just a pile of folders.

- **Thin `app/`, logic in `features/`.** The most common way a Next.js codebase rots
  is logic creeping into route files until nothing can change safely. Feature folders
  keep things that change together located together; a feature can be understood — or
  deleted — as one unit. The project scales by *adding* folders, not editing shared
  ones.
- **Colocation over grouping-by-type.** A global `components/`, `hooks/`, `types/`
  split looks tidy on day one and becomes a scavenger hunt by month two. Globals exist
  only for genuinely shared code; the default home for new code is a feature.
- **Config-as-data.** Navigation and site metadata change constantly. As plain arrays
  in `config/`, changes are one-liners with no JSX touched, and access rules live in
  exactly one auditable place.
- **One place for auth.** Middleware runs before render — protection in one file beats
  scattered per-page checks you will eventually forget in one of them.
- **Own your primitives.** shadcn copies component source into the repo: no version
  lock, fully editable. For a long-lived project that beats an opaque dependency.
- **Be boring.** Use Next.js's own conventions and widely-documented patterns.
  Surprise is a cost paid every time someone new reads the code.
- **Defer complexity.** Do not add roles, global state, a database, or i18n the
  project did not ask for. A scaffold that guesses wrong is worse than one that stays
  neutral. Match the tier and the stated needs — nothing more.

---

## References

- `references/structure-reference.md` — full folder trees for the Minimal, Standard,
  and Large tiers, and the precise folder additions for each requirement axis. Read
  before generating the tree.
- `references/file-templates.md` — exact, copy-ready contents for every base file
  (middleware with/without RBAC, theme provider, providers composition, `config/`
  files, `lib/` helpers, example feature folder, app-shell components, `.env.example`).
  Read before writing files in Step 4.
