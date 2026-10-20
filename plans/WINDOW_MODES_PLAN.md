# Window modes: titlebar buttons, Explode, Join, Spatial Canvas

Status: proposed

Builds on `POPOUT_WINDOWS_PLAN.md` (floats are OS windows, P6), `app/layout/SPLITS.md` "Floating a
view", and `EMBEDDED_EDITOR_PLAN.md` step 1 (`ViewHost`). Requested by the maintainer on 2026-10-19.

## What and why

Three titlebar buttons, each a Command, so each is also in the palette and the keymap:

- **Explode** (`fizzy.explodeWindows`): every region except the explorer becomes a child OS window,
  sized and placed exactly where it sat in the main window, then movable on its own. The main window
  shrinks (animated) to the explorer region and stays the parent of the child windows.
- **Join** (`fizzy.joinWindows`): the inverse. It pulls every child window back into its place in
  the main layout (rules below). It is its own Command, not an undo, and it works however the layout
  changed meanwhile.
- **Canvas** (`fizzy.toggleCanvas`): the main window becomes one zoomable, pannable viewport with
  every available surface drawn in it at natural size, fully interactive, laid out by a smart
  arrangement.

## The shapes rule (AGENTS.md "Shipped shapes")

- **Neither is a new shipped shape, and neither is a flag on a shape.** Both are runtime user
  state, like closing a place or floating a view. Shape code does not change.
- **Explode and Join are Layout operations:** `Floats.explode(layout, keep_keywords)` and
  `Floats.join(layout)`, built on existing floats (`app/layout/Floats.zig:257 add`, `:401 close`); a
  float is a place with a window. Fizzy's command passes the sidebar keywords as `keep`; Layout has
  no notion of "explorer".
- **Canvas is a Layout operation plus one seam:** `Layout.stage(@src(), opts)`, a container any
  shape may call around its places. Normally a plain box that costs nothing; in canvas mode it
  presents the places it contains on a transformed viewport. Fizzy's shape (`src/editor/layout.zig`,
  the `work` region) wraps its work area in it. No special case: example-app copies it unprivileged
  in the last step.
- Everything uses public Layout API (`State.regions` with `bounds`, `assign`, `order`, `closed`).

## Titlebar buttons (step 2)

- **Side:** opposite the window controls. macOS traffic lights are on the left, so buttons go on
  the right of the title strip (`titlebar_box`, `src/editor/Editor.zig:~4670`; collapses to 0 in full
  screen). Windows/Linux caption buttons are on the right (`src/editor/caption_buttons.zig`), so
  buttons go on the left (`titlebar_top_buffer + titlebar_height`, `Editor.zig:~4653`).
- **Hit rects:** `Editor.pushTitleStripRect` (`Editor.zig:4921`, as Sidebar does) and
  `registerFloatingTitleBarRects` on macOS, so clicks reach dvui.
- **Look:** the rail's `Chooser.icon` (`app/layout/Chooser.zig:401`), 20pt glyphs — selected
  `theme.color(.highlight,.fill)`, hovered `.window,.text`, rest `.window,.fill`. Tooltips as
  `Chooser.drawTooltip` (`Chooser.zig:~215-233`: `core.dialogs.tooltipOptions`,
  `tooltipBeginFor(350_000)`). Extract the colour rule and tooltip into a small shared helper rather
  than copying; new `src/editor/TitleModes.zig` draws the buttons.
- **Commands:** three entries in `src/editor/Keybinds.zig:~166` (`.bind = null`, lucide icons,
  `isEnabled`). Explode: enabled when any non-kept region has views; lit while any exploded float
  exists. Join: enabled when any OS-window float exists; never lit. Canvas: a toggle, lit while on.
- **No OS windows** (web, Wayland, `-Dnative-backend=sdl3`): Explode makes in-window floats at the
  same rects — same code path; the button stays enabled.
- **Full screen:** no titlebar buttons (they live with the window controls, which full screen
  removes); the commands still work. Where fizzy draws its own menu bar (Windows/Linux), the buttons
  sit at its right end. Explode leaves full screen first, then explodes.
