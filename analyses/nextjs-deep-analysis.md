# Next.js Framework Deep Analysis

**Version Analyzed:** 16.1.0-canary.31 **Branch:** canary **Analysis Date:**
December 2024 **Repository:** https://github.com/vercel/next.js

---

## Executive Summary

Next.js is a production-grade React meta-framework developed by Vercel that
provides a complete full-stack solution for building web applications. At its
core, Next.js abstracts the complexity of server-side rendering, static
generation, and modern React patterns into a cohesive developer experience.

### Key Capabilities

| Capability                                | Description                                         |
| ----------------------------------------- | --------------------------------------------------- |
| **Server-Side Rendering (SSR)**           | Dynamic HTML generation per request                 |
| **Static Site Generation (SSG)**          | Pre-rendered HTML at build time                     |
| **Incremental Static Regeneration (ISR)** | Background regeneration with stale-while-revalidate |
| **Partial Pre-Rendering (PPR)**           | Hybrid static shell with dynamic holes              |
| **React Server Components (RSC)**         | Server-first component model with streaming         |
| **App Router**                            | File-system based routing with nested layouts       |

### Architecture Highlights

- **Monorepo Structure:** 18 packages managed via pnpm workspaces and Turbo
- **Rust-Based Tooling:** SWC for transpilation, Turbopack for bundling
- **Three Bundler Options:** Webpack 5 (default), Rspack (Rust-based), Turbopack
  (next-gen)
- **React 19 Integration:** Full support for Server Components, Flight protocol,
  and new APIs
- **Edge Runtime:** First-class support for edge computing environments

### Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                      Next.js Framework                       │
├─────────────────────────────────────────────────────────────┤
│  App Router  │  Pages Router  │  API Routes  │  Middleware  │
├─────────────────────────────────────────────────────────────┤
│     React 19     │     React Server Components (RSC)        │
├─────────────────────────────────────────────────────────────┤
│  Webpack 5  │  Turbopack  │  Rspack  │  SWC Compiler        │
├─────────────────────────────────────────────────────────────┤
│  Node.js Runtime  │  Edge Runtime  │  Serverless Functions  │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Core Architecture & Runtime Dynamics

### 1.1 Repository Structure

Next.js is organized as a monorepo containing 18 distinct packages:

| Package                 | Purpose                           |
| ----------------------- | --------------------------------- |
| `next`                  | Core framework (~100KB compiled)  |
| `next-swc`              | SWC compiler bindings (Rust/NAPI) |
| `next-env`              | Environment variable loading      |
| `create-next-app`       | Project scaffolding CLI           |
| `@next/font`            | Font optimization utilities       |
| `@next/mdx`             | MDX integration                   |
| `@next/bundle-analyzer` | Bundle analysis tools             |
| `@next/codemod`         | Automated migration scripts       |
| `eslint-config-next`    | ESLint configuration preset       |
| `eslint-plugin-next`    | Custom ESLint rules               |

The build orchestration uses Turbo for task scheduling with dependency-aware
caching:

```json
{
  "packageManager": "pnpm@9.6.0",
  "workspaces": ["packages/*"],
  "devDependencies": {
    "turbo": "2.5.5",
    "@swc/core": "1.11.24",
    "webpack": "5.98.0"
  }
}
```

### 1.2 Entry Points and CLI Commands

The framework exposes several CLI entry points defined in
`packages/next/src/cli/`:

| Command       | File             | Purpose                     |
| ------------- | ---------------- | --------------------------- |
| `next dev`    | `next-dev.ts`    | Development server with HMR |
| `next build`  | `next-build.ts`  | Production build            |
| `next start`  | `next-start.ts`  | Production server           |
| `next lint`   | `next-lint.ts`   | ESLint integration          |
| `next info`   | `next-info.ts`   | System diagnostics          |
| `next export` | `next-export.ts` | Static HTML export          |

### 1.3 Server Initialization Pipeline

The server bootstrapping follows a layered architecture:

```
Binary Entry (dist/bin/next)
         │
         ▼
    CLI Parser (cli/next-*.ts)
         │
         ▼
    NextServer Wrapper (server/next.ts)
         │
         ▼
    ┌────┴────┐
    │         │
    ▼         ▼
NextDevServer  NextNodeServer
(Development)   (Production)
    │              │
    └──────┬───────┘
           ▼
    BaseServer (Abstract)
    - handleRequest()
    - run()
    - render()
```

**Key Source:** `packages/next/src/server/next.ts`

