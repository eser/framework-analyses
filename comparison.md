# Web Framework Comparison Matrix

**Analysis Date:** December 2024/2025 **Frameworks Analyzed:** 7

---

## Executive Summary

This comparison covers seven modern web frameworks spanning different
architectural approaches:

| Framework           | Category                   | Best For                                          |
| ------------------- | -------------------------- | ------------------------------------------------- |
| **Astro**           | Content meta-framework     | Content-heavy websites, blogs, marketing sites    |
| **Elysia**          | Backend API framework      | High-performance Bun APIs, real-time applications |
| **Fresh**           | Full-stack framework       | Deno-native apps, minimal JS footprint            |
| **Next.js**         | Full-stack React framework | Production React apps, enterprise applications    |
| **Redwood SDK**     | Edge-native framework      | Cloudflare Workers apps, real-time edge apps      |
| **Remix 3**         | Composable web framework   | Portable web apps, standards-first development    |
| **TanStack Router** | Routing library            | Type-safe routing, existing React/Vue/Solid apps  |

---

## 1. Core Characteristics

| Framework       | Version       | Runtime                         | UI Library                                          | Build Tool                     | Primary Language |
| --------------- | ------------- | ------------------------------- | --------------------------------------------------- | ------------------------------ | ---------------- |
| Astro           | 5.16.6        | Node.js (>=18.20.8)             | Multi-framework (React, Vue, Svelte, Solid, Preact) | Vite                           | TypeScript       |
| Elysia          | 1.4.19        | Bun (primary), Node.js, Workers | N/A (backend)                                       | tsup                           | TypeScript       |
| Fresh           | 2.2.0         | Deno                            | Preact + @preact/signals                            | ESBuild + Vite plugin          | TypeScript       |
| Next.js         | 16.1.0-canary | Node.js, Edge Runtime           | React 19                                            | Webpack 5 / Turbopack / Rspack | TypeScript       |
| Redwood SDK     | 1.0.0-beta.41 | Cloudflare Workers              | React 19                                            | Vite 7.x                       | TypeScript       |
| Remix 3         | remix-the-web | Node.js, Bun, Deno, Workers     | Custom JSX runtime                                  | tsgo + esbuild                 | TypeScript       |
| TanStack Router | 1.141.5       | Any (framework-agnostic core)   | React, Solid, Vue                                   | Vite (primary)                 | TypeScript       |

---

## 2. Architecture Comparison

### Rendering Strategies

| Framework       | SSG | SSR | ISR | Streaming | Islands | RSC | PPR |
| --------------- | :-: | :-: | :-: | :-------: | :-----: | :-: | :-: |
| Astro           | ✅  | ✅  | ❌  |    ✅     |   ✅    | ❌  | ❌  |
| Elysia          | N/A | N/A | N/A |    N/A    |   N/A   | N/A | N/A |
| Fresh           | ✅  | ✅  | ❌  |    ✅     |   ✅    | ❌  | ❌  |
| Next.js         | ✅  | ✅  | ✅  |    ✅     |   ❌    | ✅  | ✅  |
| Redwood SDK     | ❌  | ✅  | ❌  |    ✅     |   ❌    | ✅  | ❌  |
| Remix 3         | ❌  | ✅  | ❌  |    ✅     |   ❌    | ❌  | ❌  |
| TanStack Router | ✅  | ✅  | ❌  |    ✅     |   ❌    | ❌  | ❌  |

**Legend:** SSG=Static Site Generation, SSR=Server-Side Rendering,
ISR=Incremental Static Regeneration, RSC=React Server Components, PPR=Partial
Pre-Rendering

### Routing System

| Framework       | Routing Type | Dynamic Routes    | Catch-All            | Route Groups | Layouts | Middleware      |
| --------------- | ------------ | ----------------- | -------------------- | ------------ | ------- | --------------- |
| Astro           | File-based   | `[slug].astro`    | `[...path].astro`    | ❌           | ✅      | ✅              |
| Elysia          | Config-based | `.get('/:id')`    | `.get('/*')`         | ❌           | ❌      | ✅ (hooks)      |
| Fresh           | File-based   | `[slug].tsx`      | `[...path].tsx`      | `(group)`    | ✅      | ✅              |
| Next.js         | File-based   | `[slug]/page.tsx` | `[...slug]/page.tsx` | `(group)`    | ✅      | ✅              |
| Redwood SDK     | Config-based | `/users/:id`      | `/files/*`           | ❌           | ✅      | ✅              |
| Remix 3         | Config-based | `/posts/:slug`    | `/files/*path`       | ❌           | ❌      | ✅              |
| TanStack Router | Hybrid       | `$postId.tsx`     | `$.tsx`              | ❌           | ✅      | ✅ (beforeLoad) |