- **Shortcuts:** defaults ⌃⌘E Explode, ⌃⌘J Join, ⌃⌘O Canvas (Ctrl+Alt+E/J/O on Windows/Linux). Step 2
  checks them against fizzy's keymap and every bundled and store plugin's chords, and picks another
  only for a real conflict.

## Explode

1. Regions come from `State.regions` (published each frame; `bounds` physical, `State.zig:888
   setPlaceMetrics`). Skip any matching `keep`. Plugin-declared inner regions (`host.region`, e.g.
   workbench panes) travel inside their surface, which is the unit that floats.
2. Per region, `Floats.add` a `Float N` holding the region's assignment, order and selection, with
   `home` = the place name.
3. **New:** a per-view **origin** (surface id → place name + order slot) in `State.homes`. Join
   needs it because `Float.home` is per float but views move between windows.
4. Mark the float `exploded = true`.
5. **Rect:** the window's content rect = the place's border box in screen space (main window origin
   + `bounds / scale`). Windows are `os_frame` and the float draws its header inside, so the frame
   grows upward by the header height to keep content exactly in place; clamped to the display
   (`float_rules.reachable`, `float_rules.zig:85`).
6. **Windows through a seam:** `Floats.Windows { ctx, vtable: available, open(screen_rect, title)
   -> ?handle }`. Today implemented by `Popout.popOut` (`src/editor/Popout.zig:256` →
   `viewports.open`); after `EMBEDDED_EDITOR_PLAN` step 1, by `ViewHost` — explode's child windows
   are `ViewHost`s with no SDL or viewport access of their own. If step 1 hasn't merged, a
   one-function adapter over `Popout`; never a second windowing path.
7. **Vacated places:** the split tree is untouched; vacated places are marked closed, keeping their
   sash (like `Region.close`).
8. **The main window shrinks to the explorer region** and stays the parent of the child windows. Its
   pre-explode frame is saved (`State.pre_explode_frame`) before the shrink. The new frame is the
   explorer's border box in screen space (plus the titlebar/chrome the window keeps), clamped to the
   display; the child rects in step 5 are computed from the pre-shrink geometry, so content does not
   move when the frame does. The shrink is animated together with the lift-out (next section).

## Join: what "its place" means

Join never replays a snapshot or pixel rects; it resolves each view against the layout as it is
now. The tree is never modified by Explode, so ratios and extents persist. Per OS-window float, per
view v, in order:

1. **Origin place exists:** the place in `State.homes[v]`; insert at its saved order slot (later
   arrivals append). Reselect v only if nothing there was selected since.
2. **Origin place closed** (by Explode or by the person): reopen it with its kept extent
   (`State.requestSlideOpen`), since a view is returning; collapsible / `hide_when_empty` rules
   still apply.
3. **Origin place gone** (a minted split removed, `forgetPlace`): `State.siblingLeaf`; if none,
   clear the assignment so keywords place it (`Layout.matching`); if no region accepts it,
   `Layout.unplaced()` for the picker.
4. **Views moved between child windows:** the move doesn't re-home a view; it goes to its own
   origin. A view the person dragged into a main-window place is already docked; Join leaves it.
5. **A sub-split inside a child window** (`Float 1/r1`): each view to its own origin, falling back
   to the float's `home`.
6. **Child windows opened meanwhile:** a float made after Explode has `home` set at float-out, so it
   joins the same way. One the person closed already sent its views home (existing close semantics).
7. **Main window frame:** Join grows the main window back to its pre-explode frame (animated), or, if
   the person resized or moved it since Explode, leaves it at its current frame. Either way it then
   reflows to that size via the tree's extents and collapse rules.
8. **A view whose surface vanished** (plugin unloaded or hidden): dropped silently.

**Floats that existed before Explode** are not regions, so Explode leaves them alone. On Join: an
in-window float stays a float; a pre-existing OS-window float merges back into the main window as
an in-window float (as `Popout` merge does), keeping its place and home. Only `exploded` floats and
floats opened after Explode dock home. ("Join and Dock All" is an open question.)

There is no global mode: Explode and Join read float flags, so partial states are fine — close some
child windows by hand, press Explode again, and the rest explode.

