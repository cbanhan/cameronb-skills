# Cursor Rule: Expo Router Inner Pages & Modals (Navigation Contract)

This project uses **Expo Router with file-based routing**.
Do NOT apply legacy React Navigation patterns.

This rule governs how inner pages and modals are created.
Follow it strictly to avoid navigation breakage.

---

## Core Principles (Non-Negotiable)

1. Expo Router is file-based
   - Navigation behavior is inferred from file structure
   - Do NOT manually compose Stack or Tab navigators unless explicitly instructed

2. Tabs are opt-in
   - Only routes explicitly declared in the Tabs layout are tabs
   - Inner pages must NEVER appear in the footer navigation

3. Ownership determines placement
   - Inner pages belong to a tab
   - Modals belong to the app

---

## Tabs (Primary Navigation)

Tabs live under `(app)` and are declared in `(app)/_layout.tsx`.

### Tab Structure Types

| Structure | Example | Owns Inner Pages? |
|-----------|---------|-------------------|
| File-based | `account.tsx` | No |
| Folder-based | `account/index.tsx` | Yes |

### Rules

- Tabs WITHOUT inner pages may use file-based structure (`tabname.tsx`)
- Tabs WITH inner pages MUST use folder-based structure (`tabname/index.tsx`)
- Tab names in the Tabs layout reference the **folder**, not `index`
  - Example: use `name="home"`, not `name="home/index"`
- Current tabs:
  - `home/index.tsx` (Home) - folder-based, has inner pages
  - `account/index.tsx` (Account) - folder-based, has inner pages
- Do NOT auto-register routes as tabs
- Do NOT allow wildcard tab registration

If a new route appears in the footer nav unintentionally, it is a bug.

---

## Inner Pages (Stack Screens)

Use inner pages for:
- Drill-down views
- Detail pages
- Content navigated from within a tab

### Rules for Inner Pages

Inner pages are stack screens owned by a tab.

### Required Placement (Preferred)

- Inner pages must live under a folder owned by their tab
  - Example: `app/(app)/home/inner.tsx`
  - Example: `app/(app)/account/privacy.tsx`
- Inner pages must NOT be placed directly under `(app)`
- Inner pages must NOT be added to the Tabs layout
- Inner pages use implicit stack navigation provided by Expo Router
- Bottom tab bar remains visible by default
- Inner pages should be registered via a nested Stack inside their tab folder
  - Example: `app/(app)/home/_layout.tsx` with a `<Stack />`

When placed correctly, inner pages will NOT appear in the tab bar and do NOT require `href: null`.

---

### Fallback: Hiding Routes from Tabs (Use Sparingly)

If a route exists directly under `(app)` and must not appear in the tab bar:

- It MUST be explicitly hidden in `(app)/_layout.tsx` using `href: null`
- This is a fallback mechanism, not the default architecture

This approach should be avoided when possible.


### Prohibited for Inner Pages

- Do NOT wrap tabs in a custom Stack
- Do NOT arbitrarily restructure tabs; use folder-based structure only when inner pages are needed
- Do NOT use legacy React Navigation stacks

---

## Modals (Overlay Screens)

Use modals for:
- Create / compose flows
- Edit screens
- Temporary overlays

### Rules for Modals

- Modals must live in a dedicated `(modal)` route group
- Modals must NOT live under `(app)` or any tab folder
- Modal presentation is configured in the root stack (`app/_layout.tsx`)
- Modals must not appear in the footer navigation
- Modals overlay the app and dismiss back to the previous screen

### Modal Presentation

- Default to half-height or sheet-style when appropriate
- Modals should not replace tab navigation

---

## Safe Area & Layout Expectations

- Safe area handling is applied at layout level, not per screen
- Inner pages and modals must respect safe areas automatically
- Do NOT apply manual padding hacks using `useSafeAreaInsets` for navigation containers

---

## Validation Checklist (Run Before Coding)

Before creating or modifying routes, confirm:

1. Is this screen a tab, an inner page, or a modal?
2. Does its file location match its ownership?
3. Will this route accidentally appear in the tab bar?
4. Am I relying on Expo Router inference, not manual navigation?
5. Will this structure scale if more tabs/pages are added?

If any answer is unclear, STOP and fix the plan first.

---

## Final Enforcement Rule

If an inner page or modal:
- breaks the tab bar
- appears as a new footer button
- requires wrapping tabs in a Stack
- requires moving tab roots
- is referenced using `/index` in a tab name or redirect

→ The implementation is incorrect and must be revised.

Follow this contract at all times.