### Hydration Strategy

| Framework       | Hydration Type      | Description                                                |
| --------------- | ------------------- | ---------------------------------------------------------- |
| Astro           | Selective (Islands) | Only `client:*` components hydrate; zero JS by default     |
| Elysia          | N/A                 | Backend framework, no client hydration                     |
| Fresh           | Selective (Islands) | Only `islands/` directory components hydrate               |
| Next.js         | Full + RSC          | RSC components stay server-only; client components hydrate |
| Redwood SDK     | RSC + Streaming     | RSC payload streams; selective component hydration         |
| Remix 3         | Selective           | `hydrated()` function marks interactive components         |
| TanStack Router | Full                | Standard React hydration with state restoration            |

---

## 3. Data Handling

### Data Loading Patterns

| Framework       | Pattern               | Server Actions      | Form Handling | Validation                   |
| --------------- | --------------------- | ------------------- | ------------- | ---------------------------- |
| Astro           | Frontmatter + Actions | ✅ `defineAction()` | ✅            | Zod integration              |
| Elysia          | Route handlers        | ❌                  | ✅            | TypeBox (built-in)           |
| Fresh           | Handlers export       | ❌                  | ✅            | Manual                       |
| Next.js         | Loaders + Actions     | ✅ `'use server'`   | ✅            | Any (Zod common)             |
| Redwood SDK     | Route handlers        | ✅ `'use server'`   | ✅            | Manual (Zod recommended)     |
| Remix 3         | Route handlers        | ❌                  | ✅            | Manual                       |
| TanStack Router | Route loaders         | ✅ (TanStack Start) | ✅            | Zod/Valibot/ArkType adapters |

### Caching Strategy

| Framework       | Data Cache           | Page Cache             | Invalidation          | SWR |
| --------------- | -------------------- | ---------------------- | --------------------- | --- |
| Astro           | Build-time           | CDN/Server             | Adapter-dependent     | ❌  |
| Elysia          | Manual               | Manual                 | Manual                | ❌  |
| Fresh           | BUILD_ID versioned   | 1-year static assets   | Build-based           | ❌  |
| Next.js         | Incremental Cache    | ISR                    | `revalidatePath/Tag`  | ✅  |
| Redwood SDK     | Edge/Durable Objects | CDN                    | Manual                | ❌  |
| Remix 3         | Manual               | Manual                 | Manual                | ❌  |
| TanStack Router | Built-in match cache | `staleTime` / `gcTime` | `router.invalidate()` | ✅  |

---

## 4. Performance Features

### Code Splitting

| Framework       |  Automatic   | Route-based |      Component-based      | Lazy Loading |
| --------------- | :----------: | :---------: | :-----------------------: | :----------: |
| Astro           |      ✅      |     ✅      |       ✅ (Islands)        |      ✅      |
| Elysia          |     N/A      |     N/A     |            N/A            |     N/A      |
| Fresh           |      ✅      |     ✅      |       ✅ (Islands)        |      ✅      |
| Next.js         |      ✅      |     ✅      |     ✅ (`dynamic()`)      |      ✅      |
| Redwood SDK     |      ✅      |     ✅      |     ✅ (client refs)      |      ✅      |
| Remix 3         | ✅ (esbuild) |     ✅      |            ❌             |    Manual    |
| TanStack Router |      ✅      |     ✅      | ✅ (`lazyRouteComponent`) |      ✅      |

### Prefetching & Preloading

| Framework       | Link Prefetch | Viewport Prefetch | Intent Prefetch |    Asset Preload     |
| --------------- | :-----------: | :---------------: | :-------------: | :------------------: |
| Astro           |      ✅       |        ✅         |   ✅ (hover)    | ✅ (`modulepreload`) |
| Elysia          |      N/A      |        N/A        |       N/A       |         N/A          |
| Fresh           |      ❌       |        ❌         |       ❌        |  ✅ (Link headers)   |
| Next.js         |      ✅       |        ✅         |       ❌        |          ✅          |
| Redwood SDK     |      ❌       |        ❌         |       ❌        |          ❌          |
| Remix 3         |      ❌       |        ❌         |       ❌        |          ❌          |
| TanStack Router |      ✅       |        ✅         |       ✅        |          ✅          |

### Streaming Support