```typescript
// Server creation factory
export function createServer(options: NextServerOptions): NextServer {
  const server = new NextServer(options);
  return server;
}

// NextServer wraps lazy initialization
class NextServer {
  private serverPromise?: Promise<Server>;

  async prepare(): Promise<void> {
    if (!this.serverPromise) {
      this.serverPromise = this.createServer();
    }
    await this.serverPromise;
  }

  getRequestHandler(): RequestHandler {
    return async (req, res, parsedUrl) => {
      const server = await this.getServer();
      return getTracer().trace(
        NextServerSpan.getRequestHandler,
        () => server.handleRequest(req, res, parsedUrl),
      );
    };
  }
}
```

### 1.4 Request Handling Flow

The core request pipeline in `BaseServer.handleRequestImpl()` processes requests
through 10 distinct stages:

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────┐
│ 1. Matchers Ready                   │  Wait for route compilation
├─────────────────────────────────────┤
│ 2. URL Normalization                │  Slashes, query strings
├─────────────────────────────────────┤
│ 3. RSC Detection                    │  Check RSC_HEADER
├─────────────────────────────────────┤
│ 4. Locale Handling                  │  i18n extraction
├─────────────────────────────────────┤
│ 5. Rewrite Processing               │  beforeFiles, afterFiles
├─────────────────────────────────────┤
│ 6. Dynamic Params                   │  [slug], [...catch]
├─────────────────────────────────────┤
│ 7. Incremental Cache                │  ISR setup
├─────────────────────────────────────┤
│ 8. Metadata Attachment              │  Route matching
├─────────────────────────────────────┤
│ 9. Middleware Execution             │  Edge runtime
├─────────────────────────────────────┤
│ 10. Route Handler                   │  Render response
└─────────────────────────────────────┘
     │
     ▼
HTTP Response
```

**Key Source:** `packages/next/src/server/base-server.ts` (lines 921-1562)

### 1.5 Internal State Management

Next.js uses Node.js `AsyncLocalStorage` for request-scoped state:

```typescript
// Request context storage
const requestAsyncStorage = new AsyncLocalStorage<RequestStore>();

interface RequestStore {
  type: "request" | "prerender" | "cache";
  phase: "action" | "render" | "after";
  waitUntil: (promise: Promise<any>) => void;
  incrementalCache?: IncrementalCache;
  isHmrRefresh?: boolean;
}
```

This enables accessing request context anywhere in the render tree without prop
drilling, powering features like `headers()`, `cookies()`, and server actions.

### 1.6 Lifecycle Hooks

The framework exposes several lifecycle integration points:

| Hook                   | Location             | Purpose                |
| ---------------------- | -------------------- | ---------------------- |
| `prepare()`            | Server startup       | Initialize resources   |
| `after()`              | Post-response        | Background tasks       |
| `middleware`           | Request interception | Edge processing        |
| `generateStaticParams` | Build time           | Static path generation |
| `revalidate`           | Runtime              | Cache invalidation     |

**After Hook Implementation:** `packages/next/src/server/after/`

```typescript
export function after(task: () => Promise<void> | void): void {
  const workUnitStore = workUnitAsyncStorage.getStore();

  if (workUnitStore) {
    workUnitStore.waitUntil(
      Promise.resolve().then(task),
    );
  }
}
```

---

## 2. Build System & Bundling

### 2.1 Bundler Architecture

Next.js supports three bundler backends with a unified configuration interface:

| Bundler       | Selection          | Use Case                          |
| ------------- | ------------------ | --------------------------------- |
| **Webpack 5** | Default            | Stable, ecosystem compatibility   |
| **Rspack**    | `NEXT_RSPACK=1`    | Faster builds, Webpack-compatible |
| **Turbopack** | `--turbopack` flag | Next-gen, Rust-native             |

**Bundler Selection Logic:**
`packages/next/src/shared/lib/get-webpack-bundler.ts`

```typescript
export default function getWebpackBundler(): typeof webpack {
  return process.env.NEXT_RSPACK ? getRspackCore() : webpack;
}
```

### 2.2 Webpack Configuration

The main configuration generator resides in
`packages/next/src/build/webpack-config.ts` (~2,800 lines). Key configuration
aspects:

**Compiler Instances:**

```typescript
// Three separate compilation targets
const compilers = {
  server: getBaseWebpackConfig(dir, {
    compilerType: "server",
    isDevFallback: false,
  }),
  edgeServer: getBaseWebpackConfig(dir, {
    compilerType: "edge-server",
  }),
  client: getBaseWebpackConfig(dir, {
    compilerType: "client",
  }),
};
```

**Optimization Settings:**

```typescript
optimization: {
  emitOnErrors: !dev,
  checkWasmTypes: false,
  nodeEnv: false,
  splitChunks: isNodeServer ? {
    // Vendor chunks per module for faster dev reload
    cacheGroups: {
      vendor: {
        chunks: 'all',
        test: /[\\/]node_modules[\\/]/,
        name: (module) => `vendor-chunks/${getModuleName(module)}`
      }
    }
  } : {
    // Production client chunking
    cacheGroups: {
      framework: { priority: 40 },  // React, Next.js
      lib: { priority: 30 },        // Large modules > 160KB
      commons: { priority: 20 }     // Shared code
    }
  }
}
```

### 2.3 Turbopack Integration

Turbopack runs in a worker thread for isolation:

**Key Source:** `packages/next/src/build/turbopack-build/impl.ts`

```typescript
export async function turbopackBuild(): Promise<TurbopackBuildResult> {
  const bindings = await getBindings();

  const project = await bindings.turbo.createProject({
    projectPath: dir,
    rootPath: config.experimental?.turbopackRoot ?? dir,
    distDir: distDir,
    nextConfig: config,
    env: loadedEnvFiles,
    defineEnv: createDefineEnv({ config, dev: false }),
    buildId,
    encryptionKey,
    memoryLimit: config.experimental?.turbopackMemoryLimit,
  });

  // Persistent file-system caching
  if (isFileSystemCacheEnabledForBuild()) {
    await project.enablePersistentCaching();
  }

  return project.buildApp();
}
```

### 2.4 Code Splitting Strategy

**Client-Side Chunk Groups:**

| Chunk       | Contents                           | Priority |
| ----------- | ---------------------------------- | -------- |
| `framework` | React, react-dom, next/dist/client | 40       |
| `lib`       | node_modules > 160KB               | 30       |
| `commons`   | Shared across 2+ entry points      | 20       |
| `polyfills` | Core-js, whatwg-fetch              | Separate |

**Dynamic Import Handling:**

The `ReactLoadablePlugin`
(`packages/next/src/build/webpack/plugins/react-loadable-plugin.ts`) tracks
dynamic imports:

```typescript
// Generates manifest for code-split chunks
{
  "componentName": {
    "id": "chunk-id",
    "files": ["chunk-hash.js", "chunk-hash.css"],
    "name": "./components/Heavy.tsx"
  }
}
```

### 2.5 CSS Processing Pipeline

**CSS Loader Chain:** `packages/next/src/build/webpack/config/blocks/css/`

```
Source File (*.css, *.scss)
         │
         ▼
