---
layout: post
title: "Making Next.js Feel Like Python: The Linear Script Pattern"
date: 2026-03-21
---


If you're a Python developer building frontends with Next.js—while keeping your backend in Python, Go, or Express—the "hooks everywhere" paradigm can feel disorienting. The constant ping-pong between `useEffect`, `useState`, and API routes makes it hard to trace where data comes from.

By leaning into **Server Components**, you can write Next.js pages that read like a linear Python script—top to bottom, with traceable data and no hidden magic. Your Next.js server acts as a **Backend for Frontend (BFF)**: it calls your real backend's API, and the browser only ever sees the final HTML.

> **Note**: This guide uses Next.js 16 conventions. The most notable change from earlier versions is that `middleware.ts` has been renamed to `proxy.ts`—the functionality is the same, but the new name clarifies that this layer acts as a network gateway, not app-level middleware.

---

## 1. The Core Idea: Server Components as Your "Main" Function

Instead of scattering data fetching across hooks and API routes, put it directly inside your Page function. The page becomes your entry point—like `main.py`.

```tsx
// app/dashboard/page.tsx

import { fetchFromBackend } from "@/lib/api";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const user = await fetchFromBackend("/me");

  if (!user) {
    redirect("/login");
  }

  const posts = await fetchFromBackend(`/users/${user.id}/posts`);

  const formatDate = (date: string) => new Date(date).toLocaleDateString();

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">{user.name}'s Dashboard</h1>

      <section className="mt-6">
        {posts.map((post: any) => (
          <div key={post.id} className="border-b py-4">
            <h2 className="text-xl">{post.title}</h2>
            <p className="text-gray-500">{formatDate(post.created_at)}</p>
          </div>
        ))}
      </section>
    </main>
  );
}
```

Why this works for a Python-trained brain:

- **No "invisible" entry**: Everything that happens when the user hits this URL starts at line 1 of this file.
- **Linear execution**: Read it top to bottom. Data is fetched (`await`), logic is checked (`if !user`), then UI is returned.
- **Traceable data**: If you wonder where `posts` comes from, look three lines up. Ctrl+Click `fetchFromBackend` to see exactly how data is fetched.
- **Colocated styles**: With Tailwind CSS (`p-8`, `text-2xl`), you don't need to hunt through `.css` files.

---

## 2. The API Layer: Your Backend Connection

Since your backend is always a separate service, create a centralized API client that every Server Component and Server Action uses:

```tsx
// lib/api.ts
import { cookies } from "next/headers";

const API_BASE = process.env.API_BASE_URL!; // e.g. https://api.yourapp.com

export async function fetchFromBackend(
  endpoint: string,
  options: RequestInit = {}
) {
  const cookieStore = await cookies();
  const token = cookieStore.get("auth_token")?.value;

  const response = await fetch(`${API_BASE}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      ...(token && { Authorization: `Bearer ${token}` }),
      "Content-Type": "application/json",
    },
  });

  if (!response.ok) {
    throw new Error(`API error: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export async function mutateBackend(
  endpoint: string,
  method: "POST" | "PUT" | "PATCH" | "DELETE",
  body?: any
) {
  return fetchFromBackend(endpoint, {
    method,
    ...(body && { body: JSON.stringify(body) }),
  });
}
```

Every page reads the same way:

```tsx
// app/projects/page.tsx
import { fetchFromBackend } from "@/lib/api";

export default async function ProjectsPage() {
  const projects = await fetchFromBackend("/projects");

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Projects</h1>
      <ul className="mt-4">
        {projects.map((p: any) => (
          <li key={p.id} className="py-2 border-b">{p.name}</li>
        ))}
      </ul>
    </main>
  );
}
```

Why this pattern matters:

- **Zero browser fetches**: The browser never sees your API URL—only the final HTML.
- **Security**: API keys and tokens stay on the Next.js server, never exposed to the browser.
- **Explicit traceability**: Ctrl+Click `fetchFromBackend` to see how data is fetched, what headers are sent, how errors are handled.

> **Future-proofing**: If you ever move to gRPC or another protocol, you only change `lib/api.ts`. Every page that imports from it continues working without modification.

---

## 3. Authentication: JWT with HTTP-Only Cookies

Store the JWT from your backend in an **HTTP-only cookie**. This makes the token available to your Next.js server on every request without browser-side global state—and protects against XSS attacks since JavaScript can't read the cookie.

### The Login Flow

```tsx
// app/login/actions.ts
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export async function loginAction(formData: FormData) {
  "use server";

  const res = await fetch(`${process.env.API_BASE_URL}/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      email: formData.get("email"),
      password: formData.get("password"),
    }),
  });

  if (!res.ok) {
    return { error: "Invalid credentials" };
  }

  const { token } = await res.json();

  const cookieStore = await cookies();
  cookieStore.set("auth_token", token, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 60 * 60 * 24 * 7, // 7 days
  });

  redirect("/dashboard");
}
```

### The Login Page

```tsx
// app/login/page.tsx
import { loginAction } from "./actions";

