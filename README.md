# My Claude Code Skills

A collection of custom Claude Code skills for accelerating development workflows.

---

## Skills

### `color-palette-preview`
Generates a polished, self-contained HTML preview from a set of CSS variables or hex color values. Given a palette, it produces a swatch grid alongside a realistic mini UI mockup — cards, buttons, text, badges — styled in a modern SaaS aesthetic. The output is a single copy-pasteable HTML file with no external dependencies, making it easy to evaluate a color scheme in context before committing to it in a real project.

### `fastapi-new-project`
Scaffolds a complete FastAPI project from scratch in one shot. It creates the full folder structure (`app/`, `services/`, `core/`, `schemas/`, `sql/`), writes all boilerplate files with correct content (`main.py`, `config.py`, `exceptions.py`, `dependencies.py`, `routes.py`), sets up a virtual environment, installs dependencies, generates `requirements.txt`, and configures `.env`, `.env.example`, and `.gitignore`. The result is a production-ready starting point that follows the layered architecture defined in the FastAPI coding rules.

### `fastapi-rules`
A single security-first ruleset for every FastAPI project. Combines architecture standards and security requirements into one file so safety is never an afterthought. Covers the full layered pattern (thin routers, service-layer logic, typed Pydantic schemas with `extra = "forbid"`, `Settings`-based config, custom exceptions), plus security built into each layer: ownership-enforced SQL `WHERE` clauses, parameterized queries, JWT algorithm whitelisting, SSRF and path traversal prevention, file upload magic-byte validation, security headers middleware, CORS configuration, rate limiting, and Argon2id password hashing. Includes a combined pre-commit checklist and a Firebase Auth override section.

### `expo-rn-rules`
A single ruleset for all React Native Expo development. Covers component structure, StyleSheet-only styling, layout rules, HTML/CSS/Tailwind-to-RN conversion (element mapping, event mapping, unsupported CSS handling), and Expo Router file-based navigation for tabs, inner pages, and modals. Navigation rules are strict: no manual navigator composition, tabs are opt-in, inner pages are owned by their tab folder, and modals live in a dedicated `(modal)` group. Includes a combined pre-commit checklist and pre-code navigation checklist.

### `swift_best_practices`
Provides a strict SwiftUI architecture prompt guide covering folder structure, state management, navigation, services, theme usage, and code style. Designed to keep new features consistent, scalable, and maintainable.

### `HTML_to_swift`
Guides full React Native or HTML/CSS-to-SwiftUI conversions with a strict multi-phase process: source reading, migration mapping, project structure, assets, color tokens, CSS translation rules, navigation patterns, and component/state management mappings.


### `swift_dev_menu`
Defines how to build a reusable SwiftUI developer menu that is debug-only (`#if DEBUG`), sectioned for safe tooling actions, wired through environment objects (without business logic in the view), and includes proper state handling for dialogs, sheets, persistent toggles, and preview setup.

### `plan-review`
Performs a senior-engineer pre-development plan review before any code is written. It reads the plan in full, cross-references every assumption against the actual codebase, and produces a structured findings table categorized by severity (CRITICAL, MAJOR, MINOR). Flags missing prerequisites, wrong assumptions, already-completed work, ambiguous instructions, missing guards, regression risks, and silent failure modes. Outputs a READY or NEEDS UPDATES status with a clear list of what the plan gets right.

### `create-plan`
Generates a detailed, codebase-grounded implementation plan before any code is written. It reads all relevant files, maps dependencies and sequencing, and produces a `plan.md` file with a clear goal, scope, prerequisites, step-by-step instructions (each naming exact file paths and changes), validation steps, and risk notes. Designed to pair with `plan-review` — the plan is created first, then reviewed before implementation begins.
