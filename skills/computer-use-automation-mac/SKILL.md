---
name: computer-use-automation-mac
description: Use this macOS Computer Use skill whenever the user wants to open, switch to, or operate a desktop app or local GUI and complete a task in it—even when they never mention "computer use" explicitly. Trigger on requests such as opening Calculator/计算器/计算机 app to compute expressions, using another app, changing System Settings, handling file pickers, or operating a user-specified local browser session. This includes controls, menus, dialogs, sheets, and windows. Use mac_computer_use_tool plane="cu" and seed_computer_use_ax with the newest AX-tree observation; verify the outcome and request user takeover for login or other manual-only steps. Not for BU page automation.
---

# macOS AX computer use

Use this skill only with:

```text
mac_computer_use_tool(plane="cu", code="...", title="...")
```

Every tool cell is a fresh Python process:

```python
import seed_computer_use_ax as cu
```

## Authorization boundaries

- This skill is CU-only. Every `mac_computer_use_tool` call must use
  `plane="cu"`. Never import or call `bu`, and never select `plane="bu"` or
  `plane="mixed"`. If the user specifies a local or named browser, or an
  existing local browser session, operate that exact browser through `cu`. If
  AX cannot complete the task, report the limitation instead of falling back to
  `bu`.
- Screenshots are transient observations by default. `screenshot=True` permits
  visual targeting or verification; it does not authorize creating a file.
  Never call `Frame.save()`, write screenshot bytes, or otherwise persist a
  screenshot unless the user explicitly asks to save, export, or deliver one.
  A requested final screenshot must be visually verified after capture before
  delivery.

## Required user takeover

When a step requires the user to act personally—for example signing in, entering
credentials, completing a CAPTCHA or MFA challenge, granting consent, or taking
manual control of the current app or browser—call the session's user-takeover
tool (currently `interaction.request_action`). Treat a sign-in or permission dialog as such a gate whenever it blocks the
requested flow. A clearly optional prompt may be dismissed once only through an
explicit `Skip` or `Not now` action (or a localized equivalent); if it remains,
reappears, or offers no such action, request takeover immediately. Calling the
tool is required: do not merely ask in chat, keep retrying or closing the dialog,
try shortcuts or alternative entry points, switch apps, or build a workaround.

After calling the takeover tool, issue no further `cu` actions until control returns. Then assume the UI may have changed. Re-observe the exact app
before any further action and use only indexes from that new observation. If no
user-takeover tool is available, report the blocker and stop instead of claiming
that a plain-text instruction transferred control.

Do not drive the GUI with shell commands, subprocesses, AppleScript, the Dock,
or Spotlight. Use file and shell tools for standalone non-GUI work. A
`mac_computer_use_tool` cell must actually call `cu`; a file-only cell is
refused.

## Terms

- **Cell**: one `mac_computer_use_tool` invocation and one fresh Python process.
- **Observation**: the blocks emitted by `cu.get_app_state(...)`; the AX tree is
  `scope=ax_tree`, and reported new windows use `scope=window_changes`.
- **Transition**: a UI change that may add, remove, reorder, open, close, reload,
  scroll into, or virtualize AX rows.

## Core invariants

1. Only `cu.get_app_state(...)` reads and emits an AX tree. Action functions
   return small records and do not refresh the tree.
2. Read `[obs ...]` blocks directly. Only rows under `scope=ax_tree` provide
   element indexes; `scope=window_changes` records never do. Do not reprint or
   expand observation content.
3. Use element indexes only from the single newest observation for that exact
   app process. Never guess an index, reuse one after another observation, carry one
   across an app switch, or mix indexes from different trees.
4. After a transition, observe before the next action. The next target may not
   have existed in the tree used to plan the transition.
5. After `get_app_state()` runs, perform no further app-bound action in that
   cell. It should normally be the final `cu.*` call.

An index is the integer printed at the start of a tree row, not its ordinal
position. Indentation identifies parentage and may be the only distinction
between otherwise identical controls. Variables and row records do not survive
between cells; copy the exact app id and printed integer needed by the next cell.

