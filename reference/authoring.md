---
dsl: [designSystem, brand, modes]
---
# Design System: Authoring & Modifying

A `ds` fence authors or extends a brand: its first row is
`file("Styles/<name>.ds")`, the `Styles/` folder beside the canvases that
use it, and the DSL body follows. It inherits the project `default` (the
catalog shown in your init context), so declare ONLY what differs, a
real brand is usually a few lines. At the closing fence the body merges
into that file statement by statement, before the fences after it, so a
later `bl` fence can `ds(name)` it. A new file is kebab-case; name the
visual direction (`fintech-warm`), not the task. An existing file keeps its
own spelling (`file("Styles/Acme.ds")`).

While the project's `Styles/default.ds` is still the default shipped with
Brilliant (no edit yet), your init context says so beside the catalog, and
`new` writes into `Styles/default.ds` itself instead of a second file: the
fence is `file("Styles/default.ds")`, each statement replaces its namesake
in the shipped default, and the session is then on `default`, now the
project's own system. Once `default.ds` has been edited, `new` writes its
own `Styles/<name>.ds` as above.

```ds
file("Styles/fintech-warm.ds")
modes { theme: [light, dark] }          // axes; first value = default

// PRIMITIVE (mode-independent): color(seed) → 11-stop OKLCH ramp .50…950,
// number([list]|seed,count,ramp) → numeric scale. SEMANTIC wraps one in a
// role vocab; mode behavior auto-applies per generator (see authoring-modes):
primary:         boldness(color(#1976D2))            // 9 roles hint…intense
brand:           boldness(color(#0080FF, dark: #66B2FF)) // own dark colour: dark ramp from #66B2FF (authoring-modes)
accent:          boldness(color(#E5484D), reach: bold) // reach: soft | bold moves how far the ramp's light and deep ends go (softer pastels and lifted darks, or near-white and near-black); .500 stays the seed; omit it for the standard ramp
spacing:         tshirt(number([4,8,16,24,32]), min: { none: 0 })  // xs…Nxl + named stop
font.lineHeight: looseness(number([1,1.25,1.5]))     // 6 steps none…loose
font.letterSpacing: looseness(number([-0.05,0,0.05,0.1])) // FRACTIONS of the font size, like line-height multipliers: 0.05 = 5% tracking (0.8px at 16). For pixels, put ls(N) on the element instead.
font.family:     "Noto Serif"                        // multi-word names need quotes; single-word fonts (Manrope, Inter) can stay bare

// Color roles need boldness(color(...)), a bare hex is one frozen value,
// no light/dark flip, no hover/container steps. Outside-catalog hue gets
// its own primitive first:
success:       boldness(color(#1AAB7A))
color.success: success.firm // correct, mode aware dynamic resolution
// WRONG: color.success: #bare-hex // incorrect, non-mode aware static resolution
```

After a `ds` fence, in any mode, the brand becomes the session's design
system: the token rows after it, in the same text, resolve against it and
unstamped frames auto-stamp it. The one exception: when the same `edit` call
passes `designSystem: "none"`, the fence writes the file and the session stays
on none. `inherits: none` makes a brand standalone (rare);
`root: true` stops the parent-folder cascade. Comments are preserved
into `.ds` and carry to future sessions, use them for design intent.
A `.ds` written any other way (the write or edit tools, a shell command, a
script) is checked like the fence and refused with the line and the first
error when it would not load, the file left as it was before the write.

In the in-app chat the design system follows what the user wants:

- Edit a design system (a `ds` fence) when the user asks for it; when their
  request clearly implies it, edit and say why in one line. No permission prompt.
- Switch this chat's design system when the current one does not fit:
  `execute_commands` `set_chat_state` with `{"state": "designSystem",
  "value": "<brand>", "reason": "<one line>"}` (`value` is `new`, `none`,
  `default`, or a design system this project has). It changes this chat
  only, never the user's default for new chats. The picker follows, and the
  chat shows `Switched to <brand>` with your reason. Elements already on the
  canvas keep their own design system.
- Ask first, in a plain chat message, when an edit is your own idea, the user
  said no to it earlier in this chat, or you are unsure what they want.

Each design-system edit shows the user one line,
`Edited the design system: 3 spacing tokens`, with Undo. The user's Revert of
your turn takes back the design-system edits and switches it made, and Redo
puts them back.
