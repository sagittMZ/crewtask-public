# CrewTask

**A Family OS for overwhelmed parents.**

Describe a task in plain language - voice or text. CrewTask parses it into a structured checklist, assigns it to the right people, and keeps the whole family in sync even without internet.

---

## The Problem

Managing a household with kids is a coordination problem. Tasks come from everywhere - voice notes, WhatsApp messages, half-formed thoughts at 11pm. Most task apps are built for solo work or office teams. None of them handle the chaos of family life.

---

## What It Does

- **AI task parsing** - type or say "dentist for both kids next Tuesday, remind dad" and get a structured task with assignees, due date, and reminders
- **Voice input** - hands-free capture with auto-restart and Voice Activity Detection
- **Offline-first** - works without internet, syncs when connection returns
- **My Day** - personal inbox across all family groups, sections: Overdue / Today / Assigned
- **Kanban board** - shared family task board with filters by status, assignee, priority
- **Recurring tasks** - pattern-based generator with calendar visualization
- **Location-aware tasks** - assign tasks to places, see them on a map
- **Android app** - native experience via Capacitor, iOS in progress

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript, Vite, TailwindCSS |
| State + Offline | TanStack Query v5 + IndexedDB persist cache |
| UI components | Ionic + custom Tailwind components |
| Backend | Supabase (PostgreSQL, Auth, Realtime, Storage) |
| Serverless | Supabase Edge Functions (Deno) |
| AI | Multi-model pipeline via Edge Functions |
| Mobile | Capacitor (Android shipped, iOS in progress) |
| CI/CD | GitHub Actions |
| E2E testing | Playwright |

---

## Architecture

- **[Architecture Decisions](docs/public/architecture/)** - why we chose Supabase over a custom backend, Capacitor over React Native, offline-first via TanStack Query
- **[Engineering](docs/public/engineering/)** - how the AI orchestration pipeline and offline sync work
- **[Changelog](CHANGELOG.md)** - feature history in human-readable format

---

## Key Design Principles

**Offline-first, not offline-capable.**
The app works fully without internet. Data lives locally first, syncs in the background. Connectivity drops everywhere - mobile data gaps, subway, spotty WiFi.

**AI as input layer, not AI as a feature.**
The AI pipeline is the entry point for task creation. The entire UX assumes structured input is generated, not typed. Voice and natural language are first-class citizens.

**One codebase, three platforms.**
React web app wrapped with Capacitor for Android and iOS. One engineer, three platforms, no compromise on UX.

**Security by default.**
Row Level Security enforced at database level. Every data access pattern goes through RLS policies. Service role key never exposed to user-facing flows.

---

## About

Built by a solo founder with 10+ years in QA and startup engineering.

- [Portfolio](https://crewtask.app/portfolio)
- [About the Founder](https://crewtask.app/founder)
- [Presentation](https://crewtask.app/pitch)
- [Release Notes](https://crewtask.app/releasenotes)
- [Twitter / X](https://x.com/s_a_g_i_t_t)
- [LinkedIn](https://www.linkedin.com/in/marina-zorina)

---

*Architecture docs sync automatically from the private repository on every release.*
