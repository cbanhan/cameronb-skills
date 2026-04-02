# Swift Conversion Skill

You are an expert iOS engineer converting an existing app (React Native or HTML/CSS mockups) into a production-ready SwiftUI app. Follow this skill precisely every time it is invoked.

---

## Invocation

Use this skill when the user says any of the following:
- "Convert this to SwiftUI"
- "Migrate this RN app to Swift"
- "Convert this HTML screen to SwiftUI"
- `/swift-convert`

---

## Phase 0 — Always do this first: Read before writing

Before writing a single line of Swift:

1. **Read the source files** — HTML, CSS, RN component, or mockup provided by the user
2. **Read any linked CSS** — gradient values, colors, transforms, font sizes, border radii all come from CSS. Never guess these values.
3. **Check the existing Swift project** for:
   - What color tokens already exist in `Color+Theme.swift`
   - What components already exist in `Components/`
   - What assets already exist in `Assets.xcassets`
4. **Never reuse a component you haven't read** — always read the current file state first

---

## Phase 1 — Migration Map (new projects only)

When starting a brand new conversion, create a migration map before any code. Save it to `.cursor/rn_to_swift.md`.

The map must cover:
- **Data models & types** — TypeScript interfaces → Swift structs
- **API / networking** — fetch/axios → URLSession or Alamofire
- **State management** — Redux/Zustand/Context → `@State`, `@ObservableObject`, `@EnvironmentObject`
- **Components** — inventory all RN components, list Swift equivalents, note dependencies
- **Screens** — list all screens with their navigation destinations
- **Navigation** — Expo Router / React Navigation → `NavigationStack` + `TabView`
- **RN packages** — list every package and its Swift equivalent or "drop" if iOS-only removes the need
- **Platform-specific code** — `Platform.OS`, `Platform.select()` → drop for iOS-only builds
- **Theme tokens** — colors, fonts, spacing from constants files

Convert in this order to minimize rework:
1. Data models & types
2. API / networking layer
3. State / business logic
4. Reusable components (leaf components first, composites after)
5. Screens / full views
6. Navigation structure

---

## Phase 2 — Project Structure

Always use this folder structure inside the Xcode target folder:

```
AppName/
├── Theme/
│   ├── Color+Theme.swift       — Color extension with named tokens
│   └── Font+Theme.swift        — Font extension + TextVariant enum
├── Components/                 — Reusable views, leaf → composite order
├── Screens/
│   ├── Onboarding/
│   └── Main/
├── Navigation/
│   ├── AppRouter.swift         — Root view, onboarding gate
│   └── MainTabView.swift       — TabView with tabs
└── Assets.xcassets/            — All colors and images
```

---

## Phase 3 — Assets (images)

**All images must go into `Assets.xcassets` as image sets. Never reference loose files.**

To add an image:

1. Create the folder: `Assets.xcassets/<name>.imageset/`
2. Copy the image file into that folder
3. Create `Contents.json` in that folder:

```json
{
  "images" : [
    {
      "filename" : "image-name.jpg",
      "idiom" : "universal"
    }
  ],
  "info" : {
    "author" : "xcode",
    "version" : 1
  }
}
```

4. Reference in SwiftUI as: `Image("image-name")`

**For @2x / @3x variants** (icons, illustrations — not photos):

```json
{
  "images" : [
    { "idiom": "universal", "scale": "1x" },
    { "filename": "icon@2x.png", "idiom": "universal", "scale": "2x" },
    { "filename": "icon@3x.png", "idiom": "universal", "scale": "3x" }
  ],
  "info" : { "author": "xcode", "version": 1 }
}
```

---

## Phase 4 — Color Tokens

**Every color used more than once must be a named token**, not a hardcoded value.

### Add a color to the asset catalog

Create `Assets.xcassets/<TokenName>.colorset/Contents.json`:

```json
{
  "colors" : [
    {
      "color" : {
        "color-space" : "srgb",
        "components" : { "alpha": "1.000", "red": "0xEC", "green": "0x48", "blue": "0x99" }
      },
      "idiom" : "universal"
    },
    {
      "appearances" : [{ "appearance": "luminosity", "value": "dark" }],
      "color" : {
        "color-space" : "srgb",
        "components" : { "alpha": "1.000", "red": "0xEC", "green": "0x48", "blue": "0x99" }
      },
      "idiom" : "universal"
    }
  ],
  "info" : { "author": "xcode", "version": 1 }
}
```

Omit the dark appearance block if the color is the same in both modes.

### Register in Color+Theme.swift

```swift
extension Color {
    static let appCTA        = Color("CTAColor")       // #ec4899
    static let appText       = Color("TextColor")      // #11181C / #ECEDEE
    static let appBackground = Color("BackgroundColor")// #fff / #151718
    static let appTint       = Color("AccentColor")    // #0a7ea4 / #fff
    static let appIcon       = Color("IconColor")      // #687076 / #9BA1A6
}
```

**Hex → component reference:**
`#ec4899` → `red: 0xEC, green: 0x48, blue: 0x99`
`rgba(0, 80, 255, 0.55)` → `red: 0.0, green: 0.314, blue: 1.0` with `.opacity(0.55)`

---

## Phase 5 — CSS → SwiftUI Translation Rules

### Layout

| CSS | SwiftUI |
|---|---|
| `display: flex; flex-direction: column` | `VStack` |
| `display: flex; flex-direction: row` | `HStack` |
| `display: grid; place-items: center` | `ZStack` with centered content, or `frame(maxWidth:.infinity, alignment:.center)` |
| `margin-top: auto` | `Spacer()` above content, or `ZStack(alignment: .bottom)` |
| `position: absolute; inset: 0` | `.ignoresSafeArea()` on a layer inside `ZStack` |
| `z-index: 2` | Order in `ZStack` (last = top) |
| `gap: 14px` | `spacing: 14` on `VStack`/`HStack` |
| `padding: 16px` | `.padding(16)` |
| `padding: 20px 28px` | `.padding(.vertical, 20).padding(.horizontal, 28)` |
| `flex: 1` | `.frame(maxWidth: .infinity)` or `Spacer()` |
| `overflow: hidden` | `.clipped()` or `.clipShape(...)` |
| `overflow: auto` / `overflow-y: scroll` | `ScrollView { }` |
| `display: none` / conditional render | `if condition { SomeView() }` |
| `align-items: flex-start` | `VStack(alignment: .leading)` |
| `align-items: flex-end` | `VStack(alignment: .trailing)` |
| `justify-content: space-between` | `HStack` with `Spacer()` between items |
| `width: 100%` | `.frame(maxWidth: .infinity)` |
| `height: 100%` | `.frame(maxHeight: .infinity)` |

### Typography

| CSS | SwiftUI |
|---|---|
| `font-size: 32px; font-weight: 700` | `.font(.system(size: 32, weight: .bold))` |
| `font-weight: 900` | `.fontWeight(.black)` |
| `font-weight: 800` | `.fontWeight(.heavy)` |
| `font-weight: 700` | `.fontWeight(.bold)` |
| `font-weight: 600` | `.fontWeight(.semibold)` |
| `font-weight: 500` | `.fontWeight(.medium)` |
| `font-weight: 400` | `.fontWeight(.regular)` |
| `font-weight: 300` | `.fontWeight(.light)` |
| `text-transform: uppercase` | `.textCase(.uppercase)` |
| `white-space: nowrap` | `.lineLimit(1)` |
| `min-height: 88px` | `.frame(minHeight: 88)` |
| `letter-spacing: -0.6px` | `.tracking(-0.6)` |
| `line-height: 21px` on `font-size: 15px` | `.lineSpacing(6)` — value = lineHeight - fontSize |
| `text-shadow: 0 2px 8px rgba(0,0,0,0.35)` | `.shadow(color: .black.opacity(0.35), radius: 4, x: 0, y: 2)` |
| `text-align: center` | `.multilineTextAlignment(.center)` |
| `color: #eaf2ff` | `.foregroundStyle(Color(red: 0.918, green: 0.949, blue: 1.0))` |

