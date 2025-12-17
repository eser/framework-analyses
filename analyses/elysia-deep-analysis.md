# Elysia Framework v1.4.19

## Deep Technical Analysis Report

**Version Analyzed:** 1.4.19 **License:** MIT **Primary Author:** saltyAom
**Repository:** https://github.com/elysiajs/elysia **Analysis Date:** December
2024

---

# Executive Summary

Elysia is a TypeScript-first web framework designed for the Bun runtime, with
the tagline "Ergonomic Framework for Humans." At its core, Elysia differentiates
itself through three key innovations:

**1. End-to-End Type Safety:** Unlike traditional web frameworks where types are
often manually maintained or inferred at runtime, Elysia leverages TypeScript's
type system to provide complete type inference from route definition through to
client consumption. The framework uses a sophisticated generic type system with
seven type parameters to track state, routes, and schema definitions throughout
the application lifecycle.

**2. Dual Compilation Modes:** Elysia implements both Ahead-of-Time (AOT) and
Just-in-Time (JIT) compilation strategies. In AOT mode (the default), route
handlers are compiled into optimized JavaScript functions at registration time,
eliminating runtime overhead. This approach generates actual code strings that
are then compiled via the `Function()` constructor, resulting in handlers that
have minimal abstraction overhead.

**3. Static Code Analysis via Sucrose:** The framework includes a custom static
analysis engine called "Sucrose" that parses handler functions to determine
which context properties (body, headers, query, cookies, etc.) are actually
accessed. This inference drives optimization—if a handler doesn't access
`query`, the generated code won't include query parsing logic.

**Performance Philosophy:** Elysia optimizes for request latency over developer
convenience where the two conflict. The framework generates specialized code per
route rather than using generic middleware chains, trades startup time for
faster request handling, and leverages Bun's native capabilities when available.

**Multi-Runtime Support:** While optimized for Bun, Elysia supports Web Standard
runtimes (Node.js via adapters) and Cloudflare Workers through a pluggable
adapter system that abstracts runtime-specific behaviors.

---

# Architecture Overview

## Core Module Structure

The Elysia source is organized into focused modules, with the main class file
containing the bulk of the framework logic:

| File                    | Lines | Responsibility                                       |
| ----------------------- | ----- | ---------------------------------------------------- |
| `src/index.ts`          | 8,285 | Main Elysia class, routing, lifecycle, plugin system |
| `src/compose.ts`        | 2,764 | AOT handler composition, code generation             |
| `src/types.ts`          | 2,673 | TypeScript type definitions and utilities            |
| `src/schema.ts`         | 1,342 | Schema validation with TypeBox integration           |
| `src/sucrose.ts`        | 762   | Static code analysis for inference                   |
| `src/dynamic-handle.ts` | 748   | JIT request handling                                 |
| `src/error.ts`          | 616   | Error classes and validation error handling          |
| `src/context.ts`        | 256   | Request context type definitions                     |
| `src/cookies.ts`        | 461   | Cookie parsing and management                        |

## Module Dependencies

```
index.ts (Main Elysia Class)
    ├── compose.ts ──── sucrose.ts
    ├── dynamic-handle.ts
    ├── schema.ts ──── type-system/
    ├── adapter/
    │   ├── bun/
    │   ├── web-standard/
    │   └── cloudflare-worker/
    └── ws/
```

## Build and Distribution

