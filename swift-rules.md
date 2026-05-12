---
name: swift-rules
description: SwiftUI iOS coding rules. Covers project architecture (tiered Small/Medium/Large), state management, navigation, services, theme, models, code style, HTML/CSS/RN-to-SwiftUI conversion, and the DEBUG-only developer menu pattern. Use when starting a new Swift project, adding a screen or feature, converting HTML or React Native to SwiftUI, or scaffolding a dev menu.
---

# Swift iOS Rules
> Directives for every file generated in this SwiftUI project.
> Read the codebase before writing. Infer tier, backend, and patterns from what already exists.

---

## 🧠 Step 0 — Read Context First

Before writing any code:
- Check existing files, folder structure, and imports to infer the tier and backend
- If `FirebaseAuth` / `Firestore` is imported → Firebase mode
- If `URLSession` + `APIService.swift` exists → REST mode
- If no service files exist → Local-only
- Screen count and store structure reveal the tier

**Only ask the user when** starting a brand-new project with no existing code, or when context is genuinely contradictory. Never ask about something already established in the codebase.

---

## 📐 Architecture Tiers

### Tier 1 — Small App (1–5 screens, solo builder)

**State pattern:** Single global store + `@EnvironmentObject`

```
Screens/{FeatureName}/     → full screen views only
Components/                → reusable views used across 2+ screens
Models/                    → Swift structs, Codable, Identifiable, Hashable
Services/                  → API or Firebase calls only, no business logic
Store/                     → single AppStore, ObservableObject, @MainActor
Navigation/                → AppRouter, tab view, NavigationStack only
Theme/                     → Color+Theme.swift and Font+Theme.swift only
```

- All state in a single `{AppName}Store` class — `@MainActor`, `ObservableObject`, `@Published`
- Persist to `UserDefaults` via a nested `PersistedState` struct with `Codable`
- Never store UI state (sheet open, alert shown) in the global store — use `@State` locally
- Pass the store via `@EnvironmentObject`, not as an init parameter
- Local-only apps: delete `Services/` entirely

### Tier 2 — Medium App (6–15 screens)

**State pattern:** Feature-scoped `@Observable` view models per screen. Shared `AppStore` for global state only.

```
Screens/{FeatureName}/
    {Feature}View.swift         → the screen
    {Feature}ViewModel.swift    → @Observable view model for this screen only
Components/                     → reusable views, no view model inside
Models/                         → shared data structs
Services/                       → API or Firebase calls only
Store/                          → AppStore for global state only (auth, user, config)
Navigation/                     → AppRouter only
Theme/                          → Color+Theme.swift, Font+Theme.swift
```

- Use `@Observable` (Swift 5.9 / iOS 17+) for view models — not `ObservableObject`
- Each screen: `@State var viewModel = {Feature}ViewModel()` in the screen
- `@EnvironmentObject` only for `AppStore` — never for feature view models
- Promote state to `AppStore` only when needed across 3+ screens or must survive navigation

### Tier 3 — Large App (15+ screens or team)

**State pattern:** Full dependency injection. No `@EnvironmentObject`. Explicit init injection only.

- Each feature is a self-contained module: own models, view model, and service protocol
- All services are protocol-backed for testability (`protocol ServiceNameProtocol`)
- `AppStore` replaced by a `DependencyContainer` passed from the root
- Unit tests inject mock services without needing the full app

---

## 🗂️ Navigation

- Use `NavigationStack` with a typed enum for all route transitions — never `NavigationLink` for programmatic navigation
- Tab navigation lives in `MainTabView` only
- Onboarding navigation lives in `AppRouter` only — enum-driven step tracking
- Use `@AppStorage("hasOnboarded")` for first-launch routing, not state

