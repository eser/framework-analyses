# Redwood SDK Deep Analysis

**Framework**: rwsdk (Redwood SDK) **Version Analyzed**: 1.0.0-beta.41
**Analysis Date**: December 2025 **Repository**:
https://github.com/redwoodjs/sdk

---

## Executive Summary

Redwood SDK (rwsdk) is a full-stack React 19 framework purpose-built for
Cloudflare Workers. It provides a complete solution for building server-driven
web applications at the edge, combining React Server Components (RSC), streaming
Server-Side Rendering (SSR), and real-time capabilities through Cloudflare's
Durable Objects.

### Key Differentiators

- **Edge-Native Architecture**: Built from the ground up for Cloudflare Workers,
  not adapted from Node.js patterns
- **React Server Components**: First-class RSC support with streaming payloads
  and progressive hydration
- **Stream Stitching**: Sophisticated 6-phase algorithm that interleaves
  document and app streams for optimal Time-to-First-Byte
- **Type-Safe Routing**: Compile-time route validation with the `linkFor<App>()`
  utility
- **Vite 7.x Integration**: Multi-environment build system with 20+ custom
  plugins
- **Real-Time Built-In**: WebSocket support via Durable Objects without
  additional infrastructure

### Technology Stack

| Layer              | Technology                       |
| ------------------ | -------------------------------- |
| Runtime            | Cloudflare Workers (V8 isolates) |
| UI Library         | React 19.2.3                     |
| RSC Implementation | react-server-dom-webpack 19.2.3  |
| Build Tool         | Vite 7.2.x                       |
| Database           | Kysely ORM + D1/Durable Objects  |
| Package Manager    | pnpm 10.x                        |
| Language           | TypeScript 5.9.x (strict mode)   |

---

## Core Architecture & Runtime

### Application Entry Point

The framework's entry point is the `defineApp()` function, which creates a
Cloudflare Workers fetch handler from route definitions:

```typescript
// src/worker.tsx
import { defineApp, render, route } from "rwsdk/worker";
import { Document } from "./Document";

export default defineApp([
  render(Document, [
    route("/", () => <HomePage />),
    route("/users/:id", ({ params }) => <UserPage id={params.id} />),
  ]),
]);
```

The `defineApp()` function returns an `AppDefinition` object containing:

1. **`__rwRoutes`**: The route definition array (used for type-safe routing)
2. **`fetch`**: The Cloudflare Workers fetch handler

### Request Lifecycle

When a request arrives, the following sequence executes:

```
Request → fetch() handler
    ├── 1. Initialize webpack require for RSC
    ├── 2. Handle asset requests (/assets/*)
    ├── 3. Create RwContext with nonce, Document, databases
    ├── 4. Create RequestInfo context
    ├── 5. runWithRequestInfo() - wrap in AsyncLocalStorage
    ├── 6. router.handle() - match route and execute
    │       ├── Execute global middleware
    │       ├── Match path pattern
    │       ├── Execute route middleware
    │       └── Render page component
    └── 7. Return Response (streaming HTML or RSC payload)
```

### Request Context Management

The framework uses Node.js's `AsyncLocalStorage` to provide request context
throughout the async call chain:

```typescript
// sdk/src/runtime/requestInfo/worker.ts
const requestInfoStore = defineRwState(
  "requestInfoStore",
  () => new AsyncLocalStorage<Record<string, any>>(),
);

export function runWithRequestInfo<Result>(
  nextRequestInfo: RequestInfo,
  fn: () => Result,
): Result {
  return requestInfoStore.run(nextRequestInfo, fn);
}

export function getRequestInfo(): RequestInfo {
  const store = requestInfoStore.getStore();
  if (!store) {
    throw new Error("Request context not found");
  }
  return store as RequestInfo;
}
```

### RequestInfo Interface

Every route handler receives a `RequestInfo` object:

```typescript
interface RequestInfo<Params = any, AppContext = DefaultAppContext> {
  request: Request; // The incoming fetch Request
  params: Params; // Route parameters (e.g., { id: "123" })
  ctx: AppContext; // User-defined application context
  rw: RwContext; // Framework internals
  cf: ExecutionContext; // Cloudflare execution context
  response: ResponseInit; // Mutable response headers/status
  isAction: boolean; // True if this is a server action call
}
```

### RwContext Internal State