### Backgrounds & Shapes

| CSS | SwiftUI |
|---|---|
| `background: #ec4899` | `.background(Color.appCTA)` |
| `background: rgba(0,20,60,0.28)` | `.background(Color(red:0, green:20/255, blue:60/255).opacity(0.28))` |
| `border-radius: 20px` | `.clipShape(RoundedRectangle(cornerRadius: 20))` |
| `border-radius: 999px` (pill) | `.clipShape(Capsule())` |
| `box-shadow: 0 8px 16px rgba(0,0,0,0.08)` | `.shadow(color: .black.opacity(0.08), radius: 8, x: 0, y: 8)` |
| `border: 2px solid #e5e5e5` | `.overlay(RoundedRectangle(cornerRadius: r).stroke(Color(...), lineWidth: 2))` |
| `border: 2px solid color` (active state) | Conditional `.stroke(isSelected ? .appCTA : .gray, lineWidth: 2)` |
| `opacity: 0.91` | `.opacity(0.91)` |

### Gradients

| CSS | SwiftUI |
|---|---|
| `linear-gradient(to bottom, a, b)` | `LinearGradient(colors: [a, b], startPoint: .top, endPoint: .bottom)` |
| `linear-gradient(to bottom, a 0%, b 50%, c 100%)` | `LinearGradient(stops: [.init(color:a, location:0), .init(color:b, location:0.5), .init(color:c, location:1)], startPoint:.top, endPoint:.bottom)` |
| `radial-gradient(...)` | `RadialGradient(...)` |
| `linear-gradient(135deg, a, b)` | `LinearGradient(colors:[a,b], startPoint:.topLeading, endPoint:.bottomTrailing)` |
| `linear-gradient(90deg, a, b)` | `LinearGradient(colors:[a,b], startPoint:.leading, endPoint:.trailing)` |

### Transforms

| CSS | SwiftUI |
|---|---|
| `transform: rotate(5deg)` | `.rotationEffect(.degrees(5))` |
| `transform: scale(1.15)` | `.scaleEffect(1.15)` |
| `transform: rotate(5deg) scale(1.15)` | `.rotationEffect(.degrees(5)).scaleEffect(1.15)` |

### Images

| CSS | SwiftUI |
|---|---|
| `background: url(...) center/cover` | `Image("name").resizable().scaledToFill()` |
| `object-fit: contain` | `.scaledToFit()` |
| `object-fit: cover` | `.scaledToFill().clipped()` |
| `filter: drop-shadow(...)` | `.shadow(...)` |
| Full-bleed background image | `Image(...).resizable().scaledToFill().ignoresSafeArea()` inside a `ZStack` |

---

## Phase 6 — Navigation

### Onboarding flow (stack with completion gate)

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

### Tab bar (main shell)

```swift
struct MainTabView: View {
    var body: some View {
        TabView {
            EarnView()
                .tabItem { Label("Earn", systemImage: "dollarsign.circle") }
            AppsView()
                .tabItem { Label("Apps", systemImage: "square.grid.2x2") }
        }
        .tint(.appTint)
    }
}
```

### Ionicons → SF Symbols mapping

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
| `chevron-back` | `chevron.left` |
| `close-outline` | `xmark` |
| `checkmark-circle-outline` | `checkmark.circle` |
| `star-outline` | `star` |
| `heart-outline` | `heart` |
| `share-outline` | `square.and.arrow.up` |

---

## Phase 7 — State Management Mapping

| React Native | SwiftUI |
|---|---|
| `useState<T>` | `@State var x: T` |
| `useEffect(() => {}, [])` | `.onAppear { }` |
| `useEffect(() => {}, [dep])` | `.onChange(of: dep) { }` |
| `useContext` | `@Environment` or `@EnvironmentObject` |
| `useRef` | `@State` or a stored property |
| Redux store | `@ObservableObject` class + `@StateObject` / `@EnvironmentObject` |
| Zustand store | `@Observable` class (Swift 5.9+ / iOS 17+) or `@ObservableObject` for iOS 15/16 support |
| `useColorScheme()` | `@Environment(\.colorScheme) var colorScheme` |
| `AsyncStorage` | `@AppStorage` (simple values) or `UserDefaults` |
| `Platform.OS === 'ios'` | Drop — you are building iOS only |
| `Platform.select({ios, default})` | Drop — use the iOS value directly |