┌─────────────────────────┐
│ SASS/SCSS Preprocessor  │  (if applicable)
├─────────────────────────┤
│ PostCSS Processing      │  Autoprefixer, etc.
├─────────────────────────┤
│ CSS Loader              │  URL resolution
│ OR Lightning CSS        │  (experimental)
├─────────────────────────┤
│ Mini CSS Extract        │  (production)
│ OR Style Loader         │  (development)
└─────────────────────────┘
```

**CSS Modules Configuration:**

```typescript
// Class name generation
getCssModuleLocalIdent: ((context, _, localName, options) => {
  const hash = crypto.createHash("md5")
    .update(relativePath + localName)
    .digest("base64")
    .slice(0, 5);
  return `${localName}__${hash}`;
});
```

**Pattern Matching:**

```typescript
const regexCssGlobal = /(?<!\.module)\.css$/;
const regexCssModules = /\.module\.css$/;
const regexSassGlobal = /(?<!\.module)\.(scss|sass)$/;
const regexSassModules = /\.module\.(scss|sass)$/;
```

### 2.6 Static Asset Handling

**Image Loader:** `packages/next/src/build/webpack/loaders/next-image-loader/`

```typescript
export default async function nextImageLoader(content: Buffer) {
  const { width, height } = await getImageSize(content, extension);
  const blurDataURL = await getBlurImage(content, extension, width, height);

  // Emit to /_next/static/media/[hash].[ext]
  const outputPath = `${assetPrefix}/_next/static/media/${name}.${hash}.${ext}`;
  this.emitFile(outputPath, content);

  return `export default ${
    JSON.stringify({
      src: outputPath,
      width,
      height,
      blurDataURL,
    })
  }`;
}
```

**Font Loader:** `packages/next/src/build/webpack/loaders/next-font-loader/`

- Parses font function calls via SWC transform
- Downloads Google Fonts at build time
- Generates @font-face CSS with fallback metrics
- Emits font files to `static/media/[hash].[ext]`

### 2.7 Development Server & HMR

**Hot Module Replacement Architecture:**

```
┌──────────────────────────────────────────┐
│           NextDevServer                  │
├──────────────────────────────────────────┤
│  ┌─────────────────┐  ┌────────────────┐ │
│  │ HotReloaderWebpack │ HotReloaderTurbo│ │
│  └────────┬────────┘  └───────┬────────┘ │
│           │                   │          │
│           ▼                   ▼          │
│  ┌─────────────────────────────────────┐ │
│  │         WebSocket Server            │ │
│  │    (/__nextjs_hmr endpoint)         │ │
│  └─────────────────────────────────────┘ │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│              Browser                      │
│  ┌─────────────────────────────────────┐ │
│  │     React Fast Refresh Runtime      │ │
│  └─────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

