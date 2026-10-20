# An agent's own view of a document

Status: proposed.

An agent drawing in pixi should be able to keep working while the person pans, zooms, resizes the
pane, switches tabs or draws in the same sprite, and it should see what the person changes as they
change it. Today it can't. It looks at the document through the person's window, so anything that
moves the person's view moves what the agent sees, and a hidden tab leaves it blind. This plan
gives an agent its own view of a document: a picture the document's owner draws for it, at a
region and scale the agent picks, offscreen. That view is independent of every pane. It also lists
what has to change for an agent's work to carry on unseen, and for the person and the agent to see
each other's work in one document.

It builds on `AGENTS_PLAN.md` ("Senses": reading fizzy at four depths), on
`SHARED_HISTORY_PLAN.md` (fizzyedit/plans#15: who made each step, an agent's `expect`, "the agent
hears it") and on `RECORDING_PLAN.md` (fizzyedit/plans#17, the per-document track). It amends
none of them; its additions to them are listed at the end.

## What there is

- **An agent's edits already ignore the view.** `pixi.draw` (its `stroke` op included) edits layer
  pixels through `PixelEdit` in document coordinates. It reads no canvas widget and no view
  transform. Its Reveal (the pixels drawn again at a steady pace) advances in `beginFrame`
  whether or not the canvas draws. `pixi.read` and `pixi.exportImage` work on a hidden document.
- **An agent sees only what the window drew.** `ui.capture` reads back the window's
  `FrameTarget`, the frame as fizzy drew it; `ui.snapshot` lists that frame's widgets. A document
  in a hidden tab, or in a pane that isn't drawn, is in neither. The capture also costs about 2.4 s
  on the UI thread in Debug (AGENTS_PLAN, "Senses"). There is no tool that pictures a document
  apart from the view.
- **No owner hook draws a document offscreen.** The one drawing hook is `drawDocument`, called by
  `drawDocSurface` after `bindDocumentToPane`. The pieces for one exist, though:
  - pixi composites visible layers into `layer_composite_target` (`render.syncLayerComposite`, at
    most once a frame), and `Export.exportFlattened` reads it back with `Texture.readTarget`;
  - `pixels.composite` does the same on the CPU;
  - `Layout.captureUnplaced` already draws a surface that drew nowhere offscreen (360×480, for the
    view picker);
  - dvui's `renderTarget` and `FrameTarget.snapshot` are used by the blur pyramid and the drag
    glass.
- **pixi keeps one camera per document.** `file.editor.canvas` (scale, origin, scroll) belongs to
  the file, and every pane drawing it rebinds it. `pixi.view` and `pixi.fitView` read the last
  drawn viewport, so they fail when the sprite isn't on screen. Per-pane UI only (rulers, the Edit
  pill) is in `CanvasData.forGrouping`.
- **A claim holds everything, unevenly** (`sdk.Claim`, `Host.claimDocument`):
  - `holdInput` (`src/editor/Editor.zig`) marks every mouse event in the canvas rect handled
    before `drawDocument` runs (presses, motion, wheel), so pan and wheel zoom are held as well as
    strokes. Trackpad pinch is not: it comes from a global ratio (`takeTrackpadPinchRatio`), not
    an event.
  - Every keybind of the active document's owner is skipped, navigation keys included.
  - `runCommand` refuses only for the active document, and `callCommand` is never gated.
  - Closing is refused. Tabs and splitters stay usable.
  - The owner is never told; it can only poll `host.documentClaim`.
- **Claims are on their way out of ordinary work.** fizzyedit/agent#16 (waiting on
  fizzyedit/fizzy#442, `OpenOptions.beside`): an agent's commands no longer claim. They open the
  document beside the person and start a recorded session, and pixi waits for the person's open
  stroke (`personBusy` → `afterPerson`). `fizzy_claim` stays, for work that can't share a document.
