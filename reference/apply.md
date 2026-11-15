---
assumes: design-systems/core
dsl: [set_default_design_system, set_session_design_system]
---
# Design System: Applying

## `ds()` picks brand and mode

A top-level frame auto-stamps the active DS; children inherit.

```
ds(editorial)              // switch brand
ds(, theme(dark))          // keep brand, go dark
ds(editorial, theme(dark)) // both
```

One row in a `ds` fence, `active { modes: { theme: dark } }`, switches the
whole canvas to dark (an element's own `ds()` mode still wins) and leaves the
design system file unchanged. A new axis switches from the fence after the one
that declares it. The fallback to the canvas on screen belongs to the in-app chat only, the person's own assistant: an MCP call whose fences bound no canvas has its `active { modes: { ... } }` row refused by name, and binds its canvas with a ```bl fence earlier in the same call.

A brand name with a comma, double-quote, or newline rides quoted, same as an
element name: `ds("My, Brand", theme(dark))`. Plain names stay unquoted.

A brand from a **library** the project depends on is named by the library's
handle: `ds(@acme/design-kit)` is the library's default look,
`ds(@acme/design-kit/marketing)` one of its named brands. It resolves against
the library's own tokens (see `reference/libraries/updating`).


## Keeping a design system across restarts

The `designSystem` argument on `edit` (`default`, `new`, `none`, or a brand you authored this session) lasts for your session. The `set_default_design_system` pin is saved in the user's app settings, so it survives restarts: every session `init` starts afterwards begins on it, the calling session switches too, and it can name any brand the project has. Which design system an `edit` call uses (first match wins):

1. The `designSystem` argument on this call. It then sticks for your later calls.
2. Your session's current choice: an earlier argument, the brand of a `ds` fence you wrote, a pin set from this session, or the pin your session started on at `init`.
3. Nothing chosen and nothing pinned: `none` (bare values allowed). A block with `$token` rows on a canvas whose elements and rows name only `ds(default)` then runs on `default`, which sticks, and the answer says so.

In the in-app chat the switch is `set_chat_state` (`{state: "designSystem", value, reason}`; `value` is `new`, `none`, `default`, or a brand the project has; `reason` is one line the user reads beside the switch). It switches the chat only and writes no pin; the picker follows it, and when to switch is in `design-systems/authoring`. `set_session_design_system` (`{value}`: `default`, a brand the project has, or `null` for None; optional `reason`) is its alias and switches your session only. In the in-app chat the `designSystem` argument on `edit` may also name any brand the project has; for an external agent it names `default`, `new`, `none`, or a brand you authored this session.

Pin values: `default`, a brand the project has, `<none>`, or `<new>`; a brand this project lacks counts as no pin. `null` clears the pin. After an app restart your session starts on the pin again, whether you keep your `sessionId` or call `init` again; a kept `sessionId` is told at its next `edit` that its refs and checkpoints are gone.

```
{ "canvasId": "...", "sessionId": "mcp:...", "commands": [{ "commandId": "set_default_design_system", "params": { "value": "default" } }] }
{ "sessionId": "mcp:...", "designSystem": "none", "text": "..." }
```
