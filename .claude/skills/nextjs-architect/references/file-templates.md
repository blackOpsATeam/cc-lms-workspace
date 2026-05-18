# File Templates

Copy-ready contents for every base file the scaffold needs. Adapt each to the enabled
requirements — notes on conditionals are inline. These encode the current, correct
patterns; use them rather than improvising per project.

Paths assume the `--src-dir` layout (`src/...`).

## Table of contents

1. `lib/utils.ts`
2. `lib/api-client.ts`
3. `config/site.ts`
4. `config/navigation.ts`
5. `config/roles.ts` (RBAC only)
6. `middleware.ts` (two variants)
7. `components/providers/theme-provider.tsx`
8. `components/providers/query-provider.tsx` (TanStack Query only)
9. `components/providers/index.tsx`
10. `app/layout.tsx`
11. App-shell components
12. Example feature folder
13. `.env.example`
14. `STRUCTURE.md` outline

---

## 1. `lib/utils.ts`

Created by `shadcn init`. If shadcn is not used, create it manually:

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## 2. `lib/api-client.ts`

A thin fetch wrapper. Keeps base URL, headers, and error handling in one place so
feature code stays declarative.

```ts
const BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "";

type RequestOptions = RequestInit & { json?: unknown };

export async function apiClient<T>(
  path: string,
  { json, headers, ...init }: RequestOptions = {},
): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...headers,
    },
    body: json !== undefined ? JSON.stringify(json) : init.body,
  });

  if (!res.ok) {
    const message = await res.text().catch(() => res.statusText);
    throw new Error(message || `Request failed: ${res.status}`);
  }

  return res.status === 204 ? (undefined as T) : ((await res.json()) as T);
}
```

---

## 3. `config/site.ts`

Single source of truth for app identity. Renaming the project starts here.

```ts
export const siteConfig = {
  name: "My App",
  description: "A short description of the app.",
  url: process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000",
} as const;
```

---

## 4. `config/navigation.ts`

Navigation as data. The sidebar maps over this; adding a page is a one-line edit.
Include the `roles` field only when RBAC is enabled.

```ts
import { LayoutDashboard, Settings, type LucideIcon } from "lucide-react";
// import type { Role } from "./roles"; // RBAC only

export interface NavItem {
  label: string;
  href: string;
  icon: LucideIcon;
  // roles?: Role[]; // RBAC only — omit field entirely if no RBAC
}

export const navItems: NavItem[] = [
  { label: "Dashboard", href: "/dashboard", icon: LayoutDashboard },
  { label: "Settings", href: "/settings", icon: Settings },
];

// RBAC variant — uncomment with the roles import and field above:
// export const navFor = (role: Role) =>
//   navItems.filter((i) => !i.roles || i.roles.includes(role));
```

---

## 5. `config/roles.ts` (RBAC only)

Roles and the access map in one auditable place. Replace the role names with the
project's actual roles.

```ts
export const ROLES = ["ADMIN", "USER"] as const;
export type Role = (typeof ROLES)[number];

/** Path prefix → roles permitted to enter it. */
export const ROUTE_ACCESS: { prefix: string; roles: Role[] }[] = [
  { prefix: "/dashboard", roles: ["ADMIN", "USER"] },
  { prefix: "/admin", roles: ["ADMIN"] },
];

export function canAccess(path: string, role: Role): boolean {
  const match = ROUTE_ACCESS.find((r) => path.startsWith(r.prefix));
  return match ? match.roles.includes(role) : true;
}
```

---

## 6. `middleware.ts` (two variants)

Runs before any protected page renders. Use **Variant A** for auth without roles,
**Variant B** when RBAC is enabled.

**Variant A — auth only**

```ts
import { NextRequest, NextResponse } from "next/server";

const PUBLIC_ROUTES = ["/login", "/register", "/forgot-password"];

export function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const token = req.cookies.get("session")?.value;
  const isPublic = PUBLIC_ROUTES.some((p) => pathname.startsWith(p));

  if (!token && !isPublic) {
    const url = new URL("/login", req.url);
    url.searchParams.set("from", pathname);
    return NextResponse.redirect(url);
  }
  if (token && isPublic) {
    return NextResponse.redirect(new URL("/dashboard", req.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```