**HMR Message Types:** `packages/next/src/server/dev/hot-reloader-types.ts`

```typescript
enum HMR_MESSAGE_SENT_TO_BROWSER {
  ADDED_PAGE,
  REMOVED_PAGE,
  RELOAD_PAGE,
  SERVER_COMPONENT_CHANGES,
  MIDDLEWARE_CHANGES,
  CLIENT_CHANGES,
  SYNC,
  BUILT,
  BUILDING,
  TURBOPACK_MESSAGE,
  SERVER_ERROR,
}
```

---

## 3. Performance Optimizations

### 3.1 Rendering Strategies

Next.js implements multiple rendering strategies selectable per route:

| Strategy | Build Time   | Request Time  | Use Case             |
| -------- | ------------ | ------------- | -------------------- |
| **SSG**  | Full render  | Serve static  | Marketing pages      |
| **SSR**  | None         | Full render   | Personalized content |
| **ISR**  | Full render  | Revalidate    | Blog posts, products |
| **PPR**  | Static shell | Dynamic holes | Hybrid pages         |

### 3.2 Partial Pre-Rendering (PPR)

PPR enables rendering a static shell with dynamic "holes" that stream in:

**Key Source:** `packages/next/src/server/lib/experimental/ppr.ts`

```typescript
export type ExperimentalPPRConfig = boolean | "incremental";

// Enable globally
// next.config.js: { experimental: { ppr: true } }

// Or per-route (incremental mode)
// page.tsx: export const experimental_ppr = true
```

**Staged Rendering Controller:**
`packages/next/src/server/app-render/staged-rendering.ts`

```typescript
enum RenderStage {
  Before = 1, // Initialization
  Static = 2, // Pre-render static content
  Runtime = 3, // First dynamic resolution
  Dynamic = 4, // Full dynamic render
  Abandoned = 5, // Cancelled
}

class StagedRenderingController {
  currentStage: RenderStage = RenderStage.Before;

  advanceStage(stage: RenderStage): void {
    this.currentStage = stage;
    // Notify listeners, trigger streaming
  }
}
```

### 3.3 Streaming with Suspense

Streaming enables sending HTML chunks as they become available:

**Key Source:**
`packages/next/src/server/stream-utils/node-web-streams-helper.ts`

```typescript
export function chainStreams<T>(
  ...streams: ReadableStream<T>[]
): ReadableStream<T> {
  // Fast path: single stream
  if (streams.length === 1) return streams[0];

  const { readable, writable } = new TransformStream();

  // Pipe streams sequentially
  let promise = streams[0].pipeTo(writable, { preventClose: true });

  for (let i = 1; i < streams.length - 1; i++) {
    promise = promise.then(() =>
      streams[i].pipeTo(writable, { preventClose: true })
    );
  }

  // Last stream closes
  promise.then(() => streams[streams.length - 1].pipeTo(writable));

  return readable;
}
```

### 3.4 Caching Strategies

**ISR with Stale-While-Revalidate:**

**Key Source:** `packages/next/src/server/lib/cache-control.ts`

```typescript
export function getCacheControlHeader({
  revalidate,
  expire,
}: CacheControl): string {
  const swrHeader =
    typeof revalidate === "number" && expire && revalidate < expire
      ? `, stale-while-revalidate=${expire - revalidate}`
      : "";

  if (revalidate === 0) {
    return "private, no-cache, no-store, max-age=0, must-revalidate";
  }

  if (typeof revalidate === "number") {
    return `s-maxage=${revalidate}${swrHeader}`;
  }

  return `s-maxage=${CACHE_ONE_YEAR}${swrHeader}`;
}
```

**Fetch Request Deduplication:**

**Key Source:** `packages/next/src/server/lib/dedupe-fetch.ts`

```typescript
export function createDedupeFetch(originalFetch: typeof fetch) {
  const getCacheEntries = React.cache((url: string) => []);

  return function dedupeFetch(resource, options) {
    // Skip dedup for non-GET or requests with signals
    if (options?.signal) return originalFetch(resource, options);

    const cacheKey = generateCacheKey(request);
    const entries = getCacheEntries(url);

    // Return cached promise if exists
    for (const [key, promise] of entries) {
      if (key === cacheKey) {
        return promise.then(() => cloneResponse(cached));
      }
    }

    // Execute and cache
    const promise = originalFetch(resource, options);
    entries.push([cacheKey, promise]);
    return promise;
  };
}
```