- **Working unseen has traps:**
  - An agent call with no `path` goes to `host.activeDoc()`. Since agent#16 opens the agent's
    document beside the person, that is usually the person's document, not the agent's.
  - pixi's `active_drawing` is cleared only inside `FileWidget`. A stroke left open when its pane
    stops drawing keeps `personBusy` true, so the agent's calls queue forever and
    `needsContinuousRepaint` keeps the app awake.
  - `ui.settled` waits for the whole app to be quiet. A Reveal, the activity feed after a command,
    or that stuck stroke keep it false. A `ui.capture` with no `FrameTarget` waits and refreshes
    forever.

## Principles

1. **An agent sees the document, not the window.** A look is the owner drawing the document for
   the agent: a region in the document's own units, at a scale the agent picks. Nothing on screen
   moves, and nothing of the person's view is read.
2. **The view is the person's.** Pan, zoom, scroll, resizing and switching tabs are never held.
   A claim, when there is one, holds edits only.
3. **Working is not being drawn.** Everything an agent does to a document works with its pane
   hidden, in a background workspace, mid-resize or behind a modal dialog.
4. **Nothing costs a frame unused.** A look renders once, on request, into a target read back a
   frame later. No owner draws anything for an agent nobody is running.
5. **One contract for every owner.** The look is an optional document hook any owner can
   implement. Without it, an agent falls back to the window capture it has today.

## The look

```zig
// sdk: an optional document hook (Plugin.zig)
renderDocument: ?*const fn (state: *anyopaque, doc: DocHandle, req: Look.Request, out: *Look.Out) anyerror!void = null,

pub const Look = struct {
    pub const Request = struct {
        /// In the document's own units: pixels for pixi, lines and columns for text.
        /// Empty: the whole document.
        region: Rect = .{},
        /// Longest side of the picture, in pixels; the owner fits the region into it.
        /// 1536 keeps a look near the ~1.15 MP a model reads anyway.
        max_px: u32 = 1536,
        /// The owner's own extras, as ZON: pixi's `.{ .grid = true, .layers = "*" }`.
        options: []const u8 = "",
    };
    pub const Out = struct {
        /// Premultiplied RGBA in a host-owned target the owner renders into.
        target: *dvui.Texture.Target,
        /// What the picture shows, in document units (the region, clamped).
        shown: Rect,
        /// Document pixels per picture pixel, so a point the agent picks maps back exactly.
        scale: f32,
        /// The document's step when drawn (`SHARED_HISTORY_PLAN`'s step id), so the agent knows
        /// what it saw.
        step: u64,
    };
};
```

- **The host asks, the owner draws, the agent reads a frame later.** `Host.lookDocument(doc,
  req, callback)` queues the request. In the next `endFrame` the host makes a target, calls
  `renderDocument`, and starts the GPU readback. The pixels reach the callback a frame after that,
  the way AGENTS_PLAN's raw region captures do (3A): no frame waits, and fizzy never encodes. On
  the web the readback is synchronous; that's acceptable there.
- **The agent tool** is `doc.look .{ .path, .region, .max_px, .options }`, answered with raw
  "FZPX" pixels plus `shown`, `scale` and `step`. The bridge scales and encodes, as it does for
  `ui.capture`. The bridge's instructions say: look at a document with `doc.look`, keep
  `ui.capture` for the app around it.
- **pixi draws it from what it already has.** `syncLayerComposite`, then the region of
  `layer_composite_target` drawn into the look target at whole-pixel steps, over the
  transparency checker, with the grid when asked. The person's open stroke is in it, from the
  split composite, because it is part of what the person sees too. A look costs one textured quad
  and a readback, never a recomposite unless layers changed.
- **Text can follow,** drawing lines with its highlighting. An agent reads text faster with
  `text.read`, so text's look is for layout questions (wrapping, folding) and comes later.
- **A generic fallback**, for an owner without the hook, would draw its surface offscreen at the
  requested size, the way `Layout.captureUnplaced` does for the picker. That only works once
  owners keep their camera per view rather than per document (pixi does not), so it waits for
  step 9. Until then, no hook means `ui.capture`.

