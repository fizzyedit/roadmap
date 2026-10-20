# One history, many hands: who made each step, and undo and tapes with more than one

Status: in progress. Step 1: fizzyedit/agent#20, fizzyedit/agent#21. Amended 2026-10-16: authors
attach to the steps owners already report (`Host.recordStep`, `RECORDING_PLAN.md`), not to a
second channel; see "Recording it" and "Steps".

A document is now worked on by more than one hand at once: the person, an agent beside them (it
no longer holds the document while it works, fizzyedit/agent#16), a tape playing, another plugin
running a command on it. This plan says who made each change, what Cmd-Z does when the last change
was someone else's, how a plugin that does not own a document takes part, and how an agent's
session tape stays true when the person edited during it.

It builds on `HISTORY_TIMELINE_PLAN.md` (a document's history as a line you can scrub: the hook
`historyLine`/`historyGoTo`, on main and implemented by text; the strip; going back is a step).
That plan read a step's author from the claim holder, and agents no longer claim; this one gives
it authors another way, and lists its other amendments at the end.

## What there is

- **Each owner keeps its document's history.** Text: `textcore/History`, transactions grouped into
  a step per run of typing, `id` bumped on every mutation (dirty tracking) and `step` stable for an
  entry's life. Pixi: `internal/History`, an undo and a redo stack of `Change`s (pixels, layers,
  animations, grid, resize, reorders), one step per `append`, an exact duplicate of the top change
  dropped. Neither caps its depth. Image, chat and archive keep none.
- **Undo is a command per owner.** `fizzy.undo` (Cmd-Z) runs `<active document's owner>.undo`
  (`Editor.runActiveDocCommand`). It takes no arguments and acts on the active document. The
  owners' `pixi.undo` and `text.undo` take none either, and pixi's runs even during the person's
  stroke (only `pixi.draw` and the transform wait for one).
- **Other plugins change a document through its owner's commands.** Atlas's "convert wikilinks"
  runs `text.format`; zig supplies text's formatter; chat writes with `text.write`; an agent draws
  with `pixi.draw`. Each is a step in the owner's history. Nothing keeps a second history of a
  document.
- **`Host.documentEdited(doc_id)`** is the owner's report of an edit. It becomes an activity
  bubble only while a command runs, titled with the command, so typing and hand-drawn strokes
  never show. Text's typing does not call it. Pixi calls it before `append` decides whether to keep
  the change.
- **Nobody records who made a step.** Entries have no author, `documentEdited` carries none,
  `Command.Call` has no caller, and `runCommand` versus `callCommand` is not a person/agent split
  (the palette and keybinds with arguments use `callCommand`). Claims said who held a document,
  never who made an edit. A tape's injected events are marked (`automation.Input.synthetic`), for
  native-menu dedupe only.
- **Where changes start, in a frame.** Input is consumed during the draw: by the keymap's tick, by
  plugins' keybind ticks, and by the widgets of the surfaces, menus and palette as they draw. The
  agent's commands, chat's writes and pixi's waiting calls (a command that arrived during the
  person's stroke, queued by `afterPerson` and run when they let go) run earlier, from plugins'
  `beginFrame`. A demo or a live tape (a tape of input played into the running app, as an agent's
  `fizzy_act` is) injects its events before all of it.
- **Step ids do not last.** Text's `reloadFromDisk` and `restoreState` start history over, ids
  from 1. Pixi has no ids.
- **The agent's session tape is beside history, not from it.** A keyframe (the document's bytes when
  the session began, which playback cuts to), then each command the agent ran. What the person did
  meanwhile is not in it, so a replay can come out different from what the document became; until
  fizzyedit/agent#16 the person could not edit meanwhile. A pixi draw that waited out the person's
  stroke is recorded when it was asked for, not when it ran. A session on a sprite with no path yet
  (`untitled-1`) records nothing.
- **The black box** (`RECORDER_PLAN.md`) keeps the last minute of raw input, frame-exact, to
  reproduce a bug or a stutter, and its step 6 turns a stretch into a portable tape. A different
  record for a different job; this plan uses it for the person's own recording and for races.

## Principles

1. **A document has one history, its owner's.** Anything that changes a document, from any plugin,
   does it through the owner's commands, so the change is a step there: undoable, shown, attributed.
   A plugin that needs a change no command makes asks the owner for the command; it never keeps a
   history of someone else's document.
