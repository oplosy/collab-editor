# Shared database migration workflow

The web and Go services share one PostgreSQL schema, but they use different migration tools. Prisma owns schema creation; the Go service uses Goose for its baseline and foreign-key constraints. Keep that order consistent to avoid competing schema histories.

## Fresh local database

From the repository root:

1. Copy .env.example to .env and set DATABASE_URL and JWT_SECRET.
2. Start PostgreSQL with docker compose up -d.
3. Apply Prisma migrations with pnpm db:migrate.
4. Apply the Go baseline and constraints with pnpm migrate:go.
5. Regenerate the Prisma client with pnpm db:generate.
6. Start the web and Go processes with pnpm dev.

## Authoring a Prisma migration

Use pnpm db:migrate:dev to create a migration. Run this command against a Prisma-only database: the Go Goose tracking table appears as schema drift to Prisma, so do not author migrations against a database after the Go migration step has run.

After changing the Prisma schema, regenerate the client with pnpm db:generate. Check the migration and shared schema assumptions before changing either service; do not introduce a second schema owner.