| Framework       | HTML Streaming | Data Streaming | Suspense | Progressive Rendering  |
| --------------- | :------------: | :------------: | :------: | :--------------------: |
| Astro           |       ✅       |       ❌       |    ❌    |           ✅           |
| Elysia          |       ❌       |       ✅       |   N/A    |          N/A           |
| Fresh           |       ✅       |       ❌       |    ❌    |           ✅           |
| Next.js         |       ✅       |       ✅       |    ✅    |        ✅ (PPR)        |
| Redwood SDK     |       ✅       |       ✅       |    ✅    | ✅ (6-phase stitching) |
| Remix 3         |       ✅       |       ❌       |    ❌    |           ❌           |
| TanStack Router |       ✅       |  ✅ (NDJSON)   |    ✅    |           ✅           |

---

## 5. Security Features

### XSS Prevention

| Framework       | Auto-Escaping |   HTML Sanitization   | Method                                 |
| --------------- | :-----------: | :-------------------: | -------------------------------------- |
| Astro           |      ✅       |    `html-escaper`     | HTMLString class for safe content      |
| Elysia          |      N/A      |          N/A          | Backend only                           |
| Fresh           |      ✅       |      `@std/html`      | Preact built-in + `escapeHtml()`       |
| Next.js         |      ✅       |    React built-in     | JSX auto-escaping                      |
| Redwood SDK     |      ✅       |    React built-in     | JSX auto-escaping                      |
| Remix 3         |      ✅       | SafeHtml branded type | `html` template tag escapes by default |
| TanStack Router |      ✅       |    React built-in     | Framework delegates to React           |

### Security Mechanisms

| Framework       |    CSRF Protection     | CSP Nonce |  Cookie Signing  |   Input Validation   |         Encryption          |
| --------------- | :--------------------: | :-------: | :--------------: | :------------------: | :-------------------------: |
| Astro           |         Manual         |    ✅     |        ❌        |     Zod schemas      |             ❌              |
| Elysia          |         Manual         |    ❌     |        ❌        |   TypeBox built-in   |             ❌              |
| Fresh           |    ✅ (middleware)     |    ✅     |        ❌        |        Manual        |             ❌              |
| Next.js         | ✅ (Origin validation) |    ✅     |        ❌        |        Manual        | ✅ (AES-GCM for bound args) |
| Redwood SDK     |         Manual         |    ✅     |        ❌        |        Manual        |             ❌              |
| Remix 3         |         Manual         |    ❌     | ✅ (HMAC-SHA256) |        Manual        |             ❌              |
| TanStack Router |          N/A           |    ✅     |       N/A        | Zod/Valibot adapters |             ❌              |

---

## 6. Developer Experience

### TypeScript Support

| Framework       |   Type-Safe Routes    | Type-Safe Params |      Type-Safe Data      |    Generated Types    |
| --------------- | :-------------------: | :--------------: | :----------------------: | :-------------------: |
| Astro           |          ❌           |        ❌        | ✅ (Content Collections) |          ✅           |
| Elysia          |          ✅           |        ✅        |            ✅            | ✅ (7 generic params) |
| Fresh           |          ❌           |        ❌        |            ❌            |          ❌           |
| Next.js         |        Partial        |     Partial      |            ✅            |          ❌           |
| Redwood SDK     | ✅ (`linkFor<App>()`) |        ✅        |            ✅            |          ❌           |
| Remix 3         |          ✅           |        ✅        |            ❌            |          ❌           |
| TanStack Router |          ✅           |        ✅        |            ✅            |    ✅ (route tree)    |

### Development Tools

| Framework       |        HMR        |   Dev Server   | CLI |   DevTools   | Error Overlay |
| --------------- | :---------------: | :------------: | :-: | :----------: | :-----------: |
| Astro           |        ✅         |   ✅ (Vite)    | ✅  | ✅ (Toolbar) |      ✅       |
| Elysia          |        ✅         |       ❌       | ❌  |      ❌      |      ❌       |
| Fresh           |        ✅         |   ✅ (Deno)    | ✅  |      ❌      |      ✅       |
| Next.js         | ✅ (Fast Refresh) |       ✅       | ✅  |      ❌      |      ✅       |
| Redwood SDK     |  ✅ (RSC-aware)   | ✅ (Miniflare) | ✅  |      ❌      |      ❌       |
| Remix 3         |  ❌ (tsx watch)   |       ✅       | ❌  |      ❌      |      ❌       |
| TanStack Router |        ✅         |   ✅ (Vite)    | ✅  |  ✅ (Panel)  |      ✅       |

### CLI Commands

