# ADR-001: Supabase Edge Functions over Custom Node.js API

**Date:** 2026-01 | **Status:** Accepted

---

## Context

CrewTask started as a personal project - a simple task manager for one family, built fast with minimal infrastructure. Supabase was the database from day one: managed PostgreSQL, built-in auth, good free tier. No ambitions beyond "solve our own problem."

As the product grew - more features, more ideas, a real product taking shape - the architecture needed to evolve. v1 had grown a custom Node.js REST API layer sitting between the frontend and Supabase. This API handled business logic, called Supabase as a database client, and ran on a DigitalOcean server.

The problem with this setup as complexity increased:
- Monthly server cost regardless of traffic
- Manual deployment pipeline
- Custom auth session handling on top of Supabase Auth (redundant)
- Any serverless AI logic would need to be hosted separately
- Maintenance overhead for one engineer building a product, not infrastructure

The v2 rewrite was an opportunity to simplify.

---

## Options Considered

**Option A: Keep and evolve the custom Node.js API**
- Existing code to build on
- Full control over every layer
- Continued server and deployment overhead
- AI functions would still need separate hosting

**Option B: Move business logic to Supabase Edge Functions**
- Deno-based serverless functions colocated with the database
- Same TypeScript, no separate server to manage
- Built-in secrets management, no separate environment config
- Scales to zero, no idle cost
- Direct database access from functions without network hop

---

## Decision

**Replace the custom Node.js API with Supabase Edge Functions** for all server-side business logic.

The database was already Supabase. Moving the logic layer there eliminated the intermediate server entirely. Edge Functions run in Deno - the TypeScript is the same, the deployment is `supabase functions deploy`, and there is nothing to provision.

The AI pipeline (the core of v2's new features) lives in Edge Functions. This was the right home for it: close to the data, serverless, easy to deploy.

---

## Consequences

**Gained:**
- Zero server management - no DO instance, no deployment pipeline for the API layer
- AI functions colocated with data - no cross-service latency
- Supabase CLI handles local development and deployment
- Infrastructure cost reduced vs v1

**Trade-offs:**
- Deno runtime - some npm packages unavailable, though most common ones work or have alternatives
- Local development requires Supabase CLI; running migrations locally uses Docker, though schema changes can also be applied directly via the Supabase dashboard
