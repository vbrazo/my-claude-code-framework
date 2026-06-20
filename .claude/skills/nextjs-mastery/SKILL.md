---
name: nextjs-mastery
description: Next.js App Router in depth — Server Components, ISR and caching, middleware, parallel routes, Server Actions, and data fetching
---

# Next.js App Router

## How the app directory is laid out

```
app/
  layout.tsx              # Root layout (wraps all pages)
  page.tsx                # Home route /
  loading.tsx             # Route-level Suspense fallback
  error.tsx               # Route-level error boundary
  not-found.tsx           # Custom 404
  (marketing)/
    about/page.tsx        # /about (grouped without URL segment)
  dashboard/
    layout.tsx            # Nested layout for /dashboard/*
    page.tsx              # /dashboard
    @analytics/page.tsx   # Parallel route slot
    @activity/page.tsx    # Parallel route slot
    settings/
      page.tsx            # /dashboard/settings
  api/
    webhooks/route.ts     # Route handler (POST /api/webhooks)
```

A route group `(name)` tidies up the folder structure without touching the URL; a parallel route `@slot` lets several pages render side by side.

## Server Components and fetching data

```tsx
async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await db.product.findUnique({ where: { id } });

  if (!product) notFound();

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />
      </Suspense>
    </div>
  );
}

async function Reviews({ productId }: { productId: string }) {
  const reviews = await db.review.findMany({ where: { productId } });
  return (
    <ul>
      {reviews.map(r => <li key={r.id}>{r.text} - {r.rating}/5</li>)}
    </ul>
  );
}
```

Components are Server Components unless you opt out. They execute on the server, talk to the database directly, and ship no JavaScript to the browser.

## ISR and caching

```tsx
export const revalidate = 3600;

async function BlogPage() {
  const posts = await fetch("https://api.example.com/posts", {
    next: { revalidate: 3600, tags: ["posts"] },
  }).then(r => r.json());

  return <PostList posts={posts} />;
}
```

```tsx
import { revalidateTag, revalidatePath } from "next/cache";

export async function createPost(formData: FormData) {
  "use server";
  await db.post.create({ data: { title: formData.get("title") as string } });
  revalidateTag("posts");
  revalidatePath("/blog");
}
```

## Middleware

```typescript
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("session")?.value;

  if (request.nextUrl.pathname.startsWith("/dashboard") && !token) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  const response = NextResponse.next();
  response.headers.set("x-request-id", crypto.randomUUID());
  return response;
}

export const config = {
  matcher: ["/dashboard/:path*", "/api/:path*"],
};
```

Middleware runs at the edge ahead of every matched request, so keep it lean.

## Server Actions

```tsx
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
});

export async function updateProfile(prevState: any, formData: FormData) {
  const parsed = schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors };
  }

  await db.user.update({
    where: { email: parsed.data.email },
    data: { name: parsed.data.name },
  });

  revalidatePath("/profile");
  return { success: true };
}
```

## What to avoid

- Slapping `'use client'` on a top-level layout or page
- Fetching on the client for data a Server Component could load
- Reaching for `useEffect` to fetch instead of a Server Component or `use()`
- Leaving slow async components without a `<Suspense>` wrapper
- Loading middleware with heavy logic — it fires on every matched request
- Skipping the `loading.tsx` and `error.tsx` conventions

## Before you ship

- [ ] Server Components are the default; `'use client'` lives only on interactive leaves
- [ ] Data is fetched in Server Components with caching set deliberately
- [ ] `<Suspense>` boundaries wrap each independent async section
- [ ] Key routes have `loading.tsx` and `error.tsx`
- [ ] Middleware stays light — auth, redirects, headers, nothing more
- [ ] Server Actions validate with Zod before any database write
- [ ] `revalidateTag` or `revalidatePath` runs after every mutation
- [ ] Route groups and parallel routes organize the complex layouts
