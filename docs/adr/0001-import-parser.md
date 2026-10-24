# 0001. Use oxc-parser to parse imports

Date: 2026-10-24 · Status: Accepted

## Context

Radar must extract import sources from every file of a JS/TS codebase — including JSX/TSX — and real projects have thousands of files, so the parser has to be fast.

The project already runs TypeScript 7 for `tsc`, while tools that need the compiler API get TS 6 through an npm alias: the TS 7 JS API is not stable yet.

## Options considered

### TS Compiler API

- Good: Same parser as `tsc`; can later resolve paths from `tsconfig`;
- Bad: Non-ESTree AST; heavy; tied to the TS 6 compatibility package until the TS 7 API is stable

### `@babel/parser` 8.0.8

- Good: Mature, best documentation, configurable via plugins
- Bad: Slowest; Babel AST is not plain ESTree (`StringLiteral`); plugins must be chosen per file extension (`jsx` breaks `<T>` casts in `.ts`)

### `oxc-parser` 0.155.0

- Good: Fastest; ESTree / TS-ESTree AST (same as ESLint); picks syntax from the file name; exposes a ready-made `module` record
- Bad: 0.x — breaking changes possible between minor versions; native binary

### Spike

Fixture `test/fixtures/imports.ts` covers `import x from`, `import type`, `export * from`, `export { x } from`, dynamic `import()` and a local `export const`.

Both Babel and oxc extracted exactly `./a`–`./e`, skipped the local export, and marked `./b` as type-only (`importKind` / `exportKind`).

### Benchmark

`typescript.js` from TypeScript 6.0.3 (9.1 MB, 201k lines), Node v24.15.0, 5 warm-up + 20 measured runs, 3 launches.

| Parser                | min, ms | average, ms |
| --------------------- | ------- | ----------- |
| `oxc-parser` 0.155.0  | 115–120 | 119–123     |
| `@babel/parser` 8.0.8 | 331–334 | 349–386     |

oxc is ~3× faster and more stable between runs.

## Decision

Use `oxc-parser`: it is the fastest option, produces a standard ESTree AST and selects the syntax from the file extension, so Radar needs no per-extension configuration.

## Consequences

- The parser is hidden behind Radar's own function, so replacing it touches one module.
- 0.x releases may break between minor versions: the `^0.155.0` range only allows patches, and minor upgrades are done deliberately, with tests.
- Dynamic `import()` can be nested (e.g. `React.lazy(() => import('./Page'))`), so the extractor must traverse the whole AST, not only `program.body`.
- `import(variable)` cannot be resolved statically; whether to skip or report such imports is decided in the extractor task.
- Type-only imports are detectable and can be treated separately in the graph.

## Revisit when

- oxc ships a breaking change that is costly to adopt;
- TypeScript 7 ships a stable JS API and Radar needs `tsconfig`-based path resolution.