The `RwContext` object tracks rendering state:

```typescript
type RwContext = {
  nonce: string; // CSP nonce for inline scripts
  Document: React.FC<DocumentProps>; // Document component
  rscPayload: boolean; // Whether to inject RSC payload
  ssr: boolean; // Whether to SSR the page
  layouts?: React.FC<LayoutProps>[]; // Active layout components
  databases: Map<string, Kysely>; // Database connections
  scriptsToBeLoaded: Set<string>; // Client component scripts
  entryScripts: Set<string>; // Entry point scripts
  inlineScripts: Set<string>; // Inline script content
  pageRouteResolved: PromiseWithResolvers<void>; // Route completion signal
  actionResult?: unknown; // Server action return value
};
```

### Global State Registry

Framework-level singletons are managed through a keyed registry:

```typescript
// sdk/src/runtime/state.ts
const state: Record<string, any> = {};

export function defineRwState<T>(key: string, initializer: () => T): T {
  if (!(key in state)) {
    state[key] = initializer();
  }
  return state[key];
}
```

This pattern ensures singleton instances survive across hot module replacement
during development.

### Module Exports Structure

The SDK exposes 63 subpath exports for different use cases:

| Export                  | Purpose                 | Environment |
| ----------------------- | ----------------------- | ----------- |
| `rwsdk/worker`          | App definition, routing | Worker      |
| `rwsdk/client`          | Client hydration        | Browser     |
| `rwsdk/router`          | Routing utilities       | All         |
| `rwsdk/auth`            | Authentication          | Worker      |
| `rwsdk/db`              | Database (Kysely)       | Worker      |
| `rwsdk/realtime/worker` | WebSocket server        | Worker      |
| `rwsdk/realtime/client` | WebSocket client        | Browser     |
| `rwsdk/vite`            | Build plugins           | Build       |

---

## Build System & Bundling

### Vite Multi-Environment Architecture

Redwood SDK uses Vite 7.x's multi-environment feature to build three separate
bundles:

```
┌─────────────────────────────────────────────────────────────┐
│                    Vite Build System                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Worker    │  │    SSR      │  │   Client    │         │
│  │ Environment │  │ Environment │  │ Environment │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│        │                │                │                  │
│        ▼                ▼                ▼                  │
│   dist/worker/     dist/__inter-     dist/client/          │
│   index.js         mediate_builds/   *.js + manifest       │
│                    ssr_bridge.js                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Environment-Specific Resolve Conditions

Each environment has tailored import resolution:

```typescript
// Worker environment
resolve: {
  conditions: ["workerd", "react-server", "module", "node"],
  noExternal: true
}

// Client environment
resolve: {
  conditions: ["browser", "module"]
}

// SSR environment
resolve: {
  conditions: ["workerd", "module", "browser"],
  noExternal: true
}
```

### Multi-Phase Production Build

The `buildApp()` orchestrator runs five sequential phases:

```
Phase 1: Plugin Setup
    │   Purpose: Allow plugins to generate code before scanning
    │   Output: None (write: false)
    ▼
Phase 2: Directive Scanning
    │   Purpose: Identify "use client" and "use server" files
    │   Output: Populates clientFiles and serverFiles sets
    ▼
Phase 3: Worker Build
    │   Purpose: Bundle server-side code with RSC transforms
    │   Output: dist/worker/index.js (intermediate)
    ▼
Phase 4: SSR Build
    │   Purpose: Pre-compile SSR bridge as IIFE
    │   Output: dist/__intermediate_builds/ssr_bridge.js
    ▼
