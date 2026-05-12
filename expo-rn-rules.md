---
name: expo-rn-rules
description: React Native Expo coding rules. Covers component structure, StyleSheet patterns, layout, HTML-to-RN conversion, and Expo Router file-based navigation for tabs, inner pages, and modals. Use when building Expo screens, converting HTML/CSS to React Native, or creating/modifying routes.
---

# React Native Expo Rules
> Directives for every component, screen, and route in this Expo project.
> File-based routing only. No legacy React Navigation patterns. No inline styles.

---

## 🧠 Core Mental Model

| Concern | Rule |
|---------|------|
| Components | Self-contained files: imports → component → `StyleSheet.create({})` |
| Styles | All styles in `StyleSheet.create({})` — no inline styles, ever |
| Layout | RN default is `flexDirection: 'column'` — horizontal layouts must set `'row'` explicitly |
| Navigation | Expo Router infers behavior from file structure — do not compose navigators manually |
| Screens | Full screens use `<SafeAreaView flex={1}>` — sections and elements use `<View>` |
| Icons | Stub all icons with `{/* Icon: name */}` — do not import icon libraries |

---

## 📁 Project Structure & Routing

```
app/
├── _layout.tsx              ← Root stack, modal presentation, security headers middleware
├── (app)/
│   ├── _layout.tsx          ← Tabs layout — only explicitly declared routes are tabs
│   ├── home/
│   │   ├── _layout.tsx      ← Nested Stack for home inner pages
│   │   └── index.tsx        ← Home tab root
│   └── account/
│       ├── _layout.tsx      ← Nested Stack for account inner pages
│       └── index.tsx        ← Account tab root
└── (modal)/
    └── create.tsx           ← Modals live here, NOT under (app)
```

**File naming:**
- Screens: `kebab-case.tsx`
- Components: `PascalCase.tsx`
- Layouts: `_layout.tsx`

---

## 🟦 Component Rules

### Output Format

Every component file must follow this structure:

```tsx
import { View, Text, StyleSheet } from 'react-native'

export default function MyComponent() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  )
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 18,
    fontWeight: '600',
  },
})
```

### Rules

- [ ] `export default` on every component
- [ ] All styles in `StyleSheet.create({})` at the bottom — no inline `style={{ }}` objects
- [ ] Use `style={styles.name}` references only
- [ ] No `react-native-vector-icons`, `@expo/vector-icons`, or any icon library — stub with `{/* Icon: name */}`
- [ ] No explanatory comments — only `{/* TODO: ... */}` for untranslatable elements

---

## 🟩 Layout Rules

### Flexbox

- RN default is `flexDirection: 'column'` — explicitly set `flexDirection: 'row'` for horizontal layouts
- Always audit the source layout direction before writing styles
- Use `flex: 1`, `'100%'`, or percentage values — avoid fixed `px` widths
- `padding`/`margin` shorthand works (`padding: 16`), but no CSS-style multi-value shorthand — use `paddingVertical` / `paddingHorizontal`

### Screen Roots

- **Full screen** → `<SafeAreaView style={{ flex: 1 }}>` as root
- **Section / partial** → `<View>` as root, no SafeAreaView
- **Single element** → most appropriate primitive directly as root
- Safe area handling belongs at the layout level, not per-screen with `useSafeAreaInsets`

### Unsupported CSS → RN Translation

| CSS | React Native |
|-----|-------------|
| `box-shadow` | `elevation` (Android) + `shadowColor/Offset/Opacity/Radius` (iOS) |
| `cursor: pointer` | Remove — no cursor on mobile |
| `hover:` / `focus:` | Remove or stub `{/* TODO: pressed state */}` |
| `transition`, `animation` | Stub `{/* TODO: Animated API */}` |
| `padding: '8px 16px'` | `paddingVertical: 8, paddingHorizontal: 16` |
| `border-radius` | ✅ Works natively |
| `overflow: hidden` | ✅ Works, required explicitly with `borderRadius` for clip |
| `text-transform` | ✅ Works natively |
| `z-index` | ✅ Works between siblings only |

---

## 🟨 HTML → Expo Conversion Rules

### Step 1: Detect Input Scope

Before converting, classify what was provided:

- **Full screen** — complete page/view filling the viewport → `<SafeAreaView style={{ flex: 1 }}>`
- **Section or partial** — card, form, nav bar, list item, modal, hero → `<View>`
- **Element** — single button, input, badge → most appropriate primitive directly

When in doubt, default to `<View>`.

### Element Mapping