| Framework       | Create                | Dev             | Build             | Preview/Start     |
| --------------- | --------------------- | --------------- | ----------------- | ----------------- |
| Astro           | `npm create astro`    | `astro dev`     | `astro build`     | `astro preview`   |
| Elysia          | `bun create elysia`   | `bun run dev`   | `bun build`       | `bun run start`   |
| Fresh           | `deno init --fresh`   | `deno task dev` | `deno task build` | `deno task start` |
| Next.js         | `npx create-next-app` | `next dev`      | `next build`      | `next start`      |
| Redwood SDK     | Template-based        | `npm run dev`   | `npm run build`   | `npm run release` |
| Remix 3         | N/A                   | `tsx watch`     | `pnpm build`      | `node server.js`  |
| TanStack Router | Template-based        | `vite`          | `vite build`      | `vite preview`    |

---

## 7. Ecosystem & Integration

### Framework Integration

| Framework       | React | Vue | Svelte | Solid | Preact | Alpine |
| --------------- | :---: | :-: | :----: | :---: | :----: | :----: |
| Astro           |  ✅   | ✅  |   ✅   |  ✅   |   ✅   |   ✅   |
| Elysia          |  ❌   | ❌  |   ❌   |  ❌   |   ❌   |   ❌   |
| Fresh           |  ❌   | ❌  |   ❌   |  ❌   |   ✅   |   ❌   |
| Next.js         |  ✅   | ❌  |   ❌   |  ❌   |   ❌   |   ❌   |
| Redwood SDK     |  ✅   | ❌  |   ❌   |  ❌   |   ❌   |   ❌   |
| Remix 3         |  ❌   | ❌  |   ❌   |  ❌   |   ❌   |   ❌   |
| TanStack Router |  ✅   | ✅  |   ❌   |  ✅   |   ❌   |   ❌   |

### Platform Deployment

| Framework       | Node.js | Deno | Bun | Cloudflare | Vercel | Netlify | Docker |
| --------------- | :-----: | :--: | :-: | :--------: | :----: | :-----: | :----: |
| Astro           |   ✅    |  ✅  | ✅  |     ✅     |   ✅   |   ✅    |   ✅   |
| Elysia          |   ✅    |  ❌  | ✅  |     ✅     |   ❌   |   ❌    |   ✅   |
| Fresh           |   ❌    |  ✅  | ❌  |     ❌     |   ❌   |   ❌    |   ✅   |
| Next.js         |   ✅    |  ❌  | ❌  |     ✅     |   ✅   |   ✅    |   ✅   |
| Redwood SDK     |   ❌    |  ❌  | ❌  |     ✅     |   ❌   |   ❌    |   ❌   |
| Remix 3         |   ✅    |  ✅  | ✅  |     ✅     |   ❌   |   ❌    |   ✅   |
| TanStack Router |   ✅    |  ✅  | ✅  |     ✅     |   ✅   |   ✅    |   ✅   |

---

## 8. Architecture Patterns

### Build System Comparison

| Framework       | Primary Bundler | Alternative Bundlers     | Build Speed | Hot Reload   |
| --------------- | --------------- | ------------------------ | ----------- | ------------ |
| Astro           | Vite            | -                        | Fast        | Vite HMR     |
| Elysia          | tsup            | Bun native               | Very Fast   | Bun watch    |
| Fresh           | ESBuild         | Vite plugin              | Fast        | WebSocket    |
| Next.js         | Webpack 5       | Turbopack, Rspack        | Medium      | Fast Refresh |
| Redwood SDK     | Vite 7.x        | -                        | Fast        | RSC HMR      |
| Remix 3         | tsgo + esbuild  | -                        | Fast        | File watch   |
| TanStack Router | Vite            | Webpack, Rspack, esbuild | Fast        | Vite HMR     |

### Lifecycle Hooks

| Framework       | Request Hooks | Build Hooks |   Route Hooks   | Component Hooks |
| --------------- | :-----------: | :---------: | :-------------: | :-------------: |
| Astro           |      ❌       |     12+     |       ✅        |       ❌        |
| Elysia          |      11       |     ❌      |       ❌        |       ❌        |
| Fresh           |      ❌       |     ❌      |       ❌        |       ❌        |
| Next.js         |      ✅       |     ❌      |       ✅        |       ❌        |
| Redwood SDK     |      ❌       |     ❌      |       ❌        |       ❌        |
| Remix 3         |      ✅       |     ❌      |       ❌        |       ❌        |
| TanStack Router |      ❌       |     ❌      | ✅ (beforeLoad) |       ❌        |

---

## 9. Use Case Recommendations

