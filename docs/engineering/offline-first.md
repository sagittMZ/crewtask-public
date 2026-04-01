# Offline-First Engineering

How CrewTask works without internet - and why it matters.

---

## Why Offline-First

Connectivity drops everywhere - mobile data gaps, subway, spotty WiFi at school. The app needs to work regardless.

"Works offline" is table stakes. The harder requirement: **everything a user does offline must persist correctly when they come back online**, including tasks created by multiple family members simultaneously.

---

## The Data Flow

```
User action (create/update/delete task)
        │
        ▼
  Optimistic update applied to local cache immediately
        │
        ▼
  Mutation added to persistent queue (IndexedDB)
        │
        ├── Online: mutation executes immediately → server confirms → cache updated
        │
        └── Offline: mutation stays in queue → survives app restart → 
                     executes on reconnect → cache updated
```

The user never waits for a network round-trip to see their action reflected in the UI.

---

## The Persist Cache

TanStack Query's cache is serialized to IndexedDB on every update. On app startup:

1. Cache loads from IndexedDB instantly - UI renders with cached data in milliseconds
2. Background refetch starts - fresh data from server merges into cache
3. User sees stale data for ~200ms then fresh data updates silently

This means the app is fully functional from the first render, regardless of network state.

Cache invalidation is explicit: after a mutation completes (online or replayed from queue), affected queries are invalidated and refetched.

---

## The Mutation Queue

The queue is the hard part. Requirements:

- Mutations must execute in order (create task before updating it)
- Queue must survive tab close, app background, device restart
- Reconnect event must not cause duplicate execution
- Multiple reconnect events in quick succession must not cause race conditions

Implementation details:
- Queue stored in IndexedDB via a custom persister
- Sequential execution guaranteed by the persister (no parallel replay)
- Idempotency handled at the server level where possible
- Post-resume cache invalidation fires after queue drains, not during replay

This was the most test-intensive part of the codebase. Reconnect races, background-to-foreground transitions, and sequential ordering under concurrent network state changes all required explicit test scenarios.

---

## Client-Side IDs

Tasks created offline need an ID before the server assigns one. We use `crypto.randomUUID()` for client-generated IDs.

When the mutation replays and the server responds with the canonical ID, the local cache entry is updated. References from other entities (e.g., a recurring task instance referencing its parent) use the client-generated ID until sync completes.

This approach avoids a class of offline bugs where a second offline mutation references an ID that doesn't exist yet on the server.

---

## Realtime Sync (Online)

When the device is online, Supabase Realtime subscriptions keep the local cache in sync with changes made by other family members:

- Task created by spouse on their phone → appears on your screen within ~500ms
- Task completed by a family member → status updates across all connected devices
- Subscription events map to TanStack Query cache updates directly

The realtime layer and the offline layer are independent. Realtime handles the online multi-user case. The persist queue handles the offline single-user case. They do not interfere.

---

## Recurring Tasks

Recurring tasks add a layer of complexity to offline sync. The architecture:

- A recurring task template stores the pattern (daily, weekly, custom)
- A server-side CRON job generates concrete instances for today each day
- Future instances appear in the calendar as "virtual" (computed from the pattern, not stored)
- Completing or modifying an instance only affects that instance - the template is untouched

This means completing "take out trash every Monday" for this Monday does not affect next Monday's instance. Modifying the recurring pattern from today forward is a separate operation.

Offline: creating and completing instances works the same as single tasks. Pattern modifications are queued and replayed on reconnect.