This rule is safety-critical: a wrong numeric index does not necessarily fail; it
can operate a different real control. The server rejects an index absent from the
newest tree or a detectable row-identity mismatch, but that check does not make a
guessed or stale index safe. Any new observation may renumber the tree, even when
the app looks idle; bringing another window forward can be enough.

Do not over-observe. Read once, plan all non-structural actions against that one
tree, act, and observe again only at the boundary where the next decision needs
new UI state. If an app action occurs after `get_app_state()`, the attached state
is marked `superseded`; treat that as an observation placed in the middle of a
cell and correct the sequence.

## Operating cycle

- **COLD**: If an exact Bundle ID or full `.app` path is known, call
  `get_app_state()` directly. Use `list_apps()` only to resolve an unknown or
  ambiguous installation. Observing activates or launches the app.
- **READY**: Act on indexes from the newest observation. Batch actions only while
  earlier actions cannot restructure the later targets.
- **TRANSITION**: Perform the transition, optionally wait, then end with
  `get_app_state(app_id)`, preserving the same `pid` only while a process pin is
  active. Continue in the next cell using the new tree. If the
  action may create a window, observe its app next: the first successful
  `get_app_state()` for any app consumes the pending window-change result.
- **SWITCH**: Observe the other app. Re-observe an app before acting when
  switching back to it.

Treat an action as a transition when it opens or closes a menu, popover, sheet,
window, page, or tab; submits or reloads content; sorts, filters, or virtualizes a
list; or changes whether other controls exist. If uncertain, stop and observe.

## Public SDK contract

`seed_computer_use_ax` exports eleven functions. Arguments after `*` are
keyword-only. Every app-bound function requires a non-empty `app_id` and accepts
optional keyword-only `pid=None`; `list_apps()` and `wait()` are the only
exceptions.

| Function | Contract |
|---|---|
| `cu.list_apps()` | Return `list[Record]` entries with `app_id`, `bundle_id`, and optional `full_path`. |
| `cu.get_app_state(app_id, *, window_id=None, screenshot=False, pid=None)` | Activate or launch one app and read the full AX tree for the requested app/window scope. Optionally include window PNG frames. Return `AppState` and emit observation blocks. The AX tree is always a full snapshot, not a tree diff. |
| `cu.click(app_id, element_index, *, x=None, y=None, mouse_button="left", click_count=1, pid=None)` | Without coordinates, `element_index` is the target control row and its default action is invoked. With both `x` and `y`, `element_index` is the printed index of the window row that defines the `0-1000` coordinate frame; it is not the child under the point or `window_id`. |
| `cu.type_text(app_id, element_index, text, *, pid=None)` | Type text into the addressed row. |
| `cu.press_key(app_id, element_index, key, *, pid=None)` | Send one key or a `+`-joined shortcut such as `"cmd+s"` to the addressed row. |
| `cu.set_value(app_id, element_index, value, *, pid=None)` | Replace the value of a `settable` row without relying on focus. |
| `cu.scroll(app_id, element_index, direction, *, pages=1, pid=None)` | Scroll the addressed row by pages. |
| `cu.select_text(app_id, element_index, text, *, selection="text", prefix=None, suffix=None, pid=None)` | Select matching text or place the cursor beside it. `selection` is `"text"`, `"cursor_before"`, or `"cursor_after"`; `prefix` and `suffix` disambiguate repeats. |
| `cu.drag(app_id, element_index, from_x, from_y, to_x, to_y, *, pid=None)` | Drag between `0-1000` points. `element_index` is the printed index of the window row that defines the coordinate frame. |
| `cu.perform_secondary_action(app_id, element_index, action, *, pid=None)` | Invoke the required `action` exactly as advertised in the row's `Actions:` list. Never omit the third argument. |
| `cu.wait(seconds=3)` | Pause for a finite `int` or `float` in `0..180`; `bool` is rejected. |

Every element-targeting action returns a `Record` with `ok=True` on success;
server-specific fields may follow. `cu.wait()` returns `None`. A `Record`
supports dictionary and attribute access.