### 3.5 Image Optimization

**Key Source:** `packages/next/src/client/image-component.tsx`

```typescript
// Attribute ordering matters for browser optimization
<img
  loading={loading} // Before src: enables lazy loading
  sizes={sizes}
  srcSet={srcSet}
  src={src} // Last: prevents premature fetch
  ref={ref}
/>;

// React 19 compatible fetchPriority
const getDynamicProps = (fetchPriority?: string) => {
  if (isReact19) {
    return { fetchPriority }; // camelCase
  }
  return { fetchpriority: fetchPriority }; // lowercase
};
```

**Optimization Strategies:**

- Automatic WebP/AVIF format selection
- Blur placeholder generation at build time
- Lazy loading with Intersection Observer
- Priority loading for above-the-fold images
- Responsive srcset generation

### 3.6 Script Loading Strategies

**Key Source:** `packages/next/src/client/script.tsx`

```typescript
interface ScriptProps {
  strategy?: "beforeInteractive" | "afterInteractive" | "lazyOnload" | "worker";
}

const strategies = {
  beforeInteractive: // SSR, blocking
    "Execute before hydration",
  afterInteractive: // Default
    "Execute after hydration",
  lazyOnload: // Idle callback
    "Execute during browser idle time",
  worker: // Partytown
    "Execute in Web Worker",
};

function handleClientScriptLoad(props: ScriptProps) {
  if (props.strategy === "lazyOnload") {
    window.addEventListener("load", () => {
      requestIdleCallback(() => loadScript(props));
    });
  } else {
    loadScript(props);
  }
}
```

---

## 4. Security Measures

### 4.1 CSRF Protection

Server Actions include robust CSRF protection:

**Key Source:** `packages/next/src/server/app-render/csrf-protection.ts`

```typescript
export function isCsrfOriginAllowed(
  originDomain: string,
  allowedOrigins: string[] = [],
): boolean {
  return allowedOrigins.some((pattern) =>
    pattern === originDomain ||
    matchWildcardDomain(originDomain, pattern)
  );
}

function matchWildcardDomain(domain: string, pattern: string) {
  // Prevent overly broad patterns
  if (pattern === "*" || pattern === "**") return false;

  // Support *.example.com and **.example.com
  // *.example.com matches sub.example.com
  // **.example.com matches sub.sub.example.com
}
```

**Server Action Validation:**
`packages/next/src/server/app-render/action-handler.ts`

```typescript
// Lines 606-682: CSRF validation
const originHeader = req.headers["origin"];
const host = parseHostHeader(req.headers);

if (!originDomain) {
  console.warn("Missing origin header");
} else if (originDomain !== host?.value) {
  if (!isCsrfOriginAllowed(originDomain, serverActions?.allowedOrigins)) {
    console.error("CSRF attack detected - aborting action");
    res.statusCode = 500;
    return;
  }
}

// Prevent caching of server actions
res.setHeader(
  "Cache-Control",
  "no-cache, no-store, max-age=0, must-revalidate",
);
```

### 4.2 Content Security Policy Integration

**Key Source:**
`packages/next/src/server/app-render/get-script-nonce-from-header.tsx`

```typescript
const ESCAPE_REGEX = /[&<>"']/;

export function getScriptNonceFromHeader(
  cspHeader: string,
): string | undefined {
  const directive = directives.find((d) => d.startsWith("script-src")) ||
    directives.find((d) => d.startsWith("default-src"));

  const nonce = directive?.split(" ")
    .find((s) => s.startsWith("'nonce-") && s.endsWith("'"))
    ?.slice(7, -1);

  // Reject nonces with escape characters (XSS prevention)
  if (ESCAPE_REGEX.test(nonce)) {
    throw new Error("Nonce contained HTML escape characters");
  }

  return nonce;
}
```

### 4.3 Server Action Encryption

Bound arguments in Server Actions are encrypted:

**Key Source:** `packages/next/src/server/app-render/encryption.ts`

```typescript
async function encryptActionBoundArg(actionId: string, arg: string) {
  const key = await getActionEncryptionKey();
  const iv = crypto.getRandomValues(new Uint8Array(16));

  // Prefix with actionId for integrity
  const plaintext = actionId + arg;
  const ciphertext = await encrypt(key, iv, plaintext);

  // Return iv + ciphertext as base64
  return btoa(String.fromCharCode(...iv, ...ciphertext));
}

async function decryptActionBoundArg(actionId: string, encrypted: string) {
  const key = await getActionEncryptionKey();
  const payload = atob(encrypted);
  const iv = payload.slice(0, 16);
  const ciphertext = payload.slice(16);

  const decrypted = await decrypt(key, iv, ciphertext);

  // Verify integrity
  if (!decrypted.startsWith(actionId)) {
    throw new Error("Invalid Server Action payload");
  }

  return decrypted.slice(actionId.length);
}
```