2. **Fizzy knows who; the owner knows what.** The author of a step is the host's to know and keep,
   by step id. Owners never store authors. The one thing an owner asks is whether it may fold an
   edit into the step before it.
3. **An agent works for the person.** Its changes are the person's to take back first: Cmd-Z takes
   back the last change, whoever made it, and says whose. An agent never takes back the person's.
4. **Nothing waits on a clock, and nothing costs a frame unused.** Authors, undo and recording
   follow events. Working alone, the author map stays empty and no recorder is subscribed.

## Who made a step

### Authors

```zig
pub const Author = struct {
    kind: Kind,
    /// Which one: the plugin acting and its own id for the session or connection (the agent
    /// plugin's connection, chat's ACP session). Two agents both called "Claude" are two authors.
    plugin: []const u8 = "",
    session: u64 = 0,
    /// As the person reads it: "Claude", "the tour". Display only, never identity.
    label: []const u8 = "",
    pub const Kind = enum { person, agent, tape, unattributed };
};

/// Act as `author` until `endAct(outer)`: the steps recorded meanwhile are theirs, and they are
/// one run (below). The innermost `act` wins, as `running()` names the innermost command; a
/// command opens no scope, so atlas running `text.format` is whoever is acting. UI thread only.
pub fn act(host: *Host, author: Author) Acting
pub fn endAct(host: *Host, outer: Acting) void
/// Who is acting now: the innermost `act`, else the frame's input author where input is being
/// consumed, else `unattributed`.
pub fn author(host: *Host) Author
```

### Who acts where

- **The frame's input author is a default, not a scope.** Once the demo player and the live driver
  have decided whose frame it is, the host sets it: a tape's while one drives (its events are its
  own, `synthetic`), the person's when a real press or key paused or stopped a driver this frame,
  and the person's when no driver runs. A real release or pointer motion reaching a tape's frame is
  not authorship (no driver stops for them). The default applies only where input is consumed:
  around the keymap's tick, plugins' keybind ticks, and the surface, menu and palette draws.
  Plugins' `beginFrame`, event delivery and background reaps run without it.