**Onboarding flow:**
```swift
enum OnboardingRoute: Hashable {
    case intro, age, gender, confirmation
}

struct AppRouter: View {
    @AppStorage("hasOnboarded") private var hasOnboarded = false
    @State private var path: [OnboardingRoute] = []

    var body: some View {
        if hasOnboarded {
            MainTabView()
        } else {
            NavigationStack(path: $path) {
                SplashView { path.append(.intro) }
                    .navigationDestination(for: OnboardingRoute.self) { route in
                        switch route {
                        case .intro:        IntroView        { path.append(.age) }
                        case .age:          AgeView          { path.append(.gender) }
                        case .gender:       GenderView       { path.append(.confirmation) }
                        case .confirmation: ConfirmationView { hasOnboarded = true }
                        }
                    }
            }
        }
    }
}
```

**Tab bar:**
```swift
struct MainTabView: View {
    var body: some View {
        TabView {
            HomeView()
                .tabItem { Label("Home", systemImage: "house") }
            AccountView()
                .tabItem { Label("Account", systemImage: "person") }
        }
        .tint(.appTint)
    }
}
```

**Screens do not own navigation** — they receive `onContinue`/`onComplete` closures. `AppRouter` drives the stack.

```swift
struct SomeView: View {
    let onContinue: () -> Void

    var body: some View {
        Button("Continue", action: onContinue)
    }
}
```

---

## 🔧 Backend Modes

### Local-only
- Delete `Services/` entirely
- Persist everything to `UserDefaults` (non-sensitive) or `Keychain` (sensitive)

### Firebase (Auth + Firestore)

**`AuthManager.swift`:**
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
    func signOut() throws { try Auth.auth().signOut() }
}
```

**`FirebaseService.swift`:**
- All Firestore reads/writes as `async throws` functions
- Use `AsyncStream` for real-time listeners — never put `.addSnapshotListener` in a view or store
- Return typed Swift structs, never raw `DocumentSnapshot`
- No business logic — only encode, read/write, decode

```swift
func itemStream(userID: String) -> AsyncStream<[Item]> {
    AsyncStream { continuation in
        let listener = db.collection("users").document(userID)
            .collection("items")
            .addSnapshotListener { snapshot, _ in
                let items = snapshot?.documents.compactMap {
                    try? $0.data(as: Item.self)
                } ?? []
                continuation.yield(items)
            }
        continuation.onTermination = { _ in listener.remove() }
    }
}
```

**Persistence with Firebase:**
| Data | Storage |
|---|---|
| Non-sensitive local state | `UserDefaults` |
| Source of truth (online) | Firestore |
| Firebase auth tokens | Never store manually — SDK handles it |

Always design offline-first: local state is the UI source of truth, Firestore syncs in the background.

### REST API

- All calls in `Services/APIService.swift` as `static async throws` functions
- Use `URLSession.shared` with `async/await` — no Combine, no callbacks
- Required headers on every request: `Content-Type`, `X-App-Version`, `X-Client`, `X-Platform: IOS`
- Return typed `Codable` responses — never return optionals silently
- No business logic in services — only encode request, fire, decode response

---

## 🎨 Theme

- All colors in `Assets.xcassets` as named colorsets, accessed via `Color+Theme.swift`
- Semantic names: `appBackground`, `appSurface`, `appText`, `appTextSecondary`, `appBorder`, `appTint`, `appCoral`
- Never use hardcoded hex strings in view files — add to `Color+Theme.swift` first
- All typography via `Font+Theme.swift` and a `TextVariant` enum
- Use `ThemedText(variant: .body, "Hello")` — not raw `Text` with manual font modifiers

```swift
extension Color {
    static let appBackground = Color("BackgroundColor")
    static let appText       = Color("TextColor")
    static let appTint       = Color("AccentColor")
    static let appCTA        = Color("CTAColor")
}
```

---

## 📦 Models

- Use `struct`, never `class`, for data models
- Conform to: `Identifiable`, `Codable`, `Hashable`
- IDs are `String` (`UUID().uuidString`), never `Int`
- Timestamps are `Double` (ms since epoch: `Date().timeIntervalSince1970 * 1000`)
- Enums for fixed sets
- Firebase: use `@DocumentID var id: String?` and conform to `Codable`

---

## 🧹 Components

- A component is reusable if it appears in 2+ places — otherwise inline it
- Every component takes only the data it needs as `let` properties
- No `@EnvironmentObject` inside a component — pass values explicitly
- Naming: `{Purpose}View.swift` (e.g. `ProgressRingView`, `StatCardView`)
- Keep screens under ~300 lines — extract any section over 80 lines into `Components/`

---

## ✍️ Code Style

- No force unwraps (`!`) — use `guard let` or `if let`
- No `print()` in production code
- Prefer `guard` early-exit over nested `if let`
- Extensions go at the bottom of the file or in `Theme/` files
- Keep `init()` clean — use `load()` or `startListening()` for side effects
- Add `#Preview` to every View