**Encryption Details:**

- Algorithm: AES-GCM (authenticated encryption)
- IV: 16 bytes random per encryption
- Integrity: ActionId prefix verification
- Key: Generated at build time or from environment

### 4.4 Cross-Site Request Blocking

**Key Source:** `packages/next/src/server/lib/router-utils/block-cross-site.ts`

```typescript
export function blockCrossSite(
  req: IncomingMessage,
  res: ServerResponse,
  allowedDevOrigins: string[],
  hostname: string,
): boolean {
  // Check Sec-Fetch headers
  if (
    req.headers["sec-fetch-mode"] === "no-cors" &&
    req.headers["sec-fetch-site"] === "cross-site"
  ) {
    return blockRequest(res);
  }

  // Validate Origin against allowed list
  const origin = req.headers["origin"]?.toLowerCase();
  const allowedOrigins = [
    "localhost",
    "*.localhost",
    hostname,
    ...allowedDevOrigins,
  ];

  if (origin && !isCsrfOriginAllowed(origin, allowedOrigins)) {
    return blockRequest(res);
  }

  return false; // Allow request
}
```

### 4.5 Taint APIs (Experimental)

Prevent sensitive data from leaking to client components:

**Key Source:** `packages/next/src/server/app-render/rsc/taint.ts`

```typescript
// Mark objects that shouldn't serialize to client
export const taintObjectReference: (
  message: string,
  object: Reference,
) => void = React.experimental_taintObjectReference;

// Mark sensitive values (strings, bigint, buffers)
export const taintUniqueValue: (
  message: string,
  lifetime: Reference,
  value: TaintableUniqueValue,
) => void = React.experimental_taintUniqueValue;

// Usage example
const user = await getUser();
taintObjectReference("User object should not be passed to client", user);
taintUniqueValue("API key must not be sent to client", user, user.apiKey);
```

### 4.6 Security Summary Table

| Threat               | Mitigation                | Location                           |
| -------------------- | ------------------------- | ---------------------------------- |
| CSRF                 | Origin/Host validation    | `csrf-protection.ts`               |
| XSS                  | CSP nonce validation      | `get-script-nonce-from-header.tsx` |
| Data tampering       | AES-GCM encryption        | `encryption.ts`                    |
| Cross-site embedding | Sec-Fetch validation      | `block-cross-site.ts`              |
| Data leakage         | Taint APIs                | `taint.ts`                         |
| Body attacks         | Size limits (1MB default) | `action-handler.ts`                |

---

## 5. Framework Features

### 5.1 Routing System

Next.js provides two routing paradigms:

**App Router (Recommended):**

```
app/
├── layout.tsx          # Root layout
├── page.tsx            # Home page
├── about/
│   └── page.tsx        # /about
├── blog/
│   ├── layout.tsx      # Blog layout
│   ├── page.tsx        # /blog
│   └── [slug]/
│       └── page.tsx    # /blog/:slug
└── (marketing)/        # Route group
    └── pricing/
        └── page.tsx    # /pricing
```

**Pages Router (Legacy):**

```
pages/
├── index.tsx           # /
├── about.tsx           # /about
├── blog/
│   ├── index.tsx       # /blog
│   └── [slug].tsx      # /blog/:slug
└── api/
    └── users.ts        # /api/users
```

### 5.2 Route Modules

**Key Source:** `packages/next/src/server/route-modules/`

```typescript
// App Page Module
class AppPageRouteModule extends RouteModule {
  render(req, res, context): Promise<RenderResult> {
    return renderToHTMLOrFlight(
      req,
      res,
      context.page,
      context.query,
      context.renderOpts,
    );
  }
}

// App Route Module (API handlers)
class AppRouteRouteModule extends RouteModule {
  async execute(req, context): Promise<Response> {
    const handler = this.handlers[req.method];
    return handler(req, context);
  }
}

// Pages Route Module
class PagesRouteModule extends RouteModule {
  render(req, res, context): Promise<RenderResult> {
    return renderToHTML(
      req,
      res,
      context.page,
      context.query,
      context.renderOpts,
    );
  }
}
```

### 5.3 Data Fetching Patterns

**Server Components (App Router):**

```typescript
// Direct async data fetching
async function ProductPage({ params }) {
  const product = await db.product.findUnique({
    where: { id: params.id },
  });
  return <ProductDetails product={product} />;
}
```