Elysia uses `tsup` for building ESM and CJS outputs, with a separate Bun-native
build for optimal performance:

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "bun": "./dist/bun/index.js",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  }
}
```

The conditional export prioritizes the Bun-specific build when running on Bun,
falling back to standard ESM/CJS for other runtimes. This allows Bun-specific
optimizations while maintaining compatibility.

## Dependencies

Elysia maintains a minimal dependency footprint:

| Dependency                  | Purpose                |
| --------------------------- | ---------------------- |
| `memoirist`                 | Trie-based HTTP router |
| `cookie`                    | HTTP cookie parsing    |
| `fast-decode-uri-component` | Optimized URI decoding |
| `exact-mirror`              | Schema normalization   |

Peer dependencies include `@sinclair/typebox` for schema validation and
`typescript` for development.

---

# The Compilation System

Elysia's most distinctive architectural feature is its dual compilation system.
Rather than interpreting route handlers at runtime, the framework generates
specialized JavaScript code for each route.

## AOT (Ahead-of-Time) Mode

When `aot: true` (the default), Elysia compiles handlers at route registration
time. The `composeHandler` function in `compose.ts` builds a string of
JavaScript code that is then compiled via `Function()`:

```typescript
export const composeHandler = ({
  app,
  path,
  method,
  hooks,
  validator,
  handler,
  allowMeta = false,
  inference,
}: {
  app: AnyElysia;
  path: string;
  method: string;
  hooks: Partial<LifeCycleStore>;
  validator: SchemaValidator;
  handler: unknown | Handler<any, any>;
  allowMeta?: boolean;
  inference: Sucrose.Inference;
}): ComposedHandler => {
  const adapter = app["~adapter"].composeHandler;
  let fnLiteral = "";

  // Build function body as string based on inference
  inference = sucrose(
    Object.assign({ handler: handler as any }, hooks),
    inference,
    app.config.sucrose,
  );

  // Generate header parsing only if headers are accessed
  if (hasHeaders) fnLiteral += adapter.headers;

  // ... more code generation ...

  return Function("arguments", fnLiteral)(); /* injected dependencies */
};
```

The generated code is highly specialized. For a simple handler that only
accesses `query`, the generated function won't include body parsing, header
extraction, or cookie handling code—only query string parsing.

## JIT (Just-in-Time) Mode

When `aot: false`, Elysia uses `createDynamicHandler` which performs validation
and context setup at request time:

```typescript
export const createDynamicHandler = (app: AnyElysia): DynamicHandler =>
  async function (request: Request): Promise<Response> {
    // Extract path from URL
    const url = request.url;
    const s = url.indexOf("/", 11);
    const qi = url.indexOf("?", s + 1);
    const path = url.substring(s, qi === -1 ? url.length : qi);

    // Route matching via Memoirist
    const route = app.router.dynamic.find(method, path);

    // Context setup and validation performed at runtime
    // ...
  };
```

JIT mode trades faster startup for slower per-request handling, useful during
development or when routes are modified dynamically.

## The Sucrose Engine

Sucrose is Elysia's static code analysis engine that powers inference-based
optimization. It parses handler functions to determine which context properties
are accessed:

```typescript
export namespace Sucrose {
  export interface Inference {
    query: boolean;
    headers: boolean;
    body: boolean;
    cookie: boolean;
    set: boolean;
    server: boolean;
    route: boolean;
    url: boolean;
    path: boolean;
  }
}
```

The analysis works by:

1. Converting the handler function to a string
2. Parsing parameter destructuring patterns
3. Detecting property access in the function body

```typescript
export const separateFunction = (
  code: string,
): [string, string, { isArrowReturn: boolean }] => {
  // Remove async keyword
  if (code.startsWith("async")) code = code.slice(5);
  code = code.trimStart();

  // JSC: Starts with '(', is an arrow function
  if (code.charCodeAt(0) === 40) {
    index = code.indexOf("=>", code.indexOf(")"));
    // ... extract parameters and body
  }
  // ... handle other function formats
};

export const findParameterReference = (
  parameter: string,
  inference: Sucrose.Inference,
) => {
  const { parameters, hasParenthesis } = retrieveRootParamters(parameter);

  if (parameters.query) inference.query = true;
  if (parameters.headers) inference.headers = true;
  if (parameters.body) inference.body = true;
  if (parameters.cookie) inference.cookie = true;
  // ...
};
```

## Code Generation Patterns

For a handler accessing query and body, the generated code might look like:

```javascript
// Generated (simplified)
"use strict";
const c = createContext(request);
c.headers = Object.fromEntries(request.headers.entries());
c.query = parseQueryFromURL(request.url);
c.body = await request.json();

if (validator.query && !validator.query.Check(c.query)) {
  throw new ValidationError("query", validator.query, c.query);
}
if (validator.body && !validator.body.Check(c.body)) {
  throw new ValidationError("body", validator.body, c.body);
}

let r = handler(c);
if (r instanceof Promise) r = await r;