## Animation

Explode and Join are one continuous motion each, not a cut.

- **Explode:** each region lifts out of its place as a window while the main window's frame eases in
  to the explorer region, together.
- **Join:** the reverse: the windows fly to their slots and dissolve in as the main window grows
  back to its pre-explode frame (or stays, if the person resized it).
- **Driver:** `core.anim` easing, setting the OS window frames (`os_frame`, per window and the main
  window) each frame over about 250-350 ms. A pure std-only `explode_rules` function maps progress
  to frames, so it is tested without a window. Frame-setting goes through the `Windows` seam.
- **Reduced motion:** with the OS/fizzy reduced-motion setting, near-instant cross-fades instead.
- **Cost:** nothing costs a frame when not animating; the animation holds no timer or wake once done,
  and a new Explode/Join mid-flight retargets from the current frames.
- **Which step:** the rules (progress to frames, retargeting) are step 3. The driver and window
  wiring are built in step 4; if that pushes step 4 over 800 lines, they land as step 4b, a PR of its
  own right after it.

## Persistence

- `layout.zon` already saves floats (`src/backend/layout_file.zig:54 SavedRegion.Floating`). Add
  `exploded: bool`, a `homes` list, the main window's pre-explode frame, and the OS window rect `os: ?{x,y,w,h,display,maximized}` — the
  planned-but-unbuilt `POPOUT_WINDOWS_PLAN` "Next 2/3"; check no one has claimed it.
- Older fizzy reads past the new fields; views fall back to their keywords. A vanished display
  clamps through `float_rules.reachable`.
- Quit while exploded closes the children but saves the state; a restart comes back exploded.
- Quit while exploded saves the main window's pre-explode frame, so Join after a restart still grows
  it back.
- The recorded window arrangement is saved in the `canvas` block, so quitting in Canvas and
  relaunching can still leave Canvas to it.
- Canvas saves `{pan, zoom}` and per-card rect overrides in a `canvas` block.

## Spatial Canvas

- **Seam:** dvui's `ScaleWidget` scales children's `rectScale`; hit-testing compares physical
  pointer points to those scaled rects, so input maps back for ordinary widgets.
  `core/widgets/CanvasWidget.zig` has pan, zoom and fit (`scaler`, `scroll_info`, `fit_move`); the
  stage reuses its maths rather than forking it.
- **Presenting places:** `Layout.stage` records the regions the shape declares inside it; in canvas
  mode it positions each in an arranged rect under the scale/pan transform — deferred drawing, or an
  absolute `.rect` per region box under a neutral container. A spike at the start of step 6 decides;
  the Layout API stays the contract.
- **Every available surface:** placed regions, `Layout.unplaced()` and hidden-but-offerable
  surfaces, as cards.
- **Natural size:** each card starts at its place's current size (`State.placeSize`), or a default
  for unplaced surfaces; the person resizes cards by their corners. No SDK change; a
  `Surface.natural_size` hint is deferred (open question).
- **Arrangement:** a pure std-only `canvas_rules.zig`, seeded from the tree's geometry so the first
  canvas resembles the window; resolves overlaps deterministically (relax outward, shelf-pack the
  unplaced cards). Unit-tested on Linux.
