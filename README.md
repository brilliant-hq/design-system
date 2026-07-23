# Design System DSL: Language Specification

The design-system DSL is Brilliant's token language: a concise text representation of a design
system that generates a full, mode-aware token catalog from a handful of seed declarations. It is
the companion language to [Blueprint](https://github.com/brilliant-hq/blueprint): every color, font,
and scale value a Blueprint design references (`$token`) resolves through a design system authored in
this language. This document is the authoritative language reference. The detailed authoring guides
live alongside it under [`reference/`](reference/).

A design system authored here is stored as a `.ds` source file (the on-disk name inside a project is
`Styles/<name>.styles`) and compiles to a resolved `.gen.yaml` catalog. The language is settled: the
primitive and semantic model described below is the current design, not a draft.

## 1. Design goals

- **Generative, not enumerated.** An author declares seeds (a brand color, a spacing base, a type
  ramp) and a small role vocabulary. The generator expands each seed into a full stop scale, so a
  real brand is usually a few lines rather than hundreds of hand-listed values.
- **Mode-aware by construction.** Light and dark, comfortable and compact, standard and high-contrast
  are axes, not separate files. A single semantic declaration re-resolves correctly across every
  active mode combination, and the mode behavior of a generator is built in rather than re-authored.
- **Two clean layers.** Primitives are mode-independent constants (Tailwind-style fixed stops).
  Semantics are mode-keyed roles built on top of primitives. External consumers (Style Dictionary,
  Tokens Studio, custom Tailwind generators) read the resolved catalog as exactly that contract.
- **Cascading and forgiving.** A brand file inherits the project default and declares only what
  differs. The parser recovers from errors and emits teaching diagnostics rather than failing hard.
- **Legible and durable.** One declaration per line, comments preserved into the resolved artifact so
  design intent carries to future sessions and to the AI that designs against the system.

## 2. Lexical structure

The lexer is a single forward pass that never throws: bad input yields an `invalid` token plus a
recovered diagnostic, and parsing continues. The token shapes are:

| Shape | Meaning |
|---|---|
| `name : value` | assignment of a token path to a value |
| `name { ... }` | a block: semantic role (per-mode branches), `modes`, `active`, `unset`, or a trailing stop-override block |
| `( )` | generator and constructor calls, `name(args)`: `color(#0080FF)`, `number([...])`, `drop(y: 2, blur: 4)` |
| `[ ]` | lists: a `number([...])` stop list, a `shadow.*` layer list, a `modes` axis value list |
| `.` | path separator: `font.size.3xl`, `color.surface.container` |
| `,` | argument, list, and combo-key separator |
| `*` | wildcard, only inside `unset { *.dark }` |

Literals: hex colors (`#RGB`, `#RRGGBB`, `#RRGGBBAA`), integers and decimals (a trailing `%` divides
by 100, so `63.7%` lexes to `0.637`), an optional unary `+` on numbers (`shift(+1)`), quoted strings
(`"Noto Serif"`, no escapes), and barewords (`Manrope`, `bold`, `none`). Identifiers may start with a
letter, `_`, or `$` (so `$default` is one token) and may contain digit-leading words like `2xl`.
`//` line comments and `/* */` block comments are trivia; line comments are preserved into the
resolved catalog as provenance.

## 3. Statements

A `.ds` file is a flat sequence of statements. Order matters only for shadowing: a later explicit
assignment overrides an earlier generator emission for the same key.

- **`modes { axis: [v1, v2, ...] }`** declares the mode axes. The first value on each axis is its
  default. The built-in system declares `theme`, `density`, and `accessibility`.
- **Primitive assignment** (`name : value`): a hex, number, string, bareword alias, or a
  primitive/semantic generator call (see §5). A trailing `{ stop: value }` block overrides individual
  generated stops (`brand: color(#1E40AF) { 300: #4B68C8 }`); the preferred modern form is a separate
  `brand.300: #4B68C8` assignment.
- **Semantic block** (`name { $default: value, mode.key: value, comboKey: value }`): a mode-aware
  role. `$default` is the fallback; a single mode value (`dark`) or a fully qualified key
  (`theme.dark`) branches on one axis; a comma-listed combo (`theme.dark, density.compact:`) fires
  only when all listed modes are simultaneously active.
- **`active { brand: <name>, modes: { axis: value } }`** records the folder-level active brand and
  starting mode set. It is metadata for visibility; runtime resolution is queried per element.
- **`hide: a, b.*`** and **`pin: a, b`** are in-app color-picker display hints (which palettes tuck
  away, which pin to the top). They do not change what designs or the AI can reference.
- **`unset { path, *.key }`** drops entries contributed by a parent in the cascade. `*.key` drops that
  key from every semantic (`*.dark` removes the dark branch everywhere).
- **`root: true`** stops the parent-folder cascade; **`inherits: none`** makes a brand standalone
  (both rare; `inherits: default` is the norm and need not be written).

## 4. Primitives versus semantics

This split is the heart of the language.

A **primitive** is a mode-independent scale. `color(...)` generates an 11-stop OKLCH ramp
(`.50 .100 .200 ... .950`); `number(...)` generates a numeric scale (`.1 .2 ... .N`). A primitive
stop is a single frozen value that never flips by mode. Bare primitive stops (`$primary.500`,
`$spacing.4`) exist but are the low-level layer.

A **semantic** wraps a primitive in a named role vocabulary and carries mode behavior. The three
semantic generators and their role vocabularies are fixed:

| Generator | Roles | Applies to |
|---|---|---|
| `boldness(...)` | `hint faint subtle soft mid firm bold strong intense` (9, center `mid`) | color tones, `font.weight`, `stroke.width`, `visibility` |
| `tshirt(...)` | `xs sm md lg xl 2xl ... 20xl` plus named boundary stops | `spacing`, `radius`, `font.size` |
| `looseness(...)` | `none tight snug normal relaxed loose` (6) | `font.lineHeight`, `font.letterSpacing` |

Roles map positionally onto the underlying primitive stops. `boldness(color(...))` maps its 9 roles
onto specific OKLCH stops (`hint` to `.50`, `mid` to `.500`, `intense` to `.900`); the unused
`.400`/`.950` stops stay available as primitives. A bare color seed reference (`$primary`) resolves
to `.mid`. Number-backed scales emit no bare form: `$spacing` alone is an error whose diagnostic
suggests a stop like `$spacing.md`.

Interfaces are built from semantic roles, never raw palette stops: roles re-tint with the brand and
flip correctly between light and dark, so a surface stays a surface and text stays legible on its own.

## 5. Generators

Generators are function-call values on the right of an assignment.

**`color(seed)`** registers a color seed. The seed is a hex literal or an `oklch(L%, C, H)` call
(converted to sRGB hex at generation time). The generator expands it into the 11-stop ramp (see §7).

**`number(...)`** is polymorphic:
- `number([4, 8, 16, 24, 32])`: an explicit list, materialized as stops `.1 .. .N` by position.
- `number(seed)`: a bare seed; the runtime catalog ramp expands it (spacing 1..32, etc.).
- `number(seed, count, ramp)`: a generated ramp of `count` stops. `ramp` is `linear()`,
  `linear(step: S)`, `geometric()`, or `geometric(ratio: R)`. Default is linear stepped by the seed.
- `number(range(min, max), count, ramp)`: a ramp spanning `[min, max]` inclusively.

**Semantic wrappers** take exactly one inner generator: `boldness(color(...))` or
`boldness(number([...]))`; `tshirt(number(...))`; `looseness(number(...))`. Each also accepts optional
`min: { name: value }` and `max: { name: value }` records that add named boundary stops below or above
the role band (for example `spacing: tshirt(number([...]), min: { none: 0 })` gives `$spacing.none`,
and `stroke.width: boldness(number([...]), max: { thick: 8, thicker: 16, thickest: 48 })`). A
`transforms:` argument tunes mode behavior (see §6).

Standalone `oklch(L%, C, H)` (no `color()` wrapper) is a single color primitive. `rgba(...)`/`rgb(...)`
and `drop(...)`/`inset(...)` are constructors used inside composites (see §8).

## 6. Mode transforms

A semantic generator bakes in default mode behavior by transforming the role's stop index per active
mode. The default programs are:

- `boldness(color(...))`: `theme.dark: mirror`, `accessibility.high-contrast: outward(1)`
- `boldness(number(...))`: `accessibility.high-contrast: shift(+1)`
- `tshirt(...)`: `density.compact: shift(-1)`, `accessibility.large-text: shift(+1)`
- `looseness(...)`: `accessibility.large-text: shift(+1)`

The three operators, each acting on the role's stop index (out-of-range clamps):

- **`shift(N)`**: offset by N; sums across simultaneously active modes.
- **`mirror`**: reflect around the middle stop (low tones become high tones, `mid` stays put). This is
  how a single `boldness(color(...))` declaration produces a correct dark palette: the low stops that
  read as pale surfaces in light mode resolve to deep surfaces in dark mode.
- **`outward(N)`**: push N stops away from the middle toward the nearer terminal (raises contrast).

Authors override with a `transforms: { ... }` record (keys axis-qualified, for example `theme.dark`),
which replaces the default entirely, so relist any default entries you want to keep. `transforms: none`
opts a scale out of all mode behavior (brand-stable knobs like `radius`). Per-mode value branches can
also be authored directly as semantic-block keys or `name.mode: value` seed overrides when an index
transform is not the right tool.

## 7. Color ramp generation (OKLCH)

`color(seed)` expands to 11 stops using OKLCH so hue stays perceptually constant across the ramp:

1. The seed is the `.500` stop, pinned exactly to its sRGB value (no round-trip drift).
2. Target lightness per stop follows a fixed ladder (`.50` near-white at L 0.97 down to `.950` at
   L 0.23), rescaled so the seed's own lightness anchors `.500` and the ladder keeps its relative
   spacing. Achromatic seeds (chroma below 0.05) use a lifted light-half ladder, matching how Tailwind
   hand-tunes neutrals lighter than chromatic palettes.
3. Chroma is softened by a parabolic curve (peaks near mid-lightness, falls toward the extremes) plus
   an asymmetric extreme-stop rolloff (bright tints desaturate faster than dark shades), then clamped
   to the largest in-gamut chroma at each `(L, H)` so warm high-chroma seeds do not silently clip.

The step numbers, lightness targets, and dark-mode mirror map are frozen constants shared by the
runtime generator and the Blueprint `$var` preprocessor, so both sides expand a seed identically.

## 8. Composites

Composites bundle several atoms into one named decision. They are a landed part of the language, not a
pending extension.

- **Typography** is a record: `typography.h1: { fontSize: font.size.3xl, fontWeight: font.weight.bold,
  lineHeight: 1.2, fontFamily: font.family.serif }`. Fields may reference other tokens or hold
  literals; a partial override merges over the inherited composite.
- **Shadow** is a list of layers: `shadow.md: [ drop(y: 2, blur: 4, spread: -1, color: rgba(0,0,0,0.06)),
  drop(y: 4, blur: 6, spread: -1, color: rgba(0,0,0,0.10)) ]`. `drop(...)` is a cast shadow, `inset(...)`
  an inner one; both take named args (`x`, `y`, `blur`, `spread`, `color`).

Shadow and glow colors are absolute on purpose (`color.shadow: neutral.950`): a shadow stays dark on
any surface, so a mode-flipping role there would turn it into a glowing halo in dark mode.

## 9. The generator pipeline

Resolution is a fixed sequence from `.ds` source to an in-memory `DesignSystem` and a `.gen.yaml`
artifact:

1. **Lex + parse** the source into an AST (statements and values), recovering from errors.
2. **Resolve** the AST into a `DesignSystemSource`: seeds, a token map, mode axes, mode-specific
   seeds, the active block, and hide/pin patterns. Semantic generators emit their role tokens here,
   carrying a transform program where one applies.
3. **Generate** the `DesignSystem`: expand color seeds into OKLCH ramps, expand numeric scales,
   apply explicit tokens (which shadow generator output), then resolve every token reference at build
   time so the runtime resolver never chain-follows. Cycles and dangling references become halt
   diagnostics.
4. **Serialize** to `.gen.yaml`, split into a `PRIMITIVES` section (fixed constants) and a
   `SEMANTICS` section (mode-keyed roles), with provenance comments and any generation warnings.

The runtime never reads `.gen.yaml` back: it always rebuilds the in-memory system from the `.ds`
source. The artifact exists purely as a human-readable and tool-readable export of the resolved
catalog.

## 10. File format

- **Source**: a `.ds` document (stored as `Styles/<name>.styles` in a project). One file per brand;
  `default` carries the project's base system. A brand file inherits `default` and declares only its
  differences.
- **Resolved artifact**: a sibling `.gen.yaml`, regenerated automatically on every source change and
  never hand-edited (it carries a "GENERATED, DO NOT EDIT" header).
- **Cascade**: folder structure forms an inheritance chain to the root `default`, unless `root: true`
  or `inherits: none` breaks it.

## 11. Versioning and migration

The DSL is a settled surface that evolves additively under a forgiving parser: new generators, roles,
and named boundary stops are added without breaking existing files, and near-miss authoring forms are
absorbed with a teaching diagnostic rather than rejected. Legacy pre-DSL YAML `.styles` files are
lifted into the DSL layout by a one-shot startup migration that re-expresses the built-in defaults as
DSL and layers the user's customizations on top (append-and-resolver-wins). Because the runtime
rebuilds from source and the resolved `.gen.yaml` is disposable, a regeneration is always safe: the
source `.ds` is the single source of truth.

## 12. Related references

- [`reference/`](reference/): the authoring guides (core concepts and stop vocabularies, authoring and
  extending a brand, mode transforms). These are the surface an author and the AI read.
- [Blueprint language spec](https://github.com/brilliant-hq/blueprint): the design language whose
  `$token` bindings resolve through a system authored here. The `.ds` extension is introduced there as
  the design-system document type complementary to Blueprint's `.bl`.
- [brilliant-hq/brilliant](https://github.com/brilliant-hq/brilliant): the agent and product docs
  (MCP and HTTP setup, design guidance) for using Brilliant end to end.