return mapResponse(r, c.set);
```

Static response optimization handles literal return values:

```typescript
if (!isHandleFn) {
  // Handler is a static value, not a function
  if (isResponse) {
    const response = handler as Response;
    handler = () => response.clone();
  }
  // Generate: return function(){return a.clone()}
}
```

---

# Router Implementation

## Memoirist Trie Router

Elysia uses `Memoirist`, a trie-based router that supports lazy initialization
and parameter decoding:

```typescript
router = {
  "~http": undefined,
  get http() {
    if (!this["~http"]) {
      this["~http"] = new Memoirist({
        lazy: true,
        onParam: fastDecodeURIComponent,
      });
    }
    return this["~http"];
  },
  "~dynamic": undefined,
  get dynamic() {
    if (!this["~dynamic"]) {
      this["~dynamic"] = new Memoirist({
        onParam: fastDecodeURIComponent,
      });
    }
    return this["~dynamic"];
  },
  static: {}, // Static path cache
  response: {}, // Native response cache
  history: [], // Route registry
} as Router;
```

The router maintains separate instances for AOT (`http`) and JIT (`dynamic`)
modes. The `lazy: true` option defers route tree building until first access.

## Route Registration Flow

When `app.get('/users/:id', handler)` is called:

1. **Path normalization**: Prefix handling, slash normalization
2. **Hook merging**: Local hooks merged with global and scoped hooks
3. **Macro application**: `applyMacro(localHook)` processes any registered
   macros
4. **Validator creation**: Schema validators compiled (lazy or eager based on
   AOT setting)
5. **Handler compilation**: `composeHandler()` generates optimized function
6. **Router registration**: `router.http.add(method, path, compiledHandler)`
7. **History tracking**: Route metadata stored in `router.history`

```typescript
private add(
  method: HTTPMethod,
  path: string,
  handle: Handler<any, any, any> | any,
  localHook?: AnyLocalHook,
  options?: { allowMeta?: boolean; skipPrefix?: boolean }
) {
  localHook ??= {}
  this.applyMacro(localHook)

  // ... validation setup ...

  const mainHandler = composeHandler({
    app: this,
    path: internalPath,
    method,
    hooks: mergedHooks,
    validator: localValidator,
    handler: handle,
    allowMeta,
    inference: cloneInference(this.inference)
  })

  this.router.http.add(method, loosePath, mainHandler)
  this.router.history.push(routeMetadata)
}
```

## Static Response Optimization

Bun supports native static responses that bypass JavaScript entirely for
matching paths. Elysia leverages this when available:

```typescript
const nativeStaticHandler = typeof handle !== "function"
  ? () => {
    const fn = adapter.createNativeStaticHandler?.(
      handle,
      hooks,
      context.set as Context["set"],
    );
    return fn?.();
  }
  : undefined;
```

For routes returning static values (`app.get('/', 'Hello')`), the response can
be served directly from Bun's routing layer with zero JavaScript execution per
request.

---

# Lifecycle Hooks Architecture

Elysia provides 11 lifecycle hooks that execute at specific points in the
request/response cycle.

## Hook Execution Order

| Order | Hook            | Execution Point                    | Can Short-Circuit |
| ----- | --------------- | ---------------------------------- | ----------------- |
| 1     | `start`         | Server initialization              | No                |
| 2     | `request`       | Before routing                     | Yes               |
| 3     | `parse`         | Body parsing                       | Yes               |
| 4     | `transform`     | Pre-validation (includes `derive`) | Yes               |
| 5     | `beforeHandle`  | Pre-handler (includes `resolve`)   | Yes               |
| 6     | Handler         | Main route logic                   | N/A               |
| 7     | `afterHandle`   | Post-handler                       | Yes               |
| 8     | `mapResponse`   | Response transformation            | No                |
| 9     | `afterResponse` | Post-response cleanup              | No                |
| 10    | `error`         | Error handling                     | Yes               |
| 11    | `stop`          | Server shutdown                    | No                |

## Hook Container Structure

Hooks are wrapped in containers that track metadata for optimization:

```typescript
export interface HookContainer<T = Function> {
  checksum?: number; // For deduplication
  fn: T; // The actual hook function
  subType?: "derive" | "resolve" | "mapResolve";
  scope?: "local" | "scoped" | "global";
  isAsync?: boolean; // Cached async detection
  hasReturn?: boolean; // Cached return detection
}
```

## Hook Registration

The `on()` method handles all hook registrations with scope-aware behavior:

```typescript
on(
  optionsOrType: { as: LifeCycleType } | string,
  typeOrHandlers: MaybeArray<Function | HookContainer> | string,
  handlers?: MaybeArray<Function | HookContainer>
) {
  // Normalize arguments
  let type: keyof LifeCycleStore

  // Convert to HookContainer array
  const handles = handlers as HookContainer[]

  for (const handle of handles) {
    handle.scope = typeof optionsOrType === 'string'
      ? 'local'
      : (optionsOrType?.as ?? 'local')

    if (type === 'resolve' || type === 'derive')
      handle.subType = type
  }

  // Run inference analysis
  if (type !== 'trace')
    this.inference = sucrose({ [type]: handles.map(x => x.fn) }, this.inference)

  // Register to appropriate event store
  switch (type) {
    case 'request':
      this.event.request ??= []
      this.event.request.push(fn)
      break
    // ... other cases
  }
}
```

## Derive vs Resolve

Both `derive` and `resolve` add computed properties to the context, but differ
in execution timing:

| Feature   | derive              | resolve                    |
| --------- | ------------------- | -------------------------- |
| Execution | `transform` phase   | `beforeHandle` phase       |
| Access to | Basic context only  | Fully validated context    |
| Use case  | Computed properties | Authentication, DB queries |
| Async     | Supported           | Supported                  |

```typescript
// derive runs during transform (before validation)
app.derive(({ headers }) => ({
  userId: headers["x-user-id"],
}));

