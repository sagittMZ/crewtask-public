# Security Architecture

**Principle: enforce access control where data lives, not where code runs.**

---

## Row Level Security as the Foundation

Every table in the database has Row Level Security enabled. There is no code path - not in Edge Functions, not in the frontend, not in any integration - that bypasses RLS.

This means: even if application code has a bug, even if an API key leaks, even if a request is forged - the database will not return data the authenticated user is not allowed to see.

The corollary: the service role key (which bypasses RLS) is never used in user-facing flows. It exists only for administrative operations and runs only in controlled server-side contexts.

---

## API Key Architecture for AI Integrations

Third-party AI agents and integrations authenticate with scoped, user-generated API keys - not with the service role key and not with the user's session JWT.

The flow:
- User generates an API key from the app settings
- Key is stored as a SHA-256 hash (never plaintext)
- Each key has explicit scopes (e.g., `tasks:create`)
- Edge Functions validate the key hash and scope before acting
- All actions performed via API key are scoped to the generating user's data

This ensures that external integrations (n8n workflows, custom AI agents, MCP connectors) operate with the minimum necessary permissions.

---

## User Data Isolation

Family data is isolated by group. A user can belong to multiple groups with different roles. RLS policies enforce that queries only return data belonging to groups the authenticated user is a member of.

Children can have profiles without email addresses (ghost profiles). These profiles are fully isolated - they cannot authenticate and their data is only accessible to group members with appropriate roles.

---

## What Is Not in This Document

Specific policy definitions, table structures, and implementation details are not published. The above describes the philosophy and approach. The implementation lives in the private repository.
