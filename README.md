# My Claude Code Skills

A collection of custom Claude Code skills for accelerating development workflows.

---

## Skills

### `color-palette-preview`
Generates a polished, self-contained HTML preview from a set of CSS variables or hex color values. Given a palette, it produces a swatch grid alongside a realistic mini UI mockup — cards, buttons, text, badges — styled in a modern SaaS aesthetic. The output is a single copy-pasteable HTML file with no external dependencies, making it easy to evaluate a color scheme in context before committing to it in a real project.

### `fastapi-new-project`
Scaffolds a complete FastAPI project from scratch in one shot. It creates the full folder structure (`app/`, `services/`, `core/`, `schemas/`, `sql/`), writes all boilerplate files with correct content (`main.py`, `config.py`, `exceptions.py`, `dependencies.py`, `routes.py`), sets up a virtual environment, installs dependencies, generates `requirements.txt`, and configures `.env`, `.env.example`, and `.gitignore`. The result is a production-ready starting point that follows the layered architecture defined in the FastAPI coding rules.

### `fastapi.coding-rules` *(coding rules / context file)*
Defines the architectural rules and checklists that govern every FastAPI file generated in a project. It enforces a strict layered pattern — thin routers (5–15 lines, one service call), business logic isolated in services, typed Pydantic schemas, all secrets via a `Settings` class, and custom exceptions that flow from services up to routers. It also covers SQL conventions (raw SQL or Supabase client, no ORMs), dependency pinning rules, a project split/scale guide, and an optional Firebase Auth override section.

### `fast_api_security`
Guides the generation of secure FastAPI endpoints by applying defense-in-depth across every common attack surface. It covers dependency-based auth with `Depends()`, IDOR prevention via ownership checks in SQL `WHERE` clauses, Pydantic mass-assignment blocking with `extra = "forbid"`, parameterized query enforcement, JWT algorithm whitelisting, file upload validation via magic bytes, SSRF and path traversal protection, open redirect prevention, security headers middleware, CORS configuration, rate limiting, safe error responses, and Argon2id password hashing. Both Railway/Postgres (raw SQL) and Supabase client patterns are supported throughout.

### `HTML-to-Expo` (`html-to-expo-rn`)
Converts HTML/CSS/Tailwind UI into production-ready Expo/React Native components. It detects the input scope (full screen, section, or element) and maps every web primitive to the correct RN equivalent — replacing `div` with `View`, `img` with `Image`, `onClick` with `onPress`, and so on. All styles are moved into `StyleSheet.create({})`, CSS properties unsupported in RN are translated or stubbed, icon references are replaced with placeholder comments, and the output is always a complete, self-contained component file ready to drop into an Expo project.

### `rn_to_swift`
Maps a React Native (Expo) app to a SwiftUI iOS target with a complete migration guide. It includes model translations, API endpoints, state management strategy, screen/component inventory, navigation plan, theme tokens, and a step-by-step conversion order to reduce rework.

### `swift_best_practices`
Provides a strict SwiftUI architecture prompt guide covering folder structure, state management, navigation, services, theme usage, and code style. Designed to keep new features consistent, scalable, and maintainable.

### `UXPilot_skill`
Generates UXPilot-ready prompts for designing calm, modern, iOS-quality app flows and screens. It emphasizes hierarchy, primary/secondary actions, card-based grouping, and functional color roles, with clear output rules for single screens and multi-screen flows.

### `swift_conversion_skill`
Guides full React Native or HTML/CSS-to-SwiftUI conversions with a strict multi-phase process: source reading, migration mapping, project structure, assets, color tokens, CSS translation rules, navigation patterns, and component/state management mappings.

### `expo-router-navigation.rules`
Defines a strict Expo Router navigation contract for tabs, inner pages, and modals so routes are placed correctly, do not appear unintentionally in bottom navigation, and avoid legacy React Navigation patterns.

### `swift_dev_menu`
Defines how to build a reusable SwiftUI developer menu that is debug-only (`#if DEBUG`), sectioned for safe tooling actions, wired through environment objects (without business logic in the view), and includes proper state handling for dialogs, sheets, persistent toggles, and preview setup.
