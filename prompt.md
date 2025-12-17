# Framework Deep Analysis Task

Analyze this framework's source code comprehensively and generate a detailed
technical report in Markdown format, optimized for laser printing (clean
formatting, no excessive nesting, proper page breaks).

## Analysis Scope

### 1. Core Architecture & Runtime Dynamics

- Entry points and initialization flow
- Module system and dependency resolution
- Lifecycle hooks and event system
- Internal state management mechanisms
- How the framework bootstraps an application

### 2. Build System & Bundling

- Build tool used (Vite, Webpack, Rollup, esbuild, Turbopack, etc.)
- Configuration system and defaults
- JavaScript bundling strategy (code splitting, tree shaking, minification)
- CSS processing pipeline (PostCSS, preprocessors, CSS modules, scoping)
- How static assets (images, fonts, SVGs) are handled and optimized
- Dev server implementation (HMR, fast refresh mechanisms)

### 3. Performance Optimizations

- Rendering strategy (SSR, SSG, ISR, partial hydration, islands)
- Lazy loading and dynamic imports
- Caching strategies (build-time, runtime, CDN hints)
- Bundle size optimization techniques
- Memory management and garbage collection considerations

### 4. Security Measures

- XSS prevention mechanisms
- CSRF protection (if applicable)
- Content Security Policy integration
- Input sanitization in templates/JSX
- Secure defaults and escape hatches

### 5. Framework Features

- Routing system (file-based, config-based, nested routes)
- Data fetching patterns (loaders, server functions, API routes)
- Form handling (actions, validation, progressive enhancement)
- Middleware/plugin system
- TypeScript integration depth
- Error boundaries and error handling

### 6. React/Preact Integration (if applicable)

- Which UI library is used and why
- Component model adaptations
- Hooks compatibility and custom hooks provided
- Server Components support (if any)
- Reconciliation and rendering pipeline modifications

### 7. Developer Experience

- CLI tools and scaffolding
- Hot Module Replacement implementation
- Error overlay and debugging tools
- Documentation quality and examples

## Output Requirements

Generate a single Markdown file with:

- Clean hierarchical structure (H1 → H2 → H3, avoid H4+)
- Code blocks with syntax highlighting for key examples
- Tables for comparisons where appropriate
- Diagrams described in text (Mermaid syntax if needed)
- Executive summary at the beginning
- Glossary of framework-specific terms at the end
- Total length: 15-25 pages when printed

File name: `{FRAMEWORK_NAME}-deep-analysis.md` Save to: Current directory

## Analysis Approach

1. Start with `package.json` to understand dependencies and scripts
2. Examine the `/src` or `/packages` directory structure
3. Trace the build pipeline from entry to output
4. Read key source files, not just documentation
5. Identify patterns by examining multiple related files
6. Note version-specific behaviors (mention the version analyzed)

Be thorough but prioritize signal over noise. Focus on "how it works" rather
than "how to use it."