export default function LoginPage() {
  return (
    <main className="p-10 max-w-md mx-auto">
      <h1 className="text-2xl font-bold mb-6">Sign In</h1>
      <form action={loginAction} className="space-y-4">
        <input name="email" type="email" placeholder="Email" className="w-full border p-2 rounded" required />
        <input name="password" type="password" placeholder="Password" className="w-full border p-2 rounded" required />
        <button type="submit" className="w-full bg-blue-500 text-white py-2 rounded">Sign In</button>
      </form>
    </main>
  );
}
```

All sensitive communication happens between Next.js and your backend. The browser is just a viewer of the final HTML.

---

## 4. Auth Layers: Proxy, Server Components, and Client Context

Next.js gives you three places to check who a user is. They are **not** interchangeable—each runs in a different environment, at a different time, and solves a different problem.

| Layer | Where It Runs | When It Runs | What It Can Do | What It Cannot Do |
|---|---|---|---|---|
| **Proxy** (`proxy.ts`) | Edge/Node, before routing | Before any page renders | Redirect, rewrite, set headers | Access your backend API in depth, render UI |
| **Server Component** (`page.tsx`) | Node.js server, during render | When the page is requested | Call your backend, verify roles, render UI | Respond to browser events |
| **Client Context** (`useContext`) | Browser, after hydration | After the page loads | Toggle UI, personalize display | Protect routes, guard data |

> Think of it as: **Proxy** = the front door bouncer (checks your ID fast). **Server Component** = the room-level security guard (verifies your access fully). **Client Context** = the name badge you wear inside (personalization only, protects nothing).

### Layer 1: Proxy — Fast Redirects (`proxy.ts`)

The proxy runs before your page renders. It handles optimistic checks: "Does this person have a cookie at all?" It should **not** be your only security layer—a 2025 vulnerability showed that middleware-based auth could be bypassed, which is partly why Next.js renamed it to "proxy."

```tsx
// proxy.ts (root of project)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const token = request.cookies.get("auth_token")?.value;
  const { pathname } = request.nextUrl;

  const publicPaths = ["/login", "/signup", "/about"];
  if (publicPaths.some((path) => pathname.startsWith(path))) {
    return NextResponse.next();
  }

  if (!token) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("redirect", pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Layer 2: Server Component — Real Authorization

This is where actual security lives. Your Server Component calls the backend API to verify the token and check roles. Even if someone bypasses the proxy, this layer rejects them.

```tsx
// lib/dal.ts (Data Access Layer)
import { cache } from "react";
import { cookies } from "next/headers";

const API_BASE = process.env.API_BASE_URL!;

// Memoized: call this 10 times across 10 components, the API is hit once per request
export const getCurrentUser = cache(async () => {
  const cookieStore = await cookies();
  const token = cookieStore.get("auth_token")?.value;

  if (!token) return null;

  try {
    const res = await fetch(`${API_BASE}/me`, {
      headers: { Authorization: `Bearer ${token}` },
    });

    if (!res.ok) return null;
    return await res.json();
  } catch {
    return null;
  }
});
```

Use it in any page:

```tsx
// app/admin/page.tsx
import { getCurrentUser } from "@/lib/dal";
import { redirect } from "next/navigation";

export default async function AdminPage() {
  const user = await getCurrentUser();

  if (!user) redirect("/login");
  if (user.role !== "admin") redirect("/unauthorized");

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Admin Panel</h1>
      <p className="text-gray-600">Logged in as {user.name} ({user.role})</p>
    </main>
  );
}
```

### Layer 3: Client Context — UI Personalization Only

Client context makes user info available to client components for display. It protects nothing—a malicious user can modify client-side state.

```tsx
// components/UserProvider.tsx
"use client";
import { createContext, useContext, ReactNode } from "react";

type User = { id: string; name: string; role: string } | null;

const UserContext = createContext<User>(null);

export function UserProvider({ user, children }: { user: User; children: ReactNode }) {
  return <UserContext.Provider value={user}>{children}</UserContext.Provider>;
}

export const useUser = () => useContext(UserContext);
```

Bridge it in your layout—the server fetches the real user, then passes it to the client:

```tsx
// app/layout.tsx
import { getCurrentUser } from "@/lib/dal";
import { UserProvider } from "@/components/UserProvider";

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser();

  return (
    <html>
      <body>
        <UserProvider user={user}>
          {children}
        </UserProvider>
      </body>
    </html>
  );
}
```

Any client component can now display personalized UI:

```tsx
// components/Header.tsx
"use client";
import { useUser } from "@/components/UserProvider";

export default function Header() {
  const user = useUser();
  return (
    <header className="flex justify-between items-center p-4 border-b">
      <h1 className="font-bold">My App</h1>
      {user ? <span>Hello, {user.name}</span> : <a href="/login">Sign In</a>}
    </header>
  );
}
```

### Which Layers Do I Need?

| Question | Answer |
|---|---|
| Can I skip the proxy and just check in `page.tsx`? | **Yes.** This is the most secure approach. The proxy just makes redirects faster. |
| Can I rely only on the proxy? | **No.** The proxy can be bypassed and can't call your backend to verify tokens properly. |
| Can I use `useUser()` context for access control? | **No.** Client state is modifiable. Use it only for UI personalization. |
| Do I need all three? | **Proxy + server component checks** is the recommended combo. Client context is optional—add it when client components need user info for display. |

| Python Pattern | Next.js Equivalent | Layer |
|---|---|---|
| `@app.before_request` (Flask) | `proxy.ts` | Proxy |
| `@login_required` / `Depends(get_current_user)` | `getCurrentUser()` in `page.tsx` | Server Component |
| `request.user` in templates | `useUser()` in client components | Client Context |

---

## 5. When You Must Use Client Components

The linear server-side pattern covers a lot of ground, but it breaks down when you need immediate browser-side feedback or access to user hardware. You must use the `"use client"` directive for:

**Direct user interactivity**: `onClick`, `onChange`, `onMouseEnter`, real-time form validation, any UI that changes without a full page reload (`useState`, `useReducer`).

**Browser-only APIs**: `window.localStorage`, `window.innerWidth`, `navigator.geolocation`, Camera API, Bluetooth, and animation libraries like Framer Motion or GSAP.

**Component lifecycle and side effects**: `useEffect` / `useLayoutEffect` for timers, WebSocket connections, analytics logging, or third-party UI libraries that rely on browser-specific hooks.

### Keeping the Linear Flow: The "Hole" Pattern

Even with client components, you can maintain clarity through **Component Composition**. Fetch data in the Server Page, then pass the rendered server content as `children` into the Client Component:

```tsx
// components/InteractiveSidebar.tsx
"use client";
import { useState } from "react";

export default function InteractiveSidebar({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>
        {isOpen ? "Close Sidebar" : "Open Sidebar"}
      </button>
      {isOpen && <aside className="border-r p-4 w-64">{children}</aside>}
    </div>
  );
}
```

```tsx
// app/dashboard/page.tsx
import { fetchFromBackend } from "@/lib/api";
import InteractiveSidebar from "@/components/InteractiveSidebar";

export default async function DashboardPage() {
  const folders = await fetchFromBackend("/folders");

  return (
    <main className="flex">
      <InteractiveSidebar>
        <nav>
          <h2 className="font-bold">My Folders</h2>
          <ul>
            {folders.map((folder: any) => (
              <li key={folder.id}>{folder.name}</li>
            ))}
          </ul>
        </nav>
      </InteractiveSidebar>
      <section className="p-10">
        <h1>Main Content Area</h1>
      </section>
    </main>
  );
}
```

> **Rule of Thumb**: If you find yourself putting `useEffect` or `fetch` inside a Client Component, stop and ask: "Can I move this data fetching up to the Page and just pass the result down?" Usually, the answer is yes.

---

## 6. Server Actions: Mutations as Functions

Server Actions let you handle data mutations (POST/PUT/DELETE) like function calls. They call your backend API, then refresh the page—no manual API endpoints or client-side `fetch()`.

```tsx
// app/folders/page.tsx
import { fetchFromBackend, mutateBackend } from "@/lib/api";
import { revalidatePath } from "next/cache";

export default async function FolderPage() {
  const folders = await fetchFromBackend("/folders");

  async function createFolder(formData: FormData) {
    "use server";
    await mutateBackend("/folders", "POST", {
      name: formData.get("folderName"),
    });
    revalidatePath("/folders");
  }

  async function deleteFolder(formData: FormData) {
    "use server";
    const id = formData.get("id");
    await mutateBackend(`/folders/${id}`, "DELETE");
    revalidatePath("/folders");
  }

  return (
    <main className="p-10">
      <h1 className="text-2xl font-bold">My Folders</h1>

      <ul className="my-4">
        {folders.map((f: any) => (
          <li key={f.id} className="p-2 border-b flex justify-between items-center">
            {f.name}
            <form action={deleteFolder}>
              <input type="hidden" name="id" value={f.id} />
              <button type="submit" className="text-red-500 text-sm">Delete</button>
            </form>
          </li>
        ))}
      </ul>

      <form action={createFolder} className="flex gap-2">
        <input name="folderName" placeholder="New folder name..." className="border p-2 rounded" required />
        <button type="submit" className="bg-blue-500 text-white px-4 py-2 rounded">Add Folder</button>
      </form>
    </main>
  );
}
```

### Adding Loading States

Use `useFormStatus` in a small client sub-component. The core Server Action stays in the main script:

```tsx
// components/SubmitButton.tsx
"use client";
import { useFormStatus } from "react-dom";

export function SubmitButton({ label = "Submit", pending_label = "Saving..." }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending} className="bg-blue-500 text-white px-4 py-2 rounded disabled:opacity-50">
      {pending ? pending_label : label}
    </button>
  );
}
```

---

## 7. Zod Validation: Pydantic for TypeScript

If you love Pydantic in Python, you'll love **Zod** in TypeScript. Define a schema, validate incoming data, get type-safe results or clear error messages.

### Side-by-Side Comparison

```python
# Python (Pydantic)
from pydantic import BaseModel, field_validator

class CreatePostInput(BaseModel):
    title: str
    content: str
    tags: list[str] = []

    @field_validator('title')
    def title_not_empty(cls, v):
        if len(v.strip()) < 3:
            raise ValueError('Title must be at least 3 characters')
        return v.strip()
```

```tsx
// TypeScript (Zod)
import { z } from "zod";

const CreatePostSchema = z.object({
  title: z.string().min(3, "Title must be at least 3 characters").transform((v) => v.trim()),
  content: z.string().min(1, "Content is required"),
  tags: z.array(z.string()).default([]),
});

type CreatePostInput = z.infer<typeof CreatePostSchema>;
```

### Validating Server Actions

Validate form data inside your Server Action before calling your backend:

```tsx
// app/posts/new/page.tsx
import { mutateBackend } from "@/lib/api";
import { redirect } from "next/navigation";
import { z } from "zod";

const CreatePostSchema = z.object({
  title: z.string().min(3, "Title must be at least 3 characters").max(100, "Title too long"),
  content: z.string().min(10, "Content must be at least 10 characters"),
  tags: z
    .string()
    .transform((v) => v.split(",").map((t) => t.trim()).filter(Boolean))
    .pipe(z.array(z.string()).max(5, "Maximum 5 tags")),
});

export default async function NewPostPage() {
  async function createPost(formData: FormData) {
    "use server";

    const result = CreatePostSchema.safeParse({
      title: formData.get("title"),
      content: formData.get("content"),
      tags: formData.get("tags") ?? "",
    });

    if (!result.success) {
      return { errors: result.error.flatten().fieldErrors };
    }

    await mutateBackend("/posts", "POST", result.data);
    redirect("/posts");
  }

  return (
    <main className="p-10 max-w-2xl">
      <h1 className="text-2xl font-bold">Create New Post</h1>
      <form action={createPost} className="mt-6 space-y-4">
        <div>
          <label className="block text-sm font-medium">Title</label>
          <input name="title" className="mt-1 w-full border p-2 rounded" required />
        </div>
        <div>
          <label className="block text-sm font-medium">Content</label>
          <textarea name="content" rows={6} className="mt-1 w-full border p-2 rounded" required />
        </div>
        <div>
          <label className="block text-sm font-medium">Tags (comma-separated)</label>
          <input name="tags" placeholder="react, nextjs, tutorial" className="mt-1 w-full border p-2 rounded" />
        </div>
        <button type="submit" className="bg-blue-500 text-white px-6 py-2 rounded">Publish</button>
      </form>
    </main>
  );
}
```

### Showing Validation Errors with `useActionState`

To display field-level errors, use `useActionState` (React 19+) in a client component:

```tsx
// components/PostForm.tsx
"use client";
import { useActionState } from "react";

type FormState = {
  errors?: { title?: string[]; content?: string[]; tags?: string[] };
  message?: string;
};

export function PostForm({
  action,
}: {
  action: (prevState: FormState, formData: FormData) => Promise<FormState>;
}) {
  const [state, formAction, isPending] = useActionState(action, {});

  return (
    <form action={formAction} className="mt-6 space-y-4">
      <div>
        <label className="block text-sm font-medium">Title</label>
        <input name="title" className="mt-1 w-full border p-2 rounded" />
        {state.errors?.title && <p className="mt-1 text-sm text-red-600">{state.errors.title[0]}</p>}
      </div>
      <div>
        <label className="block text-sm font-medium">Content</label>
        <textarea name="content" rows={6} className="mt-1 w-full border p-2 rounded" />
        {state.errors?.content && <p className="mt-1 text-sm text-red-600">{state.errors.content[0]}</p>}
      </div>
      <div>
        <label className="block text-sm font-medium">Tags</label>
        <input name="tags" placeholder="react, nextjs" className="mt-1 w-full border p-2 rounded" />
        {state.errors?.tags && <p className="mt-1 text-sm text-red-600">{state.errors.tags[0]}</p>}
      </div>
      <button type="submit" disabled={isPending} className="bg-blue-500 text-white px-6 py-2 rounded disabled:opacity-50">
        {isPending ? "Publishing..." : "Publish"}
      </button>
      {state.message && <p className="text-red-600 font-medium">{state.message}</p>}
    </form>
  );
}
```

### Validating API Responses

Zod also validates data coming *from* your backend—like Pydantic parsing an API response:

```tsx
// lib/schemas.ts
import { z } from "zod";

export const UserSchema = z.object({
  id: z.string(),
  username: z.string(),
  email: z.string().email(),
  role: z.enum(["user", "admin"]),
  created_at: z.string().transform((s) => new Date(s)),
});

export type User = z.infer<typeof UserSchema>;
```

```tsx
// lib/api.ts (add a typed fetch helper)
import { z } from "zod";

export async function fetchTyped<T>(
  schema: z.ZodType<T>,
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const data = await fetchFromBackend(endpoint, options);
  return schema.parse(data); // Throws immediately if the API shape changes
}
```

### Zod ↔ Pydantic Cheat Sheet

| Pydantic | Zod |
|---|---|
| `str` | `z.string()` |
| `int` | `z.number().int()` |
| `float` | `z.number()` |
| `bool` | `z.boolean()` |
| `list[str]` | `z.array(z.string())` |
| `Optional[str]` | `z.string().optional()` |
| `Field(default="x")` | `z.string().default("x")` |
| `Field(min_length=3)` | `z.string().min(3)` |
| `@field_validator` | `.refine()` or `.transform()` |
| `model_validate(data)` | `Schema.parse(data)` |
| `model_validate` (safe) | `Schema.safeParse(data)` |
| `ValidationError` | `ZodError` (via `.error.flatten()`) |
| Type from model | `z.infer<typeof Schema>` |

---

## 8. Error Handling

Next.js gives you two parallel systems: **file-based error boundaries** for catching render-time failures, and **try/catch inside Server Components** for granular control.

### File-Based Error Boundaries (`error.tsx`)

Drop an `error.tsx` file next to your `page.tsx`. It catches any unhandled error thrown during rendering of that route segment:

```tsx
// app/dashboard/error.tsx
"use client"; // Error boundaries MUST be client components

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="p-10 text-center">
      <h2 className="text-xl font-bold text-red-600">Something went wrong</h2>
      <p className="mt-2 text-gray-600">{error.message}</p>
      <button onClick={() => reset()} className="mt-4 bg-blue-500 text-white px-4 py-2 rounded">
        Try Again
      </button>
    </div>
  );
}
```

Error boundaries use React's `componentDidCatch` lifecycle under the hood, which only works on the client—hence the required `"use client"`. But notice it has no `useEffect` or `fetch`. It's just a UI shell.

### Inline Error Handling

For finer control—like showing a fallback for one failed API call without crashing the whole page:

```tsx
// app/dashboard/page.tsx
import { fetchFromBackend } from "@/lib/api";

export default async function DashboardPage() {
  const user = await fetchFromBackend("/me");

  let analytics = null;
  let analyticsError = null;

  try {
    analytics = await fetchFromBackend("/analytics/summary");
  } catch (err) {
    analyticsError = err instanceof Error ? err.message : "Failed to load analytics";
  }

  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">{user.name}'s Dashboard</h1>

      {analyticsError ? (
        <div className="mt-4 p-4 bg-red-50 border border-red-200 rounded">
          <p className="text-red-700">Analytics unavailable: {analyticsError}</p>
        </div>
      ) : (
        <section className="mt-4">
          <p>Page views: {analytics.pageViews}</p>
        </section>
      )}
    </main>
  );
}
```

### The `not-found.tsx` Pattern

Python frameworks have `abort(404)`. Next.js has `notFound()`:

```tsx
// app/posts/[id]/page.tsx
import { fetchFromBackend } from "@/lib/api";
import { notFound } from "next/navigation";

export default async function PostPage({ params }: { params: { id: string } }) {
  let post;
  try {
    post = await fetchFromBackend(`/posts/${params.id}`);
  } catch {
    notFound();
  }

  return (
    <article className="p-8">
      <h1 className="text-3xl font-bold">{post.title}</h1>
      <p className="mt-4">{post.content}</p>
    </article>
  );
}
```

```tsx
// app/posts/[id]/not-found.tsx
export default function PostNotFound() {
  return (
    <div className="p-10 text-center">
      <h2 className="text-xl font-bold">Post Not Found</h2>
      <p className="mt-2 text-gray-500">This post doesn't exist or was removed.</p>
    </div>
  );
}
```

### Error Handling in Server Actions

Return structured results instead of throwing:

```tsx
async function createFolder(formData: FormData) {
  "use server";

  const name = formData.get("folderName");
  if (!name || String(name).trim().length === 0) {
    return { error: "Folder name is required" };
  }

  try {
    await mutateBackend("/folders", "POST", { name: String(name) });
    revalidatePath("/folders");
    return { success: true };
  } catch (err) {
    return { error: "Failed to create folder. Please try again." };
  }
}
```

> **Mapping to Python**: `error.tsx` = route-level `except Exception`. `notFound()` = `abort(404)`. Inline `try/catch` = `try/except` with a fallback value.

---

## 9. Loading & Suspense

In Python, when you `await` a slow query, the server waits—the user sees nothing until the full response is ready. Next.js gives you **Suspense boundaries** that show parts of the page instantly while slower data loads in the background.

### The Simplest Pattern: `loading.tsx`

Drop a `loading.tsx` next to your `page.tsx`. Next.js automatically wraps the page in a Suspense boundary:

```tsx
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="p-8 animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-48 mb-6" />
      <div className="space-y-3">
        <div className="h-4 bg-gray-200 rounded w-full" />
        <div className="h-4 bg-gray-200 rounded w-3/4" />
        <div className="h-4 bg-gray-200 rounded w-5/6" />
      </div>
    </div>
  );
}
```

No `useState`, no `isLoading` flag. The file's existence is the configuration.

### Granular Suspense: Load Parts Independently

When different parts of your page have different load times, wrap the slow parts individually:

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";
import { fetchFromBackend } from "@/lib/api";

async function UserHeader() {
  const user = await fetchFromBackend("/me");
  return <h1 className="text-2xl font-bold">Welcome, {user.name}</h1>;
}

async function AnalyticsPanel() {
  const stats = await fetchFromBackend("/analytics/summary");
  return (
    <section className="mt-6 grid grid-cols-3 gap-4">
      <div className="p-4 border rounded">
        <p className="text-sm text-gray-500">Page Views</p>
        <p className="text-2xl font-bold">{stats.pageViews.toLocaleString()}</p>
      </div>
      <div className="p-4 border rounded">
        <p className="text-sm text-gray-500">Users</p>
        <p className="text-2xl font-bold">{stats.activeUsers.toLocaleString()}</p>
      </div>
      <div className="p-4 border rounded">
        <p className="text-sm text-gray-500">Bounce Rate</p>
        <p className="text-2xl font-bold">{stats.bounceRate}%</p>
      </div>
    </section>
  );
}

async function RecentPosts() {
  const posts = await fetchFromBackend("/posts?limit=10&sort=recent");
  return (
    <ul className="mt-4">
      {posts.map((post: any) => (
        <li key={post.id} className="py-2 border-b">{post.title}</li>
      ))}
    </ul>
  );
}

export default function DashboardPage() {
  return (
    <main className="p-8">
      <Suspense fallback={<div className="h-8 bg-gray-200 rounded w-48 animate-pulse" />}>
        <UserHeader />
      </Suspense>

      <Suspense fallback={
        <div className="mt-6 grid grid-cols-3 gap-4">
          {[1, 2, 3].map((i) => (
            <div key={i} className="p-4 border rounded animate-pulse">
              <div className="h-3 bg-gray-200 rounded w-20 mb-2" />
              <div className="h-6 bg-gray-200 rounded w-16" />
            </div>
          ))}
        </div>
      }>
        <AnalyticsPanel />
      </Suspense>

      <Suspense fallback={<p className="mt-4 text-gray-400">Loading posts...</p>}>
        <RecentPosts />
      </Suspense>
    </main>
  );
}
```

In Python with `asyncio`, you'd use `asyncio.gather` to run fetches concurrently—but even `gather` waits for ALL to complete before returning. Suspense is better: it **streams** each result to the browser as soon as it's ready.

> **Rule of Thumb**: If you have two `await` calls and one is significantly slower, split them into separate async components wrapped in `<Suspense>`. Your users see content faster.

---

## 10. Real-Time Data: SSE & WebSockets

Your backend likely already handles WebSockets or streaming. Next.js gives you two ways to connect: **Server-Sent Events (SSE)** for one-way server-to-client updates, and **WebSockets** for bidirectional communication.

| | SSE | WebSockets |
|---|---|---|
| Direction | Server → Client (one-way) | Bidirectional |
| Protocol | HTTP (works everywhere) | WS (needs separate server) |
| Python Equivalent | `StreamingResponse` (FastAPI) | `websockets` / `socket.io` |
| Best For | Live feeds, notifications, progress bars | Chat, collaboration, gaming |
| Reconnection | Built-in (browser auto-reconnects) | Manual |

> **Rule of Thumb**: If the client only needs to *receive* updates, use SSE. Use WebSockets only when the client needs to *send* messages back in real-time.

### Server-Sent Events

#### The Route Handler (streams from your backend)

```tsx
// app/api/notifications/stream/route.ts
export const dynamic = "force-dynamic";

export async function GET(request: Request) {
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      const send = (data: any) => {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(data)}\n\n`));
      };

      send({ type: "connected", timestamp: Date.now() });

      const interval = setInterval(async () => {
        try {
          const res = await fetch(`${process.env.API_BASE_URL}/notifications/latest`, {
            headers: { Authorization: `Bearer ${process.env.INTERNAL_TOKEN}` },
          });
          const notifications = await res.json();
          send({ type: "notifications", data: notifications });
        } catch {
          send({ type: "error", message: "Failed to fetch" });
        }
      }, 3000);

      request.signal.addEventListener("abort", () => {
        clearInterval(interval);
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

#### The Client Listener

This is one of the cases where `useEffect` is genuinely necessary—you're connecting to a persistent browser API:

```tsx
// components/NotificationFeed.tsx
"use client";
import { useEffect, useState } from "react";

type Notification = { id: string; message: string; createdAt: string };

export default function NotificationFeed() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const eventSource = new EventSource("/api/notifications/stream");

    eventSource.onopen = () => setIsConnected(true);
    eventSource.onmessage = (event) => {
      const payload = JSON.parse(event.data);
      if (payload.type === "notifications") setNotifications(payload.data);
    };
    eventSource.onerror = () => setIsConnected(false);

    return () => eventSource.close();
  }, []);

  return (
    <div>
      <div className="flex items-center gap-2 mb-4">
        <span className={`w-2 h-2 rounded-full ${isConnected ? "bg-green-500" : "bg-red-500"}`} />
        <span className="text-sm text-gray-500">{isConnected ? "Live" : "Reconnecting..."}</span>
      </div>
      <ul>
        {notifications.map((n) => (
          <li key={n.id} className="py-2 border-b">
            <p>{n.message}</p>
            <p className="text-xs text-gray-400">{n.createdAt}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

#### Named Events

For structured streams with different event types:

```tsx
// Route handler sends named events
const send = (event: string, data: any) => {
  controller.enqueue(encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`));
};
send("stats", { activeUsers: 142, cpu: 67.3 });
send("alerts", { level: "warn", message: "High memory usage" });
```

```tsx
// Client listens to specific events
const es = new EventSource("/api/dashboard/stream");
es.addEventListener("stats", (e) => setStats(JSON.parse(e.data)));
es.addEventListener("alerts", (e) => setAlerts((prev) => [JSON.parse(e.data), ...prev]));
```

### WebSockets

Next.js doesn't natively handle WebSocket upgrades, so you run a standalone server alongside it—or more likely, your backend already serves WebSocket endpoints.

#### Client Hook (connects to your backend's WS endpoint)

```tsx
// hooks/useWebSocket.ts
"use client";
import { useEffect, useRef, useState, useCallback } from "react";

type WSMessage = { user: string; text: string; timestamp: number };

export function useWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null);
  const [messages, setMessages] = useState<WSMessage[]>([]);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setIsConnected(true);
    ws.onclose = () => setIsConnected(false);
    ws.onmessage = (event) => {
      setMessages((prev) => [...prev, JSON.parse(event.data)]);
    };

    return () => ws.close();
  }, [url]);

  const send = useCallback((text: string, user: string) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ user, text }));
    }
  }, []);

  return { messages, isConnected, send };
}
```

#### The Page (Linear Flow Preserved)

```tsx
// app/chat/[room]/page.tsx
import { fetchFromBackend } from "@/lib/api";
import { getCurrentUser } from "@/lib/dal";
import { redirect } from "next/navigation";
import ChatRoom from "@/components/ChatRoom";

