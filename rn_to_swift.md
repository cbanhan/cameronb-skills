# RN → SwiftUI Migration Map
**Source:** `fasting-mobile-app` (Expo / React Native)
**Target:** `IOS-Fasting-App` (SwiftUI, iOS 16+)
**Created:** 2026-03-18

---

## 1. Data Models & Types

| TypeScript | Swift | Notes |
|---|---|---|
| `PresetSource = 'default' \| 'user'` | `enum PresetSource: String, Codable { case default_, user }` | |
| `FastingPreset` | `struct FastingPreset: Identifiable, Codable` | Same fields |
| `FastHistoryMilestone` | Inline constant array `[(thresholdHours: Int, label: String)]` | Not stored, computed |
| `FastHistoryItem` | `struct FastHistoryItem: Identifiable, Codable` | Same fields |

**Swift structs:**

```swift
// Models/FastingPreset.swift
struct FastingPreset: Identifiable, Codable, Hashable {
    let id: String
    var label: String
    var description: String
    var color: String          // hex e.g. "#E07A5F"
    let source: PresetSource
    var fastingHours: Int
    var eatingHours: Int
}

// Models/FastHistoryItem.swift
struct FastHistoryItem: Identifiable, Codable {
    let id: String
    let startedAtMs: Double
    let endedAtMs: Double
    let durationMs: Double
    let presetId: String
    let presetLabel: String
    let fastingHours: Int
    let milestoneLabel: String
}
```

---

## 2. API / Networking

**Base URL:** `https://fasting-app-api-service-production.up.railway.app`

| RN (fetch) | Swift | Method |
|---|---|---|
| `POST /v1/createinstall` | `APIService.createInstall(...)` | URLSession async/await |
| `POST /v1/updateuser` | `APIService.updateUser(...)` | URLSession async/await |
| `POST /v1/createevent` | `APIService.createEvent(...)` | URLSession async/await |
| `GET /v1/getfacts` | `APIService.getFacts()` | URLSession async/await |

**Pattern:** All calls are fire-and-forget. Use `Task { try? await APIService.X() }`.

**Common Headers:**
- `X-App-Version: 1.0.0`
- `X-Client: fasting-app`
- `X-Platform: IOS`

**Install ID:** `UUID().uuidString` stored in `UserDefaults` (replaces `expo-secure-store`).

---

## 3. State Management

| React Native | SwiftUI |
|---|---|
| `useFastingStore` (Zustand) | `FastingStore: ObservableObject` + `@StateObject` injected at root |
| `AsyncStorage` persistence | `Codable` → `UserDefaults` (or JSON file in Documents) |
| `expo-secure-store` (install ID, flags) | `UserDefaults` (non-sensitive) |
| `useColorScheme()` | `@Environment(\.colorScheme)` |

**Store class:** `FastingStore.swift` — `@Published` properties matching Zustand state.

**Persistence strategy:**
- Encode full `FastingStore` snapshot to JSON → save to `UserDefaults` key `"fasting-store"`
- Decode on app launch (implement in `init()`)

**Migrations:** Version field in persisted JSON; handle in `init()` if schema changes.

---

## 4. Components Inventory

| RN Component | Swift Equivalent | Status |
|---|---|---|
| `OnboardingHeader` | `OnboardingHeaderView` | Build new |
| `ThemedText` | `ThemedText` (same pattern) | Build new |
| `IconSymbol` | Drop — use SF Symbols directly | N/A |
| `FloatingTabBar` (custom) | Custom `TabView` with `.tabViewStyle` | Build in `MainTabView` |
| SVG progress ring | `ProgressRingView` using `Canvas` / `Path` | Build new |
| Facts carousel | `FactsCarouselView` | Build new |
| Preset color picker | `PresetColorPickerView` | Build new |
| Week streak tracker | `WeekStreakView` | Build new |

---

## 5. Screens Inventory

| Screen | Route | Swift File | Priority |
|---|---|---|---|
| Intro | `/onboarding/intro` | `Screens/Onboarding/IntroView.swift` | 1 |
| Goal | `/onboarding/goal` | `Screens/Onboarding/GoalView.swift` | 2 |
| Fasting Time | `/onboarding/fasttime` | `Screens/Onboarding/FastingTimeView.swift` | 3 |
| Fast Start | `/onboarding/faststart` | `Screens/Onboarding/FastStartView.swift` | 4 |
| Confirmation | `/onboarding/confirmation` | `Screens/Onboarding/ConfirmationView.swift` | 5 |
| Home | `/app/home` | `Screens/Main/HomeView.swift` | 6 |
| History | `/app/history` | `Screens/Main/HistoryView.swift` | 7 |
| Presets | `/app/presets` | `Screens/Main/PresetsView.swift` | 8 |

