# Swift iOS App Structure — AI Prompt Guide

---

## STEP 0 — Read Context First, Ask Only When Stuck

This document applies to both new projects and ongoing code changes. Do not ask clarifying questions unless you genuinely cannot determine the answer from context.

**Infer from context first:**
- Existing files, folder structure, and imports tell you the tier and backend mode
- If `FirebaseAuth` or `Firestore` is imported → Firebase mode
- If `URLSession` + `APIService.swift` exists → REST mode
- If there are no service files → Local-only
- Screen count and store structure reveal the tier

**Only ask the user when:**
- Starting a brand-new project with no existing code to read
- The request involves a pattern not covered anywhere in this document
- Context is genuinely contradictory or unclear

**Never ask about something already established in the existing codebase.** If the project is already in Tier 2 with Firebase, continue in that pattern without confirming it again.

---

## Tier 1 — Small App (1–5 screens, solo builder)

**When to use:** MVP, prototype, solo project, <5 screens.

**State pattern:** Single global store + `@EnvironmentObject`

**Tradeoffs:** Fastest to build. Re-renders are acceptable at this scale. God-object risk is low when the app is small. Use this unless the app is clearly medium or large.

### Folder Structure
```
- Screens/{FeatureName}/        → full screen views only
- Components/                   → reusable views used across 2+ screens
- Models/                       → Swift structs, Codable, Identifiable, Hashable
- Services/                     → API or Firebase calls only, no business logic
- Store/                        → single AppStore, ObservableObject, @MainActor
- Navigation/                   → AppRouter, tab view, NavigationStack only
- Theme/                        → Color+Theme.swift and Font+Theme.swift only
```

Local-only apps: delete `Services/` entirely. No network layer needed.

### State Management
- All state lives in a single class: `{AppName}Store`
- Mark it `@MainActor` and `ObservableObject`
- Use `@Published` for every piece of state that drives UI
- Persist state to `UserDefaults` using a nested `PersistedState` struct with `Codable`
- Never store state inside a View — pass it via `@EnvironmentObject`
- Use computed properties on the store for derived values
- Never put UI state (sheet open, alert shown) in the global store — use `@State` locally

### Navigation
- Use `NavigationStack` with a typed enum for all route transitions
- Use `@AppStorage("hasOnboarded")` for first-launch routing, not state
- Tab navigation lives in `MainTabView` only
- Onboarding navigation lives in `AppRouter` only — use enum-driven step tracking
- Never use `NavigationLink` for programmatic navigation — use the router enum

### Views & Screens
- Keep screens under ~300 lines — extract any section over 80 lines into `Components/`
- Pass the store via `@EnvironmentObject`, not as an init parameter
- Use `Timer.publish(every: 1, on: .main, in: .common)` for real-time updates
- Use `.onReceive` to bind the timer to state updates

---

## Tier 2 — Medium App (6–15 screens)

**When to use:** Multi-feature app, growing solo project, or small team.

**State pattern:** Feature-scoped `@Observable` view models per screen. Single shared store only for truly global state (auth session, user profile, app config).

**Tradeoffs:** Eliminates the god-object. Views only re-render when the specific property they access changes. Requires iOS 17+. More files, but cleaner separation.

### Folder Structure
```
- Screens/{FeatureName}/
    - {Feature}View.swift          → the screen
    - {Feature}ViewModel.swift     → @Observable view model for this screen only
- Components/                      → reusable views, no view model inside
- Models/                          → shared data structs
- Services/                        → API or Firebase calls only
- Store/                           → AppStore for global state only (auth, user, config)
- Navigation/                      → AppRouter only
- Theme/                           → Color+Theme.swift, Font+Theme.swift
```

### State Management
- Use `@Observable` (Swift 5.9 / iOS 17+) for view models — not `ObservableObject`
- Each screen has its own view model: `{Feature}ViewModel`
- `AppStore` holds only: current user session, auth state, global config
- View models are created with `@State var viewModel = {Feature}ViewModel()` in the screen
- Pass only what a child view needs — never the whole view model
- `@EnvironmentObject` is used only for `AppStore` — never for feature view models
- Persist global state to `UserDefaults` via `AppStore`. Persist feature state locally in the view model (in-memory only unless explicitly needed).

