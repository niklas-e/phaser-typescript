---
name: migrating-js-to-typescript
description: Use when converting Phaser src/**/*.js modules to TypeScript, adding Phaser TypeScript source modules, or diagnosing phaser.d.ts regressions caused by a TS migration
---

# Migrating Phaser JS Modules to TypeScript

## Overview

Phaser uses a hybrid JS/TS codebase with a custom tsgen pipeline that generates `phaser.d.ts` from JSDoc + tsc output. Migrating a module means converting `.js` to `.ts` while preserving the public API surface exactly, keeping non-type JSDoc annotations, and integrating with the overlay pipeline.

**Core principle:** The generated `phaser.d.ts` signatures must preserve the public API surface. Accepted improvements (correct `this` returns, accessor syntax, optional params, generic constraints) are fine; breaking changes are not.

## When to Use

- Converting a Phaser `src/**/*.js` module to TypeScript
- Adding a new TypeScript module to the Phaser source tree
- Debugging `.d.ts` regressions after a TS migration
- Investigating tsgen overlay behavior while converting or validating a migrated module

**Do NOT use for:**
- Pure JS changes (no migration involved)
- Changes to the tsgen pipeline itself (that's infrastructure)
- Non-Phaser TypeScript projects

## Migration Workflow

Track the migration steps when the change spans multiple files or verification rounds.

### 1. Assess the module

- Read the `.js` source completely
- Identify all imports, exports, JSDoc annotations, and public API surface
- Check if the module depends on other JS `Class()` factory types (triggers interop patterns)
- Plan to keep a sibling `.js` compatibility wrapper during the hybrid period

### 2. Convert `.js` to `.ts`

- Rename `src/path/Module.js` to `src/path/Module.ts`
- Commit the rename before making any edits to preserve git history
- Add TypeScript type annotations to all parameters, return types, and properties
- Prefer named declarations that match the canonical symbol (`export class Vector2`, `export function Clamp`)
- Add `export default Name` when existing runtime consumers or wrappers expect the module default
- Replace `module.exports` in the implementation with TS exports; keep compatibility in the sibling `.js` wrapper

### 3. Clean JSDoc annotations

**Keep (semantic/non-type annotations):**
- `@author`, `@copyright`, `@license` (file header)
- `@classdesc` with description text
- identity and ownership tags such as `@memberof Phaser.X.Y` or `@function Phaser.X.Y` when present
- `@since x.x.x` on classes, methods, and properties
- `@default` on properties with default values
- `@readonly` on read-only properties
- `@constant` on constants
- `@example` and other semantic documentation tags
- `@param` descriptions (strip the `{Type}` part, keep the text)
- `@returns` / `@return` descriptions (strip `{Type}`, keep text)

**Remove type-bearing syntax (replaced by TS):**
- `@type {Type}` (replaced by TS annotation)
- `@param {Type}` braces (TS signature provides type)
- `@returns {Type}` braces (TS signature provides type)
- `@generic {T}` (replaced by TS `<T>` syntax)
- `@callback` (replaced by TS type alias or inline)
- `@extends`, `@implements` (TS syntax replaces these)

Preserve tags such as `@class`, `@function`, `@namespace`, or `@module` when they carry API identity or ownership information. Remove them only when they become duplicate noise after the TS declaration is clear.

### 4. Fix imports

**Runtime/value imports - use `.js` extensions:**
```ts
import { objectKeys } from '../utils/object/TypedObjectUtils.js';
import Contains from './Contains.js';
```
This is standard TS ESM practice; `moduleResolution: "bundler"` resolves `.js` to `.ts`.

**Type-only imports (used only in type positions):**
```ts
import type { Vector2 } from '../../math/Vector2';
```
Current migrated modules allow extensionless `import type` paths.

**JS module imports (unchanged):**
```ts
import Line from '../line/Line.js';
import GEOM_CONST from '../const.js';
```

### 5. Keep CJS compatibility wrapper

Every migrated runtime module should keep a sibling `.js` CJS compatibility wrapper during the hybrid period unless CI has proven wrapper-free operation across build and type gates.

**Default export:**
```js
// src/structs/Map.js
module.exports = require('./Map.ts').default;
```

**Named exports:**
```js
// src/utils/object/TypedObjectUtils.js
const mod = require('./TypedObjectUtils.ts');
module.exports = mod;
module.exports.objectKeys = mod.objectKeys;
```

### 6. Update CJS consumers of migrated modules

After creating a bridge wrapper, check whether any CJS `.js` files `require()` the migrated module **with an explicit `.js` extension**. If so, remove the extension.

**Why:** Webpack's `extensionAlias: { '.js': ['.ts', '.js'] }` tries `.ts` first. A CJS `require('./Equal.js')` resolves to `Equal.ts` (ESM namespace object `{ default: fn }`) instead of the bridge `Equal.js`. The caller gets an object, not the function, causing runtime errors like "X is not a function".

```js
// ❌ BROKEN: extensionAlias redirects to .ts, bypassing the bridge
var Equal = require('../../../math/fuzzy/Equal.js');

// ✅ CORRECT: resolve.extensions finds bridge .js first
var Equal = require('../../../math/fuzzy/Equal');
```

**Note:** This only affects CJS `require()` calls in `.js` files. TypeScript `import` statements with `.js` extensions are fine — `moduleResolution: "bundler"` handles those correctly.

### 7. Add type normalization rules (if needed)

If the migrated module uses unqualified peer type references (e.g., `Vector2` instead of `Phaser.Math.Vector2`), add an entry to the `MODULE_TYPE_RULES` table in `scripts/tsgen/src/MigratedOverlay.ts`. Only two categories of transforms are acceptable:

**a) Namespace qualification** - tsc emits local names; `.d.ts` needs fully-qualified `Phaser.*` paths:
```ts
'src/math/Vector2.ts': [
    { type: 'qualify', shortName: 'Vector2Like', qualifiedName: 'Phaser.Types.Math.Vector2Like' },
],
```

