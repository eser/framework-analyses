# Fresh Framework Deep Analysis

**Version Analyzed:** 2.2.0 (@fresh/core) **Analysis Date:** December 2024
**Runtime:** Deno

---

## Executive Summary

Fresh is a Deno-native web framework built on the **Islands Architecture**
paradigm. It prioritizes server-side rendering with minimal client-side
JavaScript, shipping only the code necessary for interactive components
("islands").

### Key Characteristics

| Aspect     | Implementation                         |
| ---------- | -------------------------------------- |
| UI Library | Preact 10.27+ with @preact/signals     |
| Build Tool | ESBuild (primary) + Vite (plugin)      |
| Rendering  | Server-Side Rendering (SSR) by default |
| Hydration  | Selective (Islands only)               |
| Routing    | File-based with URL patterns           |
| TypeScript | First-class support                    |

### Architecture Philosophy

Fresh follows a "zero-JS-by-default" approach. Pages render entirely on the
server as static HTML. Only components explicitly placed in the `islands/`
directory receive client-side JavaScript, enabling selective hydration. This
results in:

- **Faster initial page loads** - no framework JS unless needed
- **Better SEO** - fully rendered HTML at first paint
- **Progressive enhancement** - works without JavaScript
- **Smaller bundles** - only interactive code is shipped

---

## Core Architecture

### Application Entry Point

The `App<State>` class serves as the central orchestrator. It uses a **command
pattern** to defer route and middleware registration until the handler is built.

```typescript
// packages/fresh/src/app.ts
export class App<State> {
  #commands: Command<State>[] = [];
  config: ResolvedFreshConfig;

  constructor(config: FreshConfig = {}) {
    this.config = {
      root: ".",
      basePath: config.basePath ?? "",
      mode: config.mode ?? "production",
    };
  }

  // Route registration methods return `this` for chaining
  use(...middleware): this {/* ... */}
  get(path, ...middlewares): this {/* ... */}
  post(path, ...middlewares): this {/* ... */}
  route(path, route, config?): this {/* ... */}
  fsRoutes(pattern?): this {/* ... */}

  handler(): (request: Request, info?) => Promise<Response> {
    // Builds router from accumulated commands
    const router = new UrlPatternRouter<MaybeLazyMiddleware<State>>();
    const { rootMiddlewares } = applyCommands(
      router,
      this.#commands,
      this.config.basePath,
    );
    // Returns async handler function
  }
}
```

### Command System

Routes are registered via 8 command types, processed when `handler()` is called:

| Command Type | Purpose                             |
| ------------ | ----------------------------------- |
| `Middleware` | Global or path-scoped middleware    |
| `Route`      | Full route with handler + component |
| `Handler`    | HTTP method-specific handlers       |
| `Layout`     | Layout wrapper components           |
| `Error`      | Error boundary routes               |
| `NotFound`   | 404 handler                         |
| `App`        | Global app wrapper                  |
| `FsRoute`    | File-system discovered routes       |

```typescript
// packages/fresh/src/commands.ts
export const enum CommandType {
  Middleware = "middleware",
  Layout = "layout",
  App = "app",
  Route = "route",
  Error = "error",
  NotFound = "notFound",
  Handler = "handler",
  FsRoute = "fsRoute",
}
```

### Request Lifecycle

```
HTTP Request
     |
     v
App.handler()
     |
     v
URL parsed, method extracted
     |
     v
Router.match(method, url)
     |
     +---> params, handlers, pattern
     |
     v
Context<State> created
     |
     v
runMiddlewares(handlers, ctx)
     |
     +---> Each middleware calls ctx.next()
     |
     v
Response returned
```

### Context Object

The `Context<State>` class is the central state container passed to every
middleware:

```typescript
// packages/fresh/src/context.ts
export class Context<State> {
  readonly config: ResolvedFreshConfig;
  readonly url: URL;
  readonly req: Request;
  readonly route: string | null;
  readonly params: Record<string, string>;
  readonly state: State = {} as State;
  readonly isPartial: boolean;

  data: unknown = undefined;
  error: unknown | null = null;
  next: () => Promise<Response>;

  // Response helpers
  redirect(pathOrUrl: string, status = 302): Response;
  render(vnode, init?, config?): Promise<Response>;
  text(content: string, init?): Response;
  html(content: string, init?): Response;
  json(content: any, init?): Response;
  stream(stream, init?): Response;
}
```

### Router Implementation

Fresh uses a dual-strategy router with static and dynamic route separation:

```typescript
// packages/fresh/src/router.ts
export class UrlPatternRouter<T> implements Router<T> {
  #statics = new Map<string, StaticRouteDef<T>>(); // O(1) lookup
  #dynamics = new Map<string, DynamicRouteDef<T>>(); // URLPattern matching

  match(method: Method, url: URL): RouteResult<T> {
    // 1. Check static routes first (exact pathname match)
    const staticMatch = this.#statics.get(url.pathname);
    if (staticMatch) return; /* ... */

    // 2. Fall back to dynamic routes (pattern matching)
    for (const route of this.#dynamicArr) {
      const match = route.pattern.exec(url);
      if (match) return; /* ... */
    }
  }
}
```

Pattern detection uses regex: `/[*:{}+?()]/` to identify dynamic routes.

### Segment Tree

Middleware and layout inheritance is managed via a hierarchical segment tree:

```typescript
// packages/fresh/src/segments.ts
interface Segment<State> {
  pattern: string;
  middlewares: MaybeLazyMiddleware<State>[];
  layout: { component; config } | null;
  errorRoute: Route<State> | null;
  notFound: Middleware<State> | null;
  app: RouteComponent<State> | null;
  children: Map<string, Segment<State>>;
  parent: Segment<State> | null;
}
```

Middleware execution walks from root to leaf, accumulating handlers.

---

## Build System

### Dual Build Strategy

Fresh supports two build approaches:

1. **ESBuild** - Primary bundler for islands and runtime
2. **Vite Plugin** - Modern development workflow integration

### ESBuild Configuration

```typescript
// packages/fresh/src/dev/esbuild.ts
const bundle = await esbuild.build({
  entryPoints: options.entryPoints,
  platform: "browser",
  target: ["chrome99", "firefox99", "safari15"],
  format: "esm",
  bundle: true,
  splitting: true, // Code splitting enabled
  treeShaking: true, // Dead code elimination
  minify: !options.dev,
  jsx: "automatic",
  jsxImportSource: "preact",

  plugins: [
    preactDebugger(PREACT_ENV),
    buildIdPlugin(options.buildId),
    windowsPathFixer(),
    denoPlugin({
      preserveJsx: true,
      publicEnvVarPrefix: "FRESH_PUBLIC_",
    }),
  ],
});
```

### Bundle Output Structure

```
_fresh/
  static/
    fresh-runtime.js    # Client-side hydration runtime
    island-*.js         # One chunk per island
    chunk-*.js          # Shared chunks
    *.css               # Processed stylesheets
  snapshot.js           # Build metadata
  metafile.json         # ESBuild analysis data
```

### Code Splitting Strategy

Each island becomes a separate entry point:

```typescript
const entryPoints: Record<string, string> = {
  "fresh-runtime": runtimePath, // Always included
  [islandName]: islandSpecifier, // One per island
};
```

The `entryToChunk` map tracks which islands produce which output files.

### Asset Hashing

Fresh uses BUILD_ID for cache invalidation:

```typescript
// BUILD_ID injected via plugin
build.onLoad({
  filter: /.*/,
  namespace: "fresh-internal-build-id",
}, () => {
  return {
    contents: `export const BUILD_ID = "${buildId}";`,
  };
});
```

Assets are served with 1-year cache lifetime, versioned by BUILD_ID.

### Development Server

The dev server provides:

- **Live Reload** via WebSocket at `/_frsh/alive`
- **HMR** for CSS and module changes
- **Error Overlay** with code frame extraction
- **CSS Eager Loading** to prevent FOUC