**`onChange` — version-aware:**
```swift
// iOS 17+ (preferred)
.onChange(of: value) { oldValue, newValue in ... }

// iOS 14–16
.onChange(of: value) { newValue in ... }
```

---

## 🔄 HTML / CSS / React Native → SwiftUI Conversion

### Phase 0 — Read before writing

1. Read all source files (HTML, CSS, RN components)
2. Read linked CSS — colors, gradients, transforms, font sizes come from here. Never guess.
3. Check the existing Swift project for color tokens in `Color+Theme.swift`, components in `Components/`, assets in `Assets.xcassets`
4. Never reuse a component you haven't read

### Phase 1 — Migration Map (new projects only)

Create `.cursor/rn_to_swift.md` covering: data models, API/networking, state management, components, screens, navigation, RN packages → Swift equivalents, theme tokens.

Convert in this order: data models → networking → state/business logic → leaf components → composite components → screens → navigation.

### Phase 2 — Assets

All images go in `Assets.xcassets` as imagesets — never loose file references.

```json
{
  "images": [{ "filename": "image.jpg", "idiom": "universal" }],
  "info": { "author": "xcode", "version": 1 }
}
```

Reference in SwiftUI as `Image("image-name")`.

### Phase 3 — Color Tokens

Every color used more than once → named token in `Color+Theme.swift` + colorset in `Assets.xcassets`.

**Hex → component:** `#ec4899` → `red: 0xEC, green: 0x48, blue: 0x99`

### Phase 4 — CSS → SwiftUI Translation

**Layout:**

| CSS | SwiftUI |
|---|---|
| `flex-direction: column` | `VStack` |
| `flex-direction: row` | `HStack` |
| `grid; place-items: center` | `ZStack` with centered content |
| `margin-top: auto` | `Spacer()` |
| `position: absolute; inset: 0` | `.ignoresSafeArea()` in `ZStack` |
| `z-index: 2` | Order in `ZStack` (last = top) |
| `gap: 14px` | `spacing: 14` on `VStack`/`HStack` |
| `padding: 20px 28px` | `.padding(.vertical, 20).padding(.horizontal, 28)` |
| `flex: 1` | `.frame(maxWidth: .infinity)` or `Spacer()` |
| `overflow: hidden` | `.clipped()` or `.clipShape(...)` |
| `overflow-y: scroll` | `ScrollView { }` |
| `width: 100%` | `.frame(maxWidth: .infinity)` |
| `align-items: flex-start` | `VStack(alignment: .leading)` |
| `justify-content: space-between` | `HStack` with `Spacer()` between items |

**Typography:**

| CSS | SwiftUI |
|---|---|
| `font-size: 32px; font-weight: 700` | `.font(.system(size: 32, weight: .bold))` |
| `font-weight: 900/800/700/600/500/400/300` | `.black/.heavy/.bold/.semibold/.medium/.regular/.light` |
| `text-transform: uppercase` | `.textCase(.uppercase)` |
| `letter-spacing: -0.6px` | `.tracking(-0.6)` |
| `line-height: 21px` on `font-size: 15px` | `.lineSpacing(6)` (lineHeight − fontSize) |
| `text-shadow: 0 2px 8px rgba(0,0,0,0.35)` | `.shadow(color: .black.opacity(0.35), radius: 4, x: 0, y: 2)` |
| `text-align: center` | `.multilineTextAlignment(.center)` |
| `white-space: nowrap` | `.lineLimit(1)` |

**Backgrounds & Shapes:**

