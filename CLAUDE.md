# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

**Done — Faz 0–12 implemented.** The monorepo, Prisma schema, auth, document
CRUD, Go sync server, persistence, frontend, history/replay, export, and
sharing/roles/public-links are in place; the spec bundle under
`docs/` remains the rationale reference. Read the relevant file before extending a phase:

- `docs/CLAUDE.md` — source-of-truth project context (Turkish): decisions, invariants, commands, roadmap.
- `docs/claude-code-playbook.md` — operational workflow, subagent routing, common-mistake warnings.
- `docs/collab-editor-spec.zip` — the full intended layout (root docs tree, `README.md`,
  ADRs 0001–0005, concept notes, and `.claude/agents/`). Retained for rationale.

This file is the English summary of that spec. When the two disagree, `docs/CLAUDE.md` wins.

## What It Is

A full-stack real-time collaborative Markdown editor: many users edit the same document
simultaneously with **no lost writes**. Conflict resolution via **CRDT (Yjs `Y.Text`)`**,
synchronization via a **custom WebSocket server**, persistence via **op-log + periodic
snapshot checkpoint** (event-sourcing-lite).

Core problem being solved: naive last-write-wins loses data. With CRDT each replica edits
independently and merges deterministically — no edit is dropped. "Optimistic concurrency"
here means **reconcile, not reject**.

## Architecture — Fixed Decisions (do not change; rationale in docs/adr)

| Concern | Choice | Why |
|---------|--------|-----|
| CRDT type | Yjs `Y.Text` | Markdown is plain text; ProseMirror node tree unnecessary |
| Editor | CodeMirror 6 + `y-codemirror.next` | Mature Yjs binding, Markdown lang support |
| Sync protocol | `y-protocols` (sync + awareness) | Battle-tested wire format; don't reinvent |
| WS server | Custom `ws` server, **separate process** from Next | Next App Router has no native WS; auth + persistence are ours |
| Persistence | Postgres: op log + periodic snapshot compaction | Replay + audit + performance balance |
| Initial load | Server Component + Server Action (create/list), then client WS takes over | SSR first state, client owns live sync |
| ORM | Prisma | Migrations + type safety |
| Auth | JWT (httpOnly cookie), verified at WS handshake | Same identity for REST and WS |

## Monorepo Layout (pnpm workspaces)

```
apps/
  web/          # Next.js App Router — UI, Server Actions, REST auth
services/
  api-go/       # Standalone Go sync server — Yjs sync + persistence + mirror JSON API
packages/
  db/           # Prisma schema + client (imported by web; Go uses pgx/goose on the same schema)
  protocol/     # Shared WS message types, auth token verification
  shared/       # Shared types, zod schemas
```

Key rule: **web and api-go are separate processes.** They share the DB schema
(Prisma for Node, pgx/goose for Go), not through Next. The sync server is
deliberately NOT inside Next (see ADR 0002).

## Invariants — Never Violate

1. Server stores/transfers Yjs state as **binary update** — never a plain-text diff.
2. Every WS connection passes **auth before its first message**. An unverified socket cannot
   read or write any document room.
3. Op log is **append-only**. Order is **snapshot commit → prune** — never prune updates
   before their covering snapshot is written.
4. A document's authoritative server `Y.Doc` is a **single instance in one process**
   (single-node scope for now; scaling discussed in ADR 0005, not implemented).
5. Version / state vector is managed **by Yjs**. Do **not** hand-maintain an integer
   `version` column — use Yjs's state vector.

If a proposed solution would violate one of these, STOP and flag it (by number) first.

## Common Mistakes to Avoid (from playbook §4)

- Opening a WebSocket inside a Next Server Component → wrong. Use the separate sync server (`services/api-go`) + a client component.
- Syncing text as a plain-string diff → violates invariant #1. Use binary updates.
- Adding and incrementing a manual `version: int` column → violates invariant #5.
- Pruning the op log before the snapshot is committed → violates invariant #3.
- Persisting awareness (cursor/presence) data to the DB → it is ephemeral; do not persist.
- Deferring the auth handshake and adding it later → violates invariant #2. Handshake precedes the first message.

## Phase Roadmap (complete — all phases shipped with green tests)

Kept as build history. Later tracks (F10 sharing polish, F11 history, F12 roles/public-links/export)
landed on top; their notes live under `docs/f12-demo/` and in git history.

0. Monorepo skeleton, Prisma schema + migration, health check
1. Auth (register/login, JWT cookie, Server Actions)
2. Document CRUD (Server Actions + Server Component listing / SSR load)
3. WS server: `y-protocols` sync, auth handshake, in-memory `Y.Doc` rooms
4. Persistence: update append, load-on-open, snapshot compaction
5. Frontend: CodeMirror + Yjs binding, awareness (cursor/presence)
6. Op-log replay endpoint + audit view (event-sourcing demo)
7. Tests (multi-client Playwright), hardening, README + architecture diagram

## Subagent Routing (from playbook §2)

Four subagents are defined inside the spec zip (`.claude/agents/`) but **not yet installed**.
Once extracted, route by domain:

| Work | Subagent |
|------|----------|
| Prisma schema, migration, op log, snapshot, replay | persistence-engineer |
| Yjs sync, y-protocols, awareness, WS message logic | crdt-sync-engineer |
| Next pages, Server Actions, auth, SSR bootstrap | nextjs-app-engineer |
| Vitest / Playwright, multi-client scenarios | test-e2e-engineer |

## Tech Versions (pinned)

Node 20 LTS · pnpm 9 · Next.js 15 (App Router, React 19) · yjs 13 · y-protocols ·
y-codemirror.next · codemirror 6 · ws 8 · PostgreSQL 16 · Prisma 6 · TypeScript 5 (strict) ·
zod · jose (JWT). Tests: vitest (unit), playwright (multi-client E2E).

## Commands (valid after Faz 0 scaffolding)

```bash
pnpm install
pnpm db:migrate            # prisma migrate deploy (apply pending; Prisma owns the schema)
pnpm db:migrate:dev        # author a NEW migration (needs a Prisma-only DB — see README)
pnpm migrate:go            # Go baseline + FK constraints, after Prisma
pnpm db:studio             # prisma studio
pnpm dev                   # web (:3000) + Go sync server (:8080) in parallel
pnpm --filter web dev
pnpm dev:go                # Go sync server alone
pnpm test                  # vitest
pnpm test:e2e              # playwright multi-client
pnpm test:go               # Go unit tests
pnpm typecheck
```

## Conventions

- Explain to the user in **Turkish**; write **code and commit messages in English**.
- View a package's existing files before writing into it — don't assume.
- State the rationale (one sentence) before adding a new dependency.
- A phase is not "done" without its tests written and green; keep `pnpm typecheck` clean.
