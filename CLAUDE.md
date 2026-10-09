### 1. Plan Mode Default
- Enter plan mode for ANY not-trivial task (3+ steps or architectural decisions)
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until the mistake rate drops
- Review lessons at session start for a project

### 3. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

### 4. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes. Don't overengineer
- Challenge your own work before presenting it

## Core Principles
- **Simplicity First**: Make every change as simple as possible. Impact minimal code
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards

## Project General Instructions

- Always use the latest stable versions of dependencies (check with `npm view <package> version`).
- Always write the application in TypeScript (`strict` mode) as a NestJS application.
- Always use npm for dependency management and commit `package-lock.json`. Use `npm ci` in CI and Docker builds.
- Use ES modules (the NestJS 12 default): `"type": "module"` in package.json, `"module": "nodenext"` in tsconfig, and `.js` extensions on relative imports.
- Target the current Node.js Active LTS release and declare it in `engines.node`.
- Always create test cases for the generated code both positive and negative (Vitest unit tests and supertest e2e tests).
- Always generate the CircleCI pipeline in the .circleci directory to verify the code (lint, build, unit and e2e tests).
- Minimize the amount of code generated.
- The `name` field in package.json must be the same as the parent directory name.
- Use semantic versioning in package.json. Each time you generate a new version, bump the PATCH section of the version number.
- Organize code in feature modules under `src/` (one folder per module, e.g. `src/products/`).
- Use TypeORM for relational data access. Never enable `synchronize` outside tests; manage schema changes with migrations.
- Do not use `any`. Prefer precise types, and `unknown` with narrowing when the type is truly unknown.
- Generate the Docker Compose file to run all components used by the application.
- Update README.md each time you generate a new version.
