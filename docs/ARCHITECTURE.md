# Adelante — Architecture

A deeper look at how Adelante is built. This describes the system and the notable
engineering decisions; it does not include source.

## Principles

1. **Widget-first.** The widgets are the product; the app configures them.
2. **Offline-first.** The app is fully usable from a bundled, curated library. The
   cloud only adds fresh content — it is never required.
3. **Local preferences.** No account is required and no third-party analytics or
   advertising SDK is used. Faith and preferences stay on the device. Optional
   aggregate event reporting is off until the reader consents.
4. **One source of truth.** The widget shows exactly what the app last published.

## Layers

### 1. Flutter app
- **State:** Riverpod, with a single controller/notifier owning preferences, the
  active widget config(s), the daily selection, and content sync.
- **Design system:** a dedicated tokens + typography layer (spacing, radii,
  durations, semantic type roles) so the whole app shares one calm rhythm; the UI
  is split into a reusable component library rather than one monolith.
- **Content model:** a rich, validated quote model (type, author, source, citation,
  translation, category, faith, tags, license status, review/approval flags,
  quality score, timestamps).

### 2. The widget bridge
State flows to the native widgets through a small, generic key/value payload written
to a shared store (iOS **App Group** `UserDefaults`; Android `SharedPreferences`).
The app publishes the current quote, styling, font token, and metadata, then asks
the OS to reload the widget. The native side maps that payload to its own resources.
Adding a capability is one new key on both sides.

### 3. iOS widget (WidgetKit · SwiftUI)
- Families: `systemSmall/Medium/Large/ExtraLarge`, `accessoryInline/Circular/Rectangular`, and StandBy.
- **Fonts** are bundled into the *extension* target (not just the app) and declared in
  the extension's `UIAppFonts`, then resolved by **PostScript name** — the family
  name silently falls back to system. The `.ttf`s are added to the target via an
  `xcodeproj` automation script.
- **Auto-fit:** a `ViewThatFits` ladder of descending sizes, each free-wrapping, with
  a final `minimumScaleFactor` — so a long quote re-wraps or shrinks, never clips.
- **Backgrounds:** `containerBackground` (iOS 17+) with a graceful fallback.

### 4. Android widget (App Widgets · RemoteViews · Kotlin)
- **Gradients & rounded corners:** rendered to a `Bitmap` and set on an `ImageView`,
  because RemoteViews can't paint gradients directly — reaching iOS parity.
- **Dynamic sizing / alignment:** padding, gravity, and uniform **XML auto-size**
  applied per-widget; notably, a runtime `setTextViewTextSize` call is avoided
  because it silently disables XML auto-size.
- **Fonts:** RemoteViews cannot set an arbitrary typeface at runtime, so each font is
  its own **baked-in layout variant** and the provider selects the layout by token.

### 5. Content system
- **Strict filter:** content selection happens before scoring. A specific-faith
  selection admits *only* that tradition; secular content appears only in
  explicitly chosen secular categories. Cross-tradition contamination is impossible
  by construction, and asserted by tests.
- **Recommendation & refresh:** a scoring engine ranks eligible content by the user's
  categories/faith, avoids recently shown items, and advances on the chosen cadence.
- **Safety:** scripture ships with exact citations and public-domain translations;
  reflections are labeled as such; nothing invents attributions.

### 6. Backend (Firebase)
- **Firestore** stores the content library, version records, sources, reviews and
  optional aggregate usage counters. It does not build user or device profiles.
- **Cloud Functions (TypeScript):**
  - `scheduledContentImport` — pulls from an **allow-list** of trusted, license-clean
    sources, validates, de-duplicates (text/citation fingerprint), quality-scores,
    then publishes eligible records or queues the rest for review, and bumps a
    content version.
  - `getContent` — an HTTPS endpoint (with **App Check**) that returns the approved
    bundle with `?since=` delta support and CDN caching.
  - admin curation callables (approve / reject).
  - opt-in aggregate event collection and an authenticated reporting endpoint.
- **Client sync** is an anonymous, read-only HTTPS fetch — it sends no preferences or
  identifiers. Approved content is merged into an on-device cache that feeds the
  widgets; any failure leaves the bundled/cached library intact.

## Privacy model

| Data | Where it lives |
| --- | --- |
| Categories, faith, favorites, widget configs, theme | On device only |
| Motivational content | Public library (device + optional backend) |
| Optional usage counts | Aggregate server counters, only after consent; no user or device profile |
| Advertising | None |

Content updates use the backend. If a reader opts into usage reporting, allowlisted
events also go to the server; preference values and free text are not sent. Fresh
event IDs support retry deduplication, not user identification. Reporting counts
consenting sessions and events, not unique people. Turning it off clears queued
events. Local preferences and saved content remain on the device.

## Quality and release status

Source inventory checked on 12 October 2026:

- 478 Dart test declarations across 77 files: 363 unit/contract and 115 widget
  declarations. Native source-text guards are included; they do not execute the
  Swift or Kotlin behaviour.
- 105 backend test declarations: 68 unit and 37 emulator/operations declarations.
- 20 authored Watch XCTest methods: 11 core and 9 UI. Generated `.build` files
  are excluded.

These are source counts, not a coverage percentage or a fresh full-suite run.
The build 21 delivery note dated 2 October records 566 executed Flutter tests
passing; parameterised declarations produce more executed cases. The configured
GitHub Actions workflow had no recorded runs when checked on 12 October.

Adelante 1.0.0 is [live on the App Store](https://apps.apple.com/us/app/adelante-daily-quotes/id6816598027)
since 4 October 2026, including its Apple Watch app and Messages extension.
Android widgets are implemented, but the Android app is not published on Google
Play. The [case study](https://yousofselim.com/work/adelante/) documents the release
scope and remaining platform checks.

## Tooling

Custom scripts generate the app icon from the brand mark, bundle fonts into the iOS
extension target, and export the curated library to JSON for backend seeding.
