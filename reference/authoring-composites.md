---
assumes: design-systems/authoring
dsl: [unset]
---
# Design System: Composites, Types, Unset, Hide

```ds
file("Styles/fintech-warm.ds")
// Composites: a record of element property paths, any name. Paths:
// fontFamily fontSize fontWeight lineHeight letterSpacing opacity radius
// gap padding, and the lists fills strokes effects. A field ref follows
// its token's modes.
typography.h1: { fontSize: font.size.3xl, fontWeight: font.weight.bold }
shadow.md: [ drop(y: 2, blur: 4, color: color.shadow) ]   // a list is effects
card.elevated: { fills: [ color.surface ], effects: [ shadow.md ], radius: radius.md }
card.elevated { $default: { effects: [ shadow.md ] }, dark: { effects: [ ] } }
// A branch names only what changes. A sub-key is a ref inside the .ds:
// typography.h1.fontSize, shadow.md.effects.0.color
// On an element a composite binds as a bare token after the props, several
// in order: r(320,200) f[($color.surface)] rd($radius.md) $card.elevated
// After a hand edit of one field, read-back shows the fields still bound
// (36:$typography.h1.fontSize, item($card.elevated.fills.0)): write them
// back as read.

// $type sets a token's type explicitly (string, boolean, number, …), winning
// over the name-prefix guess. Use it for a value no prefix classifies, so it
// isn't mis-read as a color and dropped. Mode-aware like any token:
copy.cta { $type: string, $default: "Get started" }

unset { color.primary, *.dark }   // drop inherited entries; *.key hits all semantics

// The color picker lists every color token you declare, nested and numeric
// names included (primary.main, grey.100), under its first namespace.
// hide: tucks tokens away: menus and the picker's main list leave them out,
// the color picker's Library lists them under More colours; they still resolve.
// A bare name hides that token and its ramp steps (hide: red hides the whole
// red palette, hint to intense) but never a token with its own name under it
// (hide: color.primary keeps color.primary.main); `name.*` hides everything
// under the name.
hide: tertiary, lime, yellow
```