```typescript
// WebSocket reconnection with backoff
// Sequence: 100ms -> 500ms -> 2000ms
```

---

## Islands Architecture

### Rendering Philosophy

Fresh implements the Islands Architecture pattern:

1. **Server renders full HTML** - Complete page markup
2. **Static content stays static** - No JavaScript overhead
3. **Interactive components hydrate** - Only islands receive JS
4. **Progressive enhancement** - Works without JavaScript

### Server-Side Rendering

The `RenderState` class manages SSR lifecycle:

```typescript
// packages/fresh/src/runtime/server/preact_hooks.ts
export class RenderState {
  nonce: string; // CSP nonce
  partialDepth = 0; // Nested partial tracking
  slots: Array<{ id; name; vnode }> = [];
  islandProps: any[] = []; // Serialized props
  islands = new Set<Island>(); // Used islands
  islandAssets = new Set<string>(); // CSS dependencies

  renderedHtmlTag = false;
  renderedHtmlBody = false;
  renderedHtmlHead = false;
}
```

### DOM Marker System

Fresh embeds HTML comments to mark island boundaries:

```html
<!--frsh:island:Counter:0:-->
<div>
  <button>+1</button>
  <span>0</span>
</div>
<!--/frsh:island-->

<!--frsh:partial:sidebar:0:-->
<aside>...</aside>
<!--/frsh:partial-->

<!--frsh:slot:0:children-->
<p>Slot content</p>
<!--/frsh:slot-->
```

Marker format: `frsh:{type}:{name}:{propsIdx}:{key}`

### Client Hydration

The `boot()` function initializes islands on the client:

```typescript
// packages/fresh/src/runtime/client/reviver.ts
export function boot(
  initialIslands: Record<string, ComponentType>,
  islandProps: string,
) {
  const ctx = createReviveCtx();
  _walkInner(ctx, document.body); // Find all island markers

  // Register island components
  for (const name of Object.keys(initialIslands)) {
    ISLAND_REGISTRY.set(name, initialIslands[name]);
  }

  // Deserialize props with custom handlers
  const allProps = parse<DeserializedProps>(islandProps, CUSTOM_PARSER);

  // Hydrate each island
  for (const root of ctx.roots) {
    const container = createRootFragment(
      root.start.parentNode,
      root.start,
      root.end,
    );
    revive(props, component, container, ctx.slots, allProps);
  }
}
```

### Custom Serialization

Fresh handles complex prop types through custom serialization:

```typescript
// Serialization (server)
const stringifiers: Stringifiers = {
  Signal: (value) => isSignal(value) ? { value: value.peek() } : undefined,
  Computed: (value) => isComputedSignal(value) ? { value: value.peek() } : undefined,
  Slot: (value) => isVNode(value) && value.type === Slot ? { ... } : undefined,
};

// Deserialization (client)
const CUSTOM_PARSER: CustomParser = {
  Signal: (value) => signal(value),
  Computed: (value) => computed(() => value),
  Slot: (value) => ({ kind: SLOT_SYMBOL, name: value.name, id: value.id }),
};
```

### Root Fragment Container

Islands may share the same parent node. Fresh creates virtual containers:

```typescript
// packages/fresh/src/runtime/client/reviver.ts
export function createRootFragment(parent, startMarker, endMarker) {
  return {
    _frshRootFrag: true,
    nodeType: 1,
    parentNode: parent,
    get firstChild() {
      const child = startMarker.nextSibling;
      return child === endMarker ? null : child;
    },
    appendChild(child) {
      parent.insertBefore(child, endMarker); // Insert before end marker
    },
    // ...
  };
}
```

### Async Hydration

Hydration uses non-blocking scheduling:

```typescript
"scheduler" in window
  ? scheduler.postTask(_render) // Priority scheduling API
  : setTimeout(_render, 0); // Fallback
```

---

## Security Measures

### CSRF Protection

The `csrf()` middleware validates cross-origin requests:

```typescript
// packages/fresh/src/middlewares/csrf.ts
export function csrf<State>(options?: CsrfOptions): Middleware<State> {
  return async (ctx) => {
    const { method, headers } = ctx.req;

    // Safe methods pass through
    if (method === "GET" || method === "HEAD" || method === "OPTIONS") {
      return await ctx.next();
    }

    const secFetchSite = headers.get("Sec-Fetch-Site");
    const origin = headers.get("origin");

    // Validate Sec-Fetch-Site header
    if (secFetchSite !== null) {
      if (
        secFetchSite === "same-origin" || secFetchSite === "none" ||
        isAllowedOrigin(origin, ctx)
      ) {
        return await ctx.next();
      }
      throw new HttpError(403);
    }

    // Fallback to Origin header validation
    if (origin !== null && !isAllowedOrigin(origin, ctx)) {
      throw new HttpError(403);
    }

    return await ctx.next();
  };
}
```

### Content Security Policy

The `csp()` middleware sets restrictive default headers:

```typescript
// packages/fresh/src/middlewares/csp.ts
const defaultCsp = [
  "default-src 'self'",
  "script-src 'self' 'unsafe-inline'", // Required for Fresh runtime
  "style-src 'self' 'unsafe-inline'",
  "font-src 'self'",
  "img-src 'self' data:",
  "media-src 'self' data: blob:",
  "worker-src 'self' blob:",
  "connect-src 'self'",
  "object-src 'none'",
  "base-uri 'self'",
  "form-action 'self'",
  "frame-ancestors 'none'",
  "upgrade-insecure-requests",
];
```

CSP nonces are generated per-request: `crypto.randomUUID().replace(/-/g, "")`

### CORS Handling

The `cors()` middleware provides flexible cross-origin configuration:

```typescript
// packages/fresh/src/middlewares/cors.ts
export function cors<State>(options?: CORSOptions<State>): Middleware<State> {
  const opts = {
    origin: "*",
    allowMethods: ["GET", "HEAD", "PUT", "POST", "DELETE", "PATCH"],
    allowHeaders: [],
    exposeHeaders: [],
    ...options,
  };
  // Handles preflight OPTIONS and adds headers to responses
}
```

### XSS Prevention

Fresh employs multiple XSS mitigation strategies:

1. **Preact's built-in escaping** - JSX content is escaped by default
2. **HTML entity encoding** - Via `@std/html` escape function
3. **Script escaping** - `escapeScript()` for inline scripts
4. **Attribute sanitization** - Key props are HTML-escaped

```typescript
// Attribute hook escapes dynamic values
options[OptionsType.ATTR] = (name, value) => {
  if (name === "key") {
    return `${DATA_FRESH_KEY}="${escapeHtml(String(value))}"`;
  }
};
```

### Open Redirect Prevention

The `redirect()` method sanitizes paths:

```typescript
redirect(pathOrUrl: string, status = 302): Response {
  let location = pathOrUrl;

  // Remove double slashes to prevent open redirect
  if (pathOrUrl !== "/" && pathOrUrl.startsWith("/")) {
    const pathname = /* extract pathname */;
    location = `${pathname.replaceAll(/\/+/g, "/")}${search}`;
  }

  return new Response(null, { status, headers: { location } });
}
```

---

## Framework Features

### File-Based Routing

Fresh maps filesystem paths to URL patterns:

| File Path                           | URL Pattern         |
| ----------------------------------- | ------------------- |
| `routes/index.tsx`                  | `/`                 |
| `routes/blog/[slug].tsx`            | `/blog/:slug`       |
| `routes/docs/[[version]]/index.tsx` | `/docs{/:version}?` |
| `routes/old/[...path].tsx`          | `/old/:path*`       |
| `routes/(marketing)/about.tsx`      | `/about` (group)    |

```typescript
// packages/fresh/src/router.ts
export function pathToPattern(path: string): string {
  // [id] -> :id
  // [[id]] -> {:id}?
  // [...rest] -> :rest*
  // (group) -> (removed)
}
```