// resolve runs during beforeHandle (after validation)
app.resolve(async ({ userId }) => ({
  user: await db.users.findById(userId),
}));
```

## Hook Composition in Generated Code

In AOT mode, hooks are inlined into the generated handler:

```javascript
// Generated hook chain (simplified)
for (let i = 0; i < beforeHandle.length; i++) {
  let r = beforeHandle[i](c);
  if (r instanceof Promise) r = await r;
  // Short-circuit if hook returns a value
  if (r !== undefined) return mapResponse(r, c.set);
}

// Main handler execution
let response = handler(c);
if (response instanceof Promise) response = await response;

// afterHandle hooks
for (let i = 0; i < afterHandle.length; i++) {
  let r = afterHandle[i](c, response);
  if (r instanceof Promise) r = await r;
  if (r !== undefined) response = r;
}
```

---

# State Scoping System

Elysia implements a three-tier state scoping system that controls how state and
hooks propagate through the application.

## The Three Scopes

**Singleton (Global):**

- Shared across all routes in the application
- Includes: `decorator`, `store`, `derive`, `resolve`
- Set via `app.state()`, `app.decorate()`, `app.derive({ as: 'global' })`

**Ephemeral (Scoped):**

- Shared with child instances created via `.use()`
- Propagates to plugins
- Set via `app.derive({ as: 'scoped' })`

**Volatile (Local):**

- Route-specific, does not propagate
- Default scope for hooks
- Set via `app.derive()` (no scope specified)

## Type-Level Implementation

The scoping system is encoded at the type level through generic parameters:

```typescript
export default class Elysia<
  const in out BasePath extends string = '',
  const in out Singleton extends SingletonBase = {
    decorator: {}
    store: {}
    derive: {}
    resolve: {}
  },
  const in out Definitions extends DefinitionBase = {
    typebox: {}
    error: {}
  },
  const in out Metadata extends MetadataBase = {
    schema: {}
    standaloneSchema: {}
    macro: {}
    macroFn: {}
    parser: {}
    response: {}
  },
  const in out Routes extends RouteBase = {},
  const in out Ephemeral extends EphemeralType = {
    derive: {}
    resolve: {}
    schema: {}
    standaloneSchema: {}
    response: {}
  },
  const in out Volatile extends EphemeralType = {
    derive: {}
    resolve: {}
    schema: {}
    standaloneSchema: {}
    response: {}
  }
>
```

## Scope Promotion

The `.as()` method allows promoting volatile/ephemeral state to higher scopes:

```typescript
app
  .derive(() => ({ localValue: "only this route" }))
  .as("scoped") // Now available to child plugins
  .as("global"); // Now available everywhere
```

## Singleton State Storage

```typescript
protected singleton = {
  decorator: {},  // Added to context via decorate()
  store: {},      // Shared mutable state
  derive: {},     // Computed context values
  resolve: {}     // Async resolved values
} as SingletonBase

get store(): Singleton['store'] {
  return this.singleton.store
}