| CSS | SwiftUI |
|---|---|
| `background: #ec4899` | `.background(Color.appCTA)` |
| `border-radius: 20px` | `.clipShape(RoundedRectangle(cornerRadius: 20))` |
| `border-radius: 999px` | `.clipShape(Capsule())` |
| `box-shadow: 0 8px 16px rgba(0,0,0,0.08)` | `.shadow(color: .black.opacity(0.08), radius: 8, x: 0, y: 8)` |
| `border: 2px solid #e5e5e5` | `.overlay(RoundedRectangle(cornerRadius: r).stroke(Color(...), lineWidth: 2))` |
| `opacity: 0.91` | `.opacity(0.91)` |

**Gradients:**

| CSS | SwiftUI |
|---|---|
| `linear-gradient(to bottom, a, b)` | `LinearGradient(colors: [a, b], startPoint: .top, endPoint: .bottom)` |
| `linear-gradient(135deg, a, b)` | `LinearGradient(colors: [a,b], startPoint: .topLeading, endPoint: .bottomTrailing)` |
| `linear-gradient(90deg, a, b)` | `LinearGradient(colors: [a,b], startPoint: .leading, endPoint: .trailing)` |
| Multi-stop gradient | `LinearGradient(stops: [.init(color:a, location:0), .init(color:b, location:0.5), ...], ...)` |

**Transforms:**

| CSS | SwiftUI |
|---|---|
| `transform: rotate(5deg)` | `.rotationEffect(.degrees(5))` |
| `transform: scale(1.15)` | `.scaleEffect(1.15)` |
| `transform: rotate(5deg) scale(1.15)` | `.rotationEffect(.degrees(5)).scaleEffect(1.15)` |

**Images:**

| CSS | SwiftUI |
|---|---|
| `background: url(...) center/cover` | `Image("name").resizable().scaledToFill()` |
| `object-fit: contain` | `.scaledToFit()` |
| `object-fit: cover` | `.scaledToFill().clipped()` |
| Full-bleed background | `Image(...).resizable().scaledToFill().ignoresSafeArea()` inside `ZStack` |

### State Mapping (RN → SwiftUI)

| React Native | SwiftUI |
|---|---|
| `useState<T>` | `@State var x: T` |
| `useEffect(() => {}, [])` | `.onAppear { }` |
| `useEffect(() => {}, [dep])` | `.onChange(of: dep) { }` |
| `useContext` | `@Environment` or `@EnvironmentObject` |
| Redux store | `@ObservableObject` + `@StateObject` / `@EnvironmentObject` |
| Zustand store | `@Observable` (iOS 17+) or `@ObservableObject` |
| `AsyncStorage` | `@AppStorage` or `UserDefaults` |
| `Platform.OS === 'ios'` | Drop — iOS only |

### Ionicons → SF Symbols

| Ionicon | SF Symbol |
|---|---|
| `cash-outline` | `dollarsign.circle` |
| `apps-outline` | `square.grid.2x2` |
| `clipboard-outline` | `doc.text` |
| `gift-outline` | `gift` |
| `person-outline` | `person` |
| `home-outline` | `house` |
| `settings-outline` | `gearshape` |
| `notifications-outline` | `bell` |
| `search-outline` | `magnifyingglass` |
| `chevron-forward` | `chevron.right` |
| `close-outline` | `xmark` |
| `checkmark-circle-outline` | `checkmark.circle` |
| `star-outline` | `star` |
| `heart-outline` | `heart` |
| `share-outline` | `square.and.arrow.up` |

### Conversion Rules (Always)

1. Read CSS before writing Swift — never guess colors, gradients, or transforms
2. Every color used more than once → named token in `Color+Theme.swift` + colorset
3. Every image → `Assets.xcassets` imageset — no loose file references
4. Drop all `Platform.OS` checks and web-specific code
5. Never use hardcoded hex strings in view files
6. Use `.toolbar(.hidden, for: .navigationBar)` on all onboarding screens — never deprecated `.navigationBarHidden(true)`
7. Never modify `.pbxproj` directly — instruct the user to drag new folders into Xcode navigator
8. Match the CSS exactly — do not "fix" intentional transforms without checking source first