Phase 5: Client Build
    │   Purpose: Bundle client components
    │   Output: dist/client/*.js + .vite/manifest.json
    ▼
Phase 6: Linker Pass
    │   Purpose: Inject client manifest into worker bundle
    │   Output: Final dist/worker/index.js
    ▼
Build Complete
```

### Key Vite Plugins

The framework registers 20+ plugins in specific order:

```typescript
// sdk/src/vite/redwoodPlugin.mts
return [
  staleDepRetryPlugin(), // Retry failed dep optimizations
  statePlugin(), // SDK state management
  devServerTimingPlugin(), // Performance monitoring
  devServerConstantPlugin(), // Dev server detection
  directiveModulesDevPlugin(), // Track client/server files
  configPlugin(), // Multi-environment config
  ssrBridgePlugin(), // RSC-SSR bridge
  knownDepsResolverPlugin(), // Resolve known dependencies
  cloudflarePreInitPlugin(), // Pre-init for CF plugin
  tsconfigPaths(), // TypeScript path aliases
  cloudflare(), // Cloudflare Workers plugin
  miniflareHMRPlugin(), // RSC-aware HMR
  reactPlugin(), // React transforms
  directivesPlugin(), // "use client"/"use server"
  vitePreamblePlugin(), // React refresh setup
  injectVitePreamble(), // Inject preamble script
  useClientLookupPlugin(), // Virtual client module registry
  useServerLookupPlugin(), // Virtual server module registry
  transformJsxScriptTagsPlugin(), // Script tag transforms
  moveStaticAssetsPlugin(), // Asset organization
  prismaPlugin(), // Prisma support
  linkerPlugin(), // Final asset linking
  directivesFilteringPlugin(), // Filter unused directives
];
```

### Directive Transformation

The `directivesPlugin` transforms files based on their directives:

**Client Component Transformation** (in worker environment):

```typescript
// Input: src/components/Button.tsx
"use client";
export function Button({ onClick, children }) {
  return <button onClick={onClick}>{children}</button>;
}

// Output (worker): Creates reference only
import { registerClientReference } from "react-server-dom-webpack/server";
export const Button = registerClientReference(
  {},
  "src/components/Button.tsx",
  "Button",
);
```

**Server Function Transformation** (in client environment):

```typescript
// Input: src/actions.ts
"use server";
export async function submitForm(data: FormData) {
  // Server-only code
}

// Output (client): Creates proxy
export const submitForm = (...args) =>
  globalThis.__rsc_callServer("src/actions.ts#submitForm", args);
```

### Virtual Module Registries

The build system generates virtual modules that map module IDs to lazy loaders:

```typescript
// virtual:use-client-lookup.js (generated)
export default {
  "src/components/Button.tsx": () => import("src/components/Button.tsx"),
  "src/components/Form.tsx": () => import("src/components/Form.tsx"),
  // ... all client components
};
```

These registries enable RSC to resolve client component references at runtime.

---

## Rendering Pipeline

### RSC + SSR Dual Stream Architecture

Redwood SDK implements a sophisticated rendering pipeline that produces two
streams:

1. **RSC Payload Stream**: React Server Component serialization
2. **HTML Stream**: Server-rendered HTML for initial display

```
                ┌─────────────────┐
                │   Page Element  │
                │   (JSX tree)    │
                └────────┬────────┘
                         │
          ┌──────────────┴──────────────┐
          │                              │
          ▼                              ▼
┌──────────────────┐          ┌──────────────────┐
│ renderToRscStream │          │  Document Shell  │
│ (RSC payload)     │          │  (HTML wrapper)  │
└────────┬─────────┘          └────────┬─────────┘
         │                              │
         │         tee()                │
         ├──────────────┐               │
         │              │               │
         ▼              ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ injectRSC   │  │ createFrom  │  │ renderHtml  │
│ Payload     │  │ Readable    │  │ Stream      │
│ (script)    │  │ Stream      │  │ (document)  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       │                ▼                │
       │      ┌─────────────────┐        │
       │      │ stitchDocument  │◄───────┘
       │      │ AndAppStreams   │
       │      └────────┬────────┘
       │               │
       └───────┬───────┘
               ▼
       ┌─────────────────┐
       │  Final HTML     │
       │  Response       │
       └─────────────────┘
```

### Stream Stitching Algorithm

The `stitchDocumentAndAppStreams()` function implements a 6-phase state machine:

```typescript
type Phase =
  | "read-hoisted" // 1. Buffer hoisted tags from app
  | "outer-head" // 2. Stream document <head>, inject hoisted
  | "inner-shell" // 3. Stream app's initial render
  | "outer-tail" // 4. Stream document <body> scripts
  | "inner-suspended" // 5. Stream app's suspended content
  | "outer-end"; // 6. Close </body></html>
```

**Phase transitions:**

```
┌─────────────┐    hoisted done    ┌─────────────┐
│ read-hoisted│ ─────────────────► │ outer-head  │
└─────────────┘                    └──────┬──────┘
                                          │ </head> found
                                          │ + startMarker
                                          ▼
                                   ┌─────────────┐
                                   │ inner-shell │
                                   └──────┬──────┘
                                          │ endMarker found
                                          ▼
                                   ┌─────────────┐
                                   │ outer-tail  │
                                   └──────┬──────┘
                                          │ </body> found
                                          ▼
                                   ┌──────────────┐
                                   │inner-suspended│
                                   └──────┬───────┘
                                          │ app stream done
                                          ▼
                                   ┌─────────────┐
                                   │ outer-end   │ ──► Stream complete
                                   └─────────────┘
```

**Why this matters:**

This algorithm enables **non-blocking hydration**. The client receives:

1. HTML shell with CSS → **First Paint**
2. Initial app content → **First Contentful Paint**
3. Client scripts → **Hydration begins**
4. Suspended content → **Progressive enhancement**

### RSC Payload Injection

The RSC payload is injected as inline script chunks using `rsc-html-stream`:

```html
<script>
  (self.__FLIGHT_DATA = self.__FLIGHT_DATA || []).push([
    '0:"$L1"\n1:I["src/components/Button.tsx","Button"]\n...',
  ]);
</script>
```

This allows hydration to begin before the full RSC payload arrives.

### Client Hydration

The client initializes via `initClient()`:

```typescript
// sdk/src/runtime/client/client.tsx
export const initClient = async ({ transport, hydrateRootOptions } = {}) => {
  // 1. Set up server action transport
  globalThis.__rsc_callServer = callServer;

  // 2. Create RSC stream from injected data
  const rscPayload = createFromReadableStream(rscStream, { callServer });

  // 3. Create React root with RSC content
  function Content() {
    const [streamData, setStreamData] = useState(rscPayload);
    const [, startTransition] = useTransition();

    // Update mechanism for navigation/actions
    transportContext.setRscPayload = (v) =>
      startTransition(() => setStreamData(v));

    return streamData ? use(streamData).node : null;
  }

  // 4. Hydrate
  hydrateRoot(document.getElementById("hydrate-root"), <Content />);

  // 5. Set up HMR listener
  if (import.meta.hot) {
    import.meta.hot.on("rsc:update", (e) => {
      callServer("__rsc_hot_update", [e.file]);
    });
  }
};
```

---

## Routing System

### Route Definition API

Routes are defined using composable builder functions:

```typescript
import { defineApp, index, layout, prefix, render, route } from "rwsdk/worker";

export default defineApp([
  render(Document, [
    index(() => <HomePage />),

    route("/about", () => <AboutPage />),

    route("/users/:id", ({ params }) => <UserProfile id={params.id} />),

    route("/files/*", ({ params }) => <FileViewer path={params.$0} />),

    prefix("/api", [
      route("/users", {
        get: () => Response.json(users),
        post: async ({ request }) => {
          const data = await request.json();
          return Response.json({ created: true });
        },
      }),
    ]),

    layout(DashboardLayout, [
      route("/dashboard", () => <Dashboard />),
      route("/settings", () => <Settings />),
    ]),
  ]),
]);
```

### Path Pattern Matching

The `matchPath()` function supports:

| Pattern   | Example           | Matches                            |
| --------- | ----------------- | ---------------------------------- |
| Static    | `/about`          | `/about`                           |
| Parameter | `/users/:id`      | `/users/123` → `{ id: "123" }`     |
| Wildcard  | `/files/*`        | `/files/a/b/c` → `{ $0: "a/b/c" }` |
| Combined  | `/api/:version/*` | `/api/v1/users/list`               |

```typescript
// Route matching implementation
export function matchPath(routePath: string, requestPath: string) {
  const pattern = routePath
    .replace(/:[a-zA-Z0-9]+/g, "([^/]+)") // :param → capture group
    .replace(/\*/g, "(.*)"); // * → wildcard capture

  const regex = new RegExp(`^${pattern}$`);
  const matches = requestPath.match(regex);

  if (!matches) return null;

  // Extract named params and wildcards
  const params = {};
  // ... extraction logic
  return params;
}
```

### Method-Based Routing

Routes can handle specific HTTP methods:

```typescript
route("/api/posts/:id", {
  get: ({ params }) => Response.json(getPost(params.id)),
  put: async ({ params, request }) => {
    const data = await request.json();
    return Response.json(updatePost(params.id, data));
  },
  delete: ({ params }) => {
    deletePost(params.id);
    return new Response(null, { status: 204 });
  },
  config: {
    disable405: false, // Return 405 for unhandled methods
    disableOptions: false, // Auto-handle OPTIONS requests
  },
});
```

### Middleware System

Middleware can be applied globally or per-route:

```typescript
// Global middleware
const logMiddleware = async ({ request }) => {
  console.log(`${request.method} ${request.url}`);
  // Return void to continue, Response to short-circuit
};