### Handler Pattern

Routes export handlers for data fetching:

```typescript
// routes/blog/[slug].tsx
export const handlers = define.handlers({
  async GET(ctx) {
    const post = await getPost(ctx.params.slug);
    return page({ post });
  },

  async POST(ctx) {
    const form = await ctx.req.formData();
    // Process form...
    return ctx.redirect("/thanks", 303);
  },
});

export default define.page<typeof handlers>(({ data }) => {
  return <article>{data.post.content}</article>;
});
```

### Middleware System

Middlewares compose via the `ctx.next()` pattern:

```typescript
// Logging middleware
const logger: Middleware = async (ctx) => {
  const start = Date.now();
  const res = await ctx.next();
  console.log(
    `${ctx.req.method} ${ctx.url.pathname} - ${Date.now() - start}ms`,
  );
  return res;
};

// Apply middleware
app.use(logger);
app.use("/api", authMiddleware);
app.use("/admin/*", adminGuard);
```

### Layouts

Layouts wrap child routes with shared UI:

```typescript
// routes/_layout.tsx
export default define.page(({ Component }) => {
  return (
    <div class="container">
      <Header />
      <Component /> {/* Child route renders here */}
      <Footer />
    </div>
  );
});
```

Skip layouts per-route:

```typescript
export const config: RouteConfig = {
  skipInheritedLayouts: true,
};
```

### Partials

Partials enable targeted page updates without full navigation:

```tsx
import { Partial } from "fresh/runtime";

export default function Page() {
  return (
    <main>
      <Partial name="sidebar">
        <Sidebar />
      </Partial>
      <Content />
    </main>
  );
}
```

Partial modes: `replace` (default), `append`, `prepend`

### Error Handling

Fresh provides hierarchical error boundaries:

```typescript
// Error route
app.onError("/api/*", (ctx) => {
  if (ctx.error instanceof HttpError) {
    return ctx.json({ error: ctx.error.message }, { status: ctx.error.status });
  }
  return ctx.json({ error: "Internal error" }, { status: 500 });
});

// 404 handler
app.notFound((ctx) => {
  return ctx.render(<NotFound />, { status: 404 });
});
```

The `HttpError` class enables typed HTTP errors:

```typescript
throw new HttpError(404, "Post not found");
throw new HttpError(403); // Uses default message
```

---

## Preact Integration

### Component Model

Fresh uses Preact as its UI library:

```tsx
import { signal } from "@preact/signals";

// Island component (interactive)
export default function Counter({ start }: { start: number }) {
  const count = signal(start);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => count.value++}>+1</button>
    </div>
  );
}
```

### Signals Support

@preact/signals provides reactive state:

```typescript
// Signals serialize across server/client boundary
const count = signal(0); // Signal<number>
const doubled = computed(() => count.value * 2); // ReadonlySignal<number>
```

### Preact Options Hooks

Fresh hooks into Preact's rendering lifecycle:

```typescript
// packages/fresh/src/runtime/server/preact_hooks.ts
options[OptionsType.VNODE] = (vnode) => {
  // Track ownership, asset hashing, active link detection
};

options[OptionsType.DIFF] = (vnode) => {
  // Island detection, partial tracking, head element hoisting
};

options[OptionsType.RENDER] = (vnode) => {
  // Owner stack tracking for island boundaries
};

options[OptionsType.DIFFED] = (vnode) => {
  // Cleanup after component renders
};
```

### JSX Configuration

Fresh uses Preact's precompiled JSX:

```json
{
  "compilerOptions": {
    "jsx": "precompile",
    "jsxImportSource": "preact",
    "jsxPrecompileSkipElements": [
      "a",
      "img",
      "source",
      "body",
      "html",
      "head",
      "title",
      "meta",
      "script",
      "link",
      "style",
      "base",
      "noscript",
      "template"
    ]
  }
}
```

---

## Developer Experience

### CLI Tools

Fresh provides initialization and scaffolding:

```bash
# Create new project
deno init --fresh my-project

# Development server
deno task dev

# Production build
deno task build

# Start production server
deno task start
```

### Error Overlay

Development mode includes an error overlay with:

- Stack trace display
- Code frame extraction
- Source file linking

```typescript
// packages/fresh/src/runtime/server/preact_hooks.ts
export function ShowErrorOverlay() {
  const searchParams = new URLSearchParams();
  searchParams.append("message", String(error.message));
  searchParams.append("stack", error.stack);

  const codeFrame = getCodeFrame(error.stack, ctx.config.root);
  if (codeFrame) {
    searchParams.append("code-frame", codeFrame);
  }

  return <iframe src={`${basePath}/_frsh/error?${searchParams}`} />;
}
```

### OpenTelemetry Integration

Fresh includes built-in observability:

```typescript
import { trace } from "@opentelemetry/api";

// Automatic span creation for routes
const span = trace.getActiveSpan();
if (span && pattern) {
  span.updateName(`${method} ${pattern}`);
  span.setAttribute("http.route", pattern);
}

// Render span
tracer.startActiveSpan("render", (span) => {
  span.setAttribute("fresh.span_type", "render");
  // ...
});
```

### Live Reload

WebSocket-based live reload during development:

```typescript
// packages/fresh/src/dev/live_reload.ts
// Endpoint: /{basePath}/_frsh/alive
// Reconnection backoff: 100ms -> 500ms -> 2000ms
```

---

## Performance Characteristics

### Bundle Size Optimization

- **Code splitting** - Separate chunks per island
- **Tree shaking** - Dead code elimination
- **Minification** - Production builds only
- **Lazy loading** - Routes and middleware can be lazy

### Rendering Performance

- **Streaming support** - `ctx.stream()` for large responses
- **Partial updates** - Update page sections without full reload
- **Preload hints** - Link headers for modulepreload

```typescript
// Add preload headers
let link = `<${runtimeUrl}>; rel="modulepreload"; as="script"`;
islands.forEach((island) => {
  link += `, <${island.file}>; rel="modulepreload"; as="script"`;
});
headers.append("Link", link);
```

### Caching Strategy

- BUILD_ID-based cache invalidation
- 1-year cache lifetime for static assets
- Content-addressed asset URLs

---

## Glossary

| Term            | Definition                                                 |
| --------------- | ---------------------------------------------------------- |
| **Island**      | Interactive component that receives client-side JavaScript |
| **Partial**     | Page section that can be updated independently             |
| **Handler**     | Server-side function that processes requests               |
| **Middleware**  | Function that intercepts requests/responses                |
| **Context**     | Request-scoped state container                             |
| **Segment**     | Hierarchical route unit for middleware inheritance         |
| **Command**     | Deferred route registration instruction                    |
| **BUILD_ID**    | Unique identifier for each build, used for cache busting   |
| **Slot**        | Placeholder for child content passed to islands            |
| **RenderState** | Server-side rendering context tracking islands and props   |

---

## Source File Reference

| Component        | Location                                            | Lines |
| ---------------- | --------------------------------------------------- | ----- |
| App class        | `packages/fresh/src/app.ts`                         | 481   |
| Context          | `packages/fresh/src/context.ts`                     | 476   |
| Router           | `packages/fresh/src/router.ts`                      | 313   |
| Commands         | `packages/fresh/src/commands.ts`                    | 369   |
| Island hydration | `packages/fresh/src/runtime/client/reviver.ts`      | 578   |
| Server hooks     | `packages/fresh/src/runtime/server/preact_hooks.ts` | 692   |
| ESBuild config   | `packages/fresh/src/dev/esbuild.ts`                 | 241   |
| CSRF middleware  | `packages/fresh/src/middlewares/csrf.ts`            | 114   |
| CSP middleware   | `packages/fresh/src/middlewares/csp.ts`             | 73    |
| CORS middleware  | `packages/fresh/src/middlewares/cors.ts`            | 196   |

---

_Generated from Fresh framework source code analysis_
