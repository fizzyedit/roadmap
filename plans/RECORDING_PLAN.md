# One record: history, rewind and shareable tapes

Status: proposed.

Fizzy has four plans that each record what happens, in four different ways: a document's undo
history shown as a timeline (`HISTORY_TIMELINE_PLAN.md`), who made each step and the agent's
session tape (`SHARED_HISTORY_PLAN.md`, fizzyedit/plans#15), the black box of raw input
(`RECORDER_PLAN.md`), and the demo recorder of anchored input (`AUTOMATION_PLAN.md`, milestone 5).
This plan makes them one system, and gives it the job none of them does alone. Draw something in
pixi, scrub back through how it was made, select the part worth showing, and share it as a file of
a few kilobytes that anyone can replay, the web build included.

The idea in one line: **each document keeps a track, and a track is its history with the time put
back in.** Tracks are always on and kept whole, never in a ring, because they grow with the work
done rather than with time. Undo and redo become seeks along the track: one step per press, and
scrubbing when held. Tracks are recorded per document, never per window. A session is a set of
tracks plus a note of which one the person was looking at. The black box stays, as a ring of raw
input whose length is a setting, for frame-exact bug and performance captures. It is never what
gets shared.

## What there is

- **Every owner already records each change exactly.** Pixi's `internal/History` keeps every
  `Change` as an effect: pixels as `{layer_id, indices, values}`, layers, animations, grid and
  resize. Text's `textcore/History` keeps transactions, grouped into one step per run of typing.
  The history hooks `historyLine` and `historyGoTo` are on `main` (fizzyedit/fizzy#434) and text
  implements them. Neither history keeps the time a step was made, and neither outlives its
  document.
- **State as data.** `captureDocumentState`, `restoreDocumentState` and `documentFingerprint` are
  optional owner hooks. Text and pixi both implement them. The demo player's snapshots use them:
  a seek restores the nearest snapshot and replays the ops since then in silent frames (frames
  run but not shown), landing in 2.5 ms on average.
- **The tape.** `sdk/tape` (std-only) holds keyframes, ops, captions and chapters, written as ZON
  or binary (ten thousand ops load in 0.3 ms). `sdk/replay` holds the player, the live driver,
  silent frames and the `Stage` seam. A keyframe's documents are `files: {path, text}`, so only
  text documents fit in one today.
- **Pixi animates a drawing it got in one go.** `internal/Reveal` takes pixels written all at once
  (`pixi.draw`), puts them back and draws them again at a steady pace, nearest-next, scaled by
  `core.motion` (fizzyedit/fizzy#417). So pixi can already show a pixel effect being drawn.
- **The web loads plugins** as wasm side modules (`app/store/PluginLoader_web.zig`) and plays
  tapes. That is how the site's embedded demos run.
- **The black box** (`replay.BlackBox`, fizzyedit/fizzy#386) keeps the last minute of every
  frame's dvui events, in the window's pixels. That replays exactly in the same window and layout,
  and nowhere else.
- **The agent's session tape** (fizzyedit/agent) is a separate recorder: a keyframe, then the
  agent's commands. `SHARED_HISTORY_PLAN` patches it with a keyframe wherever another hand edited
  in between.

## Principles

1. **Record effects, not input.** A step is what the document became, in the owner's own terms:
   the pixels a stroke changed, the text a transaction inserted. Replaying an effect is exact on
   any machine, at any window size, under any layout, and after the brush algorithm changes (fizzyedit/pixi#29
   changed pixi's). Input replays only where it was recorded, so it is the black box's job alone.
2. **Per document, not per window.** A track follows a document, whoever changes it. OS windows,
   splits, floats, pan and zoom are not recorded. Five windows mean five places tracks are shown,
   never five things to record. What the person was looking at is one small extra track (the
   session's *focus*).
3. **The history is the track.** A track's steps are the owner's history entries, with the same
   ids `historyLine` reports, plus the time each was made and who made it (`SHARED_HISTORY_PLAN`'s
   `Author`). Cmd-Z is a one-step seek along it, the transport scrubs it, and a tape is cut from it:
   one record, not four. At first the track mirrors the owner's undo stack. Later the owner keeps
   only recent steps in memory and reads older ones back from the track.
4. **Invisible, or it isn't done.** Always-on recording is acceptable only when nobody can measure
   it in a frame. Every piece has a budget (Performance targets), a profiler scope
   (`core.profile`, under `track`) and a kept benchmark. A step that misses its budget doesn't ship
   on by default.
5. **Nothing runs while the app sleeps.** Recording happens only when an edit happens. No timer
   takes a checkpoint, nothing flushes to disk on a clock, and nothing requests a frame. A sleeping
   fizzy records nothing and costs nothing. An hour spent thinking is one gap in the times, and
   playback shortens it. The one exception is the transport's auto-hide (below): a single wake,
   scheduled only while the transport is showing.
6. **Motion is optional detail on an effect.** A step can carry how it was made, such as a pen's
   path and timing in art pixels, so a replay draws the way the hand drew. Without that, the owner
   animates the effect itself (pixi's `Reveal`). The track is complete without motion; motion
   only makes the replay more lifelike.
7. **Tapes stay tapes.** A shared recording is a `.tape`, played by the existing player on its own
   stage (`demo://`), with seeking, chapters and captions. That is also how the web build and the
   site already play tapes.
8. **One path on every target.** The web has no threads, so nothing in recording depends on one.
   Work small enough for a frame runs on the frame that needs it. Work too long for one is spread
   over the frames after, a slice each within a budget, which works everywhere and stays
   deterministic. A native worker could only ever be an optimisation of that, and none is
   planned until a measurement asks for one.

## The pieces

### A track

Each open document whose owner records steps has a track, kept by the host. It holds:

| Part | What |
|---|---|
| `base` | The document's state when the track began (`captureDocumentState`), or the file on disk when the document opened unchanged (stored by path and hash until a tape needs the bytes). |
| `steps` | Per step: the history id, `at` (ms since the track began), `ms` (how long it took), the author index, the owner's encoded effect, and optional motion. Appended to the track's byte buffer as they come. |
| `checkpoints` | The document's state every so often (below), so a seek never replays from the base. |
| `undo marks` | Undo and redo are steps of their own kind (`undone`, `redone`), naming the entry they move. An undo's effect is that entry's reverse, so every step applies forward and a replay never runs one backwards. |

The owner **pushes** each step as it happens, since only it knows what an edit became (a new
entry, or a keystroke joining its run), and applies recorded ones (fizzyedit/fizzy#456):

```zig
// Host: the owner asks, then reports. With no recorder installed, nothing is encoded.
pub fn recording(host: *const Host, doc_id: u64) bool
pub fn recordStep(host: *Host, doc_id: u64, step: sdk.Step) void
// sdk.Step: kind (added, joined, undone, redone, restarted), id (the history entry), effect,
// motion, ms.

/// Apply a recorded step to `doc`, forward, as a new history entry. `ms` asks for it to be shown
/// being made over that long (pixi: `Reveal`, following `motion`); 0 lands it at once (a seek).
applyStep: ?*const fn (state: *anyopaque, doc: DocHandle, effect: []const u8, motion: []const u8, ms: u32) anyerror!void = null,
```

The host pulling steps instead (an `encodeStep` hook) would need to hear of each step first.
Pushing needs nothing of it, and when authors land, a step's author is the host's to attach by
its `id`: `SHARED_HISTORY_PLAN.md` attaches it in `recordStep` itself (amended 2026-10-16), owners
reporting every step there and encoding its `effect` only while `recording`.

- **The owner's format is its own and versioned.** `Plugin.step_format` is a `u32`. A tape names
  the owner's id and format, and an owner that cannot read a format refuses the tape before it
  plays (no partial replays).
- **Pixi's effect** is the change it already appends, encoded forward ("after" values), with
  indices delta-varint and colours run-length. That is most of a stroke's bytes. Text's is the
  transaction's edits.
- **Recording is zero-cost when off,** since the owner never encodes. When on, it costs one encode
  per history step, at the moment of the edit (budget: 20 µs for a pixi stroke on the UI thread).
  Nothing is done per frame.

### Checkpoints: deltas, without a clock

**A checkpoint is what differs from a whole state,** not a copy of the document
(`app/state_delta.zig`, fizzyedit/fizzy#460). The track keeps its base whole. Each checkpoint is
the bytes that differ between the latest whole state and the document now, until a delta comes to
half the document: that checkpoint is kept whole, and the next ones differ from it, as video's
keyframes and deltas do.
- **The diff is generic.** It works on `captureDocumentState`'s bytes, so no owner writes
  anything for it. The common prefix and suffix are trimmed, a 64-byte block at a time. A middle
  as long in both (nothing shifted: a canvas) is kept as the runs of blocks that differ, so
  strokes far apart are a few runs. Any other middle (text inserted, shifting what follows) is
  kept as it is.
- **Restoring one** rebuilds the whole state from its reference first, then restores in place.
- An owner's `composeSteps` (compacting, rung 4) could make tighter deltas for its own documents
  later. The byte diff needs nothing from it.

**When.** A step asks for a checkpoint once the steps since the last weigh as much as the last
delta did, and then after 256 steps or a 30 s pause before one. At 8,192 steps one is asked for
whatever it costs, so a seek never replays more than that.
- The checkpoint is taken at the **top of the next frame**, before anything draws, never from
  inside an edit, where the owner's state may be half applied.
- After a pause, it is taken just after the step that ended it: near enough the pause's start for
  a seek.
- No timer is involved. A frame with nothing asked of it does one branch.

**Measured** (`zig build bench-tracks`, a stand-in owner whose state is a buffer, at load 7–8):
- Twenty thousand strokes on an 8 MB sprite hold 12.9 MB with 78 checkpoints. That's the 8 MB
  base, 4 MB of steps and about 1 MB of checkpoints. Whole copies held 29.6 MB with 2 checkpoints.
- A checkpoint of the 8 MB document costs 1.2 ms on its frame, against a 2 ms budget.

When the document closes, the track ends and stays in memory until the cap or the session's end
(persistence, below).

### A session

A session is a set of tracks plus a *focus* track. Each time the person's active document
changes, the focus track records `{at, track}`. It records only focus, never window geometry, so
a whole-session replay can cut between documents the way a video editor cuts between cameras.
A session tape has every included track's base in its keyframe, and its ops interleave the
tracks' steps in time order.

### What a shared tape is

Export takes a range of one track, or of a session, and writes a `.tape`:

- **The keyframe holds each document as its file**, bytes as on disk, naming its owner and step
  format: `File{ path, text, owner, format }`. A shared tape has to create the document from
  nothing, which a file does and `captureDocumentState` (a restore in place) does not. The file
  is the nearest checkpoint with the steps up to the range's start applied, written as the owner
  saves it, so the range plays from exactly where it starts.
- **Ops are `step` actions**: `Tape.Step{ file, effect, motion }`, at the step's time, with its
  `ms`, plus the focus cuts. The binary form keeps effects and motion in a blob table. Captions and chapters are the tape's as they are today, so `Script` can
  annotate an exported tape.
- **Time is real in the file, shortened at playback.** The player plays gaps over 2 s as 600 ms
  and marks them on the scrubber. A "real time" toggle plays them as recorded. Nothing is
  re-timed on export.
- **Authors come along** as an interned table ("Claude", "the person"), so the replay labels
  steps the way the activity feed does.
- **Size.** The whole tape is deflated (`std.compress.flate`, pure Zig, so it works on the web).
  Target: under 100 KB for an hour of drawing a 64×64 sprite, measured by a kept benchmark over a
  recorded session.

### Playing one

- **In fizzy:** opening a `.tape` plays it on its own stage, as a tab, never over the person's
  documents. A tape of one document plays in the document's own view, with the transport. The
  view frames the art itself (Fit to View, fizzyedit/fizzy#444), since pan and zoom were never
  recorded.
- **Seeking** uses the snapshots the player already keeps. The first pass also gets the tape's
  documents restored from blobs. Silent frames apply the steps between, unpaced. The determinism
  check carries on: each checkpoint recorded the document's fingerprint, and a replay that
  reaches it compares. With effects, a mismatch is a bug in an owner's encode, not drift.
- **On the web:** `fizzyed.it/app?tape=<url>` (and the site's embed) reads the tape's owners,
  loads each from the store as the web already loads plugins, and plays the tape in a player
  shape: one view and the transport, nothing else. A shape is copyable code over `Layout`, so an
  app built on fizzy gets the same player.

### Undo is a seek

Undo and redo work as they do today, one step per press, and are now moves along the track.
That makes them the way into everything else:

- **Pressing Cmd-Z** goes back one step (`historyGoTo` to the step before the top) and shows the
  view's **transport**: a small glass bar at the bottom centre of the document's view (the view's,
  not the window's), with the line, the playhead, the steps' authors and times, and undo/redo
  buttons. It is one bar per view, since tracks are per document.
- **Holding Cmd-Z (or Cmd-Shift-Z) scrubs.** Key repeat steps faster the longer it is held: one
  step, then several a frame, then whole seconds of the track. Letting go stops where it is.
  Dragging the playhead scrubs freely.
- **Scrubbing the live document never loses anything.** Going back and then editing is a step (the
  timeline plan's "going back is a step"), so the line only ever grows. What was gone back over is
  still on it, earlier.
- **The transport hides itself** after 3 s untouched (a setting). It is the one timer in the
  system: a single scheduled wake (`dvui.timer`), only while the bar is showing.
- **The hot zone.** Moving the pointer toward the bottom centre of a view, with no button held,
  brings the transport up. It shows after a short dwell (about 150 ms), so a quick pass across the
  bottom does nothing. It is a band the bar's width, a few points tall, floating over the view and
  never taking layout. It never appears during a stroke or a drag, and a press outside it hides it.
  On touch, the same band takes an upward swipe.
- **During scrubbing, owners apply steps unpaced** (no `Reveal`), and the bar shows where
  checkpoints are, so a long jump lands in one frame.
- **Watching how it was made** is different from scrubbing: it is a tape cut from the track and
  played on its own stage, at pace and with motion, so the person's document is never touched by
  playback. The transport has a "play from here" that opens one.

Where this conflicts with work near the bottom edge (pixi's canvas runs edge to edge), the dwell
and the "no button held" rule are what keep it out of the way. If testing shows that isn't enough,
the zone moves just outside the art's rect when the art reaches the bottom.

### What is kept, and for how long

There are two records, and they get two different answers.

**The black box is a ring.** It is per frame and only answers "what just happened", so it keeps a
window of time. The length is a setting, from 1 to 5 minutes, default 1. Measured at 1.3 MB a
minute, 5 minutes is 6.5 MB. Changing the length reallocates the ring and keeps what fits.

**Tracks are never a ring. Every document is always recorded, whole.** A track grows with the
work done, not with the clock: a document nobody edits adds nothing, and a sleeping app adds
nothing. That is what makes "always" affordable. Rough estimates, to be replaced by measurements in
step 3:

| Work | Steps | Track (effects, deflated) |
|---|---|---|
| An hour of typing | ~2,000 runs | ~100 KB |
| An hour of drawing a 64×64 sprite | ~2,000 strokes | ~200–400 KB |
| An hour on a 512×512 sprite, fills and pastes | ~2,000 steps | a few MB |

Effects store before and after (what undo needs), which is what pixi's `Change` already holds in
memory today. So while a document is open, a whole track costs little more than the undo history
does now. **Checkpoints are what can grow**, so they are thinned with age, the way backups are:
every 256 steps for the last hour, every few thousand before that, with at most a fixed share of
the track's bytes spent on them (the weighing above, and only deltas are thinned: a whole state is
what the deltas after it are made from). Seeking far back costs more applied steps. Applying
effects is memcpy-shaped, so that is milliseconds, not seconds.

- **While the document is open:** everything is kept. Recent steps and checkpoints stay in memory.
  Steps older than an hour spill to the profile (`<config>/tracks/`, deflated, written on edit
  and on quit, never on a timer, a slice a frame where a write is too long for one) and are read
  back when a seek reaches them. Phase
  two moves the owner's own undo stack the same way: recent entries hot, older ones paged from the
  track, so a long session's memory is bounded even though its history isn't.
- **After the document closes:** its track stays in the profile, keyed as `KeptDocuments` keys
  documents, and reopening the file picks it up if the file's hash matches the track's end. A file
  changed outside fizzy starts a new track, with the old one kept beside it until it ages out.
  This is the timeline plan's "history across a restart". It needs the owner to restore its
  history from a track (`restoreHistory`, named here, built in step 10).
- **The cap:** a total for the profile's tracks, default 2 GB (a setting). Reaching it compacts
  the oldest tracks first (below), and only drops whole closed tracks once compacting frees too
  little, never an open one. Devtools shows the total and the largest tracks, per the
  disk-hygiene rule.
- **What isn't kept:** documents whose owner records no steps (the image viewer, an archive), and
  anything that isn't a document (the explorer, layout, settings).

### Compacting a track

With room to spare, a track keeps every step. When it needs room, it **compacts**: steps that make
no difference to where the document ends up are removed, oldest first. Compacting is exact at the
moments it keeps (each checkpoint that survives, each chapter, the end) and loses the detail
between them: a replay no longer shows the stroke that was painted over. So it never touches the
recent part of a track that undo walks (the last hour, or what the owner holds in memory), and it
runs only when the cap needs it, never as housekeeping.

It is a ladder, cheapest loss first, each rung run only if the one before freed too little:

1. **Thin the checkpoints** (as above). Nothing is lost; old seeks replay more steps.
2. **Drop old motion.** The effects stay, so the document is exact; an old replay is drawn the
   owner's way (`Reveal`) instead of the hand's.
3. **Remove cancelled steps.** A step that was undone and never redone, together with its undo,
   leaves the document as it was: both go. The host does this for every owner, since it
   recorded which steps were undone (`undone`/`redone`), with no knowledge of what an effect
   holds. This is also what removes text typed and then taken back.
4. **Squash runs.** Consecutive steps between two kept moments become one net step. Only the owner
   knows how to compose its effects, through an optional hook:
   `composeSteps(a, b, allocator) ![]u8`, pure, reading only the bytes, so it can run a slice at
   a time. Pixi's composition keeps each pixel's first "before" and last "after", and a
   pixel that ends as it began drops out: a pixel drawn and drawn over is one change, or none.
   Text composes transactions. An owner without the hook keeps its runs as they are.
5. **Rebase.** The oldest surviving checkpoint becomes the track's base, and the steps before it
   go. The track then starts later, and is exact from there.

The rule a test holds it to is the determinism check: compacting never changes the document's
fingerprint at any moment it keeps. Compacting runs a slice a frame over steps already spilled,
within a frame budget, only while the cap needs it. It writes the compacted steps beside the old
ones and swaps them in, so a crash midway loses nothing.

Export offers the same ladder on purpose, independent of the cap: "the result's steps only"
(rungs 3–4 at each chapter) for a short tape of how something was made without every false
start, and as the privacy option for text.

## Games, and other streams

A game is a surface whose document is its world. Its track uses the same parts with different
contents:

- **Steps are its input stream**, one fixed record per tick (`tape.ring`, which already takes any
  stream), recorded at the game's tick rather than per dvui frame.
- **Checkpoints are its world snapshots** (`captureDocumentState`, cheap and in memory every N
  ticks), and its fingerprint is a hash of the world.
- **`applyStep` steps the simulation one tick** with that input. Scrubbing is the nearest checkpoint
  plus silent re-simulation, the same seek as a demo's.

The seam: a track whose steps are a stream, keyed by surface instead of document (`Surface.track`).
It is not built until a game plugin asks. The timeline strip and the transport work over any
track. A terminal (ghostty's output) is the same seam.

## Privacy

- **In memory, nothing is new:** a track holds what undo holds, plus times. On disk it is new:
  closed documents' tracks persist in the profile. A setting turns that off (tracks then end
  with the document, as undo does today), and "Forget this document's history" clears one track.
- **Nothing leaves the machine** except a tape the person exports. Export shows what is in it
  (which documents, how long, whose steps) before writing.
- **Text tracks keep what was typed and then deleted.** Export offers "the result's steps only"
  (checkpoints at chapter marks and no steps in between) for documents where that matters.
- **Password fields: a note for now.** A password is not a document, so tracks never see one. Only
  the black box does, and it never leaves the machine except in a capture someone asked for. If
  dvui ever marks a field as a password, the black box drops its text events (`RECORDER_PLAN`'s
  open item).

## What happens to the other plans

- **`HISTORY_TIMELINE_PLAN.md`:** stands. The strip, `historyLine`/`historyGoTo` and "going back
  is a step" are this plan's rewind. Its step 5 (recording a session, opt-in, an infobar dot) and
  step 6 (a pixi stroke recorded as a command) are replaced by this plan: tracks record effects,
  so no stroke needs to become a command to replay. Tracks are on by default. Its strip becomes
  the view's transport: undo shows it, the hot zone shows it, and it hides itself. There is no
  recording dot, because recording is not a mode.
- **`SHARED_HISTORY_PLAN.md`:** stands, amended 2026-10-16 to share this plan's channel: authors and
  `act` attach to the steps owners report through `recordStep` (no `historyStepped`), and a track
  reads each step's author from the host by `id`. Its step 4 (the
  agent's tape with keyframes where other hands acted) is replaced: the track already has every
  hand's steps, so an agent's session tape is the track's range during the session, filtered or
  labelled by author. The agent's own recorder goes.
- **`RECORDER_PLAN.md`:** its steps 1–5 stand (the black box, anchors, `--replay`, agent tools,
  hitch captures). The ring's length becomes a setting from 1 to 5 minutes. Step 6 (raw events made into portable tapes) is dropped. Its "timeline" seam is
  this plan's strip, and its "streams in a capture" is the games section above.
- **`AUTOMATION_PLAN.md`:** milestone 5 (recording input aimed at anchors) is narrowed to tours of
  fizzy itself (menus, navigation), which tracks don't hold, and is no longer on the path to
  sharing work. Milestone 6 (plugin demos) can use exported tapes as demos.

## Performance targets

| What | Target |
|---|---|
| Recording, any frame | 0: nothing per frame |
| Recording, while asleep | 0: no timers, no wakes |
| Recording, a pixi stroke | < 20 µs on the UI thread |
| Recording, the worst single step (a 512×512 fill, a paste, a resize) | < 1 ms on its frame |
| Checkpoint, 512×512 × 8 layers | < 2 ms on its frame (copy and diff); measured 1.2 ms for an 8 MB stand-in |
| Undo or redo, one step | the same frame, as today |
| Held Cmd-Z or a drag of the playhead | every frame lands; < 4 ms of applied steps a frame |
| Memory, an open document after an 8-hour session | no more than its undo history today, plus 10% |
| An hour of drawing a 64×64 sprite, exported | < 100 KB |
| Export a range | < 50 ms for an hour of steps |
| Seek anywhere in a tape | the same frame within a checkpoint's span; < 50 ms worst |
| Tape on the web, after its plugins load | first frame < 200 ms |

### Profiling it

Always-on is a performance claim, so it is checked from the first step and kept checked:

- **Profiler scopes** for every piece (`track.encode`, `track.checkpoint`, `track.seal`,
  `track.spill`, `track.apply`), so devtools and `fizzy_profile` show recording's share of a frame
  next to everything else.
- **A heavy session as a tape.** An agent draws for an hour of track time through `pixi.draw`
  strokes, fills, pastes and layer changes, on 64×64 and 512×512 sprites. Played `--fast`, it is
  the benchmark for encode, checkpoints, memory and spill, and the same session scrubbed end to
  end is the benchmark for seeking. Both are kept (`bench-replay` grows a track case) and run in CI
  as numbers to compare, not just pass/fail.
- **A frame-budget check in `test-integration`:** with recording on, the bundled tapes' frame times
  must match recording off within noise. A regression fails the build.
- **On the person's machine:** a recording step that runs over budget is a frame cause
  (`core.profile`'s causes), so a hitch capture names it.

Each target is measured before a step claims it, on macOS and in the Windows and Ubuntu VMs, and
on the phone for the web.

## Steps

Each a PR, useful without the next. Steps 1 and 2 do not wait on `SHARED_HISTORY_PLAN`: owners
push their steps, and authors attach to them by id when that plan lands.

1. **tape: steps.** Files as bytes with an `owner` and step `format`, the `step` action,
   `Check.steps`, `Stage.step`, binary version 3 with a blob table and an optional deflated body.
   Std-only, tested, benchmarked. No fingerprint move (the tape is not the plugin boundary).
   fizzyedit/fizzy#453.
2. **sdk + text: steps.** `Host.recording`/`recordStep`, `sdk.Step`, `applyStep`, `step_format`;
   text implements both. The fingerprint moves. fizzyedit/fizzy#456.
3. **app: tracks, measured.** The host keeps a track per recording document: base, steps,
   checkpoints as deltas taken at the next frame's top and thinned, the focus track, the cap.
   Profiler scopes, `fizzy.tracks` (devtools' and an agent's view), `bench-tracks`, and a check
   in `test-integration` that idle frames do no tracks work. Replays to the document's
   fingerprint. Behind `FIZZY_TRACKS` (fizzyedit/fizzy#459, #460). On by default in a PR of its
   own once a running fizzy's numbers meet the targets (a setting replaces the switch).
4. **pixi: steps.** Its `Change`s encoded with before and after, `applyStep` with `Reveal` for
   pace. The heavy-session tape and its benchmark. A store release after the SDK release that
   carries step 2.
5. **app: the transport.** Undo and redo as seeks that show the view's bar, held keys scrubbing,
   the playhead drag, auto-hide, the hot zone. Built over `historyGoTo`, so text gets it at once
   and pixi once its step 3 of `SHARED_HISTORY_PLAN` lands.
6. **app: export and play.** "Share how this was made" from the transport (a range of the track),
   the export summary, opening a `.tape` as a tab on its own stage, the one-view player, and the
   idle-gap rule in the player (moved here from step 1: nothing plays steps before this).
7. **web: play a tape from a link.** `?tape=<url>`, loading the tape's owners from the store, the
   player shape; the site embeds one.
8. **pixi: motion.** The pencil, eraser and line record their path and timing in art pixels, so a
   replay draws as the hand drew. Fill, paste and transforms keep `Reveal`.
9. **agent: its tape is the track's range.** The agent's session tape is cut from the document's
   track, labelled by author. Its own recorder is removed.
10. **app: tracks outlive the document.** Spill to the profile, closed tracks kept and picked up on
    reopen, `restoreHistory` in text and pixi, the cap, "Forget this document's history".
11. **owners page their undo from the track.** Recent entries in memory, older ones read back,
    so an 8-hour session's memory is bounded.
12. **app: compacting.** The ladder under the cap, a slice a frame: thinning, old motion, cancelled
    steps, then `composeSteps` (pixi's and text's) and rebasing; the fingerprint test over a
    compacted heavy session; export's "the result's steps only".

Steps 1–5 make recording invisible and undo a scrub. Steps 6–8 are sharing, through to the web.
Steps 9–12 fold the other recorders in and make "always" last.

## Open

- **Steps that cross documents** (a rename editing ten files) are a step in each track, with one
  `run`. A session tape shows them at the same moment. Nothing more is planned.
- **A plugin's state outside documents** (pixi's palette panel, a tool's settings) is not on any
  track. A tape of a drawing shows the drawing, not the tool choices. If people want the tool
  shown, it is motion detail (`motion` naming the tool), not a new track.
- **Owner formats across versions.** An owner reads every format it has ever written, or refuses
  the tape. Whether the store should keep old plugin versions for old tapes is undecided until a
  format actually changes.
- **Big documents.** A 4096² sprite's checkpoint is large. Owners may checkpoint by part (the
  layers touched since the last one) through the same hook, with the format saying so. Not
  built until it is measured to matter.
