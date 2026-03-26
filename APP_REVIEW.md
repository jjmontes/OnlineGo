# OnlineGo — In-Depth App Review

## What It Is

OnlineGo is an Android client for [OGS (Online-Go Server)](https://online-go.com), one of the most
popular platforms for playing Go online. The app lets you play correspondence, live, and blitz games
against other humans, challenge bots, solve tsumego puzzles, explore joseki, follow interactive
tutorials, and track your rating history — all from a single, well-organized mobile interface.

The codebase spans roughly 31,000 lines of Kotlin across 190 source files, and has clearly evolved
over years of active development from a Java/MVP origin into a modern Kotlin-first stack built on
Jetpack Compose, coroutines, Room, and the Molecule library for reactive state management.

---

## What It Does Well

### A Genuinely Complete Feature Set

This is not a half-finished side project. The app covers the full breadth of what a Go player needs:
live and correspondence game play with real-time clock management, an automatch queue, challenge
creation and acceptance, a tsumego solver with hundreds of puzzle collections, a joseki explorer
backed by OGS's opening database, interactive tutorials for beginners, detailed player statistics
with rating charts across board sizes and time controls, and even a local pass-and-play mode and AI
opponent (via KataGo). Very few open-source Go apps — on any platform — offer this range.

### Modern, Well-Layered Architecture

The architecture is clean and consistent. A single-Activity design hosts a Compose `NavHost` with
typed routes. ViewModels manage screen state via Cash App's Molecule library, which composes
reactive flows into immutable `@Immutable` state objects — a sophisticated and increasingly popular
approach that keeps UI recomposition predictable. The 17 repository classes cleanly abstract over
REST, WebSocket, and Room data sources, and Koin provides straightforward dependency injection
without the annotation processing overhead of Dagger/Hilt.

The separation of concerns is strong: the UI layer never touches OkHttp directly, repositories own
caching and synchronization logic, and the game rules engine is isolated behind a JNI boundary with
its own LRU cache. This layering makes the codebase approachable despite its size.

### Excellent Real-Time Communication

The WebSocket layer is the beating heart of the app, and it's implemented with care. Game
connections are reference-counted and pooled, so multiple screens can observe the same game without
redundant socket subscriptions. The protocol handles game data, moves, clock ticks, phase
transitions, stone removal negotiation, undo requests, and chat — all as distinct flows that
repositories can compose independently. Reconnection uses exponential backoff (750ms to 10s), and
the `SocketDebugRepository` provides a debug screen for inspecting live traffic, which is a
thoughtful touch for a project this complex.

### The Board Renderer

The 794-line `BoardComposable` is impressive. It renders the board and stones via Compose Canvas,
supports non-square boards, draws grid lines with star points, handles coordinate labels, shows
territory estimation, AI ownership overlays, move hints with numbered annotations, candidate move
highlights, and animated stone placement and removal (fade-in/fade-out with `Animatable`). It
accepts a rich set of parameters (`hints`, `ownership`, `removedStones`, `drawTerritory`) that make
it reusable across the game, puzzle, joseki, and tutorial screens.

### Thoughtful Data Caching

The offline experience is well considered. Room stores 13 entity types with smart insert/update
logic — for example, `GameDao` preserves existing player data (country, icon) when the server
returns incomplete payloads, preventing UI flicker. Finished games track pagination metadata to
avoid redundant fetches. Puzzle collections enforce a 24-hour refresh cooldown. Messages are
deduplicated by UUID. The cookie jar is persisted across restarts. These are the kinds of details
that distinguish a polished app from a prototype.

### Dependency Choices

The dependency stack is modern and sensible: Retrofit 3 + OkHttp 5, Moshi (lean and fast), Room with
auto-migrations, Coil for image loading, WorkManager for background sync, and Firebase for crash
reporting, analytics, and push notifications. The app targets SDK 36 with a min SDK of 23, covering
a wide device range while staying current.

---

## Where It Can Improve

### Test Coverage Is Minimal

This is the most significant gap. The entire test suite consists of three unit test files (
`GamePresenterTest`, `FaceToFaceViewModelTest`, `RulesManagerTest`) and one Espresso idling
resource. For a 31,000-line codebase with complex real-time state management, WebSocket protocol
handling, and game rule validation, this is a risk. The repository layer — where most of the tricky
synchronization and caching logic lives — has no tests at all. The Molecule-based ViewModels, with
their composed reactive state, would especially benefit from snapshot testing using Turbine (which
is already a dependency). Adding tests for `ActiveGamesRepository`, `OGSWebSocketService`, and the
board position calculations would provide the highest return on investment.

### Some ViewModels Are Oversized

`GameViewModel` weighs in at 1,304 lines and `GameUI` at 1,241 lines. These files handle game state
composition, timer management, move validation, undo logic, stone removal negotiation, analysis
mode, chat, navigation events, sound effects, and more — all in a single class. Breaking out a
`GameTimerManager`, a `MoveSubmissionHandler`, or an `AnalysisModeController` would improve
readability and testability without changing the architecture. The same applies to
`BoardComposable`, which could extract its drawing logic into focused extension functions on
`DrawScope`.

### Mixed Material 2 and Material 3

The theme system supports both Material 2 and Material 3 simultaneously, with parallel color
schemes, typography definitions, and shape specs. This is understandable during a migration, but it
creates ambiguity: some screens use `MaterialTheme.colorScheme` (M3), others use
`MaterialTheme.colors` (M2), and the two can produce inconsistent styling. Completing the M3
migration and removing the M2 fallbacks would simplify the theming code and ensure visual
consistency.

### Legacy View System Still Present

There's a `BoardView.kt` in `ui/views/` — the old Android View-based board renderer — still in the
codebase alongside the Compose `BoardComposable`. If the Compose version has fully replaced it,
removing the legacy code would reduce maintenance surface. If both are still active, consolidating
onto Compose would eliminate the need to maintain two rendering paths.

### Accessibility Is Basic

Content descriptions are applied to some interactive elements, and decorative icons are properly
marked with `contentDescription = null`. But there's no evidence of semantic grouping, custom
accessibility actions for the board (e.g., announcing stone placements or captured groups), focus
management for keyboard/switch navigation, or touch target size enforcement. For a board game where
spatial awareness is critical, investing in TalkBack support — even something as simple as
announcing "Black plays at D4" — would make the app usable for visually impaired players.

### Error Handling Could Be More User-Facing

The data layer has solid error handling: HTTP 429 backoff, IOException retries, Crashlytics logging,
and diagnostic headers on failed requests. But it's not always clear how these errors surface in the
UI. When a move submission fails over WebSocket, or a puzzle collection fetch hits a rate limit, the
user should see a meaningful message rather than a silent failure or a generic loading state. A
lightweight error-channel pattern (e.g., a `SharedFlow<UserMessage>` consumed by a Snackbar) would
close this gap.

### Navigation Could Use Type Safety

The navigation routes are string-based (`"game/{gameId}/{gameWidth}/{gameHeight}"`), which is
fragile — a typo in a route string or a mismatched argument type won't fail until runtime. The newer
Navigation Compose type-safe APIs (using `@Serializable` route classes) would catch these issues at
compile time and reduce boilerplate around argument parsing.

### Only Two DAOs for 13 Entities

`GameDao` and `PuzzleDao` handle all 13 Room entities, which makes `GameDao` in particular a large
interface covering games, messages, challenges, notifications, joseki positions, and opponent
history. Splitting into focused DAOs (e.g., `MessageDao`, `ChallengeDao`, `JosekiDao`) would improve
organization and make queries easier to locate.

### No Offline-First Guarantee

While caching is thoughtful, the app doesn't appear to offer a true offline experience — you can
browse cached games and puzzles, but there's no explicit offline queue for moves or challenges. For
correspondence players who often have spotty connectivity, queuing moves locally and syncing when
reconnected would be a meaningful improvement.

---

## Summary

OnlineGo is an impressive, feature-rich open-source Go client built on a modern and well-structured
Android stack. It handles the complexity of real-time multiplayer gaming, game rule enforcement via
JNI, and a rich interactive board renderer with composure and consistency. The architecture is
clean, the dependency choices are sound, and the feature set rivals commercial apps.

The main areas for growth are test coverage (the single biggest risk for long-term maintainability),
decomposing a few oversized files, completing the Material 3 migration, and investing in
accessibility. None of these are architectural problems — they're the natural next steps for a
mature, actively developed project.

For a primarily solo-developed open-source app, this is high-quality work.