// Per-route middleware (array syntax)
route("/admin", [
  authMiddleware,
  adminOnlyMiddleware,
  () => <AdminDashboard />,
]);

// Layout middleware
layout(AuthenticatedLayout, [
  // All nested routes require authentication
  route("/profile", () => <Profile />),
  route("/settings", () => <Settings />),
]);
```

### Type-Safe Links

The `linkFor<App>()` utility provides compile-time route validation:

```typescript
// Define app with routes
const app = defineApp([
  route("/", () => <Home />),
  route("/users/:id", ({ params }) => <User id={params.id} />),
  route("/posts/:postId/comments/:commentId", () => <Comment />),
]);

// Create type-safe link function
import { linkFor } from "rwsdk/router";
const link = linkFor<typeof app>();

// Type-safe usage
link("/"); // OK
link("/users/:id", { id: "123" }); // OK
link("/users/:id"); // Error: missing params
link("/users/:id", { id: "123", foo: "x" }); // Error: extra param
link("/invalid"); // Error: route doesn't exist
```

**Type Implementation:**

```typescript
type PathParams<Path extends string> = Path extends
  `${string}:${infer Param}/${infer Rest}`
  ? { [K in Param]: string } & PathParams<Rest>
  : Path extends `${string}:${infer Param}` ? { [K in Param]: string }
  : {};