| HTML | React Native |
|------|-------------|
| `div`, `section`, `article`, `header`, `footer` | `<View>` |
| `p`, `span`, `h1`–`h6`, `label` | `<Text>` |
| `img` | `<Image source={require('...')} />` |
| `input[type=text]` | `<TextInput>` |
| `button` | `<TouchableOpacity>` or `<Pressable>` |
| `a` | `<TouchableOpacity>` + Expo Router `<Link>` |
| `ul` / `li` | `<FlatList>` or `<View>` + mapped items |
| `form` | `<View>` |
| `select` | `<Picker>` or custom modal picker |

### Event Mapping

| HTML | React Native |
|------|-------------|
| `onClick` | `onPress` |
| `onChange` | `onChangeText` |
| `onSubmit` | `onPress` on submit button |
| `onFocus` / `onBlur` | ✅ Same in RN |

### Tailwind → RN Styles

- Convert Tailwind color classes to their hex equivalents
- Convert spacing classes to numeric values (`p-4` → `padding: 16`, `mt-2` → `marginTop: 8`)
- Convert `flex-row` → `flexDirection: 'row'`, `items-center` → `alignItems: 'center'`, `justify-between` → `justifyContent: 'space-between'`

### Scope Rules

- Convert ONLY what is provided — do not invent missing UI, behaviors, or states
- Do not expand scope beyond what was given

---

## 🟧 Navigation Rules (Expo Router)

This project uses **Expo Router with file-based routing**. Navigation behavior is inferred from file structure. Do NOT manually compose Stack or Tab navigators unless explicitly instructed.

### Core Principles (Non-Negotiable)

1. **File-based only** — do not manually compose Stack or Tab navigators
2. **Tabs are opt-in** — only routes explicitly declared in the Tabs layout are tabs
3. **Ownership determines placement** — inner pages belong to a tab, modals belong to the app

### Tabs

- Declared in `(app)/_layout.tsx`
- Tabs WITHOUT inner pages → file-based: `tabname.tsx`
- Tabs WITH inner pages → folder-based: `tabname/index.tsx` with a nested `_layout.tsx`
- Tab names reference the **folder**, not `index` — use `name="home"`, not `name="home/index"`
- Do NOT auto-register routes as tabs
- Do NOT allow wildcard tab registration

If a new route appears in the footer nav unintentionally, it is a bug.

### Inner Pages (Stack Screens)

Use for drill-down views, detail pages, and content navigated from within a tab.

- Must live under a folder owned by their parent tab
  - ✅ `app/(app)/home/detail.tsx`
  - ✅ `app/(app)/account/privacy.tsx`
  - ❌ Do NOT place directly under `(app)`
- Must NOT be added to the Tabs layout
- Must NOT wrap tabs in a custom Stack
- Register via a nested `<Stack />` inside the tab's `_layout.tsx`
- Bottom tab bar remains visible by default
- Do NOT use legacy React Navigation stacks

When placed correctly, inner pages will NOT appear in the tab bar and do NOT require `href: null`.

**Fallback (use sparingly):** If a route must exist directly under `(app)` and must not appear in the tab bar, hide it explicitly in `(app)/_layout.tsx` with `href: null`. Avoid this when possible.

### Modals

Use for create/compose flows, edit screens, and temporary overlays.

- Must live in a dedicated `(modal)` route group
- Must NOT live under `(app)` or any tab folder
- Presentation configured in the root stack (`app/_layout.tsx`)
- Must not appear in the footer navigation
- Default to half-height or sheet-style when appropriate
- Do not replace tab navigation

### Navigation Pre-Code Checklist

Before creating or modifying any route:

1. Is this a tab, an inner page, or a modal?
2. Does its file location match its ownership?
3. Will this route accidentally appear in the tab bar?
4. Am I relying on Expo Router inference, not manual navigation?
5. Will this structure scale if more tabs/pages are added?

If any answer is unclear, stop and fix the plan first.

### Enforcement

If an inner page or modal breaks the tab bar, appears as a new footer button, requires wrapping tabs in a Stack, requires moving tab roots, or is referenced using `/index` in a tab name → the implementation is incorrect and must be revised.

---

## ✅ Pre-Commit Checklist

Before any screen or component is considered done:

- [ ] All styles in `StyleSheet.create({})` — no inline styles
- [ ] Horizontal layouts have explicit `flexDirection: 'row'`
- [ ] Full screens use `<SafeAreaView flex={1}>`, sections use `<View>`
- [ ] Icons stubbed with `{/* Icon: name */}`
- [ ] No unsupported CSS properties
- [ ] No uninvented UI or behaviors beyond what was specified
- [ ] `onClick` → `onPress`, `onChange` → `onChangeText`, `<img>` → `<Image>`
- [ ] Tailwind colors converted to hex, spacing converted to numeric values
- [ ] New routes placed in the correct folder (tab, inner page, or modal)
- [ ] No new routes appearing unintentionally in the tab bar
- [ ] No manual Stack/Tab navigator composition
- [ ] No legacy React Navigation patterns