- **LOD:** pure functions of `(view, zoom, budget)` with hysteresis, so a parked camera is stable
  (Atlas's contract: `fizzyedit/atlas` `docs/design/organic-lod.md`, `scale-architecture.md`).
  Off-view cards are culled (no draw call); tiny cards draw the cached snapshot (`State.snapshots`,
  `Layout.drawCaptured`, `Layout.zig:780`) at a low cadence; cards near natural size draw live.
  Nothing costs a frame when canvas is off.
- **Hazards:** popups and tooltips from a scaled card must stay unscaled; surfaces reading
  `dvui.windowRect()` for responsive rules misbehave; frost and backdrop blur fall back to opaque
  fills at scale ≠ 1; text re-rasterises per zoom (snap zoom stops, or LOD); edge auto-pan during
  drags; `ViewDrag` uses physical `bounds`, so drops work under translate and scale (no rotation).
- **With child windows:** Canvas and child windows are exclusive — entering Canvas runs Join first.
  Floats draw above the stage at unscaled size.
- **Leaving Canvas restores the arrangement from before it.** Entering Canvas records a *window
  arrangement*: floats with their OS rects, `exploded` flags, `homes`, and the main window frame. It
  then runs Join for display. Leaving restores exactly that, including an exploded state and any
  popped-out floats: windows fly back out to their previous OS rects and floats return, animated in
  the same motion language as Explode/Join (see Animation). It never re-explodes from scratch; it
  replays the record.
- **Explode while in Canvas** leaves Canvas straight into Explode (the cards become their windows),
  one motion, not restore-then-explode.
- **Edge cases while in Canvas:** a view closed is dropped from the arrangement; a view opened goes
  to its keyword place on leaving (`Layout.matching`, else `Layout.unplaced()`); a display removed
  meanwhile clamps each recorded rect through `float_rules.reachable`; a recorded float whose
  surface vanished is dropped.
- **Card titles:** every card shows its title (the view's title, plus the document name where there
  is one) at every zoom, so cards can always be told apart. When a card is too small for its live
  content and LOD drops it to a snapshot or nothing, a fixed-size legible title label stays (e.g.
  above the card), unscaled by the zoom and still culled when off-view.

## Steps

One PR each, ≤800 lines outside tests and docs. "Cloud" = Linux, tests only; "Mac" = needs macOS, a
GPU or the maintainer's eye.

1. **`src/backend`, `app/layout`: persist child-window rects.** `Floating.os`, restored at launch and
   clamped; rules in a std-only tested module. Cloud for rules, Mac for windows.
   Kickoff — Task: persist and restore OS window rects for popped-out floats. Plan:
   WINDOW_MODES_PLAN step 1. Builds on: POPOUT P6. Workspace: ../fizzy-os-rects. Branch:
   layout/os-window-persist. Notes: check POPOUT "Next 2/3" isn't claimed; older files read past
   the new field; `float_rules.reachable` for vanished displays.
2. **`src/editor`: titlebar buttons and three commands.** `TitleModes.zig`, the shared rail look and
   tooltip, Keybinds entries, per-OS side, hit rects, `isEnabled`; the commands call stubs. Cloud for
   command/enablement tests, Mac for the look.
   Kickoff — Task: three titlebar icon buttons (Explode, Join, Canvas) as Commands. Plan: step 2.
   Builds on: main. Workspace: ../fizzy-mode-buttons. Branch: editor/mode-buttons. Notes: opposite
   the window controls; none in full screen, at the menu bar's right end where fizzy draws one;
   default chords ⌃⌘E/J/O (Ctrl+Alt on Windows/Linux), checked against the keymap and plugins'
   chords; reuse `Chooser.icon`'s colours and tooltip, don't copy; nothing costs a
   frame when hidden or disabled; register hit rects.
3. **`app/layout`: Explode/Join rules and operations.** Std-only `explode_rules.zig` (bounds → screen
   rects, header growth, display clamp, the Join resolution order); `Floats.explode`/`join`,
   `State.homes`, `exploded`, the `Windows` seam, the in-window fallback. Tests for every Join case:
   region closed, window resized, view moved between windows, windows opened/closed meanwhile,
   pre-existing floats. Cloud.
   Kickoff — Task: Explode and Join as Layout operations. Plan: step 3. Builds on: step 2. Workspace:
   ../fizzy-explode. Branch: layout/explode-join. Notes: never modify the split tree; Join resolves
   against the current layout, never pixel rects; the Windows seam matches EMBEDDED step 1's
   ViewHost; the main window shrinks to the explorer, Join restores its pre-explode frame (or keeps
   the current one if resized); the animation progress rules are std-only and tested.
4. **`src/editor`: child windows for Explode.** The seam through `Popout`/`ViewHost`; exact placement
   and frame growth; persist `exploded` and `homes`; restart restore; quit saves state. Mac.
   Kickoff — Task: wire the Windows seam to real OS windows; persist exploded state. Plan: step 4.
   Builds on: steps 1, 3 and EMBEDDED_EDITOR_PLAN step 1. Workspace: ../fizzy-explode-windows.
   Branch: editor/explode-windows. Notes: windows through ViewHost only; content lands exactly where
   the place sat on Retina and on two displays; check restart and a vanished display; the main window
   shrinks to the explorer and persists its pre-explode frame; leave full screen before exploding;
   build the animation (core.anim, 250-350 ms, reduced motion, free when idle) here, or as step 4b
   (editor/explode-animation) if the PR passes 800 lines.
5. **`app/layout`: `canvas_rules.zig`.** Arrangement, view transform, fit, LOD with hysteresis;
   std-only, tested. Cloud.
   Kickoff — Task: std-only canvas maths. Plan: step 5. Builds on: main. Workspace:
   ../fizzy-canvas-rules. Branch: layout/canvas-rules. Notes: deterministic, parked-camera-stable;
   reuse CanvasWidget's maths; read Atlas's scale-architecture.md for the LOD contract.
6. **`app/layout`: the `Layout.stage` seam.** Spike the two placement approaches; present places
   under the transform; an input-mapping test on dvui's testing backend (a click at a transformed
   point hits the widget); fizzy's shape wraps its work area. Partly cloud, Mac to see.
   Kickoff — Task: `Layout.stage`, presenting places on a transformed viewport. Plan: step 6. Builds
   on: step 5. Workspace: ../fizzy-stage. Branch: layout/stage. Notes: a plain box at zero cost when
   canvas is off; shape code stays copyable and unprivileged; test click mapping at scale ≠ 1;
   popups stay unscaled.