### When to promote state to AppStore
- Auth/session state
- User profile data needed across 3+ screens
- Global flags (onboarding complete, feature flags)
- Anything that must survive screen dismissal

---

## Tier 3 — Large App (15+ screens or team)

**When to use:** Multi-developer team, complex domain, enterprise app.

**State pattern:** Full dependency injection. No `@EnvironmentObject`. Explicit init injection only.

**Tradeoffs:** Maximum testability. Compile-time safety. Slower to set up. Use only when team size or complexity demands it.

### Key rules
- No `@EnvironmentObject` anywhere — inject dependencies through init
- Each feature is a self-contained module: its own models, view model, and service protocol
- All services are protocol-backed for testability (`protocol FastingServiceProtocol`)
- `AppStore` is replaced by a `DependencyContainer` passed from the root
- Unit tests can inject mock services without needing the full app

---

## Backend Mode

**Ask the user which backend they're using before writing any service code.**

### Local-only
- Delete `Services/` entirely
- Persist everything to `UserDefaults` (non-sensitive) or `Keychain` (sensitive)
- No `AuthManager`, no `APIService`

### Firebase (Auth + Firestore)
Add two files to `Services/`:

**`AuthManager.swift`**
```swift
@MainActor
final class AuthManager: ObservableObject {
    @Published var currentUser: User? = nil
    private var handle: AuthStateDidChangeListenerHandle?

    func startListening() {
        handle = Auth.auth().addStateDidChangeListener { [weak self] _, user in
            self?.currentUser = user
        }
    }

    func stopListening() {
        if let handle { Auth.auth().removeStateDidChangeListener(handle) }
    }

    func signOut() throws {
        try Auth.auth().signOut()
    }
}
```

**`FirebaseService.swift`**
- All Firestore reads/writes go here as `async throws` functions
- Use `AsyncStream` for real-time listeners — never put `.addSnapshotListener` directly in a view or store
- Return typed Swift structs, never raw Firestore `DocumentSnapshot`
- No business logic — only encode, read/write, decode

**Firestore listener pattern:**
```swift
func fastStream(userID: String) -> AsyncStream<[Fast]> {
    AsyncStream { continuation in
        let listener = db.collection("users").document(userID)
            .collection("fasts")
            .addSnapshotListener { snapshot, _ in
                let fasts = snapshot?.documents.compactMap {
                    try? $0.data(as: Fast.self)
                } ?? []
                continuation.yield(fasts)
            }
        continuation.onTermination = { _ in listener.remove() }
    }
}
```

**Persistence rules with Firebase:**
- `UserDefaults` — non-sensitive local state (streak cache, onboarding flag)
- `Keychain` — never store Firebase tokens manually; Firebase SDK handles this
- `Firestore` — source of truth for user data when online
- Always design for offline-first: local state is the UI source of truth, Firestore syncs in background

### REST API
- All calls go in `Services/APIService.swift` as `static async throws` functions
- Use `URLSession.shared` with `async/await` — no Combine, no callbacks
- Required headers on every request:
  ```
  Content-Type: application/json
  X-App-Version: 1.0.0
  X-Client: {app-name}
  X-Platform: IOS
  ```
- Return typed `Codable` responses — never return optionals silently
- No business logic in services — only encode request, fire, decode response

---

## Components

- A component is reusable if it appears in 2+ places — otherwise inline it
- Every component takes only the data it needs as `let` properties
- No `@EnvironmentObject` inside a component — pass values explicitly
- Named pattern: `{Purpose}View.swift` (e.g., `ProgressRingView`, `WeekStreakView`)

---

## Models

- Use `struct`, never `class`, for data models
- Conform to: `Identifiable`, `Codable`, `Hashable`
- IDs are `String` (`UUID().uuidString`), never `Int`
- Timestamps are `Double` (milliseconds since epoch: `Date().timeIntervalSince1970 * 1000`)
- Enums for fixed sets: `PresetSource (.default / .user)`, `OnboardingStep`, etc.
- For Firebase: add `@DocumentID var id: String?` and conform to `Codable` — Firestore SDK handles the rest

