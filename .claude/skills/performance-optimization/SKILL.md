---
name: performance-optimization
description: Speeding up the web frontend — code splitting, image optimization, caching headers, list virtualization, and Core Web Vitals
---

# Web performance

## Analyzing and splitting the bundle

```typescript
// Dynamic import for route-level code splitting
const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

```javascript
// vite.config.ts - manual chunk splitting
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          charts: ["recharts", "d3"],
          editor: ["@monaco-editor/react"],
        },
      },
    },
  },
});
```

```bash
# Analyze bundle composition
npx vite-bundle-visualizer
npx source-map-explorer dist/assets/*.js
```

## Optimizing images

```tsx
import Image from "next/image";

function ProductImage({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={800}
      height={600}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      placeholder="blur"
      blurDataURL={generateBlurHash(src)}
      loading="lazy"
    />
  );
}
```

```html
<!-- Native lazy loading with aspect ratio -->
<img
  src="product.webp"
  alt="Product photo"
  width="800"
  height="600"
  loading="lazy"
  decoding="async"
  fetchpriority="low"
/>

<!-- Preload LCP image -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />
```

## Caching headers

```typescript
function setCacheHeaders(res: Response, options: CacheOptions) {
  if (options.immutable) {
    res.setHeader("Cache-Control", "public, max-age=31536000, immutable");
    return;
  }

  if (options.revalidate) {
    res.setHeader("Cache-Control", `public, max-age=0, s-maxage=${options.revalidate}, stale-while-revalidate=${options.staleWhileRevalidate ?? 86400}`);
    return;
  }

  res.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
}

app.use("/assets", (req, res, next) => {
  setCacheHeaders(res, { immutable: true });
  next();
});

app.use("/api", (req, res, next) => {
  setCacheHeaders(res, { revalidate: 60, staleWhileRevalidate: 3600 });
  next();
});
```

## Virtualizing large lists

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: "600px", overflow: "auto" }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: "absolute",
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: `${virtualRow.size}px`,
              width: "100%",
            }}
          >
            <ItemRow item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Monitoring Core Web Vitals

```typescript
import { onCLS, onINP, onLCP } from "web-vitals";

function sendMetric(metric: { name: string; value: number; id: string }) {
  navigator.sendBeacon("/api/vitals", JSON.stringify(metric));
}

onCLS(sendMetric);
onINP(sendMetric);
onLCP(sendMetric);
```

- **LCP** (Largest Contentful Paint), target < 2.5s — preload the hero image and trim server response time
- **INP** (Interaction to Next Paint), target < 200ms — break up long tasks and lean on `requestIdleCallback`
- **CLS** (Cumulative Layout Shift), target < 0.1 — give images and embeds explicit dimensions

## What to avoid

- Shipping all the JavaScript upfront instead of splitting by route
- Serving unoptimized images — no WebP/AVIF, no responsive sizes
- Images with no `width`/`height`, which jolt the layout as they load
- `Cache-Control: no-cache` on content-hashed static assets
- Rendering thousands of DOM nodes where a virtualized list belongs
- Blocking the main thread with synchronous computation

## Before you ship

- [ ] Routes are lazy-loaded with dynamic `import()` and Suspense
- [ ] The bundle has been analyzed and vendor chunks are split out
- [ ] Images ship as WebP/AVIF with a responsive `sizes` attribute
- [ ] The LCP image is preloaded with `fetchpriority="high"`
- [ ] Static assets are cached immutable behind content hashes
- [ ] Lists of 100+ items are virtualized
- [ ] Core Web Vitals (LCP, INP, CLS) are monitored in production
- [ ] Nothing render-blocking sits in the critical path
