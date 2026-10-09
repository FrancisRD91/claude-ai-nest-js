# Enforcing the Architecture

Rules that a machine can check are checked by a machine, on every push:

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "lint": "eslint \"{src,test}/**/*.ts\"",
    "lint:arch": "depcruise src --config .dependency-cruiser.cjs",
    "verify": "npm run typecheck && npm run lint && npm run lint:arch && npm test && npm run test:e2e"
  }
}
```

The CI pipeline runs `npm run verify`, then `npm run build`. When a rule gets in the way, don't disable it inline: either the code is in the wrong layer, or the rule needs an ADR and a reviewed change.

## dependency-cruiser

```bash
npm i -D dependency-cruiser
```

Conventions the rules rely on:

```
src/<context>/domain/...            pure TypeScript
src/<context>/application/port/in   input ports (use cases, commands, queries)
src/<context>/application/port/out  output ports
src/<context>/application/service   use-case implementations
src/<context>/adapter/<in|out>/<x>  adapters, one folder per technology
src/<context>/api                   public contract for other contexts
src/<context>/<context>.module.ts   composition root
src/shared/...                      shared kernel + cross-cutting infrastructure
```

```javascript
// .dependency-cruiser.cjs - .cjs because package.json has "type": "module"
/** Matches any resolved module inside node_modules (also through symlinks). */
const NPM = '(^|/)node_modules/';

/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    // ───────────────────────── hygiene ─────────────────────────
    {
      name: 'no-circular',
      severity: 'error',
      comment: 'Circular imports break module initialisation order and usually hide a missing abstraction.',
      from: {},
      to: { circular: true },
    },
    {
      name: 'not-to-unresolvable',
      severity: 'error',
      from: {},
      to: { couldNotResolve: true },
    },
    {
      name: 'no-dev-dependencies-in-production-code',
      severity: 'error',
      from: { path: '^src/', pathNot: '\\.spec\\.ts$' },
      to: { dependencyTypes: ['npm-dev'], dependencyTypesNot: ['type-only'] },
    },

    // ─────────────── layers inside a bounded context ───────────────
    {
      name: 'domain-is-pure',
      severity: 'error',
      comment: 'Domain code depends only on its own domain and the shared kernel: no frameworks, no Node APIs, no outer layers.',
      from: { path: '^src/([^/]+)/domain/' },
      to: { pathNot: ['^src/$1/domain/', '^src/shared/domain/'] },
    },
    {
      name: 'application-does-not-depend-on-adapters',
      severity: 'error',
      comment: 'Dependencies point inwards: the application core never knows its adapters.',
      from: { path: '^src/[^/]+/application/' },
      to: { path: '^src/[^/]+/adapter/' },
    },
    {
      name: 'application-uses-only-allowed-libraries',
      severity: 'error',
      comment: 'Application code may use @nestjs/common (@Injectable) and @nestjs/cqrs only - no ORM, HTTP or validation libraries. Node built-ins (e.g. node:crypto randomUUID) are allowed here, never in the domain.',
      from: { path: '^src/[^/]+/application/' },
      to: { path: NPM, pathNot: `${NPM}@nestjs/(common|cqrs)/` },
    },
    {
      name: 'ports-do-not-depend-on-services',
      severity: 'error',
      from: { path: '^src/([^/]+)/application/port/' },
      to: { path: '^src/$1/application/service/' },
    },
    {
      name: 'input-and-output-ports-are-independent',
      severity: 'error',
      from: { path: '^src/([^/]+)/application/port/(in|out)/' },
      to: { path: '^src/$1/application/port/', pathNot: '^src/$1/application/port/$2/' },
    },
    {
      name: 'adapters-do-not-depend-on-each-other',
      severity: 'error',
      comment: 'Every adapter (adapter/in/web, adapter/out/persistence, …) talks only to the application core, never to a sibling adapter.',
      from: { path: '^src/([^/]+)/adapter/([^/]+/[^/]+)/' },
      to: { path: '^src/$1/adapter/', pathNot: '^src/$1/adapter/$2/' },
    },

    // ─────────────── bounded contexts ───────────────
    {
      name: 'public-api-is-a-self-contained-contract',
      severity: 'error',
      comment: 'src/<context>/api is a published language: plain classes and primitives, no internals, no libraries.',
      from: { path: '^src/([^/]+)/api/' },
      to: { pathNot: '^src/$1/api/' },
    },
    {
      name: 'contexts-only-meet-through-public-apis',
      severity: 'error',
      comment: 'A context reaches another one only through its public API (src/<other>/api) or, in its module file, through the other Nest module.',
      from: { path: '^src/([^/]+)/', pathNot: '^src/shared/' },
      to: {
        path: '^src/[^/]+/',
        pathNot: ['^src/$1/', '^src/shared/', '^src/config/', '^src/[^/]+/api/', '^src/[^/]+/[^/]+\\.module\\.ts$'],
      },
    },
    {
      name: 'only-module-files-import-other-modules',
      severity: 'error',
      from: { path: '^src/([^/]+)/', pathNot: '^src/[^/]+/[^/]+\\.module\\.ts$' },
      to: { path: '^src/[^/]+/[^/]+\\.module\\.ts$', pathNot: '^src/$1/' },
    },
    {
      name: 'core-never-sees-other-contexts',
      severity: 'error',
      comment: 'Domain and application code never use another context, not even its API: an outgoing adapter translates (anti-corruption layer).',
      from: { path: '^src/([^/]+)/(domain|application)/' },
      to: { path: '^src/[^/]+/api/', pathNot: '^src/$1/api/' },
    },
    {
      name: 'shared-kernel-knows-no-context',
      severity: 'error',
      from: { path: '^src/shared/' },
      to: { path: '^src/', pathNot: ['^src/shared/', '^src/config/'] },
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    exclude: { path: '\\.spec\\.ts$' },
    tsPreCompilationDeps: true, // type-only imports count too
    tsConfig: { fileName: 'tsconfig.json' },
    enhancedResolveOptions: {
      exportsFields: ['exports'],
      conditionNames: ['import', 'require', 'node', 'default', 'types'],
      mainFields: ['module', 'main', 'types', 'typings'],
    },
    reporterOptions: {
      text: { highlightFocused: true },
    },
  },
};
```

**Check once per project:** run `npm run lint:arch` and confirm `not-to-unresolvable` reports nothing. ESM sources import `./x.js` while the file on disk is `x.ts`; if those imports show up as unresolvable, adjust the resolver options until they map `.js` to `.ts`.

| Rule | Guards |
|---|---|
| `domain-is-pure` | Pure domain (no libraries, no Node APIs, no outer layers) |
| `application-does-not-depend-on-adapters` | Dependency rule |
| `application-uses-only-allowed-libraries` | Only `@nestjs/common` and `@nestjs/cqrs` from npm in `application/` |
| `ports-do-not-depend-on-services`, `input-and-output-ports-are-independent` | Clean application layer |
| `adapters-do-not-depend-on-each-other` | Adapters are isolated translators |
| `public-api-is-a-self-contained-contract` | `api/` imports nothing |
| `contexts-only-meet-through-public-apis` | Other contexts only via `api/` (adapters) or module files |
| `only-module-files-import-other-modules` | Cross-context module imports only in `<context>.module.ts` |
| `core-never-sees-other-contexts` | Domain/application never import another context's `api/` |
| `shared-kernel-knows-no-context` | `shared/` stays generic |
| `no-circular`, `not-to-unresolvable`, `no-dev-dependencies-in-production-code` | Hygiene |

## ESLint (feedback while typing)

ESLint repeats the two most important layer rules with `no-restricted-imports`, so violations show up in the editor. Type-aware rules catch the classic Node mistakes (floating promises, `any`).

```javascript
// eslint.config.mjs
// @ts-check
import eslint from '@eslint/js';
import eslintPluginPrettierRecommended from 'eslint-plugin-prettier/recommended';
import globals from 'globals';
import tseslint from 'typescript-eslint';