---

## 🛠️ Developer Menu (DEBUG only)

Scaffold a `DevMenuView.swift` that only compiles in `#if DEBUG` builds.

### Structure

```swift
#if DEBUG
import SwiftUI

struct DevMenuView: View {
    @EnvironmentObject var store: AppStore  // inject whatever stores this app uses
    @State private var showSomeTool = false
    @State private var showConfirmReset = false

    var body: some View {
        List {
            Section("Data") {
                Button("Seed Test Data") {
                    Task { await store.seedTestData() }
                }
                Button("Clear All Data", role: .destructive) {
                    showConfirmReset = true
                }
            }

            Section("Debug State") {
                HStack {
                    Text("User ID")
                    Spacer()
                    Text(store.currentUser?.id ?? "none")
                        .foregroundStyle(.secondary)
                }
                HStack {
                    Text("Auth State")
                    Spacer()
                    Text(store.isAuthenticated ? "Signed In" : "Signed Out")
                        .foregroundStyle(store.isAuthenticated ? .green : .red)
                }
            }

            Section("Tools") {
                Button("Open Paywall") { showSomeTool = true }
            }
        }
        .listStyle(.insetGrouped)
        .toolbar(.hidden, for: .navigationBar)
        .sheet(isPresented: $showSomeTool) {
            SomeDebugToolView()
        }
        .confirmationDialog("Reset all data?", isPresented: $showConfirmReset, titleVisibility: .visible) {
            Button("Reset", role: .destructive) {
                Task { await store.clearAllData() }
            }
        }
    }
}

#Preview {
    DevMenuView()
        .environmentObject(AppStore())
}
#endif
```

### Dev Menu Rules

- [ ] Entire file wrapped in `#if DEBUG` / `#endif` — never ships to production
- [ ] Use `List` with `Section` groupings and `.listStyle(.insetGrouped)`
- [ ] Destructive actions use `role: .destructive`
- [ ] Actions requiring confirmation use `confirmationDialog` before executing
- [ ] Long-running actions wrapped in `Task { await ... }`
- [ ] `@State` for sheet and dialog presentation flags
- [ ] `@AppStorage` for persistent toggles (e.g. hiding dev tab for screenshots)
- [ ] Inject stores/services via `@EnvironmentObject` — no business logic in the menu itself
- [ ] Read-only state displayed as `HStack { Text(label); Spacer(); Text(value).foregroundStyle(.secondary) }`
- [ ] `.green` / `.red` for positive / error states
- [ ] `#Preview` block with environment objects injected

---

## ✅ Pre-Commit Checklist

Before any feature is considered done:

- [ ] Tier confirmed — store pattern matches (Tier 1: single store, Tier 2: @Observable VMs, Tier 3: DI)
- [ ] No force unwraps anywhere
- [ ] No `print()` statements
- [ ] No hardcoded colors or hex strings in view files
- [ ] No `@EnvironmentObject` inside components — values passed explicitly
- [ ] No `NavigationLink` for programmatic navigation
- [ ] No `.addSnapshotListener` directly in a view — wrapped in `AsyncStream`
- [ ] No `Platform.OS` checks
- [ ] Screens do not own navigation — they call closures
- [ ] Every image in `Assets.xcassets` imageset
- [ ] Every new color token in `Color+Theme.swift` + colorset
- [ ] `#Preview` added to every new View
- [ ] Dev menu wrapped in `#if DEBUG` if present
- [ ] New Swift files dragged into Xcode navigator (not just added to filesystem)

---

## Xcode Notes

- `@Observable` requires iOS 17+, `NavigationStack` requires iOS 16+
- For iOS 15 support: use `NavigationView` and `@ObservableObject` instead
- `.toolbar(.hidden, for: .navigationBar)` requires iOS 16+ — use `.navigationBarHidden(true)` only for iOS 15 fallback
- Global accent color: set in `Assets.xcassets/AccentColor.colorset` — applies to all native controls automatically
- Files added inside `Assets.xcassets` are picked up automatically — no drag needed
- All other new Swift files must be dragged into the Xcode project navigator to be compiled
