# Changelog

All user-facing features and significant product milestones.
Technical fixes, refactors, and CI changes are not listed here.

> **Status: Early Beta.** Features listed below are implemented but not all have been
> thoroughly tested in production conditions. Expect rough edges.

---

<details>
<summary><strong>v2.0 — Q1 2026</strong> · Foundation release · 20+ features shipped</summary>

## v2.0 — Q1 2026

Complete rewrite from v1. New serverless backend (Supabase Edge Functions), new mobile architecture (Capacitor), offline-first data layer, AI task parsing pipeline.

---

### My Day

<details>
<summary><strong>My Day</strong> — personal inbox across all your family groups</summary>

One screen that shows everything that needs your attention today, across all groups you belong to. Sections: Overdue / Today / Assigned to me / High Priority.

Swipe right to complete. Optimistic update - the UI responds instantly, no waiting for network.

Deep link `?openAI=true` opens directly to AI input - accessible from lock screen shortcut.

</details>

---

### Core: AI Task Input

<details>
<summary><strong>AI Task Parsing</strong> — describe tasks in plain language, get a structured checklist</summary>

Type or say "dentist for both kids next Tuesday, remind dad the day before" and CrewTask turns it into a structured task: title, assignees, due date, location, reminders.

Reminders are created as push notifications scheduled at the extracted time. The pipeline detects conflicts with existing tasks and suggests the right assignees based on family context.

</details>

<details>
<summary><strong>Voice Input</strong> — hands-free task capture with smart silence detection</summary>

Tap once, speak naturally. Voice Activity Detection pauses recording after 7 seconds of silence and auto-restarts when you continue speaking. Hard cap at 90 seconds. Multiple voice segments append into one draft automatically.

Works on all Android devices including Huawei and non-Google Android via a three-path fallback: native STT → Web Speech API → server-side transcription.

</details>

<details>
<summary><strong>Draft Auto-Save</strong> — never lose an unfinished task</summary>

Unfinished task input is auto-saved every 15 minutes. Drafts persist across app restarts. Parse and discard when ready.

</details>

---

### Core: Task Management

<details>
<summary><strong>Kanban Board</strong> — shared family task board with filters</summary>

Columns: To Do / In Progress / Done / Cancelled. Filters by status, assignee, priority, and time period. Designed for one-handed mobile use.

</details>

<details>
<summary><strong>Checklists Inside Tasks</strong> — break any task into subtasks</summary>

Any task can have a checklist. Subtasks are part of the task, not separate items. Check them off one by one on the task detail screen.

</details>

<details>
<summary><strong>Recurring Tasks</strong> — pattern-based generator with calendar visualization</summary>

Set a pattern (daily, weekly, custom). The system generates concrete task instances for today automatically. Future instances appear in the calendar as previews. Completing one instance does not affect others.

</details>

<details>
<summary><strong>Location-Aware Tasks</strong> — assign tasks to places</summary>

Add a location to any task by typing or tapping on the map. Tasks with locations appear as markers on the map view. Useful for "pick up from school", "drop off at dojo", etc.

</details>

---

### Family & Groups

<details>
<summary><strong>Multi-Group Architecture</strong> — one account, multiple families or teams</summary>

A user can belong to multiple groups with different roles in each. Data is strictly isolated per group - you only see tasks from groups you are a member of.

</details>

<details>
<summary><strong>Children Profiles</strong> — add kids to tasks without requiring their own accounts</summary>

Add a child profile with a name and photo. Assign tasks to them. Children do not need an email or phone number. When they grow up, they can claim their profile with their own account and all their task history transfers automatically.

</details>

<details>
<summary><strong>Smart Onboarding</strong> — no setup required to get started</summary>

First time you create a task without a group, a personal group "My group" is created automatically. Add family members when you are ready.

</details>

---

### Navigation & UX

<details>
<summary><strong>Bottom Navigation</strong> — My Day / Tasks / Insights / Calendar / Map</summary>

Tab bar designed for one-handed use. Each tab is a distinct view of the family's work.

</details>

<details>
<summary><strong>Four Calendar Views</strong> — day, week, month, agenda</summary>

Switch between views based on how you want to see the week. Tasks with due dates appear across all views. Recurring task previews visible in week and month.

</details>

<details>
<summary><strong>Map View</strong> — see all location-tagged tasks on a map</summary>

All tasks with locations appear as markers. Tap a marker to see the task.

</details>

<details>
<summary><strong>Insights</strong> — family productivity overview</summary>

Task completion trends, overdue patterns, per-member load.

</details>

<details>
<summary><strong>Three Languages</strong> — English, Russian, Spanish</summary>

Full UI localization. Language switching available in settings without app restart.

</details>

---

### Android App

<details>
<summary><strong>Android App Shortcuts</strong> — quick actions from the home screen</summary>

Long-press the app icon to see shortcuts: New Task (opens AI input directly), My Day, Map.

</details>

<details>
<summary><strong>Push Notifications</strong> — task reminders and family updates</summary>

Notifications for task due dates, assignments, and family activity. Delivered even when the app is closed.

</details>

---

### Infrastructure

<details>
<summary><strong>Offline-First Data Layer</strong> — works without internet</summary>

All task operations work offline. Data is cached locally and syncs when connectivity returns. Pending actions survive app restarts - nothing is lost if you close the app before reconnecting.

See [engineering/offline-first.md](../public/engineering/offline-first.md) for the technical details.

</details>

<details>
<summary><strong>Scoped API Keys for AI Integrations</strong> — bring your own AI agents</summary>

Generate API keys from the app settings to connect external AI agents and automation tools. Keys are scoped to specific permissions. Revoke anytime.

</details>

</details>

---

*This changelog covers user-facing features only. Architecture decisions are documented in [docs/architecture/](docs/architecture/).*