export type LinkFunction<Paths extends string> = {
  <Path extends Paths>(path: Path, params?: PathParams<Path>): string;
};

export function linkFor<App>(): AppLink<App> {
  return createLinkFunction<AppRoutePaths<App>>();
}
```

---

## Developer Experience

### CLI Tools

The SDK provides three CLI commands:

| Command      | Purpose                  |
| ------------ | ------------------------ |
| `rw-scripts` | Main script runner       |
| `rwsdk`      | Alias for rw-scripts     |
| `rwsync`     | Database synchronization |

Available scripts via `rw-scripts`:

```bash
# Development
npm run dev          # Start Vite dev server with Miniflare
npm run dev:init     # Initialize Wrangler config

# Building
npm run build        # Production build
npm run build:sdk    # Build SDK package only

# Deployment
npm run release      # Deploy to Cloudflare
```

### Hot Module Replacement

The framework implements RSC-aware HMR that handles three environments:

```typescript
// sdk/src/vite/miniflareHMRPlugin.mts
async hotUpdate(ctx) {
  // Track directive changes
  const hasClientDirective = await hasDirective(ctx.file, "use client");
  const hasServerDirective = await hasDirective(ctx.file, "use server");

  // Update module registries if directives changed
  if (clientDirectiveChanged) {
    invalidateModule(server, "worker", "virtual:use-client-lookup.js");
    invalidateModule(server, "client", "virtual:use-client-lookup.js");
  }

  // For worker changes: send RSC update instead of full reload
  if (isWorkerUpdate) {
    ctx.server.environments.client.hot.send({
      type: "custom",
      event: "rsc:update",
      data: { file: ctx.file }
    });
    return []; // Prevent default HMR
  }
}
```

**HMR Flow:**

```
File Changed
    │
    ├── Is SSR file? → Invalidate SSR module graph
    │
    ├── Directive changed? → Invalidate lookup modules
    │
    ├── Is worker file? → Send "rsc:update" custom event
    │                          │
    │                          ▼
    │                    Client receives event
    │                          │
    │                          ▼
    │                    callServer("__rsc_hot_update")
    │                          │
    │                          ▼
    │                    Server re-renders page
    │                          │
    │                          ▼
    │                    Client updates via React transition
    │
    └── Is client file? → Standard Vite HMR