`app_id` accepts an exact Bundle ID, a full `.app` path, an app record from
`list_apps()`, or `AppState.app`. Records resolve `full_path`/`path` before
`app_id`/`bundle_id`, preserving an exact installation inside the same cell. If
duplicate installations require a human choice, print only concise candidate ids
and paths, then copy the chosen full path into the next cell.

`pid` is a keyword-only process selector that refines `app_id`; it never replaces
it. Omit it in ordinary workflows—do not start passing it merely because
`get_app_state()` returns one. Use it only when there is concrete evidence that
one app identity resolves to multiple running processes, such as a main app and
helpers sharing a Bundle ID. When pinning is required, use the actual
`AppState.app.pid` returned by the newest successful `get_app_state()` rather than
guessing. Every app-bound action and follow-up observation grounded in that
process's tree must carry the same `pid`. A PID is valid only for that process
lifetime; after exit or relaunch, discard it and establish a new observation.

`element_index` accepts a printed integer or a row record. Across cells, use the
integer because Python objects do not survive. The tree has no search API. Each
line contains its index, role, label/value, and optional `Actions:`; read
indentation to distinguish identical rows under different parents.

`AppState` exposes these public fields; `added_windows` is conditional:

| Field | Meaning |
|---|---|
| `app` | Exact app record with `app_id`, resolved `bundle_id`, optional `full_path`, `name`, and the actual observed `pid`, whether or not the caller supplied one. |
| `app_id`, `bundle_id`, `name` | Convenience properties forwarded from `app`. |
| `focused_element_index` | Focused row index, or `None`. |
| `view` | Always `"full"`. |
| `windows` | Window records containing only `id`, usable as `window_id`; that id is not the printed window-row action index. Window contents remain in the tree. |
| `elements` | Flattened rows addressed by printed labels. `state.elements[63]` is the row labelled `63`, not the 63rd list position. |
| `tree` | Complete rendered AX tree already emitted in the observation. Test it in code; do not print it. |
| `screenshots` | Requested screenshot `Frame` objects. Each `Frame` is PNG-like `bytes` with `index`, `window_id`, `mime_type`, and `save(target)`. Use `save(target)` only for an explicitly requested saved, exported, or delivered screenshot. |
| `added_windows` | Conditional SDK records for newly added windows. A nonempty result is summarized in `scope=window_changes`; use its `app_id` and `window_id` only to choose a subsequent scoped observation. |

Prefer `set_value()` over focus-and-type when a row is `settable`. Prefer a
semantic row action, including an advertised secondary action, over coordinates.

## Reading an observation

When one or more new windows are reported, the tool result places a separate
block immediately before the AX tree:

```text
[obs type=text scope=window_changes status=added count=1]
{"window_id":42,"app_id":"com.example.app","title":"Confirm","z_order":0}
[obs type=text scope=ax_tree]
```

The block appears only for a nonempty result; its absence makes no claim about
whether detection ran or was reliable. Its JSON lines are routing metadata, not
AX rows: they have no `element_index`, and a `window_id` must never be used as
one. `z_order:0` is frontmost. Do not print `state.added_windows`; any actionable
nonempty result is already emitted.

A rendered tree line has an index, the role as the tree words it—for example
`text entry area`, not `AXTextArea`—an optional label/value, and optional
`Actions:`. Action names such as `AXRaise` and `AXShowMenu` remain AX names.
Indentation is hierarchy:

```text
  0 standard window window_id:38 "Calculator" Actions: AXRaise
    4 group "CalculatorKeypadView"
      7 text "0"
      12 button "7"
    34 menu bar
      35 menu bar item(selectable) "Calculator"
```

In the first line, `0` is the printed window-row `element_index` used by
coordinate `click()` or `drag()`. `38` is the `window_id` used to scope
`get_app_state()` and match `Frame.window_id`. Never substitute one for the
other.

To press `button "7"`, use the printed `12`, not its position in a Python list.
Several rows may say `button "Run"`; select the one under the correct indented
parent. `state.elements[n]` means the row labelled `n`, not the nth row.

