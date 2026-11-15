---
assumes: design-systems/authoring
dsl: [modes]
---
# Design System: Per-Mode Branches and Custom Axes

```ds
file("Styles/fintech-warm.ds")
// Per-mode branch on an alias; $default is the fallback. Combo key
// (`theme.dark, density.compact:`) fires only when ALL modes active:
color.surface { $default: neutral.hint, dark: neutral.intense }
// An alias in a branch keeps the target's OTHER axes: `$default: color.ink`
// still flips with theme (the explicit branches win under every theme).
// A hex may carry alpha, #RRGGBBAA or #RGBA: a color token that defines
// alpha anywhere owns the alpha of every paint bound to it (#FFFFFF00 is
// "transparent in this mode"); an opaque token leaves a paint its own o().
// Braces after a token hold its mode values; a sub-token is a dotted name:
primary.main: #1976D2

// CUSTOM axes: `modes {}` declares any axis you like, not just the three
// built-ins (theme, density, accessibility). A branch key may be the bare
// value (`bright:`) or axis-qualified (`scheme.bright:`); both resolve the
// same, and qualified is the canonical form. Qualify a custom value that
// shares its spelling with a built-in or another custom axis: a bare `dark`
// is always theme's `dark`, so a custom `dark` is `<axis>.dark`:
modes { scheme: [normal, bright] }
color.accent { $default: brand.mid, scheme.bright: brand.strong }
// A custom-axis alias can point at a token themed on ANOTHER axis; it stays
// mode-aware across both, resolving for every combination.
// Selecting an axis value per frame/element is `ds(, <axis>(value))`
// (`ds(, scheme(bright))`), the same for custom axes as for theme; see
// design-systems/apply. A switch (canvas or element) re-resolves every bound
// token under it: colors, element opacity (`o($platform.mac)` with a
// per-mode number token flips a glyph), font family, sizes.
```
