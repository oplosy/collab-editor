# Coding Conventions

Collection of conventions applied across the workspace packages. Not strict rules — the goal is to keep packages consistent.

## General

- TypeScript everywhere; `tsconfig.base.json` is the shared baseline, packages may only tighten it.
- Lint and formatting run via `eslint.config.mjs` and prettier; run them before committing.
- Package manager is pnpm; the lockfile is the source of truth.

## Repo structure

- `apps/` — deployable applications.
- `packages/` — shared libraries.
- `services/` — backend services.
- `docs/` — documentation (this folder).

## Notes

- Keep shared types in `packages/`, not duplicated inside apps.
- Prefer small focused modules over utility grab-bags.