The tree is already emitted under `[obs type=text scope=ax_tree]`.
`print(state)` and `print(state.elements)` are folded by the SDK into one-line
summaries. Do not print `state.tree`, `state.windows`, individual rows, or an
iteration over `state.elements`; do not expand the state with `dict()` or JSON.
Those forms can reproduce the observation with a smaller output budget, cut it in
the middle, and crowd out useful results. Tool results remain in context for the
rest of the run, so every duplicated tree consumes context again on later turns.
It is fine to test `if "Radians" in state.tree:` or print a concise computed value.

## Worked patterns

All `com.example.*` ids, numeric element/window indexes, paths, and task text
below are illustrative. Replace them with values from the real task. Every
element index must come from the immediately preceding observation of the exact
app; never copy an example index into an operation.

### 1. Discover and observe an application

If the Bundle ID or full `.app` path is exact, call
`cu.get_app_state("com.apple.TextEdit")` directly. When installations are
ambiguous, filter `list_apps()` and pass a unique record in the same cell:

```python
import seed_computer_use_ax as cu

apps = cu.list_apps()
matches = [
    item for item in apps if item.get("bundle_id") == "com.example.Editor"
]

if len(matches) == 1:
    cu.get_app_state(matches[0])
else:
    print([
        (item.get("bundle_id"), item.get("full_path")) for item in matches
    ])
```

Zero matches means the app is unavailable under that identifier. With multiple
matches, choose one exact `full_path`; never guess. Copy that path into the next
cell. Repeat discovery only when the installed application set may change or an
explicit discovery error requests refresh.

### 2. Edit stable controls, then cross a transition

For a `settable` row, prefer `cu.set_value(app, 63, "replacement")`. When real
key input is required, target the same field for the shortcut and typing. Put the
first structural action last, then observe:

```python
import seed_computer_use_ax as cu

app = "com.example.Editor"
cu.press_key(app, 44, "cmd+a")
cu.type_text(app, 44, "replacement text")
cu.click(app, 80)       # Submit/Open: first structural action
cu.get_app_state(app)   # new rows are available to the next cell
```

All indexes come from one tree. If an earlier action may hide, insert, reorder,
reload, or virtualize a later target, split before it. Prefer a visible menu
command over guessing a shortcut; use `press_key()` only when its meaning is
known.

### 3. Open a menu, then complete its sheet

The application menu bar is part of its AX tree. First open the menu and observe
the revealed items:

```python
import seed_computer_use_ax as cu

app = "com.apple.TextEdit"
cu.click(app, 152)       # menu bar item from the preceding observation
cu.get_app_state(app)
```

Click the command from the open-menu observation, then observe the resulting UI:

```python
import seed_computer_use_ax as cu

app = "com.apple.TextEdit"
cu.click(app, 159)       # menu item from the open-menu observation
cu.get_app_state(app)
```

Complete the sheet with indexes from the sheet observation:

```python
import seed_computer_use_ax as cu

app = "com.apple.TextEdit"
cu.set_value(app, 91, "report.pdf")
cu.click(app, 95)        # Save/Confirm; closes the sheet
cu.wait(2)
cu.get_app_state(app)
```

Open menus and sheets persist across adjacent cells.

If a click on the underlying window is refused, inspect the tree for a modal
`sheet`; it blocks the window beneath it.

### 4. Scroll or reload a collection

Scrolling can change or virtualize rows, so observe before addressing newly
visible content:

```python
import seed_computer_use_ax as cu

app = "com.example.Browser"
cu.scroll(app, 132, "down", pages=1)
cu.get_app_state(app)
```

The same transition boundary applies after filtering, sorting, refreshing, or
switching pages or tabs.

### 5. Select text or invoke a secondary action

Use `prefix` and `suffix` when text repeats. The third argument `action` is
required and must match the target row's `Actions:` text exactly. If it opens
UI, make it the final action before observation:

```python
import seed_computer_use_ax as cu

app = "com.example.Editor"
cu.select_text(
    app,
    55,
    "target",
    selection="cursor_after",
    prefix="before ",
    suffix=" after",
)
cu.type_text(app, 55, " inserted")
cu.perform_secondary_action(
    app,
    103,
    action="AXShowMenu",  # advertised on row 103; the row stayed stable
)
cu.get_app_state(app)
```