7. **`app/layout`, `src/editor`: canvas interaction.** Pan, pinch and ctrl-wheel zoom, card move and
   resize, fit, focus a card, a view drag between cards, the `canvas` block in `layout.zon`, entering
   Canvas records the window arrangement and runs Join, leaving restores it (animated), card titles. Mac.
   Kickoff — Task: pan, zoom, move/resize cards, persistence, mutual exclusion. Plan: step 7. Builds
   on: steps 4 and 6. Workspace: ../fizzy-canvas-ui. Branch: layout/canvas-interaction. Notes: view
   drag must work under scale; persist pan, zoom and card rects; entering Canvas records the window arrangement (floats and OS rects, exploded, homes, main frame) and
   runs Join; leaving restores it, animated, windows flying back to their rects; Explode in Canvas
   leaves Canvas into Explode; closed view dropped, new view to its keyword place, removed display
   clamped; the arrangement persists in the `canvas` block so a relaunch in Canvas can still leave
   to it; each
   card carries its title (view title plus document name) at every zoom.
8. **`app/layout`: canvas LOD and culling.** Snapshot cards when tiny, cull off-view, opaque fills for
   frost at scale ≠ 1, a fixed-size legible title label that stays when a card's content is LOD'd away,
   `core.profile` numbers in the PR. Mac + GPU.
   Kickoff — Task: LOD, culling, snapshot cadence. Plan: step 8. Builds on: step 7. Workspace:
   ../fizzy-canvas-lod. Branch: layout/canvas-lod. Notes: reuse `State.snapshots` and
   `Layout.drawCaptured`; report frame times for 30 cards; the title stays legible at every zoom; nothing costs a frame when canvas is off.
9. **`fizzyedit/example-app`: `shapes/canvas.zig` and docs.** A copyable shape using `stage`, plus
   SPLITS.md and AGENTS.md notes. Cloud.
   Kickoff — Task: example canvas shape and docs. Plan: step 9. Builds on: step 6. Workspace:
   ../example-app-canvas. Branch: shapes/canvas. Notes: public Layout API only; the style of
   endless.zig; run scripts/check-examples.sh.

## Open questions

1. On Join, does a pre-existing OS-window float merge back as an in-window float (recommended) or
   dock home? And add "Join and Dock All"?
2. Canvas natural size: from the place (recommended), or a `Surface.natural_size` hint, which moves
   the fingerprint?
3. Is POPOUT "Next 2/3" (OS rect persistence) already claimed?
4. If EMBEDDED step 1 (`ViewHost`) hasn't merged, step 4 needs an adapter over `Popout`.