```

### Error Recovery

The HMR plugin includes automatic error recovery:

```typescript
if (hasErrored) {
  // Clear error state
  hasErrored = false;

  // Trigger full reload to recover
  ctx.server.hot.send({
    type: "full-reload",
    path: "*",
  });

  return [];
}
```

### TypeScript Integration

The SDK enforces strict TypeScript:

```json
// sdk/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "composite": true,
    "declaration": true
  }
}
```

Every export includes `.d.ts` type definitions for IDE support.

---

## Framework Features

### Server Functions

Server functions use the `"use server"` directive:

```typescript
// src/actions.ts
"use server";

import { getRequestInfo } from "rwsdk/worker";

export async function createUser(formData: FormData) {
  const { ctx } = getRequestInfo();

  const user = await ctx.db.insertInto("users")
    .values({
      name: formData.get("name"),
      email: formData.get("email"),
    })
    .returning("id")
    .executeTakeFirst();

  return { success: true, userId: user.id };
}
```

**Usage in components:**

```tsx
// Can be called from client components
import { createUser } from "./actions";

function SignupForm() {
  const handleSubmit = async (formData: FormData) => {
    const result = await createUser(formData);
    if (result.success) {
      router.push(`/users/${result.userId}`);
    }
  };

  return <form action={handleSubmit}>...</form>;
}
```

### Client Components

Client components use the `"use client"` directive:

```tsx
// src/components/Counter.tsx
"use client";

import { useState } from "react";

export function Counter({ initial = 0 }) {
  const [count, setCount] = useState(initial);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <button onClick={() => setCount((c) => c - 1)}>-</button>
    </div>
  );
}
```

### Database Layer

The SDK uses Kysely ORM with custom Durable Object dialect:

```typescript
// src/db.ts
import { createDb, SqliteDurableObject } from "rwsdk/db";

// Define database schema
interface Database {
  users: {
    id: string;
    name: string;
    email: string;
    created_at: Date;
  };
  posts: {
    id: string;
    user_id: string;
    title: string;
    content: string;
  };
}

// Durable Object for persistence
export class DB extends SqliteDurableObject {}

// Create typed database instance
export const db = createDb<Database>(env.DB);

// Usage in routes
route("/users", async ({ ctx }) => {
  const users = await ctx.db
    .selectFrom("users")
    .select(["id", "name", "email"])
    .execute();

  return Response.json(users);
});
```

### Real-Time WebSocket

Real-time features use Durable Objects for WebSocket management:

```typescript
// src/worker.tsx
import { defineApp, route } from "rwsdk/worker";
import { RealtimeDurableObject, realtimeRoute } from "rwsdk/realtime/worker";

export class Realtime extends RealtimeDurableObject {
  onMessage(ws: WebSocket, message: string) {
    // Broadcast to all connected clients
    this.broadcast(message);
  }

  onConnect(ws: WebSocket) {
    ws.send(JSON.stringify({ type: "connected" }));
  }
}

export default defineApp([
  realtimeRoute((env) => env.REALTIME),
  // ... other routes
]);
```

**Client usage:**

```typescript
// src/client.tsx
import { initClient, initClientNavigation } from "rwsdk/client";

initClient();

// Upgrade to WebSocket transport
await globalThis.__rw.upgradeToRealtime({ key: "chat-room-1" });
```

### Authentication Patterns

The SDK provides auth building blocks:

```typescript
// src/middleware/auth.ts
import { getRequestInfo } from "rwsdk/worker";

export async function requireAuth({ request }) {
  const { ctx } = getRequestInfo();

  const session = await getSession(request);
  if (!session) {
    return Response.redirect("/login");
  }

  ctx.user = session.user;
}

// Usage
export default defineApp([
  render(Document, [
    route("/login", () => <LoginPage />),

    layout(AuthLayout, [
      requireAuth, // Applied to all nested routes
      route("/dashboard", () => <Dashboard />),
      route("/profile", () => <Profile />),
    ]),
  ]),
]);
```

### Cloudflare Turnstile

Built-in CAPTCHA integration:

```typescript
import { verifyTurnstile } from "rwsdk/turnstile";

