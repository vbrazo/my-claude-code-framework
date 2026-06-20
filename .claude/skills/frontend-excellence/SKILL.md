---
name: frontend-excellence
description: A fast modern frontend — React Server Components, streaming SSR, code splitting, bundle trimming, and Core Web Vitals
---

# A fast modern frontend

## React Server Components

These render on the server and send finished HTML down the wire. Because they never reach the browser, they can read straight from databases, the filesystem, or internal APIs without leaking any of it.

```tsx
// app/products/page.tsx (Server Component by default)
async function ProductsPage() {
  const products = await db.query("SELECT * FROM products WHERE active = true");
  return (
    <main>
      <h1>Products</h1>
      <ProductList products={products} />
      <AddToCartButton />  {/* Client Component */}
    </main>
  );
}
```

Guidelines:
- No `useState`, `useEffect`, or browser APIs inside a Server Component
- Flag interactive components with `'use client'` on the first line of the file
- Only serializable props cross the Server→Client boundary — no functions, no class instances
- Push the `'use client'` boundary as far down the tree as it will go

## Streaming SSR

```tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <Header />  {/* renders immediately */}
      <Suspense fallback={<ChartSkeleton />}>
        <AnalyticsChart />  {/* streams when ready */}
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />  {/* streams independently */}
      </Suspense>
    </div>
  );
}
```

Each `Suspense` boundary streams on its own clock. Wrap the data-fetching components so one slow query can't hold the whole page hostage.

## Code splitting

```tsx
import dynamic from 'next/dynamic';

const HeavyEditor = dynamic(() => import('@/components/Editor'), {
  loading: () => <EditorSkeleton />,
  ssr: false,
});

const AdminPanel = dynamic(() => import('@/components/AdminPanel'));
```

Good split points:
- Route boundaries (free in the Next.js App Router)
- Components shown conditionally — modals, drawers, admin panels
- Weighty libraries — charts, rich-text editors, maps
- Anything below the fold

## Trimming the bundle

```javascript
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@heroicons/react', 'lodash-es'],
  },
};
```

Things to check:
- Build with `npx next build` and read the per-route output size
- Point `@next/bundle-analyzer` at it to find the heavy dependencies
- Drop `moment` for `date-fns` or `dayjs` — roughly 200KB back
- Import the one function you need: `import { debounce } from 'lodash-es/debounce'`
- Animate in CSS rather than JS, so there's no runtime cost
- Let icon libraries tree-shake: `import { Search } from 'lucide-react'`

## Core Web Vitals targets

| Metric | Good | Needs Work | Poor |
|--------|------|------------|------|
| LCP (Largest Contentful Paint) | <2.5s | 2.5-4.0s | >4.0s |
| INP (Interaction to Next Paint) | <200ms | 200-500ms | >500ms |
| CLS (Cumulative Layout Shift) | <0.1 | 0.1-0.25 | >0.25 |

## Improving LCP

- Preload the hero image: `<link rel="preload" as="image" href="..." />`
- Set the `priority` prop on above-the-fold `<Image>` components
- Inline the critical CSS and defer the rest
- Keep above-the-fold content off the client-render path
- Give images explicit `width`/`height` so nothing shifts

## Image handling

```tsx
import Image from 'next/image';

<Image
  src="/hero.jpg"
  alt="Descriptive alt text"
  width={1200}
  height={630}
  priority              // preload for LCP images
  sizes="(max-width: 768px) 100vw, 50vw"
  placeholder="blur"
  blurDataURL={base64}  // inline tiny placeholder
/>
```

- Reach for `next/image` or equivalent — automatic WebP/AVIF and responsive srcset
- Set `sizes` so the browser never pulls an oversized image
- Pair `placeholder="blur"` with a base64 data URL to smooth perceived load
- Let below-the-fold images lazy-load (the default)

## Loading fonts

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',       // show fallback font immediately
  preload: true,
  variable: '--font-inter',
});

export default function RootLayout({ children }) {
  return (
    <html className={inter.variable}>
      <body>{children}</body>
    </html>
  );
}
```

- Let `next/font` load fonts with zero CLS and automatic subsetting
- Set `display: 'swap'` so text never goes invisible mid-load
- Self-host rather than pulling from the Google CDN — one fewer DNS lookup
- Cap it at two font families

## Preventing CLS

- Always give images and videos a `width` and `height`
- Use the `aspect-ratio` CSS property for responsive media boxes
- Reserve space for late-arriving content (ads, embeds) with `min-height`
- Don't inject content above what's already on screen after load
- Apply `contain: layout` to components that resize themselves

## Measuring in the field

```typescript
import { onCLS, onINP, onLCP } from 'web-vitals';

onCLS(console.log);
onINP(console.log);
onLCP(console.log);
```

Trust real-user metrics (RUM) over lab scores alone — Vercel Analytics and Google Search Console both surface field data.