---

## 6. Navigation

**RN:** Expo Router file-based routing + custom floating tab bar
**Swift:** `NavigationStack` for onboarding → `MainTabView` for main app

```
AppRouter.swift
├── if hasOnboarded → MainTabView
└── else → NavigationStack(path: $path)
    ├── IntroView        { path.append(.goal) }
    ├── GoalView         { path.append(.fastingTime) }
    ├── FastingTimeView  { path.append(.fastStart) }
    ├── FastStartView    { path.append(.confirmation) }
    └── ConfirmationView { hasOnboarded = true }

MainTabView.swift
├── Tab 1: HomeView     (icon: house)
├── Tab 2: HistoryView  (icon: clock.arrow.circlepath)
└── Tab 3: PresetsView  (icon: slider.horizontal.3)
```

**Onboarding persistence:** `@AppStorage("hasOnboarded") var hasOnboarded = false`

**Tab bar:** Start with system `TabView`. Revisit floating style after functionality is working.

---

## 7. RN Packages → Swift Equivalents

| RN Package | Swift Equivalent | Action |
|---|---|---|
| `expo-router` | `NavigationStack` + `TabView` | Replace |
| `zustand` | `ObservableObject` class | Replace |
| `@react-native-async-storage/async-storage` | `UserDefaults` + `Codable` | Replace |
| `expo-secure-store` | `UserDefaults` (flags/IDs only — not sensitive) | Replace |
| `expo-crypto` | `UUID()` | Drop |
| `expo-haptics` | `UIImpactFeedbackGenerator` | Replace |
| `expo-font` | SF Pro (system fonts — no loading needed) | Drop |
| `expo-constants` | `Bundle.main.infoDictionary` | Replace |
| `react-native-svg` | `Canvas` / `Path` in SwiftUI | Replace |
| `react-native-reanimated` | SwiftUI animations (`withAnimation`, `.animation`) | Replace |
| `@expo/vector-icons` | SF Symbols (`Image(systemName:)`) | Replace |
| `@react-navigation/bottom-tabs` | `TabView` | Replace |
| `expo-splash-screen` | Xcode Launch Screen | Drop |
| `expo-status-bar` | `.statusBarStyle` modifier | Drop |
| `expo-system-ui` | Drop | Drop |
| `react-native-web` | Drop (iOS only) | Drop |

---

## 8. Platform-Specific Code → Drop

| RN Code | Action |
|---|---|
| `Platform.OS === 'ios'` | Drop check — always true |
| `Platform.select({ ios: ..., default: ... })` | Use the iOS value directly |
| `X-Platform: "AND"` | Always send `"IOS"` |
| Web font fallbacks in `theme.ts` | Use system fonts only |

---

## 9. Theme Tokens

### Colors (must become named tokens in `Color+Theme.swift` + colorsets)

| Token Name | Light | Dark | Used For |
|---|---|---|---|
| `appBackground` | `#FFFFFF` | `#151718` | Screen backgrounds |
| `appSurface` | `#FAFAFA` | `#1C1C1E` | Cards, sheets |
| `appText` | `#11181C` | `#ECEDEE` | Body text |
| `appTextSecondary` | `#9CA3AF` | `#6B7280` | Secondary labels |
| `appCoral` | `#E07A5F` | `#E07A5F` | Primary brand, header |
| `appTint` | `#0A7EA4` | `#FFFFFF` | Links, tint |
| `appIcon` | `#687076` | `#9BA1A6` | Tab icons (inactive) |
| `appBorder` | `#E5E5E5` | `#38383A` | Card borders |

### Preset Colors (for swatches — no dark variant needed)

| Name | Hex |
|---|---|
| Coral | `#E07A5F` |
| Indigo | `#4F46E5` |
| Red | `#E11D48` |
| Slate | `#334155` |
| Blue | `#3B82F6` |
| Orange | `#F97316` |
| Teal | `#14B8A6` |
| Emerald | `#10B981` |
| Amber | `#F59E0B` |
| Rose | `#F43F5E` |

### Typography

