---
layout: post
title: "Making Next.js Feel Like Python"
date: 2026-03-21
---


I'm used to starting every Python project from `main.py`. That single file is my anchor. I read the imports, follow the function calls, and the data flow reveals itself. I use FastAPI for API development, and I structure my code so that tracing the data from the entry point is always straightforward.

I wanted to try a new frontend framework. Next.js is popular, so I picked it up and opened a colleague's project to learn by example. He had `route.ts` files for API routes, `global.css`, some `module.css` files (he likes writing CSS by hand), and a structure that made no sense to me. I checked `layout.tsx`, Ctrl+clicked into components, and still couldn't orient myself. There was no `main.py` equivalent staring back at me.

I opened Claude Code and started asking questions. The key insight came quickly: **every page you see in the browser has a `page.tsx` file in the code.** That was my `main.py`. Once I had that anchor, I started visiting pages, reading top to bottom, and using the imports at the top to follow dependencies, exactly how I navigate Python.

I decided to rebuild the project from scratch using patterns that felt natural to a Python developer. Here's what emerged.

---

## The Core Pattern: `page.tsx` Is Your Entry Point

In Python, `main.py` is where you look first. In Next.js, `page.tsx` serves the same role. I made a rule for myself: **all data fetching is visible in the page file.** Instead of scattering fetch calls across `route.ts` handlers and client components, I created a single `api.ts` utility and imported it directly into each page.

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

Now every page reads the same way, like a Python script with imports at the top, data fetching in the middle, and rendering at the bottom:

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

I used Tailwind CSS because the styling lives directly in the JSX, so there's no jumping between files. Each `page.tsx` is self-contained. Another page looks almost identical:

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

---

## Authentication: Login, Cookies, and Server Actions

For login, I needed a form that submits data to the backend and stores a JWT. Next.js has "Server Actions", which are functions that run on the server when a form is submitted. I colocated the action with the page in the same folder, which felt natural (like keeping a helper function next to the script that uses it).

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

The login page itself is minimal:

```tsx
// app/login/page.tsx
import { loginAction } from "./actions";

export default function LoginPage() {
  return (
    <main className="p-10 max-w-md mx-auto">
      <h1 className="text-2xl font-bold mb-6">Sign In</h1>
      <form action={loginAction} className="space-y-4">
        <input name="email" type="email" placeholder="Email"
               className="w-full border p-2 rounded" required />
        <input name="password" type="password" placeholder="Password"
               className="w-full border p-2 rounded" required />
        <button type="submit"
                className="w-full bg-blue-500 text-white py-2 rounded">
          Sign In
        </button>
      </form>
    </main>
  );
}
```

---

## The Three Layers of Auth (and Why They're Not Redundant)

In the original project, I noticed a `proxy.ts` file listing public and protected routes, auth checks inside `page.tsx`, and `useContext` for user info in client components. The functionality seemed to overlap. I asked the LLM about it and the answer clarified something important: these three layers run in different environments, at different times, and solve different problems.

| Layer | Where It Runs | When It Runs | What It Can Do | What It Cannot Do |
|---|---|---|---|---|
| **Proxy** (`proxy.ts`) | Edge/Node, before routing | Before any page renders | Redirect, rewrite, set headers | Access your backend API in depth, render UI |
| **Server Component** (`page.tsx`) | Node.js server, during render | When the page is requested | Call your backend, verify roles, render UI | Respond to browser events |
| **Client Context** (`useContext`) | Browser, after hydration | After the page loads | Toggle UI, personalize display | Protect routes, guard data |

Think of it as: **Proxy** = the front door bouncer (checks your ID fast). **Server Component** = the room-level security guard (verifies your access fully). **Client Context** = the name badge you wear inside (personalization only, protects nothing).

### Layer 1: Proxy - Fast Redirects

The proxy runs before your page renders. It handles optimistic checks: "Does this person have a cookie at all?" It should **not** be your only security layer. A 2025 vulnerability showed that middleware-based auth could be bypassed, which is partly why Next.js renamed it to "proxy."

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

### Layer 2: Server Component - Real Authorization

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

### Layer 3: Client Context - UI Personalization Only

Client context makes user info available to client components for display. It protects nothing; a malicious user can modify client-side state.

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

Bridge it in your layout. The server fetches the real user, then passes it to the client:

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

### Which Layers Do You Need?

| Question | Answer |
|---|---|
| Can I skip the proxy and just check in `page.tsx`? | **Yes.** This is the most secure approach. The proxy just makes redirects faster. |
| Can I rely only on the proxy? | **No.** The proxy can be bypassed and can't call your backend to verify tokens properly. |
| Can I use `useUser()` context for access control? | **No.** Client state is modifiable. Use it only for UI personalization. |
| Do I need all three? | **Proxy + server component checks** is the recommended combo. Client context is optional; add it when client components need user info for display. |