**Variant B — auth + RBAC.** Decodes the role from a JWT with `jose` (Edge-safe;
`jsonwebtoken` does not run in middleware). Requires `npm i jose` and a `JWT_SECRET`.

```ts
import { NextRequest, NextResponse } from "next/server";
import { jwtVerify } from "jose";
import { canAccess, type Role } from "@/config/roles";

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);
const PUBLIC_ROUTES = ["/login", "/register", "/forgot-password"];

export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const token = req.cookies.get("session")?.value;
  const isPublic = PUBLIC_ROUTES.some((p) => pathname.startsWith(p));

  let role: Role | null = null;
  if (token) {
    try {
      const { payload } = await jwtVerify(token, SECRET);
      role = payload.role as Role;
    } catch {
      role = null; // expired or tampered
    }
  }

  if (!role && !isPublic) {
    const url = new URL("/login", req.url);
    url.searchParams.set("from", pathname);
    return NextResponse.redirect(url);
  }
  if (role && isPublic) {
    return NextResponse.redirect(new URL("/dashboard", req.url));
  }
  if (role && !isPublic && !canAccess(pathname, role)) {
    return NextResponse.redirect(new URL("/dashboard", req.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```

Note: middleware is a routing gate, not the security boundary. Server Actions, Route
Handlers, and data access must still verify the session/role themselves — middleware
can be bypassed by direct API calls.

---

## 7. `components/providers/theme-provider.tsx`

```tsx
"use client";

import { ThemeProvider as NextThemesProvider } from "next-themes";
import type { ComponentProps } from "react";

export function ThemeProvider(props: ComponentProps<typeof NextThemesProvider>) {
  return (
    <NextThemesProvider
      attribute="class"
      defaultTheme="system"
      enableSystem
      disableTransitionOnChange
      {...props}
    >
      {props.children}
    </NextThemesProvider>
  );
}
```

`next-themes` injects its own blocking script, so there is no flash of the wrong
theme — no manual inline script is needed. shadcn's default tokens swap on the `.dark`
class, so dark mode works once this provider wraps the tree.

---

## 8. `components/providers/query-provider.tsx` (TanStack Query only)

```tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState, type ReactNode } from "react";

export function QueryProvider({ children }: { children: ReactNode }) {
  const [client] = useState(() => new QueryClient());
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

---

## 9. `components/providers/index.tsx`

Composes all providers so the root layout stays clean. Include only the providers the
project actually uses.

```tsx
import type { ReactNode } from "react";
import { ThemeProvider } from "./theme-provider";
// import { QueryProvider } from "./query-provider"; // if TanStack Query

export function Providers({ children }: { children: ReactNode }) {
  return (
    <ThemeProvider>
      {/* <QueryProvider> */}
      {children}
      {/* </QueryProvider> */}
    </ThemeProvider>
  );
}
```

---

## 10. `app/layout.tsx`

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import { Providers } from "@/components/providers";
import { siteConfig } from "@/config/site";
import "./globals.css";

const inter = Inter({ subsets: ["latin"], variable: "--font-sans" });

export const metadata: Metadata = {
  title: { default: siteConfig.name, template: `%s | ${siteConfig.name}` },
  description: siteConfig.description,
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={`${inter.variable} font-sans antialiased`}>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

`suppressHydrationWarning` on `<html>` is required by `next-themes` because it sets the
theme class before React hydrates.

---

## 11. App-shell components

`components/layout/sidebar.tsx` — renders from `navigation.ts`, highlights the active
route. For RBAC, call `navFor(currentRole)` instead of using `navItems` directly.

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { navItems } from "@/config/navigation";
import { cn } from "@/lib/utils";

export function Sidebar() {
  const pathname = usePathname();
  return (
    <nav className="flex flex-col gap-1 p-3">
      {navItems.map(({ label, href, icon: Icon }) => (
        <Link
          key={href}
          href={href}
          className={cn(
            "flex items-center gap-3 rounded-md px-3 py-2 text-sm",
            pathname.startsWith(href)
              ? "bg-accent text-accent-foreground"
              : "text-muted-foreground hover:bg-accent/50",
          )}
        >
          <Icon className="h-4 w-4" />
          {label}
        </Link>
      ))}
    </nav>
  );
}
```

