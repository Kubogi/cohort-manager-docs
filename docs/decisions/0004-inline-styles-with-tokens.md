# ADR 0004 — Inline styles with design tokens; no CSS framework

**Status**: Accepted
**Date**: ~2024 (recorded retroactively 2026-05-16)

## Context

react-admin ships with Material-UI as its component library, which provides a theming system (`createTheme`, `styled(...)`, `sx={...}`). On top of MUI, the frontend ecosystem has Tailwind, Chakra, Emotion, vanilla-extract, styled-components — any of which would be reasonable for the styling layer.

The visual design here is fairly specific: a dark blue corporate-military aesthetic with Vietnamese labels and a fixed component vocabulary (`Button`, `Modal`, `SearchableSelect`, `Input`, …). Operators expect consistency across pages; deviations are noisy and confusing.

## Decision

Use **inline styles backed by a centralized design-token system**, not a CSS framework. The implementation:

- Tokens live in [`frontend/src/styles/tokens/`](../../frontend/src/styles/tokens/) — `colors`, `typography`, `spacing`, `borderRadius`, `shadows`. Each is a plain TS object/constant.
- Per-component style objects live in [`frontend/src/styles/components/`](../../frontend/src/styles/components/) — `button.styles.ts`, `modal.styles.ts`, etc. They compose tokens.
- Shared components ([`frontend/src/components/shared/`](../../frontend/src/components/shared/)) import the per-component style files and apply them as inline `style={...}` (or, where MUI is involved, as `sx={...}`).
- Page-level / resource-level styling pulls from tokens directly. No hex literals, no px literals.
- react-admin's `darkBlueTheme` extends MUI's `defaultTheme` to harmonize MUI internals with the tokens (typography sizes, drawer color).

The styling guide [`frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) (1,285 lines) is the long-form reference.

## Consequences

### Positive

- **No build-time CSS step.** No PostCSS config, no Tailwind purge, no CSS-in-JS runtime cost. Vite bundles only the TS objects, which tree-shake naturally.
- **Single source of truth.** Changing the primary blue means editing `colors.primary[700]` in one file; every component update follows.
- **Easy to audit at review time.** A diff showing `color: '#1E3A8A'` instead of `colors.primary[700]` is an obvious tell that someone bypassed the system.
- **AI-friendly.** The tokens have predictable names; coding agents pick up the pattern after seeing two examples.

### Negative

- **No pseudo-class support in inline styles** (no `:hover`, no `:focus`). The component objects work around this with explicit state (`useState` for `isHover`, key event handlers) or by using MUI's `sx={...}` (which does support pseudo-classes via emotion's compiler).
- **No media queries in inline styles.** The current design is desktop-only (the operators use a single school computer with a fixed layout), so this hasn't bitten us yet. If a mobile layout is ever needed, the inline-style approach won't scale.
- **Slightly slower runtime than static CSS** — every render evaluates the style object. Not measurable for the ~20 elements visible per page.
- **No linting against ad-hoc hex/px literals.** Review discipline is the only enforcement. The codebase is small enough that human eyes catch violations.

## Alternatives considered

- **MUI's `sx={...}` exclusively** — would have given pseudo-class and responsive support for free. Rejected because the design vocabulary diverged from MUI's defaults enough that mixing both was confusing.
- **Tailwind** — class-based, fast, well-known. Rejected because the operator-facing vocabulary is Vietnamese; Tailwind utility classes (English, terse) would add a *third* mental layer between operator vocabulary, code identifiers, and styling. The token files keep styling intent in the same language as identifiers.
- **CSS Modules** — would have separated concerns cleanly but added a build step and required filename hygiene. The inline approach is simpler and the token system supplies the "centralized" benefit a CSS module would.
- **Emotion / styled-components** — runtime CSS-in-JS. Rejected for build complexity and the runtime cost trade-off; the inline approach is "good enough" at the project's scale.

## See also

- [`/frontend/STYLING_GUIDE.md`](../../frontend/STYLING_GUIDE.md) — the long-form reference
- [`/frontend/src/styles/`](../../frontend/src/styles/) — the token and component-style sources
- [`/docs/architecture/frontend.md`](../architecture/frontend.md#8-design-tokens) — how the system fits into the broader frontend architecture