export default async function ChatPage({ params }: { params: { room: string } }) {
  const user = await getCurrentUser();
  if (!user) redirect("/login");

  const roomInfo = await fetchFromBackend(`/chat/rooms/${params.room}`);
  if (!roomInfo) redirect("/chat");

  return (
    <main className="p-8 max-w-3xl mx-auto">
      <h1 className="text-2xl font-bold mb-2">{roomInfo.name}</h1>
      <p className="text-gray-500 mb-6">{roomInfo.description}</p>
      <ChatRoom
        wsUrl={`${process.env.NEXT_PUBLIC_WS_URL}/chat?room=${params.room}`}
        username={user.name}
      />
    </main>
  );
}
```

The page reads top-to-bottom: fetch user, fetch room, render. The `ChatRoom` client component is just a shell for the WebSocket connection—the server never touches WebSocket code.

### When to Use Which

| Scenario | SSE | WebSocket |
|---|---|---|
| Live notifications | Yes | Overkill |
| Dashboard metrics / stock tickers | Yes | Overkill |
| Progress bars (file upload, AI generation) | Yes | Overkill |
| Chat / messaging | No | Yes |
| Collaborative editing | No | Yes |
| Multiplayer games | No | Yes |

### Python ↔ Next.js Real-Time Cheat Sheet

| Python | Next.js |
|---|---|
| `StreamingResponse` (FastAPI) | Route Handler with `ReadableStream` |
| `yield f"data: ...\n\n"` | `controller.enqueue(encoder.encode(...))` |
| `websockets.serve(handler, ...)` | Your backend's WS endpoint / standalone `ws` server |
| `async for message in websocket` | `ws.onmessage = (event) => ...` |
| `await websocket.send(data)` | `ws.send(JSON.stringify(data))` |

---

## 11. Global State

Split your thinking into **Server State** and **Client State**.

**Server state** (data from your backend) is handled by the Data Access Layer (`lib/dal.ts`) with React `cache`. Call `getCurrentUser()` or `fetchFromBackend()` in any server component—memoized per request, the API is hit once.

**Client state** (UI-only state like the current tab, sidebar open/closed, search filters) has two options:

- **URL Query Params**: For state that should survive page refreshes and be shareable. Server Components can read `searchParams` directly—no client hooks needed.
- **React Context**: For ephemeral UI state that client components need to share (like user info for display, as shown in the auth section).

> **Tip**: Before reaching for `useState` or Context, ask: "Should this state be in the URL?" If the user would expect to bookmark or share it (filters, pagination, tabs), put it in `searchParams`.

---

## Summary

The "Linear Script" pattern for Next.js with a separate backend:

| Concern | Where It Lives | File |
|---|---|---|
| API connection | Centralized client | `lib/api.ts` |
| Auth verification | Memoized data access | `lib/dal.ts` |
| Fast auth redirects | Proxy | `proxy.ts` |
| Data reading | Server Component | `app/**/page.tsx` |
| Data writing | Server Action | `"use server"` functions |
| Input validation | Zod schema | Co-located with the action |
| Response validation | Zod schema | `lib/schemas.ts` |
| Error recovery | Error boundary + try/catch | `error.tsx` + inline |
| Loading states | Suspense | `loading.tsx` + `<Suspense>` |
| Real-time (one-way) | SSE Route Handler | `app/api/**/route.ts` |
| Real-time (two-way) | WebSocket client hook | `hooks/useWebSocket.ts` |
| UI personalization | Client Context (optional) | `components/UserProvider.tsx` |
| Browser interactivity | Client Component shells | `"use client"` components |

The principle throughout: your `page.tsx` is the entry point. It reads top to bottom. Data comes from `fetchFromBackend()`. Mutations go through Server Actions that call `mutateBackend()`. Client components are thin shells for interactivity. Everything is one Ctrl+Click away from its source.

If you find yourself reaching for `useEffect` to fetch data, stop and ask: "Can I move this to the server?" Usually, you can—and your Python-trained brain will thank you.