**b) Non-exported type alias inlining** - tsc preserves non-exported type alias names without emitting their declarations. Inline the alias body:
```ts
'src/structs/Map.ts': [
    { type: 'replace', pattern: /\bEachMapCallback\s*<\s*K\s*,\s*V\s*>/g, replacement: '(key: K, entry: V) => boolean | void' },
],
```

**Never add cosmetic/regex hacks** - the `.d.ts` should reflect real tsc output.

### 8. Preserve typedef JS files

If the migrated `.ts` file exports a type/interface that replaces a JSDoc typedef (e.g., `Vector2Like`), keep the original `.js` typedef file:
```
src/math/typedefs/Vector2Like.js  ← keep for tsgen typedef generation
src/math/Vector2.ts               ← exports Vector2Like interface
```
The JSDoc pipeline needs the `.js` file to generate `Phaser.Types.Math.Vector2Like` in the typedef namespace.

### 9. Verify

Run the full verification pipeline in order:
```bash
npm run typecheck                 # TypeScript compiles clean
npm run build-tsgen               # Rebuild tsgen (compiles src/*.ts → bin/)
npm run ts                        # Generate phaser.d.ts + validate + test-ts
npm run build                     # Final repository-level integration gate
npm test                          # Runtime tests, especially for migrated module behavior
```

**Critical:** `npm run build-tsgen` must be run after editing `MigratedOverlay.ts` or `publish.ts` - the runtime uses compiled JS from `scripts/tsgen/bin/`.

## Type Patterns

### Static readonly constants
```ts
static readonly ZERO: Vector2 = new Vector2(0, 0);
```

### Constrained generics with defaults
```ts
export class Map<K extends string = string, V = unknown>
```

### Type-safe Object method wrappers in TypedObjectUtils.ts
```ts
// src/utils/object/TypedObjectUtils.ts
export function objectKeys<K extends string>(obj: Record<K, unknown>): K[]
{
    return Object.keys(obj) as K[];
}
```
Use `objectKeys()` instead of `Object.keys()` to preserve generic key types.

### Non-exported type aliases
```ts
type EachMapCallback<K extends string, V> = (key: K, entry: V) => boolean | void;
```
Keep file-scoped; requires inlining in the `MODULE_TYPE_RULES` table in `MigratedOverlay.ts`.

### Generic method signatures (replacing `@generic`)
```ts
getPoints<O extends Vector2[] = Vector2[]>(quantity: number, stepRate?: number, output?: O): O
```

## JS Class() Factory Interop

When a migrated `.ts` module references a JS `Class()` factory (e.g., `Line`), `typeof X.prototype` with `@ts-expect-error` may be necessary:

```ts
/**
 * @since 3.0.0
 */
// @ts-expect-error - Line is a JS Class() factory, not a TypeScript type
getLineA<O extends typeof Line.prototype = typeof Line.prototype>(line?: O): O
{
    // @ts-expect-error - Class() factory is not seen as constructable by TypeScript
    if (line === undefined) { line = new Line() as O; }
    // ...
}
```

Two `@ts-expect-error` sites per method:
1. **Signature:** `typeof X.prototype` as a type bound
2. **Body:** `new X()` on a non-constructable value

The `MODULE_TYPE_RULES` table in `MigratedOverlay.ts` replaces `typeof Line.prototype` with `Phaser.Geom.Line` in the emitted `.d.ts`.

**Only use `@ts-expect-error` for verified TS/JS interop gaps with a specific explanation.** JS `Class()` factories are one common case; mismatches in existing JS JSDoc signatures can be another.

## Pipeline Architecture

