# The Brilliant Design System Language

### A Reference for the `.ds` Token Language

This document is the authoritative language reference for the Brilliant design system language, the small declarative language in which a Brilliant design system is authored. A design system written in this language is a concise, mode-aware description of a full token catalog: a handful of seed declarations expand into hundreds of resolved color, spacing, type, and effect tokens that every Brilliant design references.

The language is the companion to [Blueprint](https://github.com/brilliant-hq/blueprint), Brilliant's design-authoring language. Every token a Blueprint design references (a `$name` binding) resolves through a design system authored in the language specified here. Where the two languages meet, this document is normative for the design system side and defers to the Blueprint specification for the design side.

This reference specifies what the language *is*: its lexical and syntactic grammar, its static and dynamic semantics, its generation algorithms, its diagnostics, and the conformance obligations of an implementation. It is written to be read cold by an engineer or an autonomous agent building against the language. It is not a tutorial. The informal authoring guides, which teach the language by example and give design guidance, live alongside this document under [`reference/`](reference/) and are cross-referenced throughout; they are the surface a designer reads, this is the surface an implementer reads.

---

## Table of contents

1. [Introduction and design goals](#1-introduction-and-design-goals)
2. [Notation and conformance](#2-notation-and-conformance)
3. [Processing model](#3-processing-model)
4. [Lexical structure](#4-lexical-structure)
5. [Syntactic grammar](#5-syntactic-grammar)
6. [Statements](#6-statements)
7. [Values](#7-values)
8. [Primitives](#8-primitives)
9. [Semantic generators](#9-semantic-generators)
10. [Mode transforms](#10-mode-transforms)
11. [Mode-keyed semantic values](#11-mode-keyed-semantic-values)
12. [Aliases and reference resolution](#12-aliases-and-reference-resolution)
13. [Composites](#13-composites)
14. [Color ramp generation](#14-color-ramp-generation)
15. [Numeric scale generation](#15-numeric-scale-generation)
16. [Transform program execution](#16-transform-program-execution)
17. [The cascade](#17-the-cascade)
18. [The resolved artifact](#18-the-resolved-artifact)
19. [Diagnostics](#19-diagnostics)
20. [Conformance requirements](#20-conformance-requirements)
21. [Versioning and evolution](#21-versioning-and-evolution)
- [Appendix A. Collected grammar](#appendix-a-collected-grammar)
- [Appendix B. Catalog constants](#appendix-b-catalog-constants)
- [Appendix C. The seed template](#appendix-c-the-seed-template)
- [Appendix D. Diagnostics index](#appendix-d-diagnostics-index)

---

## 1. Introduction and design goals

### 1.1 What the language is

A document in this language declares a design system. The unit of declaration is the *statement*, and the dominant statement is the assignment of a *token path* to a *value*. Values range from bare literals (a hex color, a number, a string) to *generator calls* that expand a single seed into a whole family of tokens. The result of processing a document is a *token catalog*: a flat map from token key to resolved value, where many keys resolve differently depending on which display *modes* are active (light or dark, comfortable or compact, standard or high-contrast).

The language is deliberately small. It has no control flow, no user-defined functions, no arithmetic beyond what a literal expresses, and no imports beyond a fixed folder cascade. Its power comes from a fixed vocabulary of built-in generators, each of which encodes a large amount of design knowledge (an OKLCH color ramp, a t-shirt sizing scale, a set of default mode behaviors) behind a one-line call.

### 1.2 Design goals

The following goals motivate the design and are stated here as intent, not as normative requirements.

- **Generative, not enumerated.** An author declares seeds (a brand color, a spacing base, a type ramp) and a small role vocabulary; the generator expands each seed into a full stop scale. A real brand is usually a few lines rather than hundreds of hand-listed values.

- **Mode-aware by construction.** Light and dark, comfortable and compact, standard and high-contrast are axes, not separate files. A single semantic declaration re-resolves correctly across every active mode combination, and the mode behavior of a generator is built in rather than re-authored per file.

- **Two clean layers.** *Primitives* are mode-independent constants (Tailwind-style fixed stops). *Semantics* are mode-keyed roles built on top of primitives. External consumers (Style Dictionary, Tokens Studio, custom Tailwind generators) read the resolved catalog as exactly that contract.

- **Cascading and forgiving.** A brand file inherits the project default and declares only what differs. The parser recovers from errors and emits teaching diagnostics rather than failing hard: a near-miss form is absorbed with a diagnostic rather than rejected.

- **Legible and durable.** One declaration per line. Comments carry design intent forward to future sessions and to the AI that designs against the system, in the surfaces that preserve them (see [4.8](#48-comments-and-trivia) for the precise, honest scope of comment preservation).

### 1.3 Scope and naming

Throughout this document the language is referred to as *the language* or *the `.ds` language*. On disk within a Brilliant project a design system source file is named `Styles/<name>.ds`: the canonical extension new writes create, and also the language's public identity (as introduced in the Blueprint documentation). The legacy extension `.styles` is still read and is migrated to `.ds` in place when a project is opened, so a not-yet-migrated project keeps working; the two extensions denote the same language. The resolved artifact a source compiles to is a sibling file `Styles/.gen/<name>.gen.yaml`. Sections that refer to on-disk layout use the `.ds` and `.gen.yaml` names; the one-shot legacy migration ([21.2](#212-legacy-migration)) is the only place the old `.styles` name still appears as a live input.

### 1.4 Relationship to Blueprint

[Blueprint](https://github.com/brilliant-hq/blueprint) is the language in which Brilliant designs are authored. A Blueprint design references a design system token with a `$` sigil (`f[($color.surface)]`, `t(..., $font.size.lg)`). That `$name` is resolved against a design system authored in the language specified here. The two languages are complementary and share the OKLCH ramp constants (so a Blueprint `$var` preprocessor and this language expand the same seed identically), but they are distinct surfaces with distinct grammars. In particular:

- The `$` sigil is a Blueprint concern. Inside a `.ds` document a token reference is a bare dotted path with **no** sigil (`color.error: red.500`). The only `$`-led identifiers in `.ds` are engine metadata keys (`$default`, `$transforms`) and the picker directives (`$hide`, `$pin`); see [4.3](#43-identifiers) and [6.6](#66-hide-and-pin).
- Alpha ordering in hex is the **same** on both surfaces: an 8-digit hex is `#RRGGBBAA` (alpha last), matching CSS and Blueprint; see [4.6](#46-hex-color-literals) and the note therein.

### 1.5 Relationship to the authoring guides

Three informal guides accompany this reference and are published alongside it under `reference/`:

- [`reference/core.md`](reference/core.md): the token vocabularies and how a design references them from Blueprint.
- [`reference/authoring.md`](reference/authoring.md): authoring and extending a brand.
- [`reference/authoring-modes.md`](reference/authoring-modes.md): mode transforms.

Those guides are the teaching surface. This document does not duplicate their tutorial voice; it specifies the mechanics they illustrate. Where a guide and this reference appear to differ, this reference describes the implemented behavior and the guide describes the intended authoring experience; [19](#19-diagnostics) and the non-normative notes throughout flag the places where implementation and documented intent diverge.

---

## 2. Notation and conformance

### 2.1 Grammar notation

Grammar fragments in this document use a compact EBNF dialect defined here. The same dialect is used in every inline fragment and in the collected grammar of [Appendix A](#appendix-a-collected-grammar).

| Form | Meaning |
|---|---|
| `lowercase` | a terminal *token kind* produced by the lexer ([4.2](#42-token-kinds)), e.g. `identifier`, `integer`, `hexColor` |
| `'text'` | a terminal matched by a token's exact source text, e.g. `'modes'`, `':'` |
| `CamelCase` | a non-terminal (a grammar production) |
| `A B` | `A` followed by `B` (concatenation) |
| <code>A &#124; B</code> | `A` or `B` (alternation) |
| `A*` | zero or more `A` |
| `A+` | one or more `A` |
| `A?` | optional `A` |
| `( A )` | grouping |
| `; text` | a comment on the production, not part of the grammar |

Two lexical conventions pervade the grammar and are stated once here rather than in every rule:

- **Newlines are insignificant.** The lexer discards all whitespace including newlines ([4.8](#48-comments-and-trivia)). The grammar is not newline-terminated; statements and fields are delimited structurally.
- **Commas are usually optional.** Inside records, lists, and semantic-block field sequences, commas are optional separators; a new field begins wherever the next key begins. Commas are *required* only where they disambiguate: between the values of a combo key, between function-call arguments, between list elements, between hide/pin patterns, and between mode-axis values. Each rule that requires commas marks them as non-optional; elsewhere a `','?` in the grammar denotes an optional separator.

### 2.2 Conformance keywords

The keywords MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as described in RFC 2119. They apply to the behavior of a *conforming implementation* (a lexer, parser, resolver, and generator that together process the language), not to the behavior of an author.

### 2.3 Conformance targets

The language is processed by a pipeline of four cooperating components. This document imposes obligations on each; [20](#20-conformance-requirements) collects them.

- **Lexer**: source text to a token stream. MUST NOT throw on any input.
- **Parser**: token stream to an abstract syntax tree (AST). MUST NOT throw on any input; MUST recover and continue past errors.
- **Resolver**: AST to a *design system source* (an intermediate, mergeable representation). MUST NOT halt; every problem is a warning.
- **Generator**: a merged design system source to a resolved *design system* (the token catalog). MAY emit halting diagnostics; a halt blocks an authoring write but does not throw.

### 2.4 Terminology

- **Token**, **token key**, **token path**: a token is a named resolved value; its *key* is the dotted string that names it (`color.surface`, `spacing.md`, `primary.500`). A *token path* is the syntactic form of a key.
- **Seed**: the input to a generator that produces a scale (a color for a ramp, a base number for a numeric scale).
- **Stop**: a member of a generated scale, named by a numeric or size suffix (`primary.500`, `spacing.4`, `font.size.lg`).
- **Role**: a semantic name mapped onto a stop by a semantic generator (`primary.mid`, `spacing.lg`, `font.lineHeight.normal`).
- **Mode**, **axis**: a *mode* is one selectable value of a display *axis*. `theme` is an axis; `light` and `dark` are its modes.
- **Primitive** vs **semantic**: a *primitive* token is mode-independent (a frozen stop). A *semantic* token carries mode behavior (mode-keyed values and/or a transform program). See [8](#8-primitives) and [9](#9-semantic-generators).
- **Halt** vs **warn**: the two diagnostic severities. A *halt* refuses an authoring write; a *warn* rides along and does not block. See [19](#19-diagnostics).

---

## 3. Processing model

A `.ds` document is processed by a fixed pipeline. Each stage is specified in the section named.

```
.ds source text
  --> Lexer          (§4)   token stream + recovered lexical errors
  --> Parser         (§5)   AST + recovered syntactic errors  (never throws)
  --> Resolver       (§6-§13)  design system source + warnings (never halts)
  --> merge          (§17)  cascade of sources merged into one source
  --> Generator      (§14-§16)  resolved design system (token catalog) + halt/warn diagnostics
  --> serialize      (§18)  .gen.yaml artifact (export only)
```

An inverse path exists for programmatic mutation and for surfacing a system to an agent:

```
design system source --> Migrator (§20.4) --> AST --> Formatter (§20.4) --> .ds text
```

Two invariants of the model are load-bearing and normative:

1. **The runtime never reads the resolved artifact back.** Runtime token resolution rebuilds the in-memory design system from the `.ds` source through the generator, then queries it ([16](#16-transform-program-execution)). The `.gen.yaml` file ([18](#18-the-resolved-artifact)) is a human- and tool-readable export only. An implementation MUST NOT make runtime resolution depend on `.gen.yaml`.

2. **The cascade is merged once, at the source level, then generated once.** When a canvas resolves against a chain of files ([17](#17-the-cascade)), the sources are merged field by field into a single source, and the generator runs a single time on the merged source. Merging at the source level (rather than generating each file and merging catalogs) preserves cross-file logic such as a child's stop override shadowing a parent's generated stop.

---

## 4. Lexical structure

The lexer performs a single forward pass over the source, producing a stream of tokens terminated by a synthetic end-of-file token. It never throws: malformed input yields an `invalid` token plus a recovered diagnostic, and scanning continues.

### 4.1 Source and positions

Source is a sequence of Unicode code points read as bytes for the purpose of the character classes below (the lexer recognizes only ASCII letters; see [4.3](#43-identifiers)). Every token carries a source *span* with an inclusive start and an exclusive end. Positions are reported as 1-indexed line and column together with a 0-indexed byte offset. A newline advances the line and resets the column.

### 4.2 Token kinds

The lexer emits exactly these token kinds.

| Kind | Emitted for | Example source |
|---|---|---|
| `identifier` | a word containing at least one letter or `_`; includes digit-leading words (`2xl`), `$`-led words (`$default`), and hyphenated words (`on-surface`) | `font`, `2xl`, `on-surface`, `$default` |
| `integer` | a pure integer with an optional single leading `-` | `42`, `-3` |
| `decimal` | a number with a `.` between digits and/or a trailing `%` | `1.5`, `-0.01`, `63.7%` |
| `string` | double-quoted content, no escapes; text excludes the quotes | `"Noto Serif"` |
| `hexColor` | `#` followed by exactly 3, 4, 6, or 8 hex digits; text retains the `#` | `#FFF`, `#FFF8`, `#FB2C36`, `#FB2C3680` |
| `colon` | `:` | |
| `comma` | `,` | |
| `dot` | `.` | |
| `star` | `*` | |
| `lbrace` `rbrace` | `{` `}` | |
| `lbracket` `rbracket` | `[` `]` | |
| `lparen` `rparen` | `(` `)` | |
| `eof` | the end sentinel (always the final token) | |
| `invalid` | a recovered token after a lexical error; the message is attached to the error list | |

There is no minus token and no plus token: numeric signs are folded into numeric tokens ([4.5](#45-numbers)). There is no boolean or keyword token kind: `true`, `false`, `none`, `bold`, `mirror`, `light`, and every other word is an `identifier`, interpreted contextually by the parser and resolver.

### 4.3 Identifiers

The character classes are:

- *digit*: `0`-`9`.
- *hex digit*: a digit or `A`-`F` or `a`-`f`.
- *letter*: `A`-`Z` or `a`-`z` only. No Unicode letters are recognized.
- *identifier start*: a letter, `_`, or `$`.
- *word part* (any character after the first in an identifier or digit-leading word): a letter, digit, `_`, or `-`.

Because `$` is an identifier start, `$default` and `$transforms` are single identifier tokens. Because a hyphen is a word part, `on-surface`, `high-contrast`, and `letter-spacing` are single identifier tokens, not subtractions. A digit does not begin an identifier through the identifier-start path; digit-leading words such as `2xl` are lexed by the number path and reclassified as identifiers ([4.5](#45-numbers)).

```
Identifier      = identifier                 ; token kind
```

### 4.4 Punctuation

The single-character tokens `: , . * { } [ ] ( )` are each emitted as their own token on sight.

### 4.5 Numbers

A `-` or a digit begins the number-or-identifier scan; a `+` immediately followed by a digit also begins a number (the `+` is dropped, so `+1` lexes as the integer `1`). The scan proceeds as follows.

1. An optional leading `-` is consumed.
2. Word-part characters are scanned.
3. If the scanned word contains any letter or `_`, the token is an **identifier**, not a number. Thus `2xl`, `3xl`, and `brand5` are identifiers. A leading `-` on such a would-be identifier is a lexical error (`identifier cannot start with "-"`) and yields an `invalid` token; so `-x` is invalid while `-5` is the integer `-5`.
4. A pure numeric word is an `integer` by default. If it is immediately followed by `.` and then a digit, the decimal part is consumed and the kind becomes `decimal`. A `.` *not* followed by a digit is left as a separate `dot` token, which is what allows a path segment like `spacing.1` to lex as `identifier dot integer`.
5. A trailing `%` is consumed and forces the kind to `decimal`, retaining the `%` in the token text. So `63.7%` and `50%` are single `decimal` tokens whose text ends in `%`.

A bare `+` not followed by a digit is not an identifier start and yields an `invalid` token.

The `%` is stripped and divided out during parsing, not lexing: a `decimal` token whose text ends in `%` carries a value of *(number)/100* once parsed ([7.3](#73-numbers)). A percentage is therefore a lexical convenience for a fraction; `63.7%` and `0.637` denote the same value.

```
Integer         = integer
Decimal         = decimal                    ; text may carry a trailing '%'
```

### 4.6 Hex color literals

A hex color is `#` followed by a run of hex digits. The lexer accepts exactly 3, 4, 6, or 8 hex digits and rejects any other count with `hex color must be 3, 4, 6, or 8 hex digits (got N)`, yielding an `invalid` token. A 4-digit hex is a valid short form (`#RGBA`) carrying alpha in its trailing digit.

```
HexColor        = hexColor                   ; '#' then 3, 4, 6, or 8 hex digits
```

> **Note (alpha ordering: alpha last).** The `.ds` color layer reads hex the way CSS and Blueprint do: **alpha last**. An 8-digit hex is `#RRGGBBAA` and the 4-digit short form is `#RGBA`, both carrying alpha in the trailing digits. Authoring `#FF000080` in a `.ds` file yields opaque-red at half alpha (red `0xFF`, green `0x00`, blue `0x00`, alpha `0x80`), and `#F008` is the same color in short form. This matches CSS and Blueprint exactly; `.ds` and Blueprint now agree at the author level. (The shared engine-side `fromHex` helper used by non-`.ds` consumers still reads 8-digit hex alpha-first, but the `.ds` color path uses its own alpha-last parser, so an author never sees the old inconsistency.)

### 4.7 Strings

A string is double-quoted content read up to the next `"`, newline, or end of file. There are no escape sequences. A string that reaches a newline or end of file before its closing quote is a lexical error (`unterminated string`) and yields an `invalid` token; a string cannot span lines. The token's text is the content without the surrounding quotes.

```
String          = string                     ; '"' ... '"', no escapes
```

### 4.8 Comments and trivia

Spaces, tabs, carriage returns, and newlines are whitespace. `//` begins a line comment that runs to end of line. `/* ... */` is a block comment; an unterminated block comment appends an `unterminated block comment` error but emits no token. Whitespace and comments are discarded entirely at lex time: they are not tokens and do not appear in the token stream.

> **Note (comment preservation, honest scope).** Because the lexer discards comments and the formatter cannot re-emit them, comments do **not** survive a parse-then-format round trip and are **not** copied into `.gen.yaml` (whose provenance comments are generated fresh from token sources). Comments survive only in two paths: (a) when a `.ds` file is edited by surgical text patching, which splices new statements into the existing text rather than reformatting it, and (b) when a raw source file is emitted verbatim into agent context. An author's design-intent comments therefore persist in a hand-maintained or surgically-patched `.ds` file, but any transformation that goes through the formatter drops them. Claims that comments are "preserved into the resolved catalog" apply only to the surgical-patch and verbatim-emit surfaces.

### 4.9 Lexical error recovery

Every lexical error (unexpected character, bad hex length, unterminated string, identifier starting with `-`, unterminated block comment) appends a `ParseError` (a message plus a span) to the lexer's error list and, where a token position is involved, emits an `invalid` token so scanning can continue. The parser prepends the lexer's errors to its own error list ([5.7](#57-error-recovery)). A conforming lexer MUST NOT throw on any input.

### 4.10 Worked example (lexical)

The source `font.size.2xl: 63.7%` lexes to:

```
identifier("font") dot identifier("size") dot identifier("2xl") colon decimal("63.7%") eof
```

`63.7%` is a single `decimal` token whose text retains the `%`. `2xl` is an `identifier` (it contains letters) though it begins with a digit. The malformed `#12345` (five hex digits) yields one `invalid` token and the error `hex color must be 3, 4, 6, or 8 hex digits (got 5)`.

---

## 5. Syntactic grammar

The parser is a recursive-descent parser over the token stream. It never throws and recovers at statement and field boundaries. It produces a `StylesFile` AST (a sequence of statements) plus a list of recovered errors.

### 5.1 File and statements

```
File            = Statement* eof

Statement       = ModesDecl
                | ActiveBlock
                | HidePinDecl
                | UnsetBlock          ; only when 'unset' is immediately followed by '{'
                | PathStatement       ; assignment or semantic block
                ; a leading token that is not an identifier is an error; one token is skipped
```

Statement dispatch keys on the leading token. If it is an identifier, the parser dispatches on its text: `modes` begins a `ModesDecl`, `active` an `ActiveBlock`, `hide` or `pin` a `HidePinDecl`, and `unset` an `UnsetBlock` **but only if the next token is `{`** (otherwise `unset` is an ordinary path and the statement is a `PathStatement`). Any other identifier begins a `PathStatement`. A leading non-identifier token produces `expected statement, got <kind> "<text>"` and the parser advances one token to recover.

### 5.2 Paths

```
Path            = ( identifier | integer ) ( '.' ( identifier | integer ) )*
```

A path is a dotted sequence of identifier or integer segments. Integer segments are how numeric stops appear in a key (`brand.300`, `spacing.1`). Paths name tokens, semantic blocks, active fields, hide/pin patterns, and unset targets.

### 5.3 Modes declaration

```
ModesDecl       = 'modes' '{' ModeAxis* '}'
ModeAxis        = identifier ':' '[' ( identifier ( ',' identifier )* )? ']'
```

A `modes` block declares the display axes. Each axis has a name and a bracketed, comma-separated list of mode values; an empty value list is permitted. By convention the **first** value on each axis is that axis's default; this convention is consumed by later stages ([16](#16-transform-program-execution)) and is not enforced by the parser. The built-in system declares three axes; see [Appendix C](#appendix-c-the-seed-template).

### 5.4 Path statements: assignment vs semantic block

```
PathStatement   = Path ( AssignmentBody | SemanticBlockBody )
                ; a path followed by neither ':' nor '{' is an error
```

After parsing the leading path the parser peeks: a `:` begins an assignment body ([5.5](#55-assignment-and-override-block)); a `{` begins a semantic block body ([5.6](#56-semantic-blocks)); anything else produces `expected ':' or '{' after '<path>'`.

> **Note (legacy mode blocks).** Every top-level `path { ... }` now parses as a semantic block. An older top-level mode-block form (`theme.dark { ... }` as a *mode* selector rather than a token) is no longer produced by the parser. The corresponding AST nodes and a resolver branch still exist for backward compatibility with hand-built ASTs, but no source text produces them, and this document does not describe top-level mode blocks as a language feature.

### 5.5 Assignment and override block

```
AssignmentBody  = ':' Value OverrideBlock?
                ; the OverrideBlock is parsed only when Value is neither a record nor a list
OverrideBlock   = '{' OverrideEntry* '}'
OverrideEntry   = ( identifier | integer | string ) ':' Value ','?
```

An assignment binds a path to a value. If the value is a scalar or a call (not a record or list) and is immediately followed by `{`, that brace begins a trailing *override block* whose entries override individual generated stops (`brand: scale(#1E40AF) { 300: #4B68C8 }`). If the value is itself a record or a list, the following `{` cannot be an override block (the record already consumed the value slot), so `typography.h1: { ... }` binds the record and does not look for overrides.

The trailing override block is retained for backward compatibility. The preferred modern form is a separate stop assignment (`brand.300: #4B68C8`); see [8.5](#85-stop-overrides).

### 5.6 Semantic blocks

```
SemanticBlockBody = '{' SemanticField* '}'
SemanticField     = SemanticKey ( ',' SemanticKey )* ':' Value ','?
SemanticKey       = Path                 ; e.g. $default, dark, theme.dark
```

A semantic block gives a token per-mode values. Each field is one or more comma-separated *keys* mapped to one value; multiple keys on a field form a *combo* that fires only when every listed mode is active. A key is a path: `$default` (the fallback), a bare mode value (`dark`), or an axis-qualified value (`theme.dark`). Combo keys are canonicalized during parsing: a single key becomes its dotted text; a combo is normalized to value-only segments, sorted alphabetically, and pipe-joined, so `theme.dark, density.compact` becomes `density.compact|theme.dark` at the AST level (and is further reduced to value-only `compact|dark` by the resolver; see [11.2](#112-key-canonicalization)).

### 5.7 Error recovery

The parser records an error on any mismatch but returns the current (unconsumed) token so callers keep progressing. Two recovery routines bound the damage: recovery to a statement boundary skips to the next `}` or end of file; recovery to a field boundary skips to the next `,` or `}`. Recovery is designed so a single mistake yields a single targeted diagnostic rather than a cascade. For example, two function-call arguments separated by a space instead of a comma yield exactly one `expected ','` diagnostic per missing comma, and the closing paren still matches. A conforming parser MUST recover and continue; it MUST NOT throw.

### 5.8 Where commas and newlines matter

- Statements are delimited structurally; there is no terminator.
- Record fields, list elements written across lines, and semantic-block fields are separated by newlines or optional commas.
- The multiple keys of a single combo field are comma-separated (the comma groups them onto one value).
- Function arguments, list elements, `hide`/`pin` patterns, and mode-axis values are comma-separated (with space-forgiveness applied to function arguments).

---

## 6. Statements

This section specifies each statement form's static semantics (what the resolver records) and, where applicable, its dynamic effect. Values are specified in [7](#7-values); generators in [8](#8-primitives) and [9](#9-semantic-generators).

The resolver's output is a *design system source*, an intermediate representation with these principal fields: `seeds` (generator seeds, hex or number-as-string), `tokens` (explicit tokens, aliases, semantic outputs, and the `$hide`/`$pin` lists), `customModes` (the flattened set of all mode values), `modeAxes` (axis name to value list), `modeSeeds` (per-mode seed overrides), `removals` (unset paths), `active` (the active block), `isRoot` and `inheritsBase` (cascade flags), `numericSeeds` (which seeds are numeric), `seedExpressions` (the original right-hand-side AST per generator seed, for round-trip), and `generatedKeys` (which keys a generator produced).

### 6.1 The modes declaration

`modes { axis: [v, ...] }` records `modeAxes[axis] = [v, ...]` and adds every value to the flat `customModes` set. Axis *names* are not added to `customModes`; only their values are. The unset resolver and the cascade use `modeAxes` to recognize mode prefixes ([17.4](#174-unset-application)).

### 6.2 Primitive assignment

A top-level assignment `name: value` records a token according to the value's shape. Before any token emission, two names are intercepted as cascade flags: `root` and `inherits` ([6.7](#67-root-and-inherits)). Otherwise:

| Value shape | Effect |
|---|---|
| hex color literal | `tokens[name] = "#..."` (a single color primitive; wrap in a generator for a ramp) |
| number literal | `tokens[name] = number` |
| string literal | `tokens[name] = string` |
| bareword | `tokens[name] = text` (the alias-vs-string decision is deferred to the generator; [12](#12-aliases-and-reference-resolution)) |
| reference path | `tokens[name] = path` (an alias, resolved at build time) |
| function call | dispatched to a generator ([6.3](#63-generator-dispatch)) |
| record | `tokens[name] = map` (a typography composite; [13.1](#131-typography)) |
| list | `tokens[name] = list` (a shadow composite; [13.2](#132-shadow)) |

An explicit assignment removes `name` from `generatedKeys` first, so a later explicit assignment shadows an earlier generator emission of the same key (this governs the migrator's round-trip in [20.4](#204-round-trip-formatter-and-migrator), and is distinct from the dynamic shadowing in [15.4](#154-double-generation)).

If the assignment carries a trailing override block, each entry writes a `name.key` scalar into `tokens` ([8.5](#85-stop-overrides)).

### 6.3 Generator dispatch

A function-call value on the right of an assignment dispatches by function name. The recognized generators are `color`, `number`, `boldness`, `tshirt`, `looseness` (the current surface), plus the legacy wrappers `scale`, `oklch` (standalone), `linear`, `explicit`, and `modular`. An unrecognized name produces the warning `<name> = <fn>(...): unknown generator function` and emits nothing. The resolver snapshots the token keyset around a generator call so that keys the generator added are recorded in `generatedKeys`, and it records the original call AST in `seedExpressions[name]` for round-trip.

The legacy wrappers are specified in [8.6](#86-legacy-generators); the primary generators in [8](#8-primitives) (`color`, `number`) and [9](#9-semantic-generators) (`boldness`, `tshirt`, `looseness`).

### 6.4 The active block

```
ActiveBlock     = 'active' '{' ActiveField* '}'
ActiveField     = ( 'brand' ':' Path | 'modes' ':' Record ) ','?
```

`active { brand: <name>, modes: { axis: value } }` records the folder-level active brand and starting mode set. The `modes:` field requires a record literal. An unknown field name produces `unknown active field "<name>" (expected 'brand' or 'modes')`. The active block is metadata for visibility (which brand and modes a folder starts in); runtime resolution is queried per element and is not driven by the active block. The resolver records an `active` value only when at least one field is present.

> **Note (active in agent-authored bodies).** Although `active { ... }` is valid grammar and resolves cleanly, the agent-facing authoring directive that writes brand files rejects a body containing `active`, because folder-level defaults are meant to be author-controlled. The same construct is therefore legal in a hand-maintained `default.ds` but refused when authored through that directive. See [19.5](#195-the-authoring-write-gate).

### 6.5 Value form summary

The value grammar is specified in full in [7](#7-values). In statement position the value follows the `:` of an assignment or the `:` of a semantic-block or active field.

### 6.6 hide and pin

```
HidePinDecl     = ( 'hide' | 'pin' ) ':' NamePattern ( ',' NamePattern )*
NamePattern     = Path ( '.' '*' )?              ; the '.*' suffix sets a glob flag
```

`hide:` and `pin:` are in-app color-picker display hints: which palettes are tucked away, which pin to the top of the picker. A pattern is a path optionally suffixed with `.*` to form a glob (`tailwind.*` matches `tailwind` and any `tailwind.<...>` key). These declarations become `tokens["$hide"]` and `tokens["$pin"]` lists on the resolved source. They do not change what a design or the AI can reference; they are purely presentation hints for the picker.

### 6.7 root and inherits

At top level, `root: true|false` sets the source's `isRoot` flag and `inherits: default|none` sets `inheritsBase`. The flag values are barewords: `true`/`false` for `root`; for `inherits`, `default` or `true` means inherit (the default), `none` or `false` means standalone. A malformed `root` value (`root: yes`) warns (`expected 'true' or 'false'`) and falls back to `false`; a malformed `inherits` value warns and keeps `true`. Neither name pollutes `seeds` or `tokens`. The cascade semantics of these flags are specified in [17.2](#172-cascade-resolution) and [17.5](#175-standalone-brands-inherits-none).

### 6.8 unset

```
UnsetBlock      = 'unset' '{' ( ',' | WildcardPath | Path )* '}'
WildcardPath    = '*' ( '.' ( identifier | integer ) )+
```

`unset { path, *.key }` drops entries a parent contributed in the cascade. Inside the braces, stray commas are skipped, a leading `*` begins a wildcard path requiring at least one dotted segment, and any other token that is not an identifier or integer produces `expected dotted path inside 'unset { ... }'` and recovers. Each path is recorded as a removal (a list of segments). Unset is meaningful only at the top level (an unset inside a mode context warns and is skipped). The removals are applied during the cascade merge, not at resolve time; see [17.4](#174-unset-application).

### 6.9 Worked example (statements)

```
modes {
  theme:   [light, dark]
  density: [comfortable, compact]
}

primary: boldness(color(#0080FF))
color.error: red.500

color.surface {
  $default: neutral.200
  dark:     neutral.900
}

hide: red, orange
pin:  primary, color.surface
```

The `modes` block records two axes and four mode values. `primary` is a semantic color generator ([9.1](#91-boldness)). `color.error` is an alias ([12](#12-aliases-and-reference-resolution)). `color.surface` is a semantic block ([11](#11-mode-keyed-semantic-values)). `hide` and `pin` record picker hints. This example resolves with zero warnings (validated).

---

## 7. Values

```
Value           = Record | List | HexColor | String
                | Integer | Decimal
                | FunctionCall | Reference | Bareword

FunctionCall    = identifier '(' ( CallArg ( ',' CallArg )* ','? )? ')'
CallArg         = identifier ':' Value           ; named
                | Value                           ; positional
Reference       = identifier ( '.' ( identifier | integer ) )+   ; multi-segment path
Bareword        = identifier                      ; single segment

Record          = '{' RecordField* '}'
RecordField     = RecordKey ':' Value             ; commas optional between fields
RecordKey       = ( identifier | integer | string ) ( '.' ( identifier | integer ) )*
List            = '[' ( Value ( ',' Value )* ','? )? ']'
```

The parser selects a value production by the leading token: `{` begins a record, `[` a list, `hexColor`/`string`/`integer`/`decimal` the corresponding literal, and an `identifier` begins a function call (if followed by `(`), a reference (if followed by `.`), or a bareword (otherwise). A value in a position where none of these apply produces `expected value, got ...` plus a fallback bareword so recovery can proceed.

### 7.1 Hex color literals

A `hexColor` token becomes a hex color literal retaining its `#`. Its interpretation as a color (including the alpha-last ordering of 4- and 8-digit forms) is specified in [4.6](#46-hex-color-literals).

### 7.2 Strings

A `string` token becomes a string literal carrying its unquoted content.

### 7.3 Numbers

An `integer` token becomes a number literal with an integer value. A `decimal` token becomes a number literal whose value is computed as follows: if the text ends with `%`, the `%` is stripped and the number is divided by 100 (`63.7%` yields `0.637`); otherwise if it contains `.` it is parsed as a double; otherwise as an integer. This normalization is where OKLCH lightness percentages become 0-to-1 fractions ([8.3](#83-the-oklch-constructor)).

### 7.4 Barewords and references

An identifier in value position with no `.` and no `(` is a bareword. A bareword may denote a string primitive (`font.family: Inter`), a boolean flag value (`true`, `none`), a transform operator (`mirror`), or an alias to another token; the decision is made downstream (in the generator for alias-vs-string, [12](#12-aliases-and-reference-resolution)). A multi-segment identifier path in value position is a reference (`red.500`), recorded as an alias.

### 7.5 Function calls

A function call is an identifier followed by a parenthesized, comma-separated argument list with an optional trailing comma. An argument is *named* when an `identifier :` lookahead matches, otherwise *positional*. Function calls appear as generators in statement position ([6.3](#63-generator-dispatch)) and as constructors in value position (`oklch(...)`, `rgba(...)`, `drop(...)`, `inset(...)`, `range(...)`, `linear(...)`, `geometric(...)`).

**Comma forgiveness.** If two arguments are separated by a space instead of a comma (the CSS `oklch(45% 0.012 140)` habit), the parser emits exactly one `expected ','` diagnostic per gap and keeps consuming, so the closing paren still matches and the error does not cascade. The comma form parses cleanly with zero diagnostics.

### 7.6 Records and lists

A record is a brace-delimited sequence of `key: value` fields with optional commas; record keys may be dotted (`density.compact: -1` inside a `transforms` record). An empty record is permitted. A list is a bracket-delimited, comma-separated sequence of values with an optional trailing comma; an empty list is permitted. Records model typography composites ([13.1](#131-typography)), `min`/`max` boundary arguments ([9.4](#94-boundary-stops)), and `transforms` programs ([10](#10-mode-transforms)); lists model numeric stop lists ([8.2](#82-the-number-generator)), shadow layer lists ([13.2](#132-shadow)), and mode-axis value lists ([5.3](#53-modes-declaration)).

### 7.7 Worked example (values)

```
red:      color(oklch(63.7%, 0.237, 25.331))
gap:      number([4, 8, 12, 16])
h1:       typography.h1
shadow.x: [ drop(y: 2, blur: 4, color: rgba(0, 0, 0, 0.06)) ]
```

Line 1 nests a constructor (`oklch(...)`) inside a generator (`color(...)`); the percentage lightness is divided to `0.637`. Line 2 is a list value. Line 3 is a reference value. Line 4 is a list of one `drop(...)` call whose `color` argument is an `rgba(...)` constructor. All four resolve with zero warnings (validated).

---

## 8. Primitives

A *primitive* is a mode-independent token or scale. Primitives are the lower layer; semantics ([9](#9-semantic-generators)) build on them. This section specifies the primitive-producing forms: the `color` generator, the polymorphic `number` generator, the `oklch` constructor, bare literals and aliases, stop overrides, and the legacy generators.

### 8.1 The color generator

```
ColorGenerator  = 'color' '(' ColorSeed ')'
ColorSeed       = HexColor | 'oklch' '(' Decimal ',' Decimal ',' Decimal ')' | String | Bareword | Integer
```

`color(seed)` registers a color seed. The seed may be a hex literal, an `oklch(L, C, H)` call (converted to sRGB hex at resolve time; [8.3](#83-the-oklch-constructor)), or a string/bareword/number carried verbatim. The generator records `seeds[name] = <hex or verbatim>` and does *not* itself emit stop tokens; the generator stage expands the seed into the 11-stop OKLCH ramp ([14](#14-color-ramp-generation)). A reference, record, or list seed is a warning.

*Static semantics.* `color(#0080FF)` records `seeds["primary"] = "#0080FF"`. `color(oklch(63.7%, 0.237, 25.331))` records a 6-digit `#RRGGBB` hex (7 characters including `#`).

*Dynamic semantics.* At generation the seed becomes the `.500` stop and the full `.50 .100 .200 .300 .400 .500 .600 .700 .800 .900 .950` ramp is produced, plus a bare `name` token equal to the seed color. See [14](#14-color-ramp-generation).

*Worked example (validated).* `primary: color(#0080FF)` resolves with `seeds["primary"] == "#0080FF"`; after generation `primary.500` equals the seed exactly, `primary.50` has OKLCH lightness `> 0.9` (measured `0.969`), and `primary.900` has lightness `< 0.35` (measured `0.289`).

### 8.2 The number generator

`number(...)` is polymorphic on its first argument. It always records a numeric seed and, in some forms, materializes explicit stop tokens `name.1 .. name.N`.

```
NumberGenerator = 'number' '(' NumberArgs ')'
NumberArgs      = List                            ; explicit stop list
                | 'range' '(' Value ',' Value ')' ',' Integer ( ',' Ramp )?
                | Value ( ',' Integer ( ',' Ramp )? )?
Ramp            = 'linear' '(' ( 'step' ':' Value )? ')'
                | 'geometric' '(' ( 'ratio' ':' Value )? ')'
```

The four forms:

1. **Explicit list**, `number([v1, ..., vN])`. Every element MUST be numeric. Records `seeds[name] = v1`, marks the seed numeric, and emits `name.1 .. name.N` = the list values (1-based). A non-numeric element is a warning and aborts the form.
2. **Range ramp**, `number(range(min, max), count, ramp?)`. `range` MUST have exactly two numeric arguments. `count` is a positive integer; `ramp` is optional. The stops span `[min, max]` inclusively. Records `seeds[name] = min` and emits `name.1 .. name.count`.
3. **Seed with count**, `number(value, count, ramp?)`. Records `seeds[name] = value` and emits `name.1 .. name.count`, stepping from `value`.
4. **Bare seed**, `number(value)`. Records only the seed; the stops are materialized by the dedicated catalog scale generator at generation time ([15](#15-numeric-scale-generation)).

**Ramp expansion.** The default ramp name is `linear`. For `linear`, the step is chosen by priority: an explicit `linear(step: S)`, else a range-derived step `(max - min) / (count - 1)`, else the seed value itself (or 1). For `geometric`, `geometric(ratio: R)` (default ratio 2) makes stop *i* equal `start * R^i`. An unknown ramp name warns and emits `count` copies of the start.

*Worked examples (validated).*

- `font.weight: number([100, 200, 300, 400, 500])` gives `font.weight.1 == 100`, `font.weight.5 == 500`, and `seeds["font.weight"] == "100"`.
- `gap: number(4, 5, linear())` gives `gap.1 == 4`, `gap.2 == 8`, `gap.3 == 12`, `gap.5 == 20` (step defaults to the seed).
- `s: number(0, 5, linear(step: 2))` gives `s.1 == 0`, `s.2 == 2`, `s.5 == 8`.
- `s: number(1, 4, geometric(ratio: 2))` gives `s.1 == 1`, `s.2 == 2`, `s.3 == 4`, `s.4 == 8`.
- `s: number(range(0, 1), 5, linear())` gives `s.1 == 0`, `s.3 == 0.5`, `s.5 == 1.0`.
- `spacing: number(4)` records only `seeds["spacing"] == "4"`; the catalog ramp materializes the stops.

### 8.3 The oklch constructor

`oklch(L, C, H)` in positional three-argument form, with all arguments numeric, denotes a color in the OKLCH space. `L` is on a 0-to-1 scale; because a `%` literal is divided by 100 during parsing, `63.7%` and `0.637` both denote the same lightness. A bare integer lightness greater than 1 is treated as a percentage, clamped, and warned. `C` is chroma and `H` is hue in degrees. The constructor converts to an uppercase 6-digit sRGB hex (no alpha) via the standard OKLCH-to-sRGB conversion, channel-clamped into gamut. Named arguments or a non-three-argument form yield no conversion (the surrounding generator then falls back to reading a positional hex or string).

A standalone `oklch(...)` on the right of an assignment (no `color()` wrapper) is a single color primitive: `tokens[name] = hex`, with no ramp and no seed.

*Worked example (validated).* `accent: oklch(63.7%, 0.237, 25.331)` records `tokens["accent"]` as a `#RRGGBB` string and leaves `seeds` empty.

### 8.4 Bare literals and aliases

A bare hex, number, or string on the right of an assignment is a single-value primitive token (`brand.300: #66B2FF`, `opacity.disabled: 0.38`, `font.family: Inter`). A bare reference path is an alias (`text.error: red.500`), resolved at build time ([12](#12-aliases-and-reference-resolution)). Neither form registers a seed.

*Worked examples (validated).* `brand.300: #66B2FF` leaves `seeds` empty and records `tokens["brand.300"] == "#66B2FF"`. `text.error: red.500` records `tokens["text.error"] == "red.500"`.

### 8.5 Stop overrides

An individual generated stop is overridden in one of two ways:

- A **trailing override block** on a ramp-generating seed: `brand: scale(#1E40AF) { 300: #4B68C8 }` records the seed and `tokens["brand.300"] = "#4B68C8"`. The override block attaches only when the value is a scalar or call, not a record or list ([5.5](#55-assignment-and-override-block)).
- A **separate stop assignment**: `primary.300: #4B68C8`. This is the preferred modern form.

Because explicit tokens are applied after the catalog and ramp generators ([15.4](#154-double-generation)), an explicit stop override shadows the generated stop of the same key.

*Worked example (validated).* Given `primary: color(#0080FF)` followed by `primary.300: #4B68C8`, the resolved `primary.300` is the overriding color `#4B68C8`, not the generated ramp stop.

### 8.6 Legacy generators

These forms are recognized for backward compatibility. New authoring SHOULD prefer `color` and `number`.

- `scale(inner)` wraps a single seed: a hex, number, string, or bareword becomes a color/number seed; an `oklch(...)` inner is converted to a hex seed; a reference, record, or list warns. This is the historical color-scale wrapper and is the form that supports a trailing override block.
- `linear(base: N)` records `seeds[name] = N`; a `count:` argument warns as unsupported.
- `explicit({ stop: value, ... })` emits each `name.stop` scalar directly into `tokens`; the bare `name` is not a seed.
- `modular(...)` warns as unsupported; only a `base:` argument becomes a seed.

*Worked examples (validated).* `brand: scale(#1E40AF)` records `seeds["brand"] == "#1E40AF"`. `spacing: scale(4)` records `seeds["spacing"] == "4"`. `gap: linear(base: 8)` records `seeds["gap"] == "8"`. `radius: explicit({ none: 0, sm: 4, md: 8 })` records `tokens["radius.none"] == 0`, `tokens["radius.sm"] == 4`, `tokens["radius.md"] == 8`.

---

## 9. Semantic generators

A *semantic* wraps exactly one inner primitive generator in a fixed role vocabulary and carries mode behavior. There are three semantic generators. Each maps its roles onto the underlying primitive stops and, unless opted out, attaches a default transform program ([10](#10-mode-transforms)).

| Generator | Role vocabulary | Applies to |
|---|---|---|
| `boldness(...)` | `hint faint subtle soft mid firm bold strong intense` (9, center `mid`) | color tones, `font.weight`, `stroke.width`, custom numeric semantics |
| `tshirt(...)` | `xs sm md lg xl 2xl 3xl ... 20xl` (24), plus named boundary stops | `spacing`, `radius`, `font.size` |
| `looseness(...)` | `none tight snug normal relaxed loose` (6) | `font.lineHeight`, `font.letterSpacing` |

```
SemanticGenerator = 'boldness' '(' Inner BoundaryArg* TransformsArg? ')'
                  | 'tshirt'   '(' Inner BoundaryArg* TransformsArg? ')'
                  | 'looseness' '(' Inner BoundaryArg* TransformsArg? ')'
Inner             = ColorGenerator | NumberGenerator
BoundaryArg       = ( 'min' | 'max' ) ':' Record
TransformsArg     = 'transforms' ':' ( 'none' | Record )
```

The inner generator MUST be `color(...)` or `number(...)` as the specific semantic allows (`tshirt` and `looseness` require `number`). A wrong inner shape is a warning and the token is not produced.

### 9.1 boldness

`boldness(inner)` produces the nine boldness roles.

- **`boldness(color(...))`.** The color seed is registered ([8.1](#81-the-color-generator)), then each of the nine roles is emitted. A role's default stop is `name.<stop>` where the stop is taken from the fixed role-to-stop map: `hint -> 50`, `faint -> 100`, `subtle -> 200`, `soft -> 300`, `mid -> 500`, `firm -> 600`, `bold -> 700`, `strong -> 800`, `intense -> 900`. Stops `.400` and `.950` are not mapped to any role and remain available as primitives. If a transform program applies, the role token is a map `{ $default: "name.stop", $transforms: { stops, baseIndex, program } }` where `stops` is the **9 role stops in role order** (`name.50 name.100 name.200 name.300 name.500 name.600 name.700 name.800 name.900`, not the full 11-stop ramp, and never the two non-role primitives `.400`/`.950`), `baseIndex` is the role's position within that 9-element role list, and `program` is the transform program ([10](#10-mode-transforms)). Building the mirror over the 9-stop role band is what makes `mirror` swap roles symmetrically across `mid`; the two non-role stops stay power-user primitives. If no transform program applies, the role token is the bare `"name.stop"` string. The bare `name` alias equals `.500` equals the `mid` role.

- **`boldness(number(...))`.** The number scale is materialized ([8.2](#82-the-number-generator)), then role *i* maps positionally to `name.<i+1>` over the 1-based stops (`hint -> .1`, `mid -> .5`, `bold -> .7`, `intense -> .9` for a nine-element list). Transform program stops are `name.1 .. name.totalStops`, base index *i*.

The default transform programs are `boldness(color)`: `theme.dark: mirror`, `accessibility.high-contrast: outward(1)`; and `boldness(number)`: `accessibility.high-contrast: shift(+1)`. See [10.3](#103-default-transform-programs).

*Worked example (validated).* `primary: boldness(color(#0080FF))` (with `theme` and `accessibility` axes declared) records `seeds["primary"] == "#0080FF"` and emits nine role maps. `primary.mid.$default == "primary.500"`, and its transform program is exactly `[{modeKey: "theme.dark", op: "mirror"}, {modeKey: "accessibility.high-contrast", op: "outward", delta: 1}]`. The dynamic resolution of these roles across modes is specified in [10.5](#105-worked-example-validated) and [16](#16-transform-program-execution).

### 9.2 tshirt

`tshirt(number(...))` produces the t-shirt roles over a numeric scale. The role-to-stop mapping depends on the inner form:

- **List form** (`tshirt(number([...]))`): role *i* maps positionally to `name.<i+1>` (`xs -> .1`, `sm -> .2`, `md -> .3`, ...).
- **Generated-ramp or range form** (`tshirt(number(seed, count, ramp))` or a range): the mapping uses the fixed catalog table `xs -> 1, sm -> 2, md -> 4, lg -> 6, xl -> 8, 2xl -> 12, 3xl -> 16, 4xl -> 20, 5xl -> 24, 6xl -> 32`. A role whose stop index exceeds the materialized stop count is dropped.

The default transform program is `density.compact: shift(-1)`, `accessibility.large-text: shift(+1)`.

*Worked examples (validated).* `font.size: tshirt(number([12, 14, 16, 20, 24, 32, 36, 40, 48, 64, 80, 96, 128]))` gives `font.size.1 == 12`, `font.size.13 == 128`, and positional roles `xs -> font.size.1`, `md -> font.size.3`, `9xl -> font.size.13`. `spacing: tshirt(number(4, 32, linear()))` materializes `spacing.1 == 4 .. spacing.32 == 128` and picks roles by the catalog table (`xs -> .1`, `sm -> .2`, `md -> .4`, `lg -> .6`, `6xl -> .32`).

### 9.3 looseness

`looseness(number(...))` maps its six roles positionally: `none -> .1`, `tight -> .2`, `snug -> .3`, `normal -> .4`, `relaxed -> .5`, `loose -> .6`. Transform stops are `name.1 .. name.max`, base index *i*. The default transform program is `accessibility.large-text: shift(+1)`.

*Worked example (validated).* `font.lineHeight: looseness(number([1.0, 1.25, 1.375, 1.5, 1.625, 2.0]))` gives `font.lineHeight.1 == 1.0` and positional roles `none -> .1 .. loose -> .6`.

### 9.4 Boundary stops

Any semantic generator accepts optional `min:` and `max:` record arguments that add named boundary stops. Each field `name: value` of the record becomes a primitive stop `tokens["seedName.<name>"] = value`. Boundary stops are fixed values (not multipliers), conceptually sitting below the role band (`min`) or above it (`max`).

*Worked examples (validated).* `spacing: tshirt(number([...]), min: { none: 0 })` gives `spacing.none == 0`. `radius: tshirt(number([...]), min: { none: 0 }, max: { full: 9999 })` gives `radius.none == 0` and `radius.full == 9999`. `stroke.width: boldness(number([...]), min: { none: 0 }, max: { thick: 8, thicker: 16, thickest: 48 })` gives those four named stops. `visibility: boldness(number([...]), min: { invisible: 0 }, max: { opaque: 1 })` gives `visibility.invisible == 0` and `visibility.opaque == 1`. A `min`-only form omits the max stop. A boundary value MAY itself be a hex (`min: { transparent: #00000000 }`), preserved verbatim.

### 9.5 The no-bare-token rule for number scales

A color seed emits a bare `name` alias (`$primary` resolves to `.mid` via `.500`). A number-backed scale emits **no** bare `name` token: `$spacing` alone has no value, and a Blueprint reference to a bare number scale is an error whose diagnostic suggests a stop (`$spacing.md`). The exceptions are the identity and zero cases inherent to the fixed catalog scales (for example the opacity scale's `0` and `100` stops); these are ordinary named stops, not a bare alias. This rule is why interfaces are built from role names, not raw number-scale seeds.

### 9.6 Error cases (validated)

A non-generator inner (`foo: boldness(#FF0000)`), a wrong inner type (`foo: tshirt(color(#FF0000))`), an argument-less `number()`, and a wrong-arity `range` (`number(range(0), 5, linear())`) each produce a warning and no token; none throws.

---

## 10. Mode transforms

A semantic generator bakes in *mode behavior* by transforming a role's stop index per active mode. The mode behavior is expressed as a *transform program*: a list of entries, each pairing an axis-qualified mode key with an operator. The program is attached to each role token as a `$transforms` payload and executed at resolution time ([16](#16-transform-program-execution)).

### 10.1 The transforms argument

```
TransformsArg   = 'transforms' ':' ( 'none' | TransformRecord )
TransformRecord = '{' TransformField* '}'
TransformField  = RecordKey ':' TransformOp ','?     ; RecordKey MUST be axis-qualified
TransformOp     = 'mirror'
                | 'shift' '(' Integer ')'
                | 'outward' '(' Integer ')'          ; Integer MUST be non-negative
```

The `transforms:` argument resolves in three shapes:

- **Omitted**: the per-generator catalog default program applies ([10.3](#103-default-transform-programs)). A bare `boldness(color(...))` therefore gets `theme.dark: mirror` plus `high-contrast: outward(1)` automatically.
- **`transforms: none`**: an empty program. The scale is mode-immune (an explicit opt-out).
- **`transforms: { axis.value: OP, ... }`**: the given program **replaces** the defaults entirely (there is no merge). Every field key MUST be axis-qualified (contain a `.`); a bare key warns and is dropped. To keep a default entry while adding another, relist it.

Each operator field value is one of: the bareword `mirror`; `shift(N)` with a single integer `N`; or `outward(N)` with a single non-negative integer `N`. A non-integer shift argument, a negative or non-integer outward argument, or any other value warns and drops that entry.

The mode key inside a transform program is **axis-qualified** (`theme.dark`), in contrast to the value-only keys of semantic-block values ([11.2](#112-key-canonicalization)). This distinction is load-bearing at resolution time; see [16.4](#164-the-mode-key-duality).

### 10.2 The operators

Each operator acts on the role's integer stop index within the role token's `stops` list, then the result is clamped to a valid index.

- **`shift(N)`**: adds `N` to the index. Shifts **sum-compose** across simultaneously active modes (two active shifts of `-1` and `+1` net to `0`).
- **`mirror`**: reflects the index across the full stop list, `idx -> (len - 1) - idx`. The exact center is a fixed point.
- **`outward(N)`**: moves `N` stops away from the center toward the nearer terminal. An index below center decreases by `N`; an index above center increases by `N`; an index exactly at center is unchanged.

> **Note (mirror over the 9-stop role band).** For `boldness(color(...))`, the `stops` list of a role token is the **9 role stops in role order** (`name.50 name.100 name.200 name.300 name.500 name.600 name.700 name.800 name.900`), not the full 11-stop ramp, and `mirror` reflects across those 9 (`idx -> 8 - idx`). Consequently the dark mirror is exactly the role-band swap: `hint` (`.50`, index 0) mirrors to `intense` at `name.900`, `faint` (`.100`) to `strong` at `name.800`, `subtle` (`.200`) to `bold` at `name.700`, `soft` (`.300`) to `firm` at `name.600`, `mid` (`.500`) stays at `name.500`, `firm` (`.600`) to `soft` at `name.300`, `bold` (`.700`) to `subtle` at `name.200`, `strong` (`.800`) to `faint` at `name.100`, and `intense` (`.900`) to `hint` at `name.50`. This is the implemented behavior (validated by dynamic resolution): a dark-mode `primary.hint` resolves to the value of `primary.900` (intense's stop). Because the two non-role stops `.400` and `.950` are excluded from the mirror band, a mirrored role lands exactly on its symmetric role-band partner, never on a power-user primitive.

### 10.3 Default transform programs

| Generator form | Default program |
|---|---|
| `boldness(color(...))` | `theme.dark: mirror`, `accessibility.high-contrast: outward(1)` |
| `boldness(number(...))` | `accessibility.high-contrast: shift(+1)` |
| `tshirt(...)` | `density.compact: shift(-1)`, `accessibility.large-text: shift(+1)` |
| `looseness(...)` | `accessibility.large-text: shift(+1)` |

These are applied when `transforms:` is omitted. They are also the tables an author relists from when overriding.

### 10.4 Per-mode values as an alternative

An index transform is the right tool when the desired per-mode value is another stop of the same scale. When it is not (a role that should point at a different token in dark mode), author a per-mode value directly with a semantic-block key ([11](#11-mode-keyed-semantic-values)) or a mode-specific seed override. The two mechanisms compose: resolution consults mode-keyed values before running the transform program ([16.2](#162-resolution-order)).

### 10.5 Worked example (validated)

```
modes {
  theme:         [light, dark]
  density:       [comfortable, compact]
  accessibility: [standard, high-contrast, large-text]
}

primary: boldness(color(#0080FF))
spacing: tshirt(number([4, 8, 12, 16, 24, 32, 48, 64, 96, 128]), min: { none: 0 })
radius:  tshirt(number([2, 4, 6, 8, 12, 16]), transforms: none)
```

With the default programs, resolving `primary.hint` in dark mode yields the value of `primary.900` (mirror over the 9-stop role band lands `hint` on `intense`), while `primary.mid` in dark stays at `primary.500`. Resolving `spacing.md` under `density.compact` yields the value of the `spacing.sm` position (a `shift(-1)`: measured `spacing.md == 12`, its compact resolution `== 8`, the value at the `sm` stop), and under `accessibility.large-text` yields the `spacing.lg` position (measured `16`). `radius` opts out entirely and never changes with mode. All values in this paragraph are validated by dynamic resolution.

---

## 11. Mode-keyed semantic values

Beyond the generator transform programs, a token MAY carry explicit *per-mode values* authored as a semantic block ([5.6](#56-semantic-blocks)). This is how an alias points at different tokens in different modes.

### 11.1 The semantic block

A semantic block `name { $default: v0, key1: v1, ... }` builds a mode-keyed token map. `$default` is the base value used when no mode key matches. Each other field is a mode key (a single mode value or a combo) mapped to a value. A value that is an `oklch(...)` literal is eagerly lowered to a hex string so the themed resolution path (which consumes hex and references) can use it.

### 11.2 Key canonicalization

Semantic-block keys are normalized to **value-only** form: `$default` is kept; any other key is split on `|`, each part's axis prefix is dropped (only the value after the last `.` is kept), the parts are sorted alphabetically, and rejoined with `|`. Thus `theme.dark` normalizes to `dark`, and `density.compact, theme.dark` normalizes to `compact|dark`. This value-only form is the key under which the value is stored in the token's `modeValues` map. (Contrast the axis-qualified keys of transform programs; see [16.4](#164-the-mode-key-duality).)

### 11.3 Resolution precedence

At resolution time ([16.2](#162-resolution-order)), a themed token's value is chosen by this precedence, highest first:

1. **Combo keys**: among `modeValues` keys containing `|`, only those whose every joined value is in the active mode set are eligible; the longest matching combo wins.
2. **Single-axis keys**: iterating the active modes in caller order, the first `modeValues[mode]` hit wins.
3. **Transform program**: starting from the base index, each program entry whose mode key is active is applied ([16.3](#163-program-execution)).
4. **`$default`**: the `modeValues["$default"]` value.
5. **Base value**: the token's plain `value`.

A combo therefore outranks a single-axis branch, which outranks the transform program, which outranks `$default`, which outranks the base value.

### 11.4 Worked examples (validated)

```
modes { theme: [light, dark] density: [comfortable, compact] }
neutral: boldness(color(oklch(55.6%, 0, 0)))
red:     color(oklch(63.7%, 0.237, 25.331))

color.surface { $default: neutral.200, dark: neutral.900 }
text.error    { $default: red.500, theme.dark, density.compact: red.200 }
```

`color.surface` resolves to the value of `neutral.200` in light mode and `neutral.900` in dark mode (single-axis branch, validated). `text.error` carries a combo key that canonicalizes to `compact|dark`; with both `dark` and `compact` active it resolves to the value of `red.200` (combo pass, validated), and otherwise to `red.500`.

---

## 12. Aliases and reference resolution

A reference value (a bare dotted path such as `red.500`) or an alias bareword that names an existing token is stored as a string on the referring token and resolved at build time.

### 12.1 The alias decision

A bareword on the right of an assignment is stored as text. The alias-vs-string decision is made by the generator: a stored string that looks like a token reference (it matches an existing token key) becomes a reference token; otherwise it stays a string primitive (a font family name, for instance). The pre-pass that collects seed names does not itself disambiguate; the membership test at generation time is the real decision point.

### 12.2 Build-time flattening

The generator collects every reference token, runs cycle detection, and then copies each referent's resolved `value`, `modeValues`, and transform program into the referrer (the referrer adopts the referent's type). This *flattening* means the runtime never chases a reference chain: a fully built system resolves a reference in one lookup. The referrer keeps its own key and records the referenced key for provenance.

A reference to a **themed** token copies the whole mode-keyed shape, so an alias of a mode-flipping role flips correctly. This is validated: `color.error: red.500` resolves to the value of `red.500`, and a `color.surface` semantic block whose branches reference palette stops resolves each branch to the referenced stop's value ([11.4](#114-worked-examples-validated)).

### 12.3 Cycles and dangling references

A reference **cycle** (`a -> b -> a`) is a halt diagnostic (`Reference cycle: a -> b -> a`); the members are excluded from resolution. A **dangling** reference (a target the active system does not define) is a halt diagnostic (`Token X references Y, which the active design system does not define`), reported once per dangling key. Both are refused writes ([19.4](#194-generator-diagnostics)).

---

## 13. Composites

A *composite* bundles several atoms into one named token. Two composite kinds exist: typography records and shadow lists.

### 13.1 Typography

A typography composite is a record token under a `typography.` key.

```
typography.h1: { fontSize: font.size.3xl, fontWeight: font.weight.bold, lineHeight: 1.2, fontFamily: font.family.serif }
```

Fields are `fontFamily`, `fontWeight`, `fontSize`, `lineHeight`, and `letterSpacing`. A string field value may be a token reference (`font.size.6xl`), looked up in the token table with expected-type gating, or a literal. A partial record merges over the catalog default typography composite for that key, so an author may override just `fontSize` and inherit the rest; a field set to the sentinel `$replace: true` opts out of the merge and takes the record as-is.

*Worked example (validated).* `typography.h1: { fontSize: font.size.3xl, fontWeight: font.weight.bold, lineHeight: 1.2 }` resolves (against the seed template's scales) to a composite whose `fontSize == 36` and `fontWeight == 700`.

### 13.2 Shadow

A shadow composite is a list of layer constructors under a `shadow.` key.

```
shadow.md: [
  drop(y: 2, blur: 4, spread: -1, color: rgba(0, 0, 0, 0.06)),
  drop(y: 4, blur: 6, spread: -1, color: rgba(0, 0, 0, 0.10)),
]
```

Each layer is a `drop(...)` (a cast shadow) or `inset(...)` (an inner shadow) constructor taking named arguments `x`, `y`, `blur`, `spread`, and `color`. Positional arguments to `drop`/`inset` warn. A `color` argument is typically an `rgba(...)` or `rgb(...)` constructor, which resolves to a literal string `"rgba(r,g,b,a)"` (comma-joined). A shadow layer defaults its color to black if omitted. The resolved value is a list of shadow layers with fields `x, y, blur, spread, color`.

*Worked example (validated).* The `shadow.md` above resolves to a two-layer shadow.

> **Note (inset flattening).** The resolver records `inset(...)` with an `$inset: true` marker, but the shadow layer built at generation time has no inset field, so the inner-vs-drop distinction does not currently survive into the resolved token: both `drop(...)` and `inset(...)` produce the same shadow-layer shape. An author authoring an inner shadow SHOULD be aware the resolved catalog does not yet carry the inset flag.

### 13.3 Absolute colors in composites

Shadow and glow colors are absolute on purpose. A shadow stays dark on any surface, so its color is authored as a fixed stop (`color.shadow: neutral.950`) rather than a mode-flipping role, which would turn the shadow into a glowing halo in dark mode. This is a convention, not a language rule, but it is baked into the seed template ([Appendix C](#appendix-c-the-seed-template)).

---

## 14. Color ramp generation

`color(seed)` expands its seed into an 11-stop OKLCH ramp at generation time. This section specifies the ramp as a behavioral contract plus a normative set of conformance ramps. The expansion is shared by the runtime generator and the Blueprint `$var` preprocessor, so both expand a seed identically.

### 14.1 Stops

The eleven steps are `50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950`. The seed itself is the `.500` stop; steps below 500 are the light half, steps above 500 are the dark half.

### 14.2 The ramp contract

A conforming implementation MUST produce ramps with the following observable properties:

- **The seed IS `.500`, byte-exact.** `name.500` equals the seed's sRGB hex exactly; the seed survives the round trip through the ramp with zero drift.
- **Constant hue.** Every stop keeps the seed's OKLCH hue.
- **Perceptually even lightness.** Stop lightness is spaced in OKLCH (perceptual) lightness, anchored to the seed: the light half runs from near-white at `.50` down to the seed at `.500`, and the dark half runs from the seed toward a fixed dark anchor that is lifted off pure black, so the bottom stops stay distinguishable from each other and from black. The spacing rescales around the seed's own lightness, so a light seed compresses its light half and a dark seed compresses its dark half rather than clipping.
- **Every stop is in sRGB gamut.** Where the seed's chroma cannot be represented at a stop's lightness and the seed's hue, chroma is reduced to the largest in-gamut value rather than channel-clipped, so high-chroma warm seeds produce clean tints instead of muddy clipped substitutes.
- **Bright tints desaturate faster than dark shades.** Chroma falls off toward both ends of the ramp, asymmetrically: the light extreme ends closer to neutral than the dark extreme (matching Tailwind's convention of pastel highlights and hue-retaining dark variants).
- **Near-achromatic seeds get a lifted light half.** A seed whose OKLCH chroma is below `0.05` is treated as a neutral: its light-half stops are lifted lighter (airier chrome surfaces, the way Tailwind hand-tunes its grays). The dark half is unaffected by the threshold.

### 14.3 Emitted tokens

For each color seed the generator emits `name.50 .. name.950` (11 generated stops) and a bare `name` token equal to the seed color. Neutral (achromatic) seeds produce a grayscale ramp with the lifted light half.

### 14.4 Conformance ramps (normative)

A conforming implementation MUST reproduce the following ramps bit-exactly. The battery covers the four seed-template brand seeds, the achromatic template neutral, a very light seed, a very dark seed, a maximum-chroma warm seed that exercises the gamut clamp, and a pair of seeds just below and just above the `0.05` neutrality threshold (their seed chromas, as round-tripped from hex, are `0.0443` and `0.0552`). All values below are generated by the shipped implementation.

| Seed | `.50` | `.100` | `.200` | `.300` | `.400` | `.500` | `.600` | `.700` | `.800` | `.900` | `.950` |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `primary` `#0080FF` | `#F2F5FA` | `#E3ECF8` | `#C0D6F4` | `#98C0F5` | `#61A2F8` | `#0080FF` | `#005FC0` | `#004998` | `#003A7B` | `#00295B` | `#001B41` |
| `secondary` `#FF3377` | `#FBF3F4` | `#FAE7EA` | `#F8CAD1` | `#FAAAB8` | `#FC7C98` | `#FF3377` | `#C50054` | `#9A0040` | `#7B0032` | `#590022` | `#3D0015` |
| `tertiary` `#FF9900` | `#F9F4EF` | `#F8EDE4` | `#F5DEC9` | `#F5CDA8` | `#F8B675` | `#FF9900` | `#B86D00` | `#8C5200` | `#6C3E00` | `#492800` | `#2D1700` |
| `quaternary` `#FFDD00` | `#F8F5E9` | `#F8F4E0` | `#F6EFCC` | `#F7EBB1` | `#F9E582` | `#FFDD00` | `#B49C00` | `#867300` | `#645600` | `#403600` | `#231D00` |
| neutral `#737373` (from `oklch(55.6%, 0, 0)`) | `#F8F8F8` | `#F0F0F0` | `#E1E1E1` | `#C6C6C6` | `#A1A1A1` | `#737373` | `#575757` | `#454545` | `#373737` | `#292929` | `#1D1D1D` |
| very light `#EAF2FF` | `#F7F8FA` | `#F6F8FB` | `#F5F7FB` | `#F3F6FC` | `#EFF4FD` | `#EAF2FF` | `#8AAADF` | `#5B7CB3` | `#3F5C8B` | `#23385B` | `#101D32` |
| very dark `#101828` | `#F8F8F9` | `#E9EAED` | `#CACED7` | `#979FB0` | `#546077` | `#101828` | `#121A28` | `#141B28` | `#151B27` | `#171C27` | `#181D26` |
| max-chroma warm `#FF4400` | `#FBF3F1` | `#F9E8E3` | `#F7CDC2` | `#F9AF9B` | `#FC8364` | `#FF4400` | `#BD3000` | `#942300` | `#761A00` | `#551000` | `#3A0800` |
| just below threshold `#60768D` (chroma `0.0443`) | `#F8F8F9` | `#EFF1F2` | `#DEE1E6` | `#BFC7D0` | `#94A3B3` | `#60768D` | `#46596C` | `#374656` | `#2D3946` | `#212A33` | `#171E25` |
| just above threshold `#5B7693` (chroma `0.0552`) | `#F4F5F6` | `#E7EAED` | `#C8CFD7` | `#A9B5C3` | `#8295AB` | `#5B7693` | `#425971` | `#34465A` | `#2A3949` | `#1F2A36` | `#161E27` |

Note the threshold pair: the below-threshold seed's `.50` is `#F8F8F9` (the lifted neutral light half) while the above-threshold seed's `.50` is `#F4F5F6` (the standard chromatic light half). The very dark seed's lightness sits below the ramp's dark anchor, so its dark half compresses into a tight band above the seed rather than descending; the table is the normative record of that behavior.

### 14.5 Worked example (validated)

For `primary: color(#0080FF)`:

| Stop | Lightness | Hex |
|---|---|---|
| 50 | 0.969 | `#F2F5FA` |
| 100 | 0.940 | `#E3ECF8` |
| 200 | 0.869 | `#C0D6F4` |
| 300 | 0.798 | `#98C0F5` |
| 400 | 0.707 | `#61A2F8` |
| 500 | 0.615 | `#0080FF` (the seed, byte-exact) |
| 600 | 0.498 | `#005FC0` |
| 700 | 0.417 | `#004998` |
| 800 | 0.358 | `#003A7B` |
| 900 | 0.289 | `#00295B` |
| 950 | 0.229 | `#001B41` |

For the achromatic `neutral: boldness(color(oklch(55.6%, 0, 0)))`, the ramp is grayscale: `neutral.50` lightness `0.979` (`#F8F8F8`), `neutral.500` (`#737373`, the seed), `neutral.950` lightness `0.231` (`#1D1D1D`). All values validated.

---

## 15. Numeric scale generation

Alongside the resolver's explicit stop tokens ([8.2](#82-the-number-generator)), the generator runs a dedicated *catalog scale generator* per numeric family whenever a seed for that family is present. This section specifies each family and the interaction between the two emitters.

### 15.1 The catalog families

Each generator fires when the source has a seed for its family key.

- **Spacing** (`spacing`): `spacing.none = 0`, then `spacing.1 .. spacing.32 = base * i` (base defaults to 4). The catalog always produces 32 stops plus `none`, independent of any list the author supplied.
- **Radius** (`radius`): the fixed step table `none: 0, sm: 1, md: 2, lg: 4, xl: 6, 2xl: 8, 3xl: 12, full: 9999`; the `full` stop is a fixed value (not multiplied), the rest are `base * multiplier` (base defaults to 4).
- **Font size** (`font.size`): the fixed multiplier table `xs: 0.75, sm: 0.875, md: 1.0, base: 1.0, lg: 1.25, xl: 1.5, 2xl: 2.0, 3xl: 2.25, 4xl: 2.5, 5xl: 3.0, 6xl: 4.0, 7xl: 5.0, 8xl: 6.0, 9xl: 8.0`; values are `base * multiplier` (base defaults to 16). `base` is a back-compat alias equal to `md`.
- **Font family** (`font.family`): the `font.family` seed and every `font.family.*` seed are emitted verbatim as font-family tokens.
- **Opacity** (`opacity`): the fixed percentage table `0 .. 100` mapping to `0.0 .. 1.0`; the seed base is validated but unused (the stops are fixed).
- **Stroke width** (`stroke.width`): the fixed step table `0: 0, 0.5: 0.5, 1: 1, 2: 2, 4: 4, 8: 8`; the `0` stop is fixed, the rest are `base * multiplier` (base defaults to 1); plus a `hairline` alias equal to `base * 0.5`.

Font weight and line height have **no** dedicated catalog generator; they are emitted exclusively by `boldness(number(...))` and `looseness(number(...))` respectively.

### 15.2 Type inference for explicit tokens

An explicit token's type is inferred from its key prefix: `spacing.`, `radius.`, `font.size.`, `font.family`, `font.weight.`, `font.lineHeight.`/`lineHeight.`, `font.letterSpacing`, `stroke.width`, `typography.`, `shadow.` each map to their type; a key in `numericSeeds` (or with such a prefix) maps to the generic numeric type (`opacity`); a long list of color-ish prefixes maps to color; and any **unrecognized** key defaults to `color`.

> **Note (unknown key defaults to color).** A custom token whose key matches none of the known prefixes and is not a number-seed stop is treated as a color, and its value is hex-parsed. A non-color custom token with an unusual name is therefore liable to be dropped as an invalid color. Author custom numeric tokens under a `number(...)` seed (so they land in `numericSeeds`) or under a known prefix.

> **Note (opacity as the catch-all numeric type).** There is no distinct "generic number" token type; the `opacity` type doubles as the catch-all 0-to-1 numeric type. Custom numeric semantics like `visibility` are typed `opacity`. The `visibility.invisible == 0` and `visibility.opaque == 1` stops resolve through the opacity path (validated).

### 15.3 The empty-system promise

A source with no seeds generates no tokens. A comment-only `.ds` file produces none of `spacing`, `spacing.1`, `spacing.none`, `radius`, `radius.md`, `font.size`, `font.family`, `font.weight`, `font.lineHeight`, `opacity`, or `stroke.width`. There are no hidden defaults. This is an architectural promise and is validated.

### 15.4 Double generation

When a family such as `spacing`, `radius`, `font.size`, or `stroke.width` is declared via `tshirt(number(...))` or `boldness(number(...))`, two emitters both produce stops: (a) the dedicated catalog scale generator (because the family's seed is present) and (b) the resolver's explicit list/role/boundary tokens. Because the resolver's explicit tokens are applied **after** the catalog generators, an explicit token **overwrites** a colliding catalog key, while a non-colliding catalog key survives. The precedence is therefore: explicit resolver tokens win; catalog stops fill the gaps.

Concretely, for the seed-template `spacing: tshirt(number([4, 8, 12, 16, 24, 32, 48, 64, 96, 128]), min: { none: 0 })`: `spacing.1 .. spacing.10` are the list values (explicit wins), `spacing.11 .. spacing.32` remain the catalog `base * i` values (base 4), `spacing.none = 0`, and the roles `spacing.xs .. spacing.6xl` reference `spacing.<positional index>`. This is validated: `spacing.8 == 64` (the list value, not the catalog `base * 8 == 32`), and `radius.md == 6` (the tshirt role reference to the list's third stop, not the catalog named `radius.md == base * 2 == 8`).

### 15.5 Worked example (validated)

Resolving the seed template ([Appendix C](#appendix-c-the-seed-template)) end to end yields, among others: `spacing.8 == 64`, `radius.md == 6`, `font.size.md == 16`, `font.family == "Manrope"`, `typography.h1.fontSize == 36`, `typography.h1.fontWeight == 700`, `shadow.md` with 2 layers, `stroke.width.thick == 8`, `stroke.width.thickest == 48`, and every color seed emitting `.50`, `.500`, `.900`, `.950`. The whole template resolves with zero warnings.

---

## 16. Transform program execution

Runtime resolution queries a token for its value given the active mode set. This section specifies the query.

### 16.1 When a token is themed

A token is *themed* when it has a non-empty mode-value map or a transform program with at least one entry. A non-themed token always resolves to its base value regardless of mode.

### 16.2 Resolution order

Given an active mode set, a themed token resolves by:

1. If the active set is null or empty, return `modeValues["$default"]` if present, else the base value.
2. **Combo pass**: among `modeValues` keys containing `|`, consider only those whose every joined value is in the active set; return the value of the longest such combo, if any.
3. **Single-axis pass**: iterate the active modes in the caller's order; return the first `modeValues[mode]` that exists.
4. **Transform pass**: if a transform program is present, run it ([16.3](#163-program-execution)); if the resulting index differs from the base index, return the stop at that index.
5. Otherwise return `modeValues["$default"]` if present, else the base value.

A single-mode convenience resolution (`resolveForMode(m)`) returns `modeValues[m] ?? modeValues["$default"] ?? value`; there is no hardcoded `light` fallback.

### 16.3 Program execution

The transform program starts at the token's base index into its `stops` list. For each program entry whose mode key is in the active set, the operator is applied ([10.2](#102-the-operators)): `shift(N)` adds `N` (summing across active shifts), `mirror` sets `idx = (len - 1) - idx`, `outward(N)` moves `N` away from center `(len - 1) / 2`. The final index is clamped to `[0, len - 1]`. If it changed from the base index, the stop at that index is the result; otherwise the token falls through to `$default` or the base value.

At build time the program's `stops` references are resolved to concrete values (an unknown stop reference warns and aborts that program), and the base index is clamped into range. So at runtime the `stops` list already holds values, and execution is pure index arithmetic.

### 16.4 The mode-key duality

Semantic-block values are keyed by **value-only** mode names (`dark`, `compact|dark`), while transform-program entries are keyed by **axis-qualified** mode names (`theme.dark`). The single-axis pass ([16.2](#162-resolution-order) step 3) matches value-only keys; the transform pass ([16.3](#163-program-execution)) matches axis-qualified keys. For both passes to fire, the active mode set supplied at query time MUST contain both forms of an active mode (both `dark` and `theme.dark`). A conforming resolver caller therefore populates the active set with each active mode in both its value-only and axis-qualified forms. This duality is a consequence of the two authoring surfaces (value-only keys are natural to write in a semantic block; axis-qualified keys are required to disambiguate a transform across axes) and is validated by the resolution of `primary.hint` in dark and `spacing.md` under `density.compact`, both of which require the axis-qualified form in the active set.

### 16.5 Worked example (validated)

Given the [10.5](#105-worked-example-validated) system and an active set `{dark, theme.dark}`: `primary.hint` resolves through the transform pass (mirror over the 9-stop role band) to the value of `primary.900`, and `primary.mid` stays at `primary.500`. Given active set `{compact, density.compact, dark, theme.dark}`: `text.error` (from [11.4](#114-worked-examples-validated)) resolves through the combo pass to `red.200`.

---

## 17. The cascade

A design system is rarely one file. A canvas resolves against a chain of `.ds` files determined by folder structure and the requested brand. This section specifies the chain, the merge, and the pruning.

### 17.1 File layout

Per brand, a file `Styles/<name>.ds`; the project baseline is `Styles/default.ds`. Generated artifacts live under `Styles/.gen/<name>.gen.yaml` (gitignored). The directory name is `Styles`, the default file is `default.ds`, and a `.legacy` subdirectory archives pre-migration files.

### 17.2 Cascade resolution

Resolving a design system for a canvas proceeds as:

1. **Base cascade.** Walk from the repository root down to the canvas's folder, collecting each ancestor folder's `Styles/default.ds` in root-to-leaf order.
2. **Brand cascade.** If a brand is requested, collect its cascade. A bare brand name collects the full brand-file chain; a path form (`./x`, `../shared/x`, `/shared/x`) resolves to a single file. An empty brand cascade logs a warning and falls back to the base.
3. **Root pruning.** Prune each cascade at the leaf-most file whose `root: true` flag is set: walk from leaf toward root, find the first such file, and drop every ancestor above it (the boundary file itself is kept). This mirrors `.editorconfig` semantics.
4. **Inherits-none boundary.** Concatenate the pruned base cascade followed by the pruned brand cascade, then truncate the combined list at the leaf-most source declaring `inherits: none` (`inheritsBase == false`): everything earlier in the chain is dropped, so a standalone sheet sheds both the base ancestors and any parent-brand layers. Default inheriting sources (`inheritsBase == true`, the norm) leave the list untouched.
5. **Merge.** Merge the surviving sources in order, each source overriding the previous, skipping empty sources.
6. **Generate once.** Run the generator a single time on the merged source. The available modes are the merged source's mode set.

### 17.3 Source merge semantics

Merging two sources (the later wins) is field by field:

- `seeds`, `tokens`, `customModes` (union), `seedExpressions`: the child wins per key.
- `modeSeeds`: deep-merged; the child's per-mode entries override.
- `modeAxes`: unioned; a redeclared axis takes the child's values.
- `generatedKeys`: for each merged token, the key is kept generated if the winning side marked it generated; an explicit child override (present in the child's tokens but not its generated keys) drops the key from generated keys so the migrator emits it alongside the seed.
- `removals`: both sides carried forward, then the child's removals are applied to the merged result ([17.4](#174-unset-application)).
- `active`: merged per field (name: child wins; modes: unioned, child wins per axis).
- `isRoot`, `inheritsBase`: taken from the child (these are per-file declarations and do not merge).
- `numericSeeds`: unioned.

The merged mode set always includes `light` and `dark` even if not declared, plus every custom mode value and every mode-seed key.

### 17.4 Unset application

The child's removals are applied to the merged source. A removal path is matched by shape: the single-segment specials `active`/`hide`/`pin`; a `*.key` wildcard (drop `key` from every mode-keyed semantic); a mode prefix (`theme.dark` drops every `dark` branch and every dark mode-seed, recognized against the declared axes plus the built-in `theme` and `density` axes); a mode-specific entry removal; a flat token key; a seed key (which also drops its mode-seeds); and a composite field walk (`typography.h1.fontSize`). An unmatched removal path appends a warning and leaves the source intact.

*Worked examples (validated at the resolver level).* `unset { color.primary, *.dark }` records two removals: the single path `[color, primary]` and the wildcard `[*, dark]`. `unset { active }` clears the active block on merge; `unset { hide, pin }` removes the `$hide`/`$pin` lists.

### 17.5 Standalone brands (inherits: none)

`inherits: none` sets `inheritsBase = false`, declaring a brand standalone (it should not layer on the base).

> **Note (inherits: none at runtime).** The `inheritsBase` flag is honored everywhere: the delta computation, the dry-run and `.gen.yaml` generation paths (which return the standalone source as-is), **and the runtime cascade merge** ([17.2](#172-cascade-resolution) step 4). At render time the merge truncates at the leaf-most `inherits: none` source, so a standalone brand sheds the base and parent-brand layers live, exactly as the surfaced delta and the generated artifact already showed. An author relying on `inherits: none` for render-time isolation gets it.

### 17.6 Brand and mode switching

Persistent brand and mode selection mutate the file's `active { ... }` block (write source, regenerate the artifact, invalidate caches, notify). Transient preview sets an in-memory overlay with no file write; a folder's active design system folds each folder file's `active` (leaf wins) and overlays the preview. Caching is keyed by canvas path and brand; a change to a root `.ds` file invalidates everything below it.

### 17.7 Worked example (cascade)

A project `default.ds` sets `primary: boldness(color(#0080FF))`; a folder brand `corporate-blue.ds` sets `primary: boldness(color(#1E40AF))`. Resolving a canvas in that folder with brand `corporate-blue` yields `primary.mid == #1E40AF` (the child seed wins). If `corporate-blue.ds` additionally sets `root: true`, the ancestor base above the folder is pruned, but a sibling `default.ds` in the same folder still merges (the base cascade and the brand cascade are collected separately).

---

## 18. The resolved artifact

Serialization writes a resolved design system to a `.gen.yaml` file. The artifact is an export only: the runtime never reads it back ([3](#3-processing-model)).

### 18.1 Header and structure

The file opens with a generated header (`GENERATED - DO NOT EDIT. Edit the sibling .ds file; this regenerates automatically.`). Then, in order:

1. A warnings block (each generation warning as a comment) if any.
2. An `$active:` block (name plus modes) if a folder-active declaration exists (visibility only; it does not affect resolution).
3. A `PRIMITIVES` section: every non-themed token, grouped by type. Colors are sub-grouped per palette prefix with a Tailwind spectrum ordering and a "Semantic aliases" header for the chrome `color.*` group. Other categories are spacing, radius, typography scales (font size/family/weight/line height/letter spacing combined), opacity, stroke width, typography composites, shadows, and custom.
4. A `SEMANTICS` section: every themed token (non-empty mode values or a transform program), same category layout. Each themed token emits its mode branches with provenance comments, and a transform program emits `$default: value` plus `# transform <modeKey>: <op>` comment lines.
5. A footer `$hide:`/`$pin:` block mirroring the source, with inline `# hidden`/`# pinned` annotations.

### 18.2 Value formatting

A color formats as `#RRGGBB` (no alpha). A double formats as an integer when whole, else a decimal. A string with spaces or colons is quoted. Token sort within a group is numeric-suffix aware, so `accent.5` sorts before `accent.10`.

### 18.3 The legacy reader

A separate legacy reader parses single-file YAML `.styles` for the one-shot startup migration only ([21.2](#212-legacy-migration)). It is not used at runtime. It preprocesses unquoted `#hex` (which YAML would read as a comment) by quoting it, handles `$modes`/`$active`, and validates axis legality.

---

## 19. Diagnostics

Two diagnostic currencies exist: lexer/parser `ParseError`s (a message plus a span, always a hard authoring error) and generator `DsDiagnostic`s (a kind of `halt` or `warn`, a message, an optional span, and an optional fix hint). The resolver produces neither; it produces plain warning strings that ride into the generator's warning stream.

### 19.1 Severity model

- **Halt**: a hard error that refuses an authoring write. A halt is surfaced with its line and message (and fix hint) and blocks the `.gen.yaml` write for that authoring action, so no half-applied system lands.
- **Warn**: a soft diagnostic that rides along and does not block. Warnings are surfaced (in the artifact, in agent context) but do not stop generation.

### 19.2 Lexer and parser errors

All lexical and syntactic errors are hard (they surface as call-halting parse errors when a source is authored), but the lexer and parser both recover and continue so a single source yields all its errors at once. The inventory: unexpected character; hex length not 3/6/8; unterminated string; identifier starting with `-`; unterminated block comment (lexer); expected statement; expected `:`/`{` after a path; unknown active field; expected `{` after `modes:`; missing `,` between call arguments (one per gap); expected value/identifier/path/stop-key; expected closing bracket/brace/paren (parser).

### 19.3 Resolver warnings

The resolver never halts. Every problem is a warning string: a generator-shape mismatch (wrong argument count or shape for any generator); an unknown generator function; an unsupported legacy form (`modular(...)`, `linear(count:)`); a non-numeric list or range element; a malformed or non-axis-qualified transform key; an unknown transform operator; a non-record `min`/`max`; positional `drop`/`inset` arguments; an OKLCH lightness clamp; an unset inside a mode context; an unsupported `active.modes` value type; and a malformed `root`/`inherits` flag.

### 19.4 Generator diagnostics

**Halts** (block the write): a mode-branch key naming no declared mode; a deferred themed branch still unresolvable after reference resolution (with a span and a type-specific fix); a reference cycle; a dangling reference.

**Warnings** (ride along): a dropped color seed (not a valid hex); a non-number numeric seed; a dropped scalar or themed token (invalid value for its type); a malformed `$transforms` payload or one referencing an unknown stop; plus every inherited resolver warning.

*Worked example (validated).* `color.x { $default: red.500, drk: red.300 }` under `modes { theme: [light, dark] }` halts with `Token "color.x": mode-branch key "drk" names no declared mode, so that branch never applies`. `foo: boldness(#FF0000)` warns (non-generator inner) and produces no token.

### 19.5 The authoring write gate

The agent-facing authoring directive (`ds_file("name") <body>`) enforces the halt gate. It runs a *dry-run generation* over the reconstructed committed cascade (base cascade plus ancestor brand siblings plus the existing self plus the candidate body, pruned and merged) and refuses the write if any halt diagnostics are present, rendering the halt lines top to bottom. It also refuses a body with parse errors and a body containing an `active { ... }` block ([6.4](#64-the-active-block)). Warnings are surfaced but do not block. The write itself is a surgical text patch, so existing comments and ordering are preserved ([4.8](#48-comments-and-trivia)). A forgiveness pass treats `--` as a line comment (warning that `//` is canonical) and auto-quotes a multi-word `font.family` value (warning). The brand name MUST match `^[a-z0-9][a-z0-9-]*$` or the call halts. A dry-run halt refuses the write so no half-applied system lands; this is the concrete realization of the halt severity.

---

## 20. Conformance requirements

This section collects the normative obligations on a conforming implementation.

### 20.1 Lexer

A conforming lexer MUST NOT throw on any input. On malformed input it MUST append a `ParseError` and, where a token position is involved, emit an `invalid` token and continue. It MUST classify tokens exactly as [4.2](#42-token-kinds) specifies, including: digit-leading words containing a letter as identifiers; `$`-led words as identifiers; a trailing `%` folded into a `decimal`; hex of exactly 3, 4, 6, or 8 digits accepted and all other counts rejected; strings without escapes that cannot span lines; and comments discarded as trivia.

### 20.2 Parser

A conforming parser MUST NOT throw on any input. It MUST recover at statement and field boundaries and continue, returning the current token on a mismatch rather than consuming it blindly. It MUST emit exactly one diagnostic per missing comma between space-separated call arguments and MUST NOT cascade. It MUST produce the AST shapes of [5](#5-syntactic-grammar) and [7](#7-values), canonicalizing combo keys as specified in [5.6](#56-semantic-blocks).

### 20.3 Resolver and generator

A conforming resolver MUST NOT halt; it MUST record every problem as a warning and produce a design system source with the fields of [6](#6-statements). A conforming generator MUST implement the OKLCH ramp of [14](#14-color-ramp-generation) exactly (so `.500 == seed` is an exact invariant and every stop stays in sRGB gamut), the numeric families of [15](#15-numeric-scale-generation), the double-generation precedence of [15.4](#154-double-generation) (explicit tokens shadow catalog stops), and the resolution order of [16](#16-transform-program-execution). It MUST resolve references at build time and MUST emit halt diagnostics for cycles and dangling references. It MUST NOT make runtime resolution depend on the `.gen.yaml` artifact.

### 20.4 Round-trip (formatter and migrator)

A conforming round trip has the property `format(parse(format(ast))) == format(ast)` (formatting is idempotent). The formatter's canonical layout is one statement per line at top level with deterministic spacing inside records and lists; comments and source whitespace are not preserved through the formatter ([4.8](#48-comments-and-trivia)). Combo keys are expanded from `a|b` back to `a, b` for readability. By default the formatter drops `unset { }` blocks (the removals were already applied to the source state); a `preserveUnsets` mode re-emits them for surfacing a delta to an agent.

Fidelity of the source-to-text migration rests on two source fields: `seedExpressions` (the original right-hand-side AST per generator seed) lets the migrator re-emit `primary: boldness(color(#0080FF))` verbatim rather than expanding it to legacy stop blocks; and `generatedKeys` lets the migrator skip generator-derived tokens (they are re-derivable from the seed) while still emitting explicit per-key overrides alongside the seed. A conforming migrator MUST re-emit a seed expression verbatim when one is recorded.

### 20.5 Determinism

Generation MUST be deterministic: the same merged source MUST produce the same token catalog on every run. The OKLCH ramp, the numeric scales, and the transform programs are pure functions of the source. Token ordering in the artifact is numeric-suffix-aware and stable ([18.2](#182-value-formatting)).

---

## 21. Versioning and evolution

### 21.1 The evolution regime

The language evolves additively under a forgiving parser. New generators, roles, and named boundary stops are added without breaking existing files, and near-miss authoring forms are absorbed with a teaching diagnostic rather than rejected. Because the runtime rebuilds from source and the `.gen.yaml` artifact is disposable, regeneration is always safe: the `.ds` source is the single source of truth.

> **Note (the `ds:vN` version stamp).** A written `.ds` file carries a grammar-version stamp as its **first line**: `ds:v1` for the current grammar, the twin of Blueprint's `bp:v1`. The on-disk serializer (the `mutateSource`/`restoreSource` writers and the startup migration) stamps every non-empty file with the current version; an empty file (the "no design system" state) is left unstamped so it still reads as empty. The agent-facing delta path stays stamp-free.
>
> The stamp is a genuine top-level statement: `ds:v1` lexes as the identifier `ds`, a `:`, and the value `v1`, exactly like `hide:` / `pin:` / `root:` / `inherits:`. The resolver recognizes it and parks the version under the `$dsVersion` meta-key (the same `$`-prefixed channel as `$hide`/`$pin`), so it survives cascade merging, reaches the generator, and never materializes as a design token or leaks into `.gen.yaml`.
>
> Reads are forgiving (the forgiving-compiler doctrine):
> - **Unstamped** legacy `.ds` (no `ds:` line) → loads as **v1**, the current shape.
> - **Known** version (`ds:v1` up to the build's current version) → loads.
> - **Unknown / higher** version (e.g. `ds:v99`, or anything greater than the build's current version) → the generator emits a **loud halt `DsDiagnostic`** (naming the unsupported version, telling the user the file was written by a newer version of Brilliant and to update to open it), the same currency the `ds_file()` dry-run refuses writes on, never a silent misparse of newer syntax as v1.
> - A `ds:` line whose value is **not** a version stamp falls through to normal token handling, so a token literally named `ds` is never swallowed (never handcuff intentional usage).
>
> Version-stamping only: the current grammar is v1. A future breaking change to the grammar or its frozen resolution defaults is a **new** version (`ds:v2`) plus a forward migration, never an in-place reinterpretation of stored v1 files. The migration runner is intentionally deferred until the first real bump needs it: the stamp and the read-gate are what had to land first.

### 21.2 Legacy migration

A one-shot startup migration lifts a legacy single-file YAML `.styles` into the DSL layout: it converts `<folder>/.styles` into `<folder>/Styles/default.ds`, archiving the original under `Styles/.legacy/` for one release. The base file embeds the seed template first, then appends the user's migrated customizations (append-and-resolver-wins), so a customization overrides the corresponding template default. Brand siblings are kept sparse (no template embed). Pre-DSL catalog renames (`brand.5 -> brand.50`, and so on) are applied before parsing. The migration is idempotent; re-running it is a no-op, and a rollback restores the archived legacy layout.

---

## Appendix A. Collected grammar

This grammar is the union of the inline fragments and is consistent with them. Token kinds (`identifier`, `integer`, `decimal`, `string`, `hexColor`, and the punctuation kinds) are produced by the lexer ([4.2](#42-token-kinds)). Newlines are insignificant; commas are optional except where a rule shows a non-optional `,`.

```
File            = Statement* eof

Statement       = ModesDecl
                | ActiveBlock
                | HidePinDecl
                | UnsetBlock                 ; only when 'unset' is followed by '{'
                | PathStatement

ModesDecl       = 'modes' '{' ModeAxis* '}'
ModeAxis        = identifier ':' '[' ( identifier ( ',' identifier )* )? ']'

ActiveBlock     = 'active' '{' ActiveField* '}'
ActiveField     = ( 'brand' ':' Path | 'modes' ':' Record ) ','?

HidePinDecl     = ( 'hide' | 'pin' ) ':' NamePattern ( ',' NamePattern )*
NamePattern     = Path ( '.' '*' )?

UnsetBlock      = 'unset' '{' ( ',' | WildcardPath | Path )* '}'
WildcardPath    = '*' ( '.' ( identifier | integer ) )+

PathStatement   = Path ( AssignmentBody | SemanticBlockBody )

AssignmentBody  = ':' Value OverrideBlock?           ; OverrideBlock only if Value is not a Record/List
OverrideBlock   = '{' OverrideEntry* '}'
OverrideEntry   = ( identifier | integer | string ) ':' Value ','?

SemanticBlockBody = '{' SemanticField* '}'
SemanticField     = SemanticKey ( ',' SemanticKey )* ':' Value ','?
SemanticKey       = Path

Path            = ( identifier | integer ) ( '.' ( identifier | integer ) )*

Value           = Record | List | HexColor | String | Integer | Decimal
                | FunctionCall | Reference | Bareword

FunctionCall    = identifier '(' ( CallArg ( ',' CallArg )* ','? )? ')'
CallArg         = identifier ':' Value               ; named
                | Value                              ; positional
Reference       = identifier ( '.' ( identifier | integer ) )+
Bareword        = identifier

Record          = '{' RecordField* '}'
RecordField     = RecordKey ':' Value
RecordKey       = ( identifier | integer | string ) ( '.' ( identifier | integer ) )*
List            = '[' ( Value ( ',' Value )* ','? )? ']'

; Generators and constructors are FunctionCalls recognized by name:
;   color(seed) | number(...) | boldness(inner ...) | tshirt(inner ...) | looseness(inner ...)
;   oklch(L,C,H) | rgba(...) | rgb(...) | drop(...) | inset(...) | range(min,max)
;   linear(step: S)? | geometric(ratio: R)?
;   scale(inner) | linear(base: N) | explicit({...}) | modular(...)    ; legacy
; Semantic argument sugar recognized inside the semantic generators:
;   min: Record | max: Record | transforms: ( 'none' | Record )
```

---

## Appendix B. Catalog constants

Verbatim from the catalog. Tables marked *historical* are retained in the code for documentation but are not consulted at runtime; an implementation MUST NOT drive behavior from them.

### B.1 Color

- **Color steps** (11): `50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950`. The per-stop output is specified by contract and conformance table in [14](#14-color-ramp-generation).
- **Neutral chroma threshold**: `0.05`.
- **Dark-mode step map** (*historical*, superseded by the index-based mirror): `50: 950, 100: 900, 200: 800, 300: 700, 400: 600, 500: 500, 600: 400, 700: 300, 800: 200, 900: 100, 950: 50`.

### B.2 Numeric families

- **Spacing max multiplier**: `32`.
- **Radius steps**: `none: 0, sm: 1, md: 2, lg: 4, xl: 6, 2xl: 8, 3xl: 12, full: 9999`; fixed stops: `{full}`.
- **Font size multipliers**: `xs: 0.75, sm: 0.875, md: 1.0, base: 1.0, lg: 1.25, xl: 1.5, 2xl: 2.0, 3xl: 2.25, 4xl: 2.5, 5xl: 3.0, 6xl: 4.0, 7xl: 5.0, 8xl: 6.0, 9xl: 8.0` (`base` is a back-compat alias equal to `md`).
- **Opacity steps**: `0: 0.0, 5: 0.05, 10: 0.1, 20: 0.2, 30: 0.3, 40: 0.4, 50: 0.5, 60: 0.6, 70: 0.7, 80: 0.8, 90: 0.9, 100: 1.0`.
- **Stroke width steps**: `0: 0, 0.5: 0.5, 1: 1, 2: 2, 4: 4, 8: 8`; fixed stops: `{0}`; aliases: `hairline: 0.5`.
- **Default numeric seeds**: `spacing: 4, radius: 4, font.size: 16, font.weight: 400, font.lineHeight: 1.5, opacity: 1, stroke.width: 1`.

Font weight and line height have no dedicated catalog table; the historical `kFontWeightSteps` and `kLineHeightSteps` were removed.

### B.3 Semantic role vocabularies

- **Boldness roles** (9, center `mid`): `hint, faint, subtle, soft, mid, firm, bold, strong, intense`.
- **Boldness color stop by role**: `hint: 50, faint: 100, subtle: 200, soft: 300, mid: 500, firm: 600, bold: 700, strong: 800, intense: 900`. Stops `.400` and `.950` are not mapped to a role and remain primitives.
- **Boldness dark role swap**: `hint <-> intense, faint <-> strong, subtle <-> bold, soft <-> firm, mid <-> mid`. This is exactly what the default dark `mirror` now produces: the transform reflects across the 9-stop role band ([9.1](#91-boldness), [10.2](#102-transform-operations)), so each role lands on the partner listed here (the code realizes the swap via the index mirror rather than by consulting this table).
- **T-shirt roles** (24): `xs, sm, md, lg, xl, 2xl, 3xl, 4xl, 5xl, 6xl, 7xl, 8xl, 9xl, 10xl, 11xl, 12xl, 13xl, 14xl, 15xl, 16xl, 17xl, 18xl, 19xl, 20xl`.
- **T-shirt index by role for a linear scale** (generated-ramp/range form only): `xs: 1, sm: 2, md: 4, lg: 6, xl: 8, 2xl: 12, 3xl: 16, 4xl: 20, 5xl: 24, 6xl: 32`.
- **Looseness roles** (6): `none, tight, snug, normal, relaxed, loose`.

### B.4 Default transform tables

- `boldness(color)`: `theme.dark: mirror`, `accessibility.high-contrast: outward(1)`.
- `boldness(number)`: `accessibility.high-contrast: shift(+1)`.
- `tshirt`: `density.compact: shift(-1)`, `accessibility.large-text: shift(+1)`.
- `looseness`: `accessibility.large-text: shift(+1)`.

The historical `kTshirtModeShift` and `kLoosenessModeShift` tables are retained for documentation only and superseded by these transform programs.

### B.5 Number-scale example stops

The suggested stop a Blueprint reference falls back to for a bare number-scale reference: `spacing: spacing.md, radius: radius.lg, font.size: font.size.lg, font.weight: font.weight.bold, font.lineHeight: font.lineHeight.normal, font.letterSpacing: font.letterSpacing.normal, stroke.width: stroke.width.md, visibility: visibility.subtle`.

---

## Appendix C. The seed template

The seed template is the canonical worked example. It is written verbatim into `<root>/Styles/default.ds` on a fresh project and is the baseline that brand deltas render against. It parses (more than 50 statements), resolves with zero warnings, and generates the built-in Brilliant design system. It exercises nearly every construct in the language: three mode axes; hex and OKLCH color seeds; an achromatic neutral; twenty-one Tailwind palettes; list-form and boundary-bearing t-shirt scales; a `transforms: none` opt-out; a `boldness(number)` scale with three max boundary stops; a custom numeric semantic (`visibility`); negative-number stop overrides; bareword and quoted font families; a `transforms:` override; role aliases; absolute shadow and glow colors; typography and shadow composites; and `hide`/`pin` picker hints.

The template is reproduced below verbatim. Its first line is the `ds:v1` grammar-version stamp ([21.1](#211-the-evolution-regime)).

```
ds:v1
// The default design system, carrying Brilliant's own visual identity.
// Every token below is something your designs reference, so one edit
// here ripples through the whole project.

modes {
  theme:         [light, dark]
  density:       [comfortable, compact]
  accessibility: [standard, high-contrast, large-text]
}

// The four colors of the Brilliant mark: a confident blue lead, with
// magenta, orange, and yellow in support.
primary:    boldness(color(#0080FF))
secondary:  boldness(color(#FF3377))
tertiary:   boldness(color(#FF9900))
quaternary: boldness(color(#FFDD00))

// Neutral is the workhorse: black, white, and greys carry nearly every
// surface, border, and label.
neutral:    boldness(color(oklch(55.6%, 0,     0)))

// Tailwind's palettes, kept here as utility accents.
red:      boldness(color(oklch(63.7%, 0.237, 25.331)))
orange:   boldness(color(oklch(70.5%, 0.213, 47.604)))
amber:    boldness(color(oklch(76.9%, 0.188, 70.08)))
yellow:   boldness(color(oklch(79.5%, 0.184, 86.047)))
lime:     boldness(color(oklch(76.8%, 0.233, 130.85)))
green:    boldness(color(oklch(72.3%, 0.219, 149.579)))
emerald:  boldness(color(oklch(69.6%, 0.17,  162.48)))
teal:     boldness(color(oklch(70.4%, 0.14,  182.503)))
cyan:     boldness(color(oklch(71.5%, 0.143, 215.221)))
sky:      boldness(color(oklch(68.5%, 0.169, 237.323)))
blue:     boldness(color(oklch(62.3%, 0.214, 259.815)))
indigo:   boldness(color(oklch(58.5%, 0.233, 277.117)))
violet:   boldness(color(oklch(60.6%, 0.25,  292.717)))
purple:   boldness(color(oklch(62.7%, 0.265, 303.9)))
fuchsia:  boldness(color(oklch(66.7%, 0.295, 322.15)))
pink:     boldness(color(oklch(65.6%, 0.241, 354.308)))
rose:     boldness(color(oklch(64.5%, 0.246, 16.439)))
slate:    boldness(color(oklch(55.4%, 0.046, 257.417)))
gray:     boldness(color(oklch(55.1%, 0.027, 264.364)))
zinc:     boldness(color(oklch(55.2%, 0.016, 285.938)))
stone:    boldness(color(oklch(55.3%, 0.013, 58.071)))

// Geometry.
spacing:      tshirt(number([4, 8, 12, 16, 24, 32, 48, 64, 96, 128]),
                     min: { none: 0 })
radius:       tshirt(number([2, 4, 6, 8, 12, 16, 20, 24, 32, 40, 48, 64, 96]),
                     min: { none: 0 }, max: { full: 9999 },
                     transforms: none)
stroke.width: boldness(number([0.5, 0.75, 1, 1.25, 1.5, 2, 2.5, 3, 4]),
                       min: { none: 0 }, max: { thick: 8, thicker: 16, thickest: 48 })
visibility:   boldness(number([0.05, 0.10, 0.20, 0.40, 0.50, 0.60, 0.80, 0.90, 0.95]),
                       min: { invisible: 0 }, max: { opaque: 1 })

// Negative spacing for overlap.
spacing.overlap.xs:  -2
spacing.overlap.sm:  -4
spacing.overlap.md:  -8
spacing.overlap.lg:  -16
spacing.overlap.xl:  -24
spacing.overlap.2xl: -32

// Type pairs two voices.
font.family:        Manrope
font.family.serif:  "Noto Serif"
font.family.mono:   monospace

// font.size opts out of the tshirt default compact shift.
font.size:          tshirt(number([12, 14, 16, 20, 24, 32, 36, 40, 48, 64, 80, 96, 128]),
                           transforms: { accessibility.large-text: shift(+1) })
font.weight:        boldness(number([100, 200, 300, 400, 500, 600, 700, 800, 900]))
font.lineHeight:    looseness(number([1.0, 1.25, 1.375, 1.5, 1.625, 2.0]))
font.letterSpacing: looseness(number([-0.05, -0.025, -0.0125, 0, 0.025, 0.1]))

// Semantic roles. Build interfaces from these, not from raw palette stops.
color.surface:                neutral.hint
color.surface.container:      neutral.faint
color.surface.container.high: neutral.subtle
color.on-surface:             neutral.strong

color.surface.hover:          neutral.faint
color.surface.pressed:        neutral.subtle
color.surface.selected:       primary.subtle
color.on-surface.selected:    primary.bold

color.outline:                neutral.soft
color.outline.variant:        neutral.subtle

color.text.primary:           neutral.bold
color.text.secondary:         neutral.firm
color.text.disabled:          neutral.soft
color.text.display:           primary.firm
color.text.display.alt:       secondary.firm

color.primary:                primary.mid
color.on-primary:             neutral.50
color.primary.container:      primary.subtle

color.secondary:              secondary.mid
color.on-secondary:           neutral.50
color.secondary.container:    secondary.subtle

color.tertiary:               tertiary.mid
color.on-tertiary:            neutral.50
color.tertiary.container:     tertiary.subtle

color.quaternary:             quaternary.mid
color.on-quaternary:          neutral.900
color.quaternary.container:   quaternary.subtle

color.success:                green.mid
color.on-success:             neutral.50
color.success.container:      green.subtle

color.error:                  red.mid
color.on-error:               neutral.50
color.error.container:        red.subtle

color.warning:                orange.mid
color.on-warning:             neutral.900
color.warning.container:      orange.subtle

color.info:                   blue.mid
color.on-info:                neutral.50
color.info.container:         blue.subtle

// shadow and glow take absolute colors on purpose.
color.shadow:                 neutral.950
color.glow:                   neutral.50

// Composites.
typography.display.lg: { fontSize: font.size.6xl, fontWeight: font.weight.bold, lineHeight: 1.1  }
typography.display.md: { fontSize: 52,            fontWeight: font.weight.bold, lineHeight: 1.1  }
typography.display.sm: { fontSize: 44,            fontWeight: font.weight.bold, lineHeight: 1.15 }

typography.h1: { fontSize: font.size.3xl, fontWeight: font.weight.bold, lineHeight: 1.2  }
typography.h2: { fontSize: 30,            fontWeight: font.weight.bold, lineHeight: 1.25 }
typography.h3: { fontSize: font.size.xl,  fontWeight: font.weight.firm, lineHeight: 1.3  }
typography.h4: { fontSize: font.size.lg,  fontWeight: font.weight.firm, lineHeight: 1.35 }

typography.body.lg: { fontSize: 18,             fontWeight: font.weight.soft, lineHeight: font.lineHeight.normal }
typography.body.md: { fontSize: font.size.md,   fontWeight: font.weight.soft, lineHeight: font.lineHeight.normal }
typography.body.sm: { fontSize: font.size.sm,   fontWeight: font.weight.soft, lineHeight: font.lineHeight.normal }

typography.button:  { fontSize: font.size.sm,   fontWeight: font.weight.firm, lineHeight: font.lineHeight.tight }
typography.input:   { fontSize: font.size.md,   fontWeight: font.weight.soft, lineHeight: 1.4 }
typography.label:   { fontSize: font.size.sm,   fontWeight: font.weight.mid,  lineHeight: 1.4 }
typography.caption: { fontSize: font.size.xs,   fontWeight: font.weight.soft, lineHeight: 1.4 }
typography.code:    { fontSize: font.size.sm,   fontWeight: font.weight.soft,
                      lineHeight: font.lineHeight.normal, fontFamily: font.family.mono }

typography.editorial.lg: { fontSize: font.size.6xl, fontWeight: font.weight.bold, lineHeight: 1.1,  fontFamily: font.family.serif }
typography.editorial.md: { fontSize: 52,            fontWeight: font.weight.bold, lineHeight: 1.1,  fontFamily: font.family.serif }
typography.editorial.sm: { fontSize: 44,            fontWeight: font.weight.bold, lineHeight: 1.15, fontFamily: font.family.serif }

shadow.2xs: [ drop(y: 1, blur: 0, color: rgba(0, 0, 0, 0.05)) ]
shadow.xs:  [ drop(y: 1, blur: 1, color: rgba(0, 0, 0, 0.05)) ]
shadow.sm:  [ drop(y: 1, blur: 2, color: rgba(0, 0, 0, 0.05)) ]

shadow.md: [
  drop(y: 2, blur: 4, spread: -1, color: rgba(0, 0, 0, 0.06)),
  drop(y: 4, blur: 6, spread: -1, color: rgba(0, 0, 0, 0.10)),
]

shadow.lg: [
  drop(y: 4,  blur: 6,  spread: -2, color: rgba(0, 0, 0, 0.05)),
  drop(y: 10, blur: 15, spread: -3, color: rgba(0, 0, 0, 0.10)),
]

shadow.xl: [
  drop(y: 10, blur: 10, spread: -5, color: rgba(0, 0, 0, 0.04)),
  drop(y: 20, blur: 25, spread: -5, color: rgba(0, 0, 0, 0.10)),
]

shadow.2xl:   [ drop(y: 25, blur: 50, spread: -12, color: rgba(0, 0, 0, 0.25)) ]
shadow.inner: [ drop(y: 2,  blur: 4,                color: rgba(0, 0, 0, 0.06)) ]

// In-app color-picker housekeeping.
hide: red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky,
      blue, indigo, violet, purple, fuchsia, pink, rose,
      slate, gray, zinc, stone
pin:  primary, secondary, tertiary, quaternary,
      color.primary, color.on-primary,
      color.secondary, color.on-secondary,
      color.tertiary, color.on-tertiary,
      color.quaternary, color.on-quaternary,
      color.surface, color.surface.container, color.on-surface,
      color.text.primary, color.text.secondary, color.text.disabled,
      color.outline, color.outline.variant
```

---

## Appendix D. Diagnostics index

### D.1 Lexical (hard)

| Condition | Message |
|---|---|
| unexpected character | `unexpected character "<c>"` |
| bad hex length | `hex color must be 3, 4, 6, or 8 hex digits (got N)` |
| unterminated string | `unterminated string` |
| identifier starting with `-` | `identifier cannot start with "-"` |
| unterminated block comment | `unterminated block comment` |

### D.2 Syntactic (hard, recoverable)

| Condition | Message (shape) |
|---|---|
| leading non-identifier | `expected statement, got <kind> "<text>"` |
| path not followed by `:`/`{` | `expected ':' or '{' after '<path>'` |
| unknown active field | `unknown active field "<name>" (expected 'brand' or 'modes')` |
| missing `{` after `modes:` | `expected '{' after 'modes:'` |
| missing comma between call args | `expected ',' between arguments` (one per gap) |
| bad path in unset | `expected dotted path inside 'unset { ... }'` |
| generic value/close-bracket | `expected value, got ...` / `expected closing '...'` |

### D.3 Resolver warnings (soft)

Generator-shape mismatch; unknown generator function; unsupported legacy form (`modular`, `linear(count:)`); non-numeric list/range element; malformed or non-axis-qualified transform key; unknown transform operator; non-record `min`/`max`; positional `drop`/`inset` argument; OKLCH lightness clamp; unset inside a mode context; unsupported `active.modes` value type; malformed `root`/`inherits` flag.

### D.4 Generator diagnostics

| Kind | Condition | Message (shape) |
|---|---|---|
| halt | mode-branch key names no declared mode | `Token "X": mode-branch key "k" names no declared mode, so that branch never applies` |
| halt | unresolvable themed branch after references | type-specific, with a span and a fix hint |
| halt | reference cycle | `Reference cycle: a -> b -> a` |
| halt | dangling reference | `Token X references Y, which the active design system does not define` |
| warn | dropped color seed (not valid hex) | dropped-with-warning |
| warn | non-number numeric seed | dropped-with-warning |
| warn | dropped scalar/themed token (bad value) | dropped-with-warning |
| warn | malformed `$transforms` / unknown stop ref | dropped-with-warning |
| warn | inherited resolver warnings | ride along |

---

## Related references

- [`reference/core.md`](reference/core.md), [`reference/authoring.md`](reference/authoring.md), [`reference/authoring-modes.md`](reference/authoring-modes.md): the informal authoring guides. They are the surface an author and the AI read; this document specifies the mechanics they illustrate.
- [Blueprint language specification](https://github.com/brilliant-hq/blueprint): the design language whose `$token` bindings resolve through a system authored here.
- [brilliant-hq/brilliant](https://github.com/brilliant-hq/brilliant): the agent and product documentation for using Brilliant end to end.