---

## When I Hit the "Client vs. Server" Wall

Things were going smoothly until I needed a Navbar with interactive state, like a sidebar that could open and close. I kept getting errors because I was trying to use `useState` inside a Server Component (`page.tsx`). The LLM suggested I extract the interactive part into a separate client component and import it into the page. That's when it clicked: **that's what all those component files were for** in my colleague's project.

The LLM also suggested a discipline that fit my style: keep client components "pure", with no `fetch` or `useEffect` in them. Data still flows through the page, and the component just handles interactivity. This way, I can always trace where data comes from by looking at `page.tsx`.

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

The data (`folders`) is fetched in the page. The component just receives it as children and handles the toggle. Clean separation.

---

## Input Validation with Zod

Once I had the core patterns down, I needed form validation. In Python, I'd reach for Pydantic. The Next.js equivalent is Zod. Same idea, different syntax. I validated form data inside Server Actions before calling the backend:

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
          <input name="tags" placeholder="react, nextjs, tutorial"
                 className="mt-1 w-full border p-2 rounded" />
        </div>
        <button type="submit"
                className="bg-blue-500 text-white px-6 py-2 rounded">
          Publish
        </button>
      </form>
    </main>
  );
}
```

### Showing Validation Errors with `useActionState`

To display field-level errors, I used `useActionState` in a client component. This is one of the cases where a client component is justified because it needs to react to form submission state:

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
        {state.errors?.title && (
          <p className="mt-1 text-sm text-red-600">{state.errors.title[0]}</p>
        )}
      </div>
      <div>
        <label className="block text-sm font-medium">Content</label>
        <textarea name="content" rows={6} className="mt-1 w-full border p-2 rounded" />
        {state.errors?.content && (
          <p className="mt-1 text-sm text-red-600">{state.errors.content[0]}</p>
        )}
      </div>
      <div>
        <label className="block text-sm font-medium">Tags</label>
        <input name="tags" placeholder="react, nextjs"
               className="mt-1 w-full border p-2 rounded" />
        {state.errors?.tags && (
          <p className="mt-1 text-sm text-red-600">{state.errors.tags[0]}</p>
        )}
      </div>
      <button type="submit" disabled={isPending}
              className="bg-blue-500 text-white px-6 py-2 rounded disabled:opacity-50">
        {isPending ? "Publishing..." : "Publish"}
      </button>
      {state.message && <p className="text-red-600 font-medium">{state.message}</p>}
    </form>
  );
}
```

### Validating API Responses

Zod also validates data coming *from* the backend, like Pydantic parsing an API response:

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

---

## Error Handling

Next.js has file-based conventions for errors that reminded me of how Python frameworks handle exceptions, but with files instead of decorators.

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
      <button onClick={() => reset()}
              className="mt-4 bg-blue-500 text-white px-4 py-2 rounded">
        Try Again
      </button>
    </div>
  );
}
```

Error boundaries use React's `componentDidCatch` lifecycle under the hood, which only works on the client, hence the required `"use client"`. But notice it has no `useEffect` or `fetch`. It's just a UI shell.

### Inline Error Handling

For finer control, like showing a fallback for one failed API call without crashing the whole page:

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

---

## Loading States

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

When different parts of your page have different load times, wrap the slow parts individually. This is where the pattern gets powerful. The page still reads top-to-bottom, but each section loads independently:

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

---

## WebSockets: The One Place Client State Is Unavoidable

Real-time features like chat genuinely need client-side state. There's no server-side alternative for maintaining a WebSocket connection. But even here, I kept the pattern: the page handles auth and data fetching, then hands everything off to a client component.

### Client Hook

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

### The Page (Linear Flow Preserved)

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

---

## A Summary of the Mental Model

**Server state** (data from your backend) is handled by the Data Access Layer (`lib/dal.ts`) with React `cache`. Call `getCurrentUser()` or `fetchFromBackend()` in any server component. It's memoized per request, so the API is hit once.

**Client state** (UI-only state like the current tab, sidebar open/closed, search filters) has two options:

- **URL Query Params**: For state that should survive page refreshes and be shareable. Server Components can read `searchParams` directly, no client hooks needed.
- **React Context**: For ephemeral UI state that client components need to share (like user info for display, as shown in the auth section).

> **Tip**: Before reaching for `useState` or Context, ask: "Should this state be in the URL?" If the user would expect to bookmark or share it (filters, pagination, tabs), put it in `searchParams`.