/** Libraries the domain layer must never import (mirrors `domain-is-pure`). */
const FRAMEWORK_PACKAGES = [
  '@nestjs/*',
  '@nestjs-cls/*',
  'nestjs-cls',
  'typeorm',
  'class-validator',
  'class-transformer',
  'express',
  'rxjs',
  'node:*',
];

export default tseslint.config(
  { ignores: ['eslint.config.mjs', '.dependency-cruiser.cjs', 'dist/**', 'coverage/**'] },
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  eslintPluginPrettierRecommended,
  {
    languageOptions: {
      globals: { ...globals.node },
      sourceType: 'module',
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
  },
  {
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      // Never enable `consistent-type-imports` with autofix in a Nest code base:
      // turning a constructor-injected class into `import type` erases the DI
      // token from the emitted decorator metadata and breaks injection at runtime.
    },
  },
  {
    files: ['src/**/domain/**/*.ts'],
    ignores: ['**/*.spec.ts'],
    rules: {
      'no-restricted-imports': ['error', {
        patterns: [
          { group: FRAMEWORK_PACKAGES, message: 'The domain layer is framework-free TypeScript.' },
          { group: ['**/application/**', '**/adapter/**', '**/api/**'], message: 'The domain layer must not depend on outer layers.' },
        ],
      }],
    },
  },
  {
    files: ['src/**/application/**/*.ts'],
    ignores: ['**/*.spec.ts'],
    rules: {
      'no-restricted-imports': ['error', {
        patterns: [
          {
            group: ['typeorm', 'class-validator', 'class-transformer', 'express', '@nestjs/typeorm', '@nestjs/config', '@nestjs/core', '@nestjs/platform-*'],
            message: 'Application services may only use @nestjs/common (@Injectable) and @nestjs/cqrs.',
          },
          { group: ['**/adapter/**'], message: 'The application layer must not depend on adapters.' },
        ],
      }],
    },
  },
  {
    // tests build fakes and mocks; relax the rules that fight them
    files: ['**/*.spec.ts', 'test/**/*.ts'],
    rules: {
      '@typescript-eslint/unbound-method': 'off',
      '@typescript-eslint/no-unsafe-assignment': 'off',
      '@typescript-eslint/no-unsafe-member-access': 'off',
    },
  },
);
```

Tests import `describe`/`it`/`expect`/`vi` from `vitest` explicitly, so no test globals are needed in the lint config.

If the project uses oxlint (the NestJS 12 default for ESM projects), mirror the two `no-restricted-imports` blocks with `overrides` in `.oxlintrc.json` and keep a type-aware check for floating promises. dependency-cruiser stays the source of truth either way.