| Usage | SwiftUI |
|---|---|
| Large heading (34px bold) | `.font(.system(size: 34, weight: .bold))` |
| Section heading (28px semibold) | `.font(.system(size: 28, weight: .semibold))` |
| Card title (18px semibold) | `.font(.system(size: 18, weight: .semibold))` |
| Body (16px regular) | `.font(.system(size: 16, weight: .regular))` |
| Label (14px medium) | `.font(.system(size: 14, weight: .medium))` |
| Micro label (11px medium) | `.font(.system(size: 11, weight: .medium))` |
| Timer display (48px bold, mono) | `.font(.system(size: 48, weight: .bold, design: .monospaced))` |
| SF Pro Rounded | `.font(.system(size: X, weight: .Y, design: .rounded))` |

---

## 10. Milestones (Hardcoded Constants)

```swift
let fastingMilestones: [(thresholdHours: Int, label: String)] = [
    (8,  "Carb Burn"),
    (10, "Glycogen Burn"),
    (12, "Fat Burn Start"),
    (14, "High Fat Burn"),
    (16, "Peak Fat Burn"),
    (18, "Ketone Production"),
    (22, "Deep Fat Burn"),
]
```

---

## 11. Conversion Order

Execute in this order to minimize rework:

- [ ] **Step 1** — `Models/FastingPreset.swift` + `Models/FastHistoryItem.swift`
- [ ] **Step 2** — `Services/APIService.swift`
- [ ] **Step 3** — `Store/FastingStore.swift` (ObservableObject, persistence)
- [ ] **Step 4** — `Theme/Color+Theme.swift` + all colorsets in Assets.xcassets
- [ ] **Step 5** — `Theme/Font+Theme.swift`
- [ ] **Step 6** — Reusable components: `ThemedText`, `OnboardingHeaderView`, `WeekStreakView`, `ProgressRingView`, `FactsCarouselView`, `PresetColorPickerView`
- [ ] **Step 7** — Onboarding screens (Intro → Goal → FastingTime → FastStart → Confirmation)
- [ ] **Step 8** — Main screens (Home, History, Presets)
- [ ] **Step 9** — Navigation: `AppRouter.swift` + `MainTabView.swift`
- [ ] **Step 10** — Wire everything in `IOS_Fasting_AppApp.swift`

---

## 12. Target Project Structure

```
IOS-Fasting-App/
├── Theme/
│   ├── Color+Theme.swift
│   └── Font+Theme.swift
├── Models/
│   ├── FastingPreset.swift
│   └── FastHistoryItem.swift
├── Services/
│   └── APIService.swift
├── Store/
│   └── FastingStore.swift
├── Components/
│   ├── ThemedText.swift
│   ├── OnboardingHeaderView.swift
│   ├── ProgressRingView.swift
│   ├── WeekStreakView.swift
│   ├── FactsCarouselView.swift
│   └── PresetColorPickerView.swift
├── Screens/
│   ├── Onboarding/
│   │   ├── IntroView.swift
│   │   ├── GoalView.swift
│   │   ├── FastingTimeView.swift
│   │   ├── FastStartView.swift
│   │   └── ConfirmationView.swift
│   └── Main/
│       ├── HomeView.swift
│       ├── HistoryView.swift
│       └── PresetsView.swift
├── Navigation/
│   ├── AppRouter.swift
│   └── MainTabView.swift
├── Assets.xcassets/
│   ├── AccentColor.colorset/
│   ├── AppBackground.colorset/
│   ├── AppSurface.colorset/
│   ├── AppText.colorset/
│   ├── AppTextSecondary.colorset/
│   ├── AppCoral.colorset/
│   ├── AppTint.colorset/
│   ├── AppBorder.colorset/
│   └── (preset color sets...)
└── IOS_Fasting_AppApp.swift
```

---

## 13. Notes & Gotchas

- **Progress ring:** RN uses `react-native-svg` with `strokeDasharray`. In SwiftUI use `Canvas` or a `Circle()` with `.trim(from:to:)` + `.stroke(style: StrokeStyle(lineWidth:, lineCap:))`.
- **Confirmation screen animation:** RN uses `Animated.loop` with scale/opacity. Use SwiftUI `withAnimation(.easeInOut.repeatForever())`.
- **Haptics:** `UIImpactFeedbackGenerator(style: .medium).impactOccurred()` or `UINotificationFeedbackGenerator`.
- **SecureStore → UserDefaults:** Install ID and flags are not sensitive; UserDefaults is sufficient. If needed later, use `Keychain` via `Security` framework.
- **Onboarding guard:** In `AppRouter`, use `@AppStorage("hasOnboarded")` — persists across launches automatically.
- **iOS 16+ required** for `NavigationStack` and `.toolbar(.hidden, for: .navigationBar)`.