```
JSDoc (.js files)                    TypeScript (.ts files)
      |                                      |
      v                                      v
  jsdoc parser                     auto-discovery (JSDoc tags)
      |                                      |
      v                                      v
  dts-dom tree                    tsc --emitDeclarationOnly
      |                                      |
      +----------- MigratedOverlay ----------+
      |        (synthetic stubs + overlay)    |
      v                                      v
                   publish.ts
                  (orchestrator)
                       |
                       v
                 phaser.d.ts
```

**Auto-discovery:** `discoverMigratedModules()` in `MigratedOverlay.ts` globs `src/**/*.ts` (excluding `.d.ts`) and discovers canonical symbols via two patterns:
- **Functions:** `@function Phaser.X.Y` JSDoc tag provides the full canonical symbol directly
- **Classes:** `@memberof Phaser.X` JSDoc tag combined with the `export class Name` declaration that follows the JSDoc block

No manual registration is needed.

**Synthetic doclets:** For migrated symbols missing from JSDoc output, `buildSyntheticDoclets()` creates minimal stubs (`...args: any[]`) so the JSDoc parser allocates namespace slots. The overlay then replaces those stubs with real tsc declarations.

**`emitMigratedModuleDeclarations()`** uses the TypeScript compiler API to emit declaration fragments in memory for auto-discovered migrated modules.

**`normalizeAuthorityFragment()`** strips tsc boilerplate before insertion:
- Removes `import` lines
- Removes `export default` lines
- Strips `export declare` to bare declarations
- Strips `export` keyword

## Compiler Config

**Main `tsconfig.json`** (root) - used by `npm run typecheck`:
- `strict: true`, `target: ES2018`, `moduleResolution: bundler`
- `allowJs: true`, `checkJs: false` (JS/TS coexist, JS not type-checked)
- `declarationDir: ./types/generated/` exists for declaration emit experiments, but the active hybrid overlay emits migrated declarations in memory

**tsgen `tsconfig.json`** (`scripts/tsgen/`) - used to build the pipeline tool:
- `target: es2018`, `module: node16`, `moduleResolution: node16`
- `strict: false` (legacy pipeline code)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Removing identity/ownership tags | Keep `@function Phaser.X` or `@memberof Phaser.X` - auto-discovery depends on them |
| Removing `@since` | Keep it - documents API version history |
| Removing `@param`/`@returns` descriptions | Keep descriptions, only remove `{Type}` braces |
| Using `Object.keys()` in generic class | Use `objectKeys<K>()` to preserve key type |
| Forgetting CJS wrapper | Runtime/test code that loads `src/**/*.js` will fail |
| CJS `require()` with `.js` ext to bridge file | `extensionAlias` skips bridge, returns ESM namespace object → "X is not a function". Drop the `.js` extension. |
| Forgetting `npm run build-tsgen` | Pipeline runs stale code; changes to tsgen source won't take effect |
| Adding regex hacks to MigratedOverlay.ts | Only namespace qualification and type alias inlining are acceptable in `MODULE_TYPE_RULES` |
| Using `.ts` extension in runtime imports | Use `.js` - `moduleResolution: bundler` resolves to `.ts` |
| Widening scope to unmigrated modules | Each migration is self-contained; don't pull in dependencies |
| Not keeping typedef `.js` files | tsgen needs them to emit `Phaser.Types.*` entries |
| Unexplained `@ts-expect-error` | Use it only for verified TS/JS interop gaps and explain the mismatch |

## Quick Reference

| Item | Location |
|------|----------|
| Migration overlay (discovery, normalization, validation) | `scripts/tsgen/src/MigratedOverlay.ts` |
| Orchestrator | `scripts/tsgen/src/publish.ts` |
| Type normalization rules | `MODULE_TYPE_RULES` in `scripts/tsgen/src/MigratedOverlay.ts` |
| Shared typed utilities | `src/utils/object/TypedObjectUtils.ts` |
| Symbol validation guard | `scripts/validate-migrated-symbols.js` |
| JSDoc types guard | `scripts/check-migrated-jsdoc-types.js` |
| Typecheck command | `npm run typecheck` |
| Final output | `types/phaser.d.ts` |
| Main TS config | `tsconfig.json` (root) |
| tsgen TS config | `scripts/tsgen/tsconfig.json` |

| Need | Action |
|------|--------|
| Converted a source module | Ensure it has `@function Phaser.X` (for functions) or `@memberof Phaser.X` (for classes) for auto-discovery |
| Module uses unqualified peer types | Add entry to `MODULE_TYPE_RULES` in `MigratedOverlay.ts` |
| Edited `MigratedOverlay.ts` or `publish.ts` | Run `npm run build-tsgen` before `npm run ts` |
| Runtime/test code loads `src/**/*.js` | Keep a sibling `.js` CJS wrapper |
| `.d.ts` contains local type names | Normalize only namespace qualification or non-exported alias inlining |