**Server Actions:**

```typescript
// Form actions
async function createPost(formData: FormData) {
  "use server";
  const title = formData.get("title");
  await db.post.create({ data: { title } });
  revalidatePath("/posts");
}

// Client-side invocation
async function handleClick() {
  "use server";
  await incrementCounter();
}
```

**Pages Router (Legacy):**

```typescript
// Static Generation
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data }, revalidate: 60 };
}

// Server-Side Rendering
export async function getServerSideProps(context) {
  const data = await fetchData(context.params.id);
  return { props: { data } };
}
```

### 5.4 Middleware System

**Key Source:** `packages/next/src/server/web/spec-extension/`

```typescript
// middleware.ts (project root or src/)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  // Authentication check
  const token = request.cookies.get("token");
  if (!token) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // Rewrite
  if (request.nextUrl.pathname.startsWith("/api/v1")) {
    return NextResponse.rewrite(
      new URL("/api/v2" + request.nextUrl.pathname.slice(7), request.url),
    );
  }

  // Headers modification
  const response = NextResponse.next();
  response.headers.set("x-custom-header", "value");
  return response;
}

export const config = {
  matcher: ["/dashboard/:path*", "/api/:path*"],
};
```

**Middleware Capabilities:**

| Feature             | Support              |
| ------------------- | -------------------- |
| Rewrites            | Yes                  |
| Redirects           | Yes                  |
| Header modification | Yes                  |
| Cookie access       | Yes                  |
| Request body        | No (Edge limitation) |
| Node.js APIs        | Limited              |

### 5.5 TypeScript Integration

Next.js provides deep TypeScript integration:

```typescript
// Typed route parameters
import { PageProps } from "next";

export default function Page({
  params,
  searchParams,
}: PageProps<{ slug: string }, { page?: string }>) {
  // params.slug: string
  // searchParams.page: string | undefined
}

// Typed metadata
import { Metadata, ResolvingMetadata } from "next";

export async function generateMetadata(
  { params }: PageProps,
  parent: ResolvingMetadata,
): Promise<Metadata> {
  return { title: `Product ${params.id}` };
}

// Typed API routes
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  return NextResponse.json({ message: "Hello" });
}
```

---

## 6. React Integration

### 6.1 React Server Components

Next.js pioneered production RSC support:

**Key Source:** `packages/next/src/server/app-render/app-render.tsx`

```typescript
async function renderToHTMLOrFlight(
  req: BaseNextRequest,
  res: BaseNextResponse,
  pagePath: string,
  query: ParsedUrlQuery,
  renderOpts: RenderOpts,
): Promise<RenderResult> {
  const isRSCRequest = req.headers[RSC_HEADER] === "1";

  if (isRSCRequest) {
    // Return Flight payload (RSC protocol)
    return renderToFlightStream(componentTree, clientModules);
  }

  // Return HTML with embedded Flight data
  return renderToHTMLStream(componentTree, clientModules);
}
```

### 6.2 Flight Protocol

The Flight protocol serializes React component trees for client/server
communication:

```typescript
// Serialization
const serialized = await streamToString(
  renderToReadableStream(args, clientModules, {
    onError(err) {
      error = err;
    },
  }),
);

// Deserialization
const deserialized = await createFromReadableStream(
  new ReadableStream({
    start(controller) {
      controller.enqueue(textEncoder.encode(payload));
      controller.close();
    },
  }),
  {
    serverConsumerManifest: {
      moduleLoading: null,
      moduleMap: rscModuleMapping,
      serverModuleMap: getServerModuleMap(),
    },
  },
);
```

### 6.3 React.cache() Integration

Next.js leverages React.cache() for request-level memoization:

```typescript
// Automatic deduplication
const getData = React.cache(async (id: string) => {
  return await db.find(id);
});

// Multiple calls resolve to single fetch
async function Page({ params }) {
  const data = await getData(params.id); // Fetches
  return <Child id={params.id} />;
}

async function Child({ id }) {
  const data = await getData(id); // Returns cached
  return <div>{data.name}</div>;
}
```

### 6.4 React 19 Features

Next.js supports React 19 features:

| Feature           | Status       | Usage                    |
| ----------------- | ------------ | ------------------------ |
| Server Components | Stable       | Default in App Router    |
| Server Actions    | Stable       | `'use server'` directive |
| useFormStatus     | Stable       | Form state access        |
| useOptimistic     | Stable       | Optimistic updates       |
| use()             | Stable       | Unwrap promises/context  |
| Taint APIs        | Experimental | Security                 |

**React Float API Integration:**

