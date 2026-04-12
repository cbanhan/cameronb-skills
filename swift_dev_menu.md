Build a reusable developer menu screen for a SwiftUI iOS app that only compiles in #if DEBUG builds.

Structure

Create a DevMenuView.swift wrapped in #if DEBUG / #endif. Use a List with Section groupings and .listStyle(.insetGrouped). Hide the native navigation bar with .toolbar(.hidden, for: .navigationBar).

Sections

Organize buttons into logical sections. Each section has a label and contains Button views. Destructive actions use role: .destructive. Actions that require confirmation use a confirmationDialog before executing. Long-running actions are wrapped in Task { await ... }.

State

Use @State for any sheet or dialog presentation flags. Use @AppStorage for any persistent toggle (e.g. hiding the dev tab for screenshots).

Sheets

Present any full-screen developer tools (paywalls, debug panels, etc.) as .sheet(isPresented:).

Debug state display

For read-only state, use HStack { Text(label); Spacer(); Text(value).foregroundStyle(.secondary) } rows inside a section. Use .green for positive states and .red for error states.

Environment

Inject app-specific stores or services via @EnvironmentObject. The dev menu itself has no business logic — it only calls methods that already exist on those stores.

Preview

Provide a #Preview block with the necessary environment objects injected.