### The agent's camera

`pixi.view` and `pixi.fitView`, called by an agent, set **the agent's camera**: the region its
next `doc.look` shows when it passes none. They never touch the person's view. The camera is kept
per author (`SHARED_HISTORY_PLAN`'s `Author`) by the owner, in document units, so it works with
no pane at all. The person's pan and zoom stay theirs. Neither of them ever moves the other's view.

## The view is the person's

With agent#16, an agent claims only by asking (`fizzy_claim`). Then a claim holds edits, not
navigation:

- **The owner decides what is an edit**, because only pixi knows that a wheel zooms its canvas and
  a press starts a stroke. `holdInput` stops consuming events for an owner that sets
  `claim_aware` in its vtable. That owner reads `host.documentClaim(doc)` while drawing and
  refuses what edits: a stroke, a transform, an edit key, a drop. Pan, zoom (wheel and pinch
  alike), scroll and selection-free hover stay live. Owners without the flag keep today's full
  hold.
- **Keybinds and commands by what they do, not by owner.** A command says whether it edits (a new
  field on the command, defaulting to edits). Under a claim, the person's keybinds and palette
  calls that don't edit run as usual. Edit commands are refused for the claimed document, not just
  for the active one: by the command's `path` argument when it has one, else the active document.
  `callCommand` and `runCommand` go through the same check.
- **What the person sees** stays the pill ("Claude is working", Stop). It gains a quiet mark on
  the tab, so a claimed document in a background tab is visible from the strip.

## Working unseen

- **An agent's call names its document.** The agent plugin fills `path` from the connection's
  session when a call leaves it empty, so it never falls to the person's active document.
  fizzy's command palette is unchanged.
- **A stroke ends with its pane.** pixi ends the person's open stroke or transform (as a release
  would) when the pane holding it stops drawing or loses the pointer's capture, so `personBusy`
  can't outlive the gesture.
- **A Reveal no one sees finishes at once.** If no pane drew the document last frame, pixi's
  Reveal completes in one step instead of refreshing at 600 px/s. Its pace is for a person
  watching; an agent working behind a closed tab shouldn't keep the app awake.
- **Settled for one document.** `doc.settled .{ .path }` waits until that document's owner is
  idle for it: no Reveal running, no queued calls (`runWaiting` empty), nothing loading. The whole
  app being quiet is still what `ui.settled` means. A `ui.capture` with no `FrameTarget` answers
  "no picture" at once instead of waiting.
- **Checked, not assumed:** every agent tool that takes a `path` gets an integration test with the
  document's tab in the background, its pane resized between calls, and the window's other
  workspace in front.

## Seeing each other

Once nothing holds the person's input, both hands are in one document. pixi edits are effects
(a `Change` per step), so their steps interleave; `SHARED_HISTORY_PLAN` names whose each one is
and what Cmd-Z does. What this plan adds is the agent knowing what changed under it:

- **Every answer says what changed since the agent last looked.** For each document the
  connection works in, the next tool answer carries the steps since the `step` of its last
  `doc.look` or edit that weren't its own: author, kind and the bounding region in document units
  (pixi derives it from a `Change`'s indices). An agent can look again at just that region. This
  is the same channel as SHARED_HISTORY's "the person undid your Draw", widened from undos to
  every step.
- **The agent's edits can say what they assumed.** `pixi.draw` takes `expect` (a step id). When
  steps by other authors have touched the region it draws in since then, it refuses with what
  changed, rather than painting over the person's new stroke from an old picture. Without
  `expect`, an edit paints, last writer wins, which is fine for pixels that don't overlap.
- **Not in the middle of the person's gesture.** pixi already queues an agent's call while the
  person's stroke is open, and runs it in `beginFrame` after the release (`afterPerson`). The queue
  keeps the call's author and `expect`.
- **The person sees the agent arrive.** The Reveal plays the agent's pixels where the person is
  looking. The activity feed names the agent's step. Nothing moves the person's view to follow
  it.

## Showing the person the agent's view (later)

A pane can show a document through the agent's camera, a second view of the same sprite labelled
"Claude's view", so a person can watch where the agent works while drawing elsewhere in the
sprite. That needs pixi's camera per view rather than per file: scale and origin move from
`file.editor.canvas` into `CanvasData.forGrouping`, with the agent's camera as one more view that
has no pane. It also lets a generic look draw any owner's surface offscreen (see the fallback
above). It is the last step because nothing before it needs a second camera on screen.