get decorator(): Singleton['decorator'] {
  return this.singleton.decorator
}
```

## Validator Layers

Schema validators also follow the three-tier model:

```typescript
protected validator: ValidatorLayer = {
  global: null,
  scoped: null,
  local: null,
  getCandidate() {
    if (!this.global && !this.scoped && !this.local)
      return emptySchema

    // Merge in precedence order: global < scoped < local
    return mergeSchemaValidator(
      mergeSchemaValidator(this.global, this.scoped),
      this.local
    )
  }
}
```

---

# Schema Validation System

Elysia integrates deeply with `@sinclair/typebox` for runtime schema validation
while supporting the Standard Schema v1 protocol for alternative validators.

## ElysiaTypeCheck Interface

The framework extends TypeBox's TypeCheck with additional metadata:

```typescript
export interface ElysiaTypeCheck<T extends TSchema>
  extends Omit<TypeCheck<T>, "schema"> {
  provider: "typebox" | "standard";
  schema: T;
  config: Object;
  Clean?(v: unknown): UnwrapSchema<T>;
  parse(v: unknown): UnwrapSchema<T>;
  safeParse(v: unknown):
    | { success: true; data: UnwrapSchema<T>; error: null }
    | { success: false; data: null; error: string; errors: MapValueError[] };
  hasAdditionalProperties: boolean;
  hasDefault: boolean;
  isOptional: boolean;
  hasTransform: boolean;
  hasRef: boolean;
}
```

## Schema Compilation

Validators are compiled via TypeBox's `TypeCompiler`:

```typescript
const validator = getSchemaValidator(schema, {
  modules: app.definitions.typebox,
  dynamic: !app.config.aot,
  additionalProperties: false,
  coerce: true,
});
```

The `additionalProperties: false` default prevents object expansion
attacks—unknown properties cause validation failure.

## Standard Schema Support

Elysia supports Standard Schema v1, enabling integration with Zod, Valibot, and
ArkType:

```typescript
// Zod integration
import { z } from "zod";

app.post("/user", ({ body }) => body, {
  body: z.object({
    name: z.string(),
    email: z.string().email(),
  }),
});

// Valibot integration
import * as v from "valibot";

app.post("/user", ({ body }) => body, {
  body: v.object({
    name: v.string(),
    email: v.pipe(v.string(), v.email()),
  }),
});
```

## Custom Type Extensions

Elysia extends TypeBox with custom types for web-specific use cases:

```typescript
// t.File - Single file upload with MIME validation
t.File({ type: "image/*", maxSize: "5m" });

// t.Files - Multiple files
t.Files({ type: ["image/png", "image/jpeg"] });

// t.Numeric - String to number coercion
t.Numeric({ minimum: 0 }); // "123" → 123

// t.ObjectString - JSON in query parameters
t.ObjectString(t.Object({ nested: t.Boolean() }));
```

## Schema Normalization

The `normalize` config controls how additional properties are handled:

```typescript
const composeCleaner = ({ schema, name, normalize }) => {
  if (!normalize || !schema.Clean) return "";

  if (normalize === true || normalize === "exactMirror") {
    // Uses exact-mirror for fast cleaning
    return `${name}=validator.${type}.Clean(${name})\n`;
  }

  if (normalize === "typebox") {
    // Uses TypeBox's Value.Clean (slower)
    return `${name}=validator.${type}.Clean(${name})\n`;
  }
};
```

---

# Multi-Runtime Adapter System

Elysia achieves runtime portability through a pluggable adapter system that
abstracts platform-specific behaviors.

## Adapter Interface

```typescript
export interface ElysiaAdapter {
  name: string
  listen(app: AnyElysia): (options, callback?) => void
  stop?(app: AnyElysia, closeActiveConnections?: boolean): Promise<void>
  isWebStandard?: boolean

  handler: {
    mapResponse(response: unknown, set: Context['set']): unknown
    mapEarlyResponse(response: unknown, set: Context['set']): unknown
    mapCompactResponse(response: unknown): unknown
    createStaticHandler?(...): (() => unknown) | undefined
    createNativeStaticHandler?(...): (() => MaybePromise<Response>) | undefined
  }

  composeHandler: {
    declare?(inference: Sucrose.Inference): string | undefined
    inject?: Record<string, unknown>
    headers: string  // fnLiteral for header parsing
    parser: {
      json: (isOptional: boolean) => string
      text: (isOptional: boolean) => string
      urlencoded: (isOptional: boolean) => string
      arrayBuffer: (isOptional: boolean) => string
      formData: (isOptional: boolean) => string
    }
  }

  composeGeneralHandler: {
    createContext(app: AnyElysia): string
    error404(hasEventHook, hasErrorHook, afterResponseHandler?): {
      declare: string
      code: string
    }
  }