`components/layout/theme-toggle.tsx`:

```tsx
"use client";

import { Moon, Sun } from "lucide-react";
import { useTheme } from "next-themes";
import { Button } from "@/components/ui/button";

export function ThemeToggle() {
  const { setTheme, resolvedTheme } = useTheme();
  return (
    <Button
      variant="ghost"
      size="icon"
      aria-label="Toggle theme"
      onClick={() => setTheme(resolvedTheme === "dark" ? "light" : "dark")}
    >
      <Sun className="h-5 w-5 dark:hidden" />
      <Moon className="hidden h-5 w-5 dark:block" />
    </Button>
  );
}
```

`app/(protected)/layout.tsx` — the shell that wraps every private page:

```tsx
import { Sidebar } from "@/components/layout/sidebar";
import { ThemeToggle } from "@/components/layout/theme-toggle";

export default function ProtectedLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex min-h-screen">
      <aside className="w-60 border-r">
        <Sidebar />
      </aside>
      <div className="flex flex-1 flex-col">
        <header className="flex h-14 items-center justify-end border-b px-4">
          <ThemeToggle />
        </header>
        <main className="flex-1 p-6">{children}</main>
      </div>
    </div>
  );
}
```

---

## 12. Example feature folder

Ship `features/_example/` so real features have a pattern to copy. Minimum contents:

`features/_example/types.ts`
```ts
export interface ExampleItem {
  id: string;
  name: string;
}
```

`features/_example/api/get-items.ts`
```ts
import { apiClient } from "@/lib/api-client";
import type { ExampleItem } from "../types";

export function getItems() {
  return apiClient<ExampleItem[]>("/api/items");
}
```

`features/_example/components/item-list.tsx`
```tsx
import type { ExampleItem } from "../types";

export function ItemList({ items }: { items: ExampleItem[] }) {
  return (
    <ul className="space-y-2">
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

`features/_example/index.ts` — the public surface:
```ts
export { ItemList } from "./components/item-list";
export { getItems } from "./api/get-items";
export type { ExampleItem } from "./types";
```

A page then stays thin:
```tsx
// app/(protected)/dashboard/page.tsx
import { getItems, ItemList } from "@/features/_example";

export default async function DashboardPage() {
  const items = await getItems();
  return <ItemList items={items} />;
}
```

---

## 13. `.env.example`

Commit this; never commit `.env`. Include only the keys the enabled axes need.

```bash
# App
NEXT_PUBLIC_APP_URL=http://localhost:3000

# API (if using an external API)
NEXT_PUBLIC_API_URL=

# Auth (if RBAC / JWT)
JWT_SECRET=

# Database (if Prisma / Drizzle)
DATABASE_URL=
```

---

## 14. `STRUCTURE.md` outline

Write this into the project root so the scaffold explains itself. Structure:

```markdown
# Project Structure

## Stack
<chosen tier, language, and the enabled requirement axes>

## Folder tree
<the final tree, annotated one line per top-level folder>

## Conventions
- `app/` is routing only — pages import from `features/` and render.
- New domain code goes in `features/<name>/`, exposed via its `index.ts`.
- Navigation and site config are data in `config/` — edit there, not in JSX.
<+ auth / RBAC / theming notes if those axes are on>

## Adding a feature
1. Copy `features/_example/` to `features/<name>/`.
2. Build inside it; export the public surface from `index.ts`.
3. Add a route under `app/(protected)/` (or `(auth)/`) that imports from it.
4. Add a `config/navigation.ts` entry if it needs a nav link.

## Environment
Copy `.env.example` to `.env` and fill in the values.
```
