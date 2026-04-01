# ADR-002: Capacitor over React Native

**Date:** 2026-01 | **Status:** Accepted

---

## Context

CrewTask needs native mobile apps on Android and iOS. The target users are parents - the app lives on their phones, not in a browser. PWA alone was not sufficient: app store distribution and home screen shortcuts require native packaging. Lock screen widgets require native code regardless of the framework chosen.

The project is built entirely with AI assistance - no traditional hand-coding. The architecture decision needed to minimize the surface area of platform-specific work while delivering a real mobile experience.

---

## Options Considered

**Option A: React Native**
- True native components
- Completely separate codebase from web
- Large ecosystem
- Would require maintaining parallel web and mobile codebases

**Option B: Capacitor (Ionic)**
- Web app wrapped in a native shell
- Single codebase: web, Android, iOS
- Ionic components handle platform-specific UI patterns (safe areas, scroll, navigation)
- Access to native APIs via plugins (push notifications, preferences, deep links, camera)
- One CI/CD pipeline

**Option C: Flutter**
- Dart - completely different language, full rewrite
- Not considered seriously given the rewrite cost

---

## Decision

**Capacitor with Ionic components.**

The single-codebase argument was decisive. Web, Android, and iOS from one codebase means every feature ships to all platforms simultaneously. There are no "web version" vs "mobile version" divergence problems.

The constraint was simple: one codebase for all platforms, minimum platform-specific code.

---

## Consequences

**Gained:**
- One codebase for web, Android, iOS
- Android build ships via GitHub Actions CI
- Capacitor plugins cover the necessary native API surface: push notifications, preferences storage, deep links, hardware back button
- iOS builds planned via Firebase App Distribution (no Mac required for CI)

**Trade-offs:**
- WebView performance ceiling - not suitable for heavy animations or computation (acceptable for task management)
- Platform-specific bugs are harder to diagnose in a WebView context
- Some platform behavior requires explicit configuration (`capacitor.config.ts`, safe area insets, edge-to-edge on Android 15)

**Current status:**
- Android: shipping
- iOS: build pipeline in progress - not yet released