  composeError: {
    mapResponseContext: string
    validationError: string
    unknownError: string
  }

  ws?(app: AnyElysia, path: string, handler: AnyWSLocalHook): unknown
}
```

## Bun Adapter

The Bun adapter leverages runtime-specific features:

- **Native static responses**: `Bun.serve({ static: { '/': response } })`
- **Direct WebSocket support**: Uses Bun's built-in WebSocket server
- **Optimized header access**: Direct property access vs. iteration

```typescript
const BunAdapter: ElysiaAdapter = {
  name: "bun",
  listen: (app) => (options, callback) => {
    app.server = Bun.serve({
      fetch: app.fetch,
      websocket: app.websocket,
      ...options,
    });
    callback?.(app.server);
  },
  handler: {
    createNativeStaticHandler(handle, hooks, set) {
      // Returns handler suitable for Bun.serve static routes
      return () => handle.clone();
    },
  },
};
```

## Web Standard Adapter

For Node.js and other Web Standard-compatible runtimes:

```typescript
const WebStandardAdapter: ElysiaAdapter = {
  name: "web-standard",
  isWebStandard: true,
  listen: (app) => (options, callback) => {
    // Implementation depends on runtime (Node, Deno, etc.)
  },
  handler: {
    mapResponse(response, set) {
      // Standard Response construction
      return new Response(body, { status, headers });
    },
  },
};
```

## Adapter Selection

The constructor selects the appropriate adapter:

```typescript
constructor(config: ElysiaConfig<BasePath> = {}) {
  this['~adapter'] =
    config.adapter ??
    (typeof Bun !== 'undefined' ? BunAdapter : WebStandardAdapter)
}
```

---

# Plugin System

## Plugin Definition

Plugins are functions that receive an Elysia instance and return a modified
instance:

```typescript
const myPlugin = (app: Elysia) =>
  app
    .state("pluginState", "value")
    .get("/plugin-route", () => "from plugin");

app.use(myPlugin);
```

## The `.use()` Method

Plugin loading supports multiple formats:

```typescript
use<Plugin>(plugin: Plugin | ((app: this) => Plugin)): this
use(plugin: Promise<{ default: Plugin }>): this
use(plugins: Plugin[]): this

// Implementation handles all cases
use(plugin) {
  if (Array.isArray(plugin)) {
    for (const p of plugin) this.use(p)
    return this
  }

  if (plugin instanceof Promise) {
    this.promisedModules.add(
      plugin.then((mod) => this._use(mod.default))
    )
    return this
  }

  if (typeof plugin === 'function') {
    return this._use(plugin(this))
  }

  return this._use(plugin)
}
```

## Checksum-Based Deduplication

Plugins are deduplicated based on name and seed:

```typescript
const pluginChecksum = checksum(
  JSON.stringify({
    name: plugin.config.name,
    seed: plugin.config.seed,
  }),
);

if (this.dependencies[pluginChecksum]) {
  // Plugin already registered, skip
  return this;
}
this.dependencies[pluginChecksum] = true;
```

This prevents the same plugin from being registered multiple times, which could
cause duplicate routes or state conflicts.

## Higher-Order Functions

The `wrap()` method enables wrapping the fetch handler:

```typescript
wrap(fn: HigherOrderFunction) {
  this.extender.higherOrderFunctions.push({
    checksum: checksum(JSON.stringify({
      name: this.config.name,
      seed: this.config.seed,
      content: fn.toString()
    })),
    fn
  })
  return this
}

