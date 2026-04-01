# ADR-003: Offline-First via TanStack Query + IndexedDB

**Date:** 2026-03 | **Status:** Accepted

---

## Context

Connectivity is unreliable everywhere - mobile data gaps, subway tunnels, spotty WiFi. The product fails its core promise if it blocks when there is no internet.

"Offline-capable" (graceful degradation) was not the goal. "Offline-first" (works fully offline, syncs when connected) was the requirement.

Initial v2 implementation loaded data via direct Supabase client calls. When network was unavailable, the UI blocked. This needed to change before the Android release.

---

## Options Considered

**Option A: WatermelonDB**
- Purpose-built offline database for React Native / mobile
- Full local database with complex query support
- Requires writing adapters for every data type
- More complexity than needed for the current data model
- Better fit when data relationships are deeply nested and frequently queried offline

**Option B: PowerSync**
- Managed sync service - handles conflict resolution, offline queue
- Excellent DX for offline-first patterns
- Additional vendor dependency and cost
- Significant migration effort from existing Supabase queries

**Option C: TanStack Query v5 + IndexedDB persist cache**
- TanStack Query already in the stack for data fetching
- `@tanstack/query-persist-client-core` + `idb-keyval` adds persistent cache
- Mutation queue with `mutationPersister` survives tab close and app restart
- Optimistic updates provide instant UI response
- No additional infrastructure, no new vendor

---

## Decision

**TanStack Query v5 with IndexedDB persistence and a custom mutation persister.**

The key insight was that TanStack Query already sat between the UI and Supabase. Adding persistence at that layer required no architectural change to the rest of the app - just configuration and a custom persister for queued mutations.

The implementation:
- Query cache serialized to IndexedDB - UI loads from cache instantly on startup, even offline
- Mutations queued with custom persister - survive browser/app restart, replay on reconnect
- Optimistic updates applied immediately - no waiting for network
- Conflict resolution: last-write-wins on the server, UI reflects server state on sync

---

## Consequences

**Gained:**
- UI loads instantly from cache regardless of network state
- All write operations work offline - task creation, status updates, assignments
- Queued mutations survive app restart - no lost actions
- Zero additional infrastructure or vendor cost
- Same API for online and offline data access throughout the codebase

**Trade-offs:**
- Custom mutation persister required careful handling of: sequential execution order, reconnect race conditions, post-resume cache invalidation
- `crypto.randomUUID()` used for client-side IDs to avoid collisions before server sync
- IndexedDB has storage limits - not an issue for task data volumes, worth monitoring at scale
- True conflict resolution (two users editing the same task offline simultaneously) is last-write-wins - acceptable for the current use case, revisit at scale

**Test coverage:**
The persister and sync logic are among the most tested parts of the codebase. Race conditions between reconnect events and queued mutation replay required specific test scenarios to get right.