route("/api/submit", {
  post: async ({ request }) => {
    const formData = await request.formData();
    const token = formData.get("cf-turnstile-response");

    const verified = await verifyTurnstile({
      token,
      secretKey: env.TURNSTILE_SECRET,
    });

    if (!verified.success) {
      return Response.json({ error: "CAPTCHA failed" }, { status: 400 });
    }

    // Process form...
  },
});
```

---

## Performance & Security

### Streaming Benefits

Streaming SSR provides significant performance advantages:

| Metric                 | Traditional SSR      | Streaming SSR |
| ---------------------- | -------------------- | ------------- |
| Time to First Byte     | Wait for full render | Immediate     |
| First Contentful Paint | After full HTML      | Progressive   |
| Time to Interactive    | After hydration      | Incremental   |

### Edge Caching

Workers run at the edge, reducing latency:

```
Traditional:
User → CDN → Origin Server (single region) → Response

Redwood SDK:
User → Cloudflare Edge (300+ locations) → Response
```

### Bundle Optimization

The build system applies multiple optimizations:

1. **Tree Shaking**: Removes unused exports
2. **Code Splitting**: Separate client/server bundles
3. **Minification**: Production builds are minified
4. **External Modules**: Cloudflare built-ins excluded from bundle

```typescript
// External modules (not bundled)
const externalModules = [
  "cloudflare:email",
  "cloudflare:sockets",
  "cloudflare:workers",
  "cloudflare:workflows",
  // Node.js polyfills handled by @cloudflare/vite-plugin
];
```

### Security Measures

**CSP Nonce:**

Every request generates a unique nonce:

```typescript
// sdk/src/runtime/lib/utils.ts
export const generateNonce = () => {
  const array = new Uint8Array(16);
  crypto.getRandomValues(array);
  return btoa(String.fromCharCode(...array));
};

// Applied to inline scripts
<script nonce={requestInfo.rw.nonce}>...</script>;
```

**XSS Prevention:**

React's JSX escapes content by default. The only escape hatch:

```tsx
<div dangerouslySetInnerHTML={{ __html: content }} />;
```

This is explicitly named to discourage unsafe usage.

**Input Validation:**

Server functions receive raw input; validation is user responsibility:

```typescript
"use server";
import { z } from "zod";

const UserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

export async function createUser(input: unknown) {
  const validated = UserSchema.parse(input);
  // Safe to use validated.name, validated.email
}
```

---

## React 19 Integration

### React Server Components

The SDK uses `react-server-dom-webpack` for RSC:

```typescript
// Package dependencies
"react": ">=19.2.0-0 <19.3.0",
"react-dom": ">=19.2.0-0 <19.3.0",
"react-server-dom-webpack": ">=19.2.0-0 <19.3.0"
```

### Server/Client Boundary Detection

Components are identified as client references via a property:

```typescript
// sdk/src/runtime/lib/router.ts
const isClientReference = (Component: React.FC) => {
  return Object.prototype.hasOwnProperty.call(Component, "$$isClientReference");
};
```

This property is set during the directive transformation:

```typescript
// Transformed client component in worker environment
import { registerClientReference } from "react-server-dom-webpack/server";

export const Button = registerClientReference(
  {}, // Placeholder object
  "src/components/Button.tsx", // Module ID
  "Button", // Export name
);
Button.$$isClientReference = true;
```

### Hydration Strategy

The client uses React 19's `use()` hook for RSC data:

```typescript
function Content() {
  const [streamData, setStreamData] = useState(rscPayload);

  // use() suspends until RSC payload resolves
  return streamData ? use(streamData).node : null;
}
```

Navigation triggers a transition:

```typescript
const [, startTransition] = useTransition();

// On navigation
startTransition(() => {
  setStreamData(newRscPayload);
});
```

### Props Serialization

Client components receive only serializable props:

```typescript
// In route handler
const createPageElement = (requestInfo, Page) => {
  if (isClientReference(Page)) {
    // Only pass serializable props
    const { ctx, params } = requestInfo;
    return <Page ctx={ctx} params={params} />;
  } else {
    // Server components get full requestInfo
    return <Page {...requestInfo} />;
  }
};
```

---

## Cloudflare Integration

### Worker Entry Point

The framework generates a standard Workers fetch handler:

```typescript
// Generated worker
export default {
  fetch: async (request: Request, env: Env, ctx: ExecutionContext) => {
    // Framework handles routing, rendering, responses
  },
};
```

### Environment Bindings

Workers access Cloudflare services via environment bindings:

```typescript
// Declared in wrangler.jsonc
{
  "main": "src/worker.tsx",
  "compatibility_date": "2025-08-21",
  "compatibility_flags": ["nodejs_compat"],
  "assets": { "binding": "ASSETS" },
  "d1_databases": [{ "binding": "DB", "database_id": "xxx" }],
  "durable_objects": {
    "bindings": [{ "name": "REALTIME", "class_name": "Realtime" }]
  }
}
```

**Type declaration:**

```typescript
declare global {
  type Env = {
    ASSETS: Fetcher;
    DB: D1Database;
    REALTIME: DurableObjectNamespace;
  };
}
```

### D1 Database

D1 is Cloudflare's serverless SQLite:

```typescript
// Direct D1 usage
const result = await env.DB.prepare(
  "SELECT * FROM users WHERE id = ?",
).bind(userId).first();