// Usage: Add timing to all requests
app.wrap((fetch) => async (request) => {
  const start = performance.now()
  const response = await fetch(request)
  console.log(`Request took ${performance.now() - start}ms`)
  return response
})
```

---

# WebSocket Implementation

Elysia provides first-class WebSocket support through the `ElysiaWS` class.

## ElysiaWS Class

```typescript
export class ElysiaWS<Context = unknown, Route extends RouteSchema = {}>
  implements ElysiaServerWebSocket {
  constructor(
    public raw: ServerWebSocket<{
      id?: string;
      validator?: TypeCheck<TSchema>;
    }>,
    public data: Prettify<
      Omit<Context, "body" | "error" | "status" | "redirect">
    >,
    public body: Route["body"] = undefined,
  ) {
    this.validator = raw.data?.validator;

    // Bind methods to raw WebSocket
    this.sendText = raw.sendText.bind(raw);
    this.sendBinary = raw.sendBinary.bind(raw);
    this.close = raw.close.bind(raw);
    this.subscribe = raw.subscribe.bind(raw);
    this.publish = this.publish.bind(this);
  }

  send(data: FlattenResponse<Route["response"]>, compress?: boolean) {
    // Validate outgoing messages
    if (this.validator?.Check(data) === false) {
      return this.raw.send(
        new ValidationError("message", this.validator, data).message,
      );
    }

    if (typeof data === "object") data = JSON.stringify(data);
    return this.raw.send(data, compress);
  }
}
```

## WebSocket Route Definition

```typescript
app.ws("/chat", {
  body: t.Object({ message: t.String() }),
  response: t.Object({ message: t.String(), from: t.String() }),

  open(ws) {
    ws.subscribe("chat-room");
  },

  message(ws, { message }) {
    ws.publish("chat-room", {
      message,
      from: ws.data.userId,
    });
  },

  close(ws) {
    ws.unsubscribe("chat-room");
  },
});
```

## Pub/Sub Support

WebSocket connections can subscribe to topics and publish messages:

```typescript
// Subscribe to a topic
ws.subscribe("notifications");

// Publish to all subscribers
ws.publish("notifications", { event: "new-message" });

// Check subscription
ws.isSubscribed("notifications");
```

---

# Error Handling System

## Error Class Hierarchy

| Class                    | Status | Code                       | Use Case                  |
| ------------------------ | ------ | -------------------------- | ------------------------- |
| `InternalServerError`    | 500    | `INTERNAL_SERVER_ERROR`    | Unhandled exceptions      |
| `NotFoundError`          | 404    | `NOT_FOUND`                | Route not found           |
| `ParseError`             | 400    | `PARSE`                    | Body parsing failure      |
| `ValidationError`        | 422    | `VALIDATION`               | Schema validation failure |
| `InvalidCookieSignature` | 400    | `INVALID_COOKIE_SIGNATURE` | Cookie tampering          |
| `InvalidFileType`        | 422    | `INVALID_FILE_TYPE`        | File MIME validation      |

## ValidationError Details

```typescript
export class ValidationError extends Error {
  code = "VALIDATION";
  status = 422;

  valueError?: ValueError; // First validation error
  expected?: unknown; // Expected schema value
  customError?: unknown; // Custom error from schema

  constructor(
    public type: string, // 'body' | 'query' | 'params' | etc.
    public validator: TSchema | TypeCheck<any> | ElysiaTypeCheck<any>,
    public value: unknown,
    private allowUnsafeValidationDetails = false,
    errors?: ValueErrorIterator,
  ) {
    // Build error message with production/development handling
    if (isProduction && !allowUnsafeValidationDetails) {
      message = JSON.stringify({
        type: "validation",
        on: type,
        found: value,
      });
    } else {
      message = JSON.stringify({
        type: "validation",
        on: type,
        property: accessor,
        message: error?.message,
        summary: mapValueError(error).summary,
        expected,
        found: value,
        errors: [...validator.Errors(value)].map(mapValueError),
      });
    }
  }

  get all(): MapValueError[] {
    return "Errors" in this.validator
      ? [...this.validator.Errors(this.value)].map(mapValueError)
      : [...Value.Errors(this.validator, this.value)].map(mapValueError);
  }
}
```

## Error Value Mapping

Validation errors are mapped to human-readable messages:

```typescript
export const mapValueError = (error: ValueError): MapValueError => {
  const property = error.path.slice(1).replaceAll("/", ".");

  switch (error.type) {
    case 42: // Additional property
      return { summary: `Property '${property}' should not be provided` };
    case 45: // Required property missing
      return { summary: `Property '${property}' is missing` };
    case 50: // Format validation
      return { summary: `Property '${property}' should be ${format}` };
    case 54: // Type mismatch
      return {
        summary: `Expected '${property}' to be ${expected} but found: ${value}`,
      };
    case 62: // Union type
      return { summary: `Property '${property}' should be one of: ${union}` };
  }
};
```

---

# Context Object

The request context provides type-safe access to all request data and response
controls.

## Context Structure

```typescript
export type Context<
  Route extends RouteSchema = {},
  Singleton extends SingletonBase = {...},
  Path extends string | undefined = undefined