```typescript
// Automatic resource preloading
ReactDOM.preload(src, { as: "script", nonce, crossOrigin });
ReactDOM.preinit(styleSrc, { as: "style" });
```

---

## 7. Developer Experience

### 7.1 CLI Tools

**Commands:**

| Command               | Purpose              |
| --------------------- | -------------------- |
| `npx create-next-app` | Project scaffolding  |
| `next dev`            | Development server   |
| `next build`          | Production build     |
| `next start`          | Production server    |
| `next lint`           | ESLint execution     |
| `next info`           | System diagnostics   |
| `@next/codemod`       | Automated migrations |

### 7.2 Error Handling

**Error Overlay:**

The development error overlay provides:

- Stack traces with source maps
- Component tree visualization
- Quick fix suggestions
- File links to editor

**Error Boundaries:**

```typescript
// app/error.tsx - Segment error boundary
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/global-error.tsx - Root error boundary
"use client";

export default function GlobalError({ error, reset }) {
  return (
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  );
}
```

### 7.3 Fast Refresh

**Key Source:** `packages/next/src/server/dev/hot-reloader-webpack.ts`

Fast Refresh preserves component state during edits:

- Hooks state preserved across edits
- Error recovery without full reload
- CSS changes apply instantly
- Server component changes stream incrementally

### 7.4 Development Optimizations

| Feature               | Benefit                      |
| --------------------- | ---------------------------- |
| On-demand compilation | Only compile accessed routes |
| Persistent caching    | Faster subsequent builds     |
| SWC transpilation     | 17x faster than Babel        |
| Incremental bundling  | Minimal rebuilds             |

---

## 8. Glossary

| Term                    | Definition                                                             |
| ----------------------- | ---------------------------------------------------------------------- |
| **App Router**          | File-system router using `app/` directory with React Server Components |
| **Pages Router**        | Legacy file-system router using `pages/` directory                     |
| **RSC**                 | React Server Components - components that render only on the server    |
| **Flight**              | React's serialization protocol for RSC payloads                        |
| **ISR**                 | Incremental Static Regeneration - background page regeneration         |
| **PPR**                 | Partial Pre-Rendering - static shell with dynamic holes                |
| **SSR**                 | Server-Side Rendering - HTML generation per request                    |
| **SSG**                 | Static Site Generation - HTML generation at build time                 |
| **Turbopack**           | Rust-based incremental bundler by Vercel                               |
| **SWC**                 | Speedy Web Compiler - Rust-based JavaScript/TypeScript compiler        |
| **Rspack**              | Rust-based Webpack-compatible bundler                                  |
| **Server Actions**      | Functions that execute on the server, callable from client             |
| **Middleware**          | Edge functions that run before request handling                        |
| **Route Segment**       | A URL path segment corresponding to a folder                           |
| **Layout**              | Shared UI wrapper that persists across navigations                     |
| **Loading**             | Loading UI shown during segment data fetching                          |
| **Error Boundary**      | Component that catches and displays errors                             |
| **Route Group**         | Folder with `(name)` syntax for organization without URL impact        |
| **Parallel Routes**     | Multiple pages rendered in the same layout simultaneously              |
| **Intercepting Routes** | Routes that intercept navigation to show in current context            |

---

## Appendix: Key File Reference

| Category        | File Path                                                | Lines  | Purpose         |
| --------------- | -------------------------------------------------------- | ------ | --------------- |
| Server Entry    | `packages/next/src/server/next.ts`                       | ~540   | Server wrapper  |
| Request Handler | `packages/next/src/server/base-server.ts`                | ~2,700 | Core pipeline   |
| Node Server     | `packages/next/src/server/next-server.ts`                | ~1,300 | Node.js impl    |
| Build Config    | `packages/next/src/build/webpack-config.ts`              | ~2,800 | Webpack setup   |
| App Render      | `packages/next/src/server/app-render/app-render.tsx`     | ~2,000 | RSC rendering   |
| Actions         | `packages/next/src/server/app-render/action-handler.ts`  | ~520   | Server actions  |
| Dev Server      | `packages/next/src/server/dev/next-dev-server.ts`        | ~1,100 | Development     |
| HMR Webpack     | `packages/next/src/server/dev/hot-reloader-webpack.ts`   | ~2,000 | Hot reload      |
| Image           | `packages/next/src/client/image-component.tsx`           | ~850   | Image component |
| Script          | `packages/next/src/client/script.tsx`                    | ~400   | Script loading  |
| CSRF            | `packages/next/src/server/app-render/csrf-protection.ts` | ~80    | Security        |
| Encryption      | `packages/next/src/server/app-render/encryption.ts`      | ~200   | Bound args      |

---

_Document generated from Next.js v16.1.0-canary.31 source code analysis._
