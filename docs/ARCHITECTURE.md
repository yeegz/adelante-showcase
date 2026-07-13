# Adelante — Architecture

A deeper look at how Adelante is built. This describes the system and the notable
engineering decisions; it does not include source.

## Principles

1. **Widget-first.** The widgets are the product; the app configures them.
2. **Offline-first.** The app is fully usable from a bundled, curated library. The
   cloud only adds fresh content — it is never required.
3. **Private by design.** No account, no analytics/ad SDKs, no per-user data leaves
   the device. Faith and preferences are local-only.
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

### 6. Backend (Firebase — optional)
- **Firestore** stores the content library, version records, sources, and reviews —
  **no user data**.
- **Cloud Functions (TypeScript):**
  - `scheduledContentImport` — pulls from an **allow-list** of trusted, license-clean
    sources, validates, de-duplicates (text/citation fingerprint), quality-scores,
    then auto-publishes trusted feeds or queues the rest for review, and bumps a
    content version.
  - `getContent` — an HTTPS endpoint (with **App Check**) that returns the approved
    bundle with `?since=` delta support and CDN caching.
  - admin curation callables (approve / reject) and a one-time admin bootstrap.
- **Client sync** is an anonymous, read-only HTTPS fetch — it sends no preferences or
  identifiers. Approved content is merged into an on-device cache that feeds the
  widgets; any failure leaves the bundled/cached library intact.

## Privacy model

| Data | Where it lives |
| --- | --- |
| Categories, faith, favorites, widget configs, theme | On device only |
| Motivational content | Public library (device + optional backend) |
| Analytics / ads / identifiers | **None** |

The only network activity is the optional, anonymous content fetch. Uninstalling
removes all local data; in-app resets clear it too.

## Quality

- ~80 automated tests: content-filter correctness, no-starvation coverage across
  every category/faith, refresh behavior, widget parity (source-level), and the
  sync/cache layer.
- `flutter analyze` clean; real-device **iOS** and **Android** release builds.

## Tooling

Custom scripts generate the app icon from the brand mark, bundle fonts into the iOS
extension target, and export the curated library to JSON for backend seeding.