- **A tape's author** is given by whoever plays it: a demo is itself ("the tour");
  `automation.play` takes its caller's, so an agent's pointer tape (`fizzy_act`, a run's input
  steps) is the agent's. A person's input stops an agent's pointer tape as it does today; agents
  draw beside the person with commands, not the pointer (fizzyedit/pixi#31).
- **An agent acts explicitly.** The agent plugin's `agentCall` opens `act` for the connection
  around each command (labelled with the MCP client's name from `initialize`, "An agent" until
  then). Chat opens `act` for its ACP session around the writes it applies.
- **Deferred work carries its author and run.** Whatever stores a call to run later stores
  `{author(), run}` with it and runs it inside `act` with both: pixi's `afterPerson` queue, so a
  draw that waited out the person's stroke is still the agent's and still part of its call.
- **A step recorded where nobody acts is `unattributed`.** That is a missed site, never quietly
  the person's. `zig build test-integration` asserts there are none after its tapes; devtools
  shows the count while the timeline is open.

### Recording it

**One channel, the recording plan's.** Owners already report each history step to the host:
`Host.recordStep(doc_id, step)` with `sdk.Step` (fizzyedit/fizzy#456, `RECORDING_PLAN.md` step 2),
`kind` one of `added`, `joined`, `undone`, `redone`, `restarted`, and the entry's `id`. That is the
channel authors attach to. There is no `historyStepped` and no second `Step`; an earlier draft of
this plan had both, and they would have been two ways to say the same thing.

- **Every step is reported; only its bytes are gated.** An owner calls `recordStep` for every step
  it makes, and encodes the step's `effect` only when `recording(doc_id)`. Working alone with no
  track, a step costs one call and nothing is kept.
- **`Host.recordStep` attaches the author.** Before it hands the step to the recorder (when the
  recorder `wants` the document), it looks at who is acting (`author()`): anyone but the person is
  kept in the document's author map under the step's `id`. The recorder, and a track, read a
  step's author from the host by `id`, so a track records every hand's steps, labelled.
- **`Step` gains `top`:** the entry on top once the step is done (0 when none is). The host keeps
  each document's top from it, so Cmd-Z's label reads the top step's author without asking for
  the line. An owner that does not fill it yet leaves it 0, and the label stays plain "Undo".
- **The draft's other kinds map onto these.** `merged` is `joined`. `moved` (a drag on the
  timeline, `historyGoTo`) is reported as the `undone` and `redone` steps it is, as text's
  `goToStep` already does. `reversed` (the timeline plan's "going back is a step") arrives with
  that plan's step 4, as `added` steps made by the jumper. `discarded` (an exact duplicate not
  kept) is simply not reported: an owner decides before it reports.

```zig
/// Whether the acting author may fold an edit into step `id`: true when `id`'s author is the
/// acting one (a step absent from the map is the person's). False ends the run of typing and
/// the owner starts a new step.
pub fn mayMerge(host: *Host, doc_id: u64, id: u64) bool
```

- The host keeps, per document, `step id → {author, time, run}` for every step not made by the
  person (`time` is for the timeline's hover); absence means the person, so working alone the map
  is empty. Authors are interned by identity (kind, plugin, session) in a small table and the map
  holds an index, so a label outlives the plugin that gave it (a reload).
- **A run** is one outermost `act`: an agent's run is one MCP call, not its whole turn, and a
  tape's is one op. Deferred work keeps the run it came with. The person's steps have no run; they
  are never stored.
- `restarted` clears the document's map. `discarded` changes nothing.
- **Going back is the jumper's.** A `moved` is recorded with the acting author (the person
  dragging, or a "Go back" from the bubble), and the `reversed` steps appended later, when an edit
  lands after it, are that jumper's, whoever made the edit that triggered them. So Claude's next
  draw after the person went back never makes Claude the author of taking back the person's work.
- The host keeps each document's top step (`top`) and looks its author up in the map, so the
  Cmd-Z label is right after an undo or a drag and never asks for the line. An owner that does not
  report steps yet (pixi until step 3) keeps a plain "Undo".
- **The activity bubble names others:** "Claude · Draw", including steps made outside a command
  (an agent's pointer tape typing). The person's own steps stay unlabelled: the feed is quiet for
  one's own work and speaks for everyone else's.

Text decides merging inside `textcore/History.end`, which knows nothing of fizzy; the editor's
edit hook that calls it (`TextEditor.editNotifyEnd`) asks `mayMerge` for the top entry's step and
passes the answer in (`end(gpa, sel, may_join)`). That is what keeps an agent's keystrokes from
joining the person's run of typing. `pushComplete` (`text.write`) already starts a step of its own.

## Undo with more than one hand

Cmd-Z keeps its meaning, the last change, and stops being a surprise. It is linear on purpose: one
line, one top, no step taken out from under another. The price is that a person typing while an
agent writes can take back the agent's step when they meant their own; the label and the bubble
say so, and Redo is one click away.

- **It says whose, before and after.** When the top step is not the person's, the menu reads "Undo
  Claude's Draw" and the bubble after it "Undid Claude's Draw", with a Redo on it. This is the case
  it is for: an agent drew, and the person says "no, not like that".
- **"Go back to before Claude's change"**, on an agent's activity bubble and on the timeline, goes
  back to just before the first step of the agent's last run. When the person's steps are mixed in,
  it says so first ("also takes back your 2 edits; they stay on the line"). Going back is a step
  (the timeline plan), so nothing is lost and dragging forward brings it all back. It is not
  offered when that run is already undone, or when history started over after it.
- **The agent hears it.** When the person undoes or goes back over an agent's steps, it hears
  so ("the person undid your Draw, step 41"), so it does not draw it again from a stale picture.
  It hears it as an event on the control server's `documents` topic (`CONTROL_SERVER_PLAN.md`
  step 2: the step, whose it was, who undid it), which any client follows, not as a notice the
  agent plugin keeps per connection. The host's part is the fact: an `undone`/`redone` whose step
  another author made.
- **An agent never takes back the person's work.** Its undo and redo are
  `fizzy.undo .{ .path, .expect }`: `expect` is the step id it means, and the undo refuses unless
  that step is on top: "the person has changed it since: fix it forward". Over the control server
  that is `POST /commands/fizzy.undo` with `{path, expect}`: the refusal is the command's (a 409),
  so it holds for any caller, curl as much as an agent, and nothing in the agent plugin checks it.
  Each command's answer lists the steps it made (the server reads them from the host by the call's
  run); a command that waited (`.waiting`) has its steps on the `documents` topic when it runs.
  An agent does not drag the line (`historyGoTo` refuses it).