## Measured before it is claimed

- `doc.look` of a 1024×1024 pixi region: under 2 ms of UI thread across its two frames in
  ReleaseFast, against `ui.capture`'s ~2.4 s in Debug, measured with `profile.report` on macOS
  and in the Windows and Ubuntu VMs.
- An idle app with an agent connected and a document hidden: zero frames per second once its
  Reveal and queue are done (the `needsContinuousRepaint` trap fixed).
- A tape where the person pans, zooms, resizes and draws for a minute while an agent draws in the
  same sprite: no agent call lost or doubled, no person stroke painted over without an `expect`
  refusal, and every frame of the person's within budget.

## Steps

One PR each, in fizzy unless named.

1. **agent: commands don't claim** (fizzyedit/agent#16, with fizzyedit/fizzy#442). Already open.
2. **sdk: the look.** `renderDocument`, `Look`, `Host.lookDocument` (queued, answered a frame
   later through the readback), and the `automation` service's plumbing. Moves the fingerprint
   (label `sdk`).
3. **pixi: draws a look,** from the composite target, with the checker, grid and the person's open
   stroke, plus the agent's camera kept per author for `pixi.view`/`pixi.fitView`.
4. **agent: `doc.look`** and `fizzy_look` in the bridge; the bridge's instructions prefer it to
   `ui.capture` for documents; the connection's session fills an empty `path`.
5. **pixi: work goes on unseen.** A stroke ends with its pane, and a Reveal no pane drew finishes
   at once.
6. **agent + app: settled for one document** (`doc.settled`), and a capture with no target
   answers at once. The integration tests with the document hidden and resized.
7. **app: a claim holds edits, not the view.** `claim_aware` owners, commands that say whether
   they edit, the check by the command's document, pinch held or passed like the wheel, and the tab
   mark. pixi opts in.
8. **agent + pixi: what changed since the agent looked,** in every answer, and `expect` on
   `pixi.draw`. Needs SHARED_HISTORY's authors, attached to steps in `recordStep` (its steps 1–2).
9. **pixi: a camera per view,** then "Claude's view" as a pane, and the generic look fallback.

## Open

- **Should a look show the person's open stroke?** This plan says yes, since it's on the canvas.
  The alternative is to show only committed steps, so an agent never reasons about half a gesture.
- **How big is a look by default?** 1536 px on the long side matches what a model reads, but pixel
  art at 1:1 may want exact pixels below that, with the agent asking for a region instead.
- **Does an agent ever get the person's camera?** "Look where they're looking" is useful
  ("draw it here"). It would be read-only, a `doc.look .{ .camera = .person }`, and never a way to
  move them.
- **Text:** whether its look is pictures at all, or the model-level read with layout facts (wrap
  points, folds) is better for an agent.

## Additions to other plans

- **`AGENTS_PLAN.md`, "Senses":** a fifth way to read: a document's own picture, owner-drawn, apart
  from the frame. It sits between the model (0) and the widget tree (1).
- **`SHARED_HISTORY_PLAN.md`:** "the agent hears it" widens from undos to every step by another
  author since the agent's last look. `expect` is taken by `pixi.draw` as well as undo.
- **`RECORDING_PLAN.md`:** none. A look reads the document; it records nothing.