---

## Theme

- All colors come from `Assets.xcassets` color sets with semantic names:
  `appBackground, appSurface, appText, appTextSecondary, appBorder, appTint, appCoral`
- Access via `Color.appBackground` etc. (defined in `Color+Theme.swift` extension)
- Never use hardcoded hex strings inline in views — add to `Color+Theme.swift` first
- All typography comes from `Font+Theme.swift` via a `TextVariant` enum
- Use `ThemedText(variant: .body, "Hello")` not raw `Text` with manual font modifiers

---

## Persistence Rules (Summary)

| Data type | Storage |
|---|---|
| Non-sensitive local state | `UserDefaults` |
| Sensitive data (never tokens — Firebase handles those) | `Keychain` |
| User data synced across devices | Firestore (Firebase mode) |
| Offline cache | `UserDefaults` or local JSON file |

---

## Telemetry

- Fire install/session tracking once on first launch using an `install_id` stored in `UserDefaults`
- Use a dedicated event method for key user actions — pass a properties dictionary
- All telemetry fires inside a `Task {}` — never block the UI thread

---

## Code Style

- No force unwraps (`!`). Use `guard let` or `if let`
- No `print()` statements in production code — remove before asking for review
- Use `@discardableResult` only when intentional
- Prefer `guard` early-exit over nested `if-let`
- Extensions go at the bottom of the file or in the `Theme/` files
- Keep `init()` methods clean — call a `load()` or `startListening()` method for side effects

---

## What NOT to Do

- Do not use `@StateObject` inside child views — only in the root app or `AppRouter`
- Do not use `NavigationLink` for programmatic navigation — use the router enum
- Do not put API or Firebase calls inside views — delegate to the store or a service
- Do not store UI state (sheet open, alert shown) in the global store — use `@State` locally
- Do not create a new file for a one-off helper — inline it or add to an existing file
- Do not store Firebase auth tokens in `UserDefaults` or `Keychain` manually — the SDK manages this
- Do not put `.addSnapshotListener` directly in a view — wrap it in an `AsyncStream` in `FirebaseService`

---

## Quick Reference Cheatsheet

### Starting a new screen
```
Before creating the screen, confirm the app tier (Small / Medium / Large).
Small: read state from {AppName}Store via @EnvironmentObject.
Medium: create a {Feature}ViewModel with @Observable and instantiate it with @State in the screen.
Large: inject dependencies via init.
Keep the screen under 300 lines — extract any section over 80 lines into Components/.
Follow the theme system (Color.app*, ThemedText).
```

### Adding a new model
```
Add a new model struct to Models/{Name}.swift.
Conform to Identifiable, Codable, Hashable.
Use String for id and Double for timestamps (ms).
Firebase apps: add @DocumentID var id: String? instead of a manual UUID id.
Add any fixed-value sets as enums inside the same file.
```

### Adding a new Firebase service call
```
Add a new async throws function to Services/FirebaseService.swift.
For one-time reads/writes: use async/await with Firestore SDK.
For real-time data: return an AsyncStream and use addSnapshotListener inside it.
Return typed Swift structs — never raw DocumentSnapshot.
No business logic in the service.
```

### Adding a new REST endpoint
```
Add a new static async function to Services/APIService.swift.
Use URLSession with async/await.
Include the standard headers: Content-Type, X-App-Version, X-Client, X-Platform.
Throw on error, return a typed Codable response.
No business logic — just the network call.
```

### Adding state (Tier 1)
```
Add @Published properties to {AppName}Store.
Add them to PersistedState if they must survive app restart.
Add a computed property if the value is derived from other state.
Call save() at the end of any mutating method.
```

### Adding state (Tier 2)
```
Add @Observable properties to the relevant {Feature}ViewModel.
Only promote to AppStore if the state is needed across 3+ screens or must survive navigation.
```