- **Not during the person's gesture.** Only the owner knows a gesture is open (pixi's
  `personBusy`), so the owner's undo, called by anyone but the person while one is, queues the
  call (`afterPerson`, with its `expect`) and checks `expect` when it runs, as `pixi.draw` already
  waits.
- **One entry point.** `fizzy.undo` and `fizzy.redo` take an optional `path` (the active document
  when empty) and `expect`, and are the only undo anyone calls; owners' `<owner>.undo` stay what
  they route to.
- **A running agent and Cmd-Z.** An agent can add a step between two presses. Each press takes the
  top step at that moment and names it, so the second press may name the agent's newest step;
  "Go back to before Claude's change" is the move that takes a whole run.

Undoing your own step from under someone else's later work ("selective undo") is left out. It
needs each owner to invert one step out of order, which pixi cannot do safely today (its layer and
animation changes name their target by index, and deleted layers come back off a stack), and an
agent can fix forward. If people ask for it, it is its own plan, starting with pixi's changes
naming layers and animations by id.

## Plugins that are not the owner

- **Changing someone else's document:** through its owner's commands, always (principle 1), so
  the change is the owner's step, made by whoever is acting.
- **A change across documents** (a rename editing ten files) is a step in each, with one run.
  Undoing it whole is not built; `run` is the seam.
- **Documents without history** (an image viewer, an archive, a chat transcript) take no part. An
  archive's contents are documents with histories of their own.
- **History that is not a document's** (moving files in the explorer, atlas's graph layout,
  settings) is on no document's line. A workbench history for file operations would use the same
  `Author` and `act`; nothing is built for it here.

## Tapes when more than one hand works

Superseded by tracks (`RECORDING_PLAN.md`; step 4 above): a track has every hand's steps, labelled
by author. Kept for the reasoning about other hands, which tracks keep.

An agent's session tape stays what it is: the agent's commands on the document after a snapshot,
failed ones and reads included, a bug report or a demo of its work. What it lacked is what other
hands did in between. It gets it at the boundaries:

- **A keyframe where someone else changed the document.** Before each agent command runs (in
  `agentCall` before `callCommand`, and in pixi's `runWaiting` before a deferred one), the recorder
  asks whether anyone else acted on the document since the agent's previous command: any
  step reported (`recordStep`) while another author acted (a step added, undone or redone, the line moved,
  reverses appended), or a `restarted`. It reads who acted in the event, not who made the step: the
  person undoing the agent's step is the person acting. If so it cuts to the document as it is now,
  then records the command. The check comes before the command, so the keyframe never holds the
  agent's own change; and it reads authors, so the agent's own typing, its deferred draws and pixi's
  reveal never cut one.
- **The person's work goes in as net changes.** A keyframe holds the document as it stood, not how
  it got there: text typed and deleted, a pasted secret taken out again, never reaches the tape.
  Whatever is still in the document at the agent's next command is in the keyframe, as the session's
  first snapshot already holds whatever was there. The person's own recording (opt-in, with the
  timeline plan's dot on the infobar) is the only thing that keeps their input.
- **A deferred command is recorded when it runs,** with its author, not when it was asked for.
- **Playback shows a keyframe** as a marked jump, a chapter titled with the other hand ("the
  person's edits"), never a silent cut. A tape plays on its own stage (`demo://`), never over the
  person's open document; a person who pauses a demo and edits has diverged it, and resuming seeks
  back to the tape's own moment, as today.
- **Cost.** At most one keyframe per agent command, and only when another hand edited since. The UI
  thread takes the owner's `documentSnapshot` (pixi first finishes any reveal, its animation of a
  command's drawing being drawn) and a worker encodes it, as a save does. Step 4 measures a 512×512,
  8-layer sprite with the person drawing throughout a 200-command session and keeps that benchmark;
  the budget is 2 ms on the UI thread per keyframe.
- **What it cannot hold:** races between the hands (a draw waiting on a stroke in flight) replay in
  order, not as they interleaved. Reproducing one is the black box's job (`RECORDER_PLAN.md`).

The person's own recording is `RECORDER_PLAN.md`'s step 6 (the black box's stretch made portable),
not a second recorder: it has their navigation and timing, which a command log does not.

## Small fixes found while mapping this

Each stands alone and can land first.

- **The agent undoes its own document.** It calls `pixi.undo`/`text.undo`, which act on the active
  document; until `fizzy.undo .{ .path }` exists, it refuses when its session's document is not
  active.
- **An untitled document's session records nothing.** The recorder takes `documentBytes` by path;
  an untitled one is recorded under its title.
- **Pixi's unsaved mark** counts steps (`bookmark`), so save, undo, then an edit reads as saved.
  Text's scheme (the top entry's id against the id at save) is right; pixi takes it.
- **Pixi's `restoreState`** puts back pixels and leaves history and `bookmark` as they were, so an
  undo after a demo seeks back applies a change to pixels that no longer match it. It drops history
  (as text's does) and restores the unsaved mark it captured.
- **Pixi's `History.append`** reports the edit and clears redo before it decides to drop a
  duplicate. It decides first.
- **Pixi's resize side stacks** are cleared unevenly (layer data, not animations or sprites), and
  `Pixels.temporary` is set nowhere and makes an undone change vanish from the line. Delete it.
- **Text's typing** never reports its edits; the doc comment says every edit does.

## Steps

Each a PR, useful without the next, in this order. Amended 2026-10-16 to share the recording
plan's channel (above).

1. **The small fixes,** the agent's own-document undo (fizzyedit/agent#21) and the untitled
   recording (fizzyedit/agent#20) first.
2. **sdk: authors.** `Author`, `act`/`endAct`/`author`, the frame's input author where input is
   consumed, the interned per-document author map filled in `Host.recordStep`, runs, `Step.top`
   and the host's top per document, `mayMerge`; tapes (`automation.play` taking an author) act.
   Bubbles name others; `test-integration` asserts nothing is unattributed. The fingerprint moves
   once, here. After the SDK release that carries it, one PR in each of agent (`agentCall` acts),
   chat (its ACP writes act) and pixi (waiting calls keep their author and run).
3. **text and pixi report every step.** Text calls `recordStep` for every step, not only while
   recording, fills `top`, and passes `mayMerge` into `History.end`. Pixi's step ids, its
   `recordStep` with effects and `applyStep`, `historyLine` and `historyGoTo` are one PR: this
   step, `RECORDING_PLAN.md`'s step 4 and the timeline plan's step 3 at once.
4. **Replaced by tracks** (`RECORDING_PLAN.md`): a track has every hand's steps, labelled by author,
   so an agent's session tape is the track's range during its session, and the agent's own
   recorder goes.
5. **app: undo says whose.** Amended 2026-10-17 for `CONTROL_SERVER_PLAN.md`, which replaces the
   agent plugin's socket: nothing here is built in the agent plugin.
   - 5a. `fizzy.undo`/`fizzy.redo` take `path` and `expect`, over any owner's `historyLine` and
     `historyGoTo`; a caller's `expect` not on top refuses. Its answer names the step and whose
     it was.
   - 5b. The label ("Undo Claude's Draw") and the bubble after it, with its Redo.
   - 5c. The steps a call made, by its run, and the fact of another's step undone, for the control
     server's answers and its `documents` topic (its step 2) to carry.
   - Pixi's undo waiting out the person's gesture is a pixi PR after 5a.
6. **app: "Go back to before Claude's change",** from the bubble and the timeline. It needs each
   owner's "going back is a step" (the timeline plan's step 4), or going back and editing would
   clear the agent's work.

Steps 2 and 3 say who did what; tracks keep it; 5 and 6 let the person take an agent's work back,
one step or a whole call.

## Amendments to `HISTORY_TIMELINE_PLAN.md`

- **Who made each step:** the acting author, attached in `Host.recordStep`, not the claim holder.
- **Read-only while claimed:** only while explicitly claimed (`fizzy_claim`). Beside the person,
  the line is shared; dragging it moves both hands' steps (agents may not drag it), and going back
  is a step, so nothing is lost.
- **The owner reports the step** (`Host.recordStep`, with `Step.top`), so the timeline need not
  ask for the whole line on every edit to learn the newest id.
- **Going back is a step:** the reverses it appends are the jumper's (whoever moved the line
  back), not the author's whose edit triggered them; owners report them as `reversed`.
- **Step 5, recording a session:** the person's is `RECORDER_PLAN.md`'s step 6; an agent's is this
  plan's step 4.

## Open

- **History across a restart.** Both owners drop it; keeping it means owners writing history into
  `captureDocumentState` and the author map going beside it in `KeptDocuments`. Worth it once the
  timeline makes history something people lean on.
- **A cap.** Neither history is capped, and going back as steps grows them: one cap per document,
  a setting, dropping the oldest steps and their authors.
- **Selective undo,** above: its own plan if asked for.