> = Prettify<{
  body: Route['body']
  query: Route['query']
  params: Route['params']
  headers: Route['headers']
  cookie: Record<string, Cookie<unknown>>

  server: Server | null
  redirect: Redirect

  set: {
    headers: HTTPHeaders
    status?: number | keyof StatusMap
    redirect?: string
    cookie?: Record<string, ElysiaCookie>
  }

  path: string      // Actual URL path: '/users/123'
  route: string     // Registered pattern: '/users/:id'
  request: Request
  store: Singleton['store']
  status: SelectiveStatus<Route['response']>
} & Singleton['decorator']
  & Singleton['derive']
  & Singleton['resolve']>
```

## Response Control via `set`

The `set` object controls response metadata:

```typescript
app.get("/example", ({ set }) => {
  set.headers["x-custom"] = "value";
  set.status = 201;
  set.cookie = {
    session: {
      value: "abc123",
      httpOnly: true,
      maxAge: 60 * 60 * 24,
    },
  };

  return { created: true };
});
```

## Status Helper

Type-safe status responses with the `status` helper:

```typescript
app.get("/user/:id", ({ status, params }) => {
  const user = db.find(params.id);

  if (!user) {
    return status(404, { error: "User not found" });
  }

  return status(200, user);
}, {
  response: {
    200: t.Object({ id: t.String(), name: t.String() }),
    404: t.Object({ error: t.String() }),
  },
});
```

---

# Performance Considerations

## Optimization Techniques

| Technique                    | Implementation                     | Benefit                            |
| ---------------------------- | ---------------------------------- | ---------------------------------- |
| Code generation              | Handler functions built as strings | Eliminates runtime abstraction     |
| Inference-based optimization | Sucrose analyzes handler access    | Removes unused parsing code        |
| Static response caching      | Native Bun static routes           | Zero JS execution for static paths |
| Lazy validation              | Schema compiled on first use       | Faster startup                     |
| Checksum deduplication       | FNV-1a hash for plugins            | Prevents duplicate work            |
| Memoized routing             | Trie-based router with lazy build  | O(1) exact, O(log n) dynamic       |

## Configuration Trade-offs

| Config                       | Performance | Flexibility   | Startup |
| ---------------------------- | ----------- | ------------- | ------- |
| `aot: true` (default)        | High        | Lower         | Slower  |
| `aot: false`                 | Medium      | Higher        | Faster  |
| `precompile: true`           | Highest     | Lowest        | Slowest |
| `normalize: true`            | Lower       | Higher safety | -       |
| `nativeStaticResponse: true` | Highest     | Bun only      | -       |

## When to Use JIT Mode

- Development with frequent code changes
- Dynamic route registration at runtime
- Plugins that modify routes after startup
- When startup time is more critical than per-request latency

---

# Glossary

| Term                | Definition                                                              |
| ------------------- | ----------------------------------------------------------------------- |
| **AOT**             | Ahead-of-Time compilation; generates handler code at route registration |
| **JIT**             | Just-in-Time compilation; performs setup at request time                |
| **Sucrose**         | Elysia's static code analysis engine for inference                      |
| **Memoirist**       | Trie-based routing library used by Elysia                               |
| **TypeBox**         | JSON Schema type builder for runtime validation                         |
| **Singleton**       | Global state scope shared across all routes                             |
| **Ephemeral**       | Scoped state that propagates to child plugins                           |
| **Volatile**        | Local state specific to individual routes                               |
| **Standard Schema** | Cross-library validation protocol (Zod, Valibot, ArkType)               |
| **HookContainer**   | Wrapper for hook functions with metadata                                |
| **Adapter**         | Runtime abstraction layer (Bun, Web Standard)                           |
| **fnLiteral**       | String containing generated JavaScript code                             |

---

# File Reference Index

| Feature             | Primary Files                 |
| ------------------- | ----------------------------- |
| Core class, routing | `src/index.ts:190-8285`       |
| AOT compilation     | `src/compose.ts:452-700`      |
| Static analysis     | `src/sucrose.ts:1-300`        |
| Schema validation   | `src/schema.ts:39-200`        |
| JIT handling        | `src/dynamic-handle.ts`       |
| Error classes       | `src/error.ts:116-616`        |
| Context types       | `src/context.ts:132-256`      |
| Adapter interface   | `src/adapter/types.ts:10-170` |
| WebSocket           | `src/ws/index.ts:44-150`      |
| Type definitions    | `src/types.ts`                |
| Cookie handling     | `src/cookies.ts`              |

---

_Report generated from Elysia v1.4.19 source code analysis_