### By Project Type

| Project Type               | Recommended              | Alternative             |
| -------------------------- | ------------------------ | ----------------------- |
| **Marketing/Content Site** | Astro                    | Fresh                   |
| **Blog/Documentation**     | Astro                    | Next.js                 |
| **E-commerce**             | Next.js                  | TanStack Router + Start |
| **Dashboard/Admin**        | Next.js, TanStack Router | Remix 3                 |
| **High-Performance API**   | Elysia                   | Remix 3                 |
| **Real-time Application**  | Elysia, Redwood SDK      | Next.js                 |
| **Edge-First App**         | Redwood SDK              | Remix 3                 |
| **Deno-Native App**        | Fresh                    | Remix 3                 |
| **Multi-Framework Site**   | Astro                    | -                       |
| **Type-Safe Routing**      | TanStack Router          | Remix 3                 |
| **Minimal JS Footprint**   | Astro, Fresh             | Remix 3                 |

### By Team Experience

| Team Background        | Recommended Frameworks                |
| ---------------------- | ------------------------------------- |
| React Experts          | Next.js, TanStack Router, Redwood SDK |
| Backend-First          | Elysia, Remix 3                       |
| Standards-First        | Remix 3, Fresh                        |
| TypeScript Enthusiasts | TanStack Router, Elysia               |
| Content Creators       | Astro                                 |
| Cloudflare Users       | Redwood SDK, Elysia                   |
| Deno Users             | Fresh                                 |

---

## 10. Feature Availability Summary

### Legend

- ✅ Full support
- ⚡ Partial/experimental support
- ❌ Not supported
- N/A Not applicable

### Quick Reference

| Feature          | Astro | Elysia | Fresh | Next.js | Redwood SDK | Remix 3 | TanStack |
| ---------------- | :---: | :----: | :---: | :-----: | :---------: | :-----: | :------: |
| SSR              |  ✅   |  N/A   |  ✅   |   ✅    |     ✅      |   ✅    |    ✅    |
| SSG              |  ✅   |  N/A   |  ✅   |   ✅    |     ❌      |   ❌    |    ✅    |
| ISR              |  ❌   |  N/A   |  ❌   |   ✅    |     ❌      |   ❌    |    ❌    |
| Islands          |  ✅   |  N/A   |  ✅   |   ❌    |     ❌      |   ❌    |    ❌    |
| RSC              |  ❌   |  N/A   |  ❌   |   ✅    |     ✅      |   ❌    |    ❌    |
| Streaming        |  ✅   |   ✅   |  ✅   |   ✅    |     ✅      |   ✅    |    ✅    |
| File Routing     |  ✅   |   ❌   |  ✅   |   ✅    |     ❌      |   ❌    |    ✅    |
| Type-Safe Routes |  ❌   |   ✅   |  ❌   |   ⚡    |     ✅      |   ✅    |    ✅    |
| Server Actions   |  ✅   |  N/A   |  ❌   |   ✅    |     ✅      |   ❌    |    ⚡    |
| WebSocket        |  ❌   |   ✅   |  ❌   |   ❌    |     ✅      |   ❌    |    ❌    |
| Edge Runtime     |  ✅   |   ✅   |  ❌   |   ✅    |     ✅      |   ✅    |    ✅    |
| Multi-Framework  |  ✅   |   ❌   |  ❌   |   ❌    |     ❌      |   ❌    |    ✅    |
| Zero JS Default  |  ✅   |  N/A   |  ✅   |   ❌    |     ❌      |   ⚡    |    ❌    |
| DevTools         |  ✅   |   ❌   |  ❌   |   ❌    |     ❌      |   ❌    |    ✅    |

---

## Appendix: Version Information

| Framework       | Version Analyzed | Repository                        | License |
| --------------- | ---------------- | --------------------------------- | ------- |
| Astro           | 5.16.6           | github.com/withastro/astro        | MIT     |
| Elysia          | 1.4.19           | github.com/elysiajs/elysia        | MIT     |
| Fresh           | 2.2.0            | github.com/denoland/fresh         | MIT     |
| Next.js         | 16.1.0-canary.31 | github.com/vercel/next.js         | MIT     |
| Redwood SDK     | 1.0.0-beta.41    | github.com/redwoodjs/sdk          | MIT     |
| Remix 3         | remix-the-web    | github.com/mjackson/remix-the-web | MIT     |
| TanStack Router | 1.141.5          | github.com/TanStack/router        | MIT     |

---

_Generated from deep analysis documents in the [analyses/](./analyses/)
directory_