// Via Kysely (recommended)
const user = await db
  .selectFrom("users")
  .where("id", "=", userId)
  .selectAll()
  .executeTakeFirst();
```

### Durable Objects

Durable Objects provide persistent state and WebSocket support:

```typescript
// Definition
export class Counter extends DurableObject {
  private value = 0;

  async increment() {
    this.value++;
    return this.value;
  }

  async getValue() {
    return this.value;
  }
}

// Usage from Worker
const id = env.COUNTER.idFromName("global");
const stub = env.COUNTER.get(id);
const value = await stub.increment();
```

### Service Bindings

Workers can call other Workers:

```typescript
// Binding in wrangler.jsonc
{
  "services": [
    { "binding": "AUTH_SERVICE", "service": "auth-worker" }
  ]
}

// Usage
const response = await env.AUTH_SERVICE.fetch(
  new Request("https://internal/verify", {
    method: "POST",
    body: JSON.stringify({ token })
  })
);
```

### Asset Serving

Static assets are served via the ASSETS binding:

```typescript
// In worker.tsx fetch handler
if (request.url.includes("/assets/")) {
  const url = new URL(request.url);
  url.pathname = url.pathname.slice("/assets/".length);
  return env.ASSETS.fetch(new Request(url.toString(), request));
}
```

---

## Glossary

| Term                 | Definition                                                                  |
| -------------------- | --------------------------------------------------------------------------- |
| **RSC**              | React Server Components - React components that render only on the server   |
| **SSR**              | Server-Side Rendering - Generating HTML on the server for initial page load |
| **Hydration**        | Process of attaching React event handlers to server-rendered HTML           |
| **Stream Stitching** | Interleaving multiple HTML streams into a single response                   |
| **Durable Objects**  | Cloudflare's stateful compute primitive with global uniqueness              |
| **D1**               | Cloudflare's serverless SQLite database                                     |
| **Workers**          | Cloudflare's V8 isolate-based serverless compute platform                   |
| **Workerd**          | Open-source Workers runtime (import condition)                              |
| **Miniflare**        | Local development simulator for Cloudflare Workers                          |
| **Wrangler**         | Cloudflare's CLI for Workers development and deployment                     |
| **Directive**        | Special string ("use client"/"use server") marking component/function type  |
| **Virtual Module**   | Vite-generated module that doesn't exist on disk                            |
| **RSC Payload**      | Serialized React component tree sent to client for hydration                |
| **Nonce**            | Single-use token for Content Security Policy                                |
| **Kysely**           | Type-safe SQL query builder for TypeScript                                  |

---

## Appendix: File Structure

```
redwood-sdk/
├── sdk/                      # Core framework package
│   ├── src/
│   │   ├── runtime/          # Runtime code (worker, client, render)
│   │   │   ├── worker.tsx    # defineApp, fetch handler
│   │   │   ├── client/       # Client hydration
│   │   │   ├── render/       # SSR and RSC rendering
│   │   │   ├── lib/          # Router, db, auth, realtime
│   │   │   └── requestInfo/  # AsyncLocalStorage context
│   │   ├── vite/             # Build system plugins
│   │   ├── lib/              # Utilities and helpers
│   │   └── scripts/          # CLI scripts
│   ├── bin/                  # CLI entry points
│   └── dist/                 # Compiled output
├── starter/                  # Project template
├── docs/                     # Documentation site (Astro)
├── addons/                   # Optional extensions
│   └── passkey/              # WebAuthn authentication
├── playground/               # Example applications (27+)
└── examples/                 # Additional examples
```

---

_Generated by analyzing rwsdk v1.0.0-beta.41 source code. For the latest
documentation, visit the official repository._
