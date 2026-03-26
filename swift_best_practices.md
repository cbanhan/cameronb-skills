# Swift iOS App Structure — AI Prompt Guide

Use this prompt at the start of any new AI conversation to get consistent architecture results.

---

## Master Prompt

Paste this when starting a new feature or screen:

```
I'm building a SwiftUI iOS app. Follow these architecture rules exactly when generating code.

## Folder Structure
Every new feature must fit into this layout:
- Screens/{FeatureName}/        → full screen views only
- Components/                   → reusable views used across 2+ screens
- Models/                       → Swift structs, Codable, Identifiable, Hashable
- Services/                     → API calls only, no business logic
- Store/                        → all app state, ObservableObject, @MainActor
- Navigation/                   → AppRouter, tab view, NavigationStack only
- Theme/                        → Color+Theme.swift and Font+Theme.swift only

Do NOT create files outside these folders. Do NOT add new folders without asking.

## State Management
- All state lives in a single class: FastingStore (or the store for this app)
- Mark it @MainActor and ObservableObject
- Use @Published for every piece of state that drives UI
- Persist state to UserDefaults using a nested PersistedState struct with Codable
- Never store state inside a View — pass it via @EnvironmentObject
- Use computed properties on the store for derived values (e.g., visiblePresets)

## Navigation
- Use NavigationStack with a typed enum for all route transitions
- Use @AppStorage("hasOnboarded") for first-launch routing, not state
- Tab navigation lives in MainTabView only
- Onboarding navigation lives in AppRouter only — use enum-driven step tracking

## Views & Screens
- Screens go in Screens/{Main}/ or Screens/{Onboarding}/
- Keep screens under ~300 lines. Extract sub-sections into Components/
- Pass the store via @EnvironmentObject, not as an init parameter
- Use Timer.publish(every: 1, on: .main, in: .common) for real-time updates
- Use .onReceive to bind the timer to state updates

## Components
- A component is reusable if it appears in 2+ places — otherwise inline it
- Every component takes only the data it needs as let properties
- No @EnvironmentObject inside a pure component — pass values explicitly
- Named pattern: {Purpose}View.swift (e.g., ProgressRingView, WeekStreakView)

## Models
- Use struct, never class, for data models
- Conform to: Identifiable, Codable, Hashable
- IDs are String (UUID().uuidString), never Int
- Timestamps are Double (milliseconds since epoch, e.g., Date().timeIntervalSince1970 * 1000)
- Enums for fixed sets: PresetSource (.default / .user), OnboardingStep, etc.

## Services / API
- All network calls go in APIService.swift as static async functions
- Use URLSession.shared with async/await — no Combine, no callbacks
- Attach these headers to every request:
    Content-Type: application/json
    X-App-Version: 1.0.0
    X-Client: {app-name}
    X-Platform: IOS
- Return typed Result<T, Error> or throw — never return optionals silently
- No business logic in services — only encode request, fire, decode response

## Theme
- All colors come from Assets.xcassets color sets with semantic names:
    appBackground, appSurface, appText, appTextSecondary, appBorder, appTint, appCoral
- Access via Color.appBackground etc. (defined in Color+Theme.swift extension)
- Never use hardcoded hex strings inline in views — add to Color+Theme.swift first
- All typography comes from Font+Theme.swift via a TextVariant enum
- Use ThemedText(variant: .body, "Hello") not raw Text with manual font modifiers

## Validation Rules (Store Logic)
- Cannot delete a preset while a fast is active
- Cannot switch presets while a fast is active (use lockedPresetId)
- Cannot delete the last remaining preset
- Minimum fast duration to record: 1 minute (60,000ms)
- Custom preset hours must be 1–24

## Telemetry
- Fire APIService.createInstall() once on first launch using install_id from UserDefaults
- Use APIService.createEvent() for key user actions — pass a properties dictionary
- All telemetry fires inside a Task {} — never block the UI thread

## Code Style
- No force unwraps (!). Use guard let or if let
- No print() statements in production code — remove before asking for review
- Use @discardableResult only when intentional
- Prefer guard early-exit over nested if-let
- Extensions go at the bottom of the file or in the Theme/ files
- Keep init() methods clean — call a load() method for side effects

## What NOT to Do
- Do not use @StateObject inside child views — only in the root app or AppRouter
- Do not use NavigationLink for programmatic navigation — use the router enum
- Do not put API calls inside views — always delegate to the store or a service
- Do not store UI state (sheet open, alert shown) in the global store — use @State locally
- Do not create a new file for a one-off helper — inline it or add to an existing file
```

---

## Quick Reference Cheatsheet

### Starting a new screen
```
Create a new screen called {Name}View in Screens/Main/.
It should read state from FastingStore via @EnvironmentObject.
Use MainTopHeaderView for the top bar.
Keep it under 300 lines — extract any section over 80 lines into Components/.
Follow the theme system (Color.app*, Font.app(*)).
```

### Adding a new model
```
Add a new model struct to Models/{Name}.swift.
Conform to Identifiable, Codable, Hashable.
Use String for id (UUID().uuidString) and Double for timestamps (ms).
Add any fixed-value sets as enums inside the same file.
```

### Adding a new API endpoint
```
Add a new static async function to Services/APIService.swift.
Use URLSession with async/await.
Include the standard headers: Content-Type, X-App-Version, X-Client, X-Platform.
Throw on error, return a typed Codable response.
Do not add any business logic — just the network call.
```

### Adding state
```
Add @Published properties to FastingStore.
Add them to the PersistedState struct if they must survive app restart.
Add a computed property if the value is derived from other state.
Call save() at the end of any mutating method.
```
