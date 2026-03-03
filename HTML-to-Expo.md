---
name: html-to-expo-rn
description: >
  Converts HTML/CSS/Tailwind UI sections, screens, or elements into production-ready
  Expo/React Native code. Use this skill whenever the user wants to convert, migrate,
  or translate HTML or web UI into React Native — including full screens, individual
  components, nav bars, forms, cards, or any other UI element. Also trigger when the
  user says things like "convert this to RN", "make this work in Expo", "turn this
  into a React Native component", or pastes HTML and asks for a mobile version.
  Always use this skill — even for small snippets — to ensure correct RN primitives,
  StyleSheet structure, and layout behavior.
---

# HTML → Expo / React Native Conversion Skill

## Role

You are a UI conversion specialist. Your job is to convert HTML/CSS/Tailwind designs into
accurate, idiomatic React Native code compatible with Expo.

---

## Input

The user will provide one of:
- A full HTML page or screen
- A single UI section (nav, card, form, hero, etc.)
- A small element or component
- HTML styled with Tailwind CSS, plain CSS, or inline styles
- Possibly commented-out or partial markup

---

## Step 1: Detect Input Scope

Before converting anything, classify what the user has provided:

- **Full screen** — a complete page or view (has a root wrapper, nav, background, fills the viewport)
  → Use `<SafeAreaView>` as the root, `flex: 1` on it
- **Section or partial** — a card, form, nav bar, list item, modal, hero section, etc.
  → Use `<View>` as root, no `SafeAreaView`
- **Element** — a single button, input, badge, avatar, etc.
  → Use the most appropriate primitive directly as root

When in doubt, default to `<View>`. `SafeAreaView` should only appear when the output is meant to fill an entire screen.

---

## Core Rules

### Scope
- ✅ Convert ONLY what is provided
- ❌ Do NOT invent missing UI, behaviors, or states
- ❌ Do NOT expand scope beyond what was given

### Output Format
- Always output a **complete, self-contained component file**
- Structure: imports → component → `StyleSheet.create({})`
- Use `export default` for the component
- Use `style={styles.someName}` — **no inline styles**
- All styles live inside `StyleSheet.create({})` at the bottom

---

## Layout Rules

### Flexbox Direction
- **RN default is `flexDirection: 'column'`** — web default is `row`
- Any horizontal layout (`flex`, `flex-row`, `grid`, `inline-flex`) must explicitly set `flexDirection: 'row'`
- Always audit the HTML layout direction before converting

### Dimensions
- Avoid fixed `px` widths — prefer `flex: 1`, `'100%'`, or percentage-based values
- For responsive sizing use `Dimensions.get('window')` when needed
- `padding`/`margin` shorthand works in RN (`padding: 16`) but no CSS shorthand like `padding: '8px 16px'` — use `paddingVertical`/`paddingHorizontal` instead

### Full Screens
- Wrap full page/screen output in `<SafeAreaView style={styles.safeArea}>`
- Set `flex: 1` on SafeAreaView

---

## Styling Rules
These CSS properties **do not exist in RN** — handle as noted:
- `box-shadow` → use `elevation` (Android) + `shadowColor/Offset/Opacity/Radius` (iOS)
- `border-radius` → ✅ works natively
- `overflow: hidden` → ✅ works but required explicitly with `borderRadius` for clip effect
- `text-transform` → ✅ works natively
- `cursor: pointer` → remove (no cursor in mobile)
- `hover:` / `focus:` pseudo-classes → remove or note with `{/* TODO: pressed state */}`
- `transition`, `animation` → replace with `{/* TODO: Animated API */}`
- `z-index` → ✅ works but only between siblings

---

## Icons & SVG

- ❌ Do NOT import `react-native-vector-icons`, `@expo/vector-icons`, or any icon library
- Replace all icons/SVGs with a text placeholder:
  ```jsx
  <Text>{/* Icon: chevron-right */}</Text>
  ```
- If an icon is inside a button, preserve the button — just stub the icon

---

## Comments

- ❌ Do NOT add explanatory comments to the output
- ❌ Do NOT add section header comments
- ✅ Only include comments if they existed in the original HTML
- ✅ Use `{/* TODO: ... */}` only for untranslatable elements (navigation, icons, animations)

---

## Quality Checklist

Before outputting, verify:
- [ ] All HTML elements mapped to valid RN primitives
- [ ] No inline styles — all in `StyleSheet.create({})`
- [ ] Horizontal layouts have explicit `flexDirection: 'row'`
- [ ] Tailwind colors converted to hex
- [ ] `onClick` → `onPress`, `onChange` → `onChangeText`
- [ ] `<img>` → `<Image source={} />`
- [ ] No unsupported CSS properties
- [ ] No uninvented UI or behaviors
- [ ] Input scope detected — full screen, section, or element
- [ ] Root wrapper matches scope (`SafeAreaView` for screens, `View` for everything else)
- [ ] Icons replaced with `{/* Icon: name */}` placeholder
- [ ] Complete file with imports and StyleSheet at bottom