---

## Phase 8 — Component Patterns

### Themed text

```swift
struct ThemedText: View {
    let text: String
    var variant: TextVariant = .default
    var color: Color? = nil

    var body: some View {
        Text(text)
            .font(.appFont(for: variant))
            .foregroundStyle(color ?? (variant == .link ? .appTint : .appText))
    }
}
```

### Full-screen background with overlay content

```swift
ZStack(alignment: .bottom) {
    Image("bg-image")
        .resizable()
        .scaledToFill()
        .ignoresSafeArea()

    LinearGradient(...)
        .ignoresSafeArea()

    VStack { /* content */ }
        .padding()
}
.toolbar(.hidden, for: .navigationBar)
```

### Interactive / selected state (`.option.active` pattern)

```swift
struct OptionButton: View {
    let label: String
    @Binding var isSelected: Bool

    var body: some View {
        Text(label)
            .font(.system(size: 18, weight: .medium))
            .frame(maxWidth: .infinity)
            .frame(minHeight: 88)
            .background(isSelected ? Color.appCTA.opacity(0.08) : .white)
            .overlay(
                RoundedRectangle(cornerRadius: 16)
                    .stroke(isSelected ? Color.appCTA : Color(hex: "#e5e5e5"), lineWidth: 2)
            )
            .clipShape(RoundedRectangle(cornerRadius: 16))
            .onTapGesture { isSelected.toggle() }
    }
}
```

### onChange — version-aware syntax

```swift
// iOS 17+ (preferred)
.onChange(of: value) { oldValue, newValue in ... }

// iOS 14–16 (use when supporting older OS)
.onChange(of: value) { newValue in ... }
```

### Screen that takes a navigation callback (never navigates itself)

```swift
struct SomeView: View {
    let onContinue: () -> Void  // navigation owned by AppRouter, not the view

    var body: some View {
        // ...
        Button("Continue", action: onContinue)
    }
}
```

---

## Rules — Always follow these

1. **Read CSS before writing Swift** — never guess colors, gradients, or transforms
2. **Every color used more than once → named token** in `Color+Theme.swift` + colorset
3. **Every image → `Assets.xcassets` imageset** — no loose file references
4. **Screens do not own navigation** — they receive `onContinue`/`onComplete` closures; `AppRouter` drives the stack
5. **Drop all Platform.OS checks and web-specific code** — iOS-only build
6. **Add `#Preview` to every View** — enables Canvas without running the app
7. **Never use hardcoded hex strings in View files** — always go through the `Color` extension
8. **Use `.toolbar(.hidden, for: .navigationBar)` on all onboarding screens** — matches the RN `headerShown: false`. Never use the deprecated `.navigationBarHidden(true)` (triggers Xcode warnings on iOS 16+)
9. **Never modify `.pbxproj` directly** — instruct the user to drag new folders into Xcode navigator
10. **Match the CSS exactly** — including intentional transforms like `rotate(5deg) scale(1.15)`. Do not "fix" things that look unusual without checking the source CSS first.

---

## Xcode Project Notes

- New Swift files added outside Xcode **must be dragged** into the Xcode project navigator to be compiled
- Exception: files added inside `Assets.xcassets` are picked up automatically — no drag needed
- Use `@AppStorage("hasOnboarded")` to persist onboarding completion across launches
- The global accent color is set in `Assets.xcassets/AccentColor.colorset` and applies to all native controls automatically
- **Minimum deployment target matters** — `@Observable` requires iOS 17+, `NavigationStack` requires iOS 16+. If supporting iOS 15, use `NavigationView` and `@ObservableObject` instead
- **`.toolbar(.hidden, for: .navigationBar)`** requires iOS 16+. For iOS 15 fallback use `.navigationBarHidden(true)` only if needed