### 6. Use a screenshot, coordinates, drag, or one window

A normal tree has no picture. Request one only for an image, chart, canvas,
unlabelled group, or another target the tree cannot name. If the whole tree is
too large, copy a window id from it and scope the new observation:

```python
import seed_computer_use_ax as cu

app = "com.apple.Preview"
cu.get_app_state(app, window_id=38, screenshot=True)
```

This read invalidates whole-app indexes. In the next cell, use a window row from
the scoped observation as the coordinate anchor:

```python
import seed_computer_use_ax as cu

app = "com.apple.Preview"
window_row_index = 0  # from: 0 standard window window_id:38 ...
cu.click(app, window_row_index, x=430, y=310)
# Alternative using endpoints derived from the same frame:
# cu.drag(app, window_row_index, 200, 300, 800, 300)
cu.get_app_state(app, window_id=38, screenshot=True)
```

Coordinates are fractions of the window in `0-1000`: `0,0` is its top-left,
`500,500` its centre, and `1000,1000` its bottom-right. They are not pixels and
the SDK reports no pixel coordinate to convert. Both `x` and `y` are required,
and `Frame.window_id` must match that window. Re-screenshot after resizing,
scrolling, toolbar changes, or canvas layout changes.

All four `drag()` coordinates also use `0-1000`.

## Failure recovery

After `status=timeout`, a partially executed failure, or condensed history,
re-observe the exact app before acting. Timed-out or partial actions may already
have run, so do not retry first.

| Error | What it means | Response |
|---|---|---|
| `CU_AX_ELEMENT_INVALID` | The index is absent from the newest tree or now identifies another row. | Observe again and use the newly printed index. Do not retry the old number. |
| `CU_AX_ARG_SHAPE` | The call was rejected before execution: invalid coordinate range/anchor, missing coordinate, unsupported key, or malformed argument. | Correct the arguments; nothing ran. |
| `CU_AX_ACTION_REFUSED` | The row cannot take that operation. A modal, `disabled`, non-frontmost app, or missing capability such as `settable` may be responsible. | Diagnose the tree and operation type. Never repeat unchanged. |
| `CU_AX_APP_NOT_RUNNING` | An action targeted an app that is not running. | Call `get_app_state(app_id)` to launch and observe it. |
| `CU_AX_APP_NOT_SURFACE` | The id addresses macOS chrome rather than an application surface. | Choose a real application; the Dock and similar chrome have no app tree. |
| `CU_AX_APP_GONE` | The app closed during the call. | Re-observe or verify the outcome first; the action may already have taken effect. |
| `CU_AX_NO_STATE` | No current observation exists for the target app. | Call `get_app_state()` before acting. |
| `CU_AX_ENVIRONMENT` | Accessibility permission is unavailable or the daemon cannot be reached. | Report and stop. Never retry or answer permission dialogs. |
| `CU_AX_CALL_FAILED` | An unclassified failure occurred. | Re-observe and check for side effects before deciding whether retry is safe. |

## Verify the outcome

Tool `status=ok` proves only that the cell returned, and action `ok=True` proves
only server acceptance; neither proves the user's requested outcome. Use the AX
tree to prove UI state and the file system to prove persistence.

Read the returned observation directly and confirm evidence specific to the
claimed outcome: for example, the expected value or selection, the intended
window or control, and the absence of an error, blocking sheet, or dialog. An
unrelated generic success label is not proof.

If the outcome includes a saved or exported artifact, verify the exact intended
path with a non-GUI file tool, then check the task-relevant type, modification,
openability, or content. Existence and nonzero size are only basic evidence; do
not create a file-only `mac_computer_use_tool` cell.

When the user explicitly requests a saved, exported, or delivered screenshot,
capture the verified target window only after all UI work is complete, save only
a `Frame` whose `window_id` matches that window, and inspect the emitted image
after the call returns. Confirm the requested app, window, and final UI state,
with no unintended foreground app, sheet, dialog, or obstruction. Do not deliver
an unverified image; if one corrective recapture also fails, report the problem.