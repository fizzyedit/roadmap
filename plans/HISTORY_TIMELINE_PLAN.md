# A document's history as a timeline

Status: proposed. Amended by `SHARED_HISTORY_PLAN.md` (who made each step, now that agents
work beside the person without a claim; recording a session), which lists the amendments.

Undo and redo walk a document's history one step at a time, and redo is lost the moment you edit
after undoing. This plan makes that history something you can see and scrub: a line under the
document with every change on it, whoever made it (you, an agent, a tape), that you drag back to
any point and forward again. Going back and editing loses nothing: the jump back becomes steps
on the line, so the work you went back over is still there, earlier on it. Undo and redo stay as
they are: they are one step along the same line.

Recording a whole session as a tape, to watch or share, is a second, separate piece (the last
part of this plan). It sits on top of the timeline. It is not what the timeline is built from.

## Why each document's own history, and not a tape

Every document plugin already keeps an exact record of its changes: its undo history. Pixi's
`History` holds every change (pixels, layers, animations, grid), and text's `textcore/History`
groups keystrokes into one entry per run of typing, which is the right size for a step on a
line. Going to a point in that history is undoing or redoing to it: exact, instant, and needing
nothing new from the document.

A tape replays forward from a snapshot. Reaching an arbitrary point means restoring the nearest
snapshot and replaying commands up to it. That is right for watching a session play, and too slow
and too fragile (input that only replays in the same window and layout) for something you lean
on every minute. So the timeline is the document's history, shown; tapes stay what they are.

## What there is

- **Pixi:** `History` has an undo stack and a redo stack of `Change`s. `append` clears redo.
  `undoRedo` applies one step either way. Since fizzyedit/pixi#27, each `append` and `undoRedo`
  tells fizzy (`host.documentEdited`). Nothing caps the stacks.
- **Text:** `textcore/History` has `undo_stack` and `redo_stack` of `Entry`s, each with an `id`
  bumped on every change (dirty tracking compares it). A new edit clears redo. Text also calls
  `documentEdited`.
- **Fizzy:** the activity feed (`app/ActivityFeed.zig`) shows each change as it happens, from
  `Host.documentEdited` and the command running at the time (`Host.running_command`). Claims say
  who is working on a document (`sdk.Claim`). Undo and redo are commands each owner implements.

## The hook

Two members on the document vtable, both optional. An owner without them has no timeline, and
undo and redo work as before.

```zig
/// `doc`'s history as a line: every step, oldest first, and how many of them are applied
/// (the rest are what redo would bring back). `version` changes whenever the line does, so
/// fizzy asks again only then. Entries and titles live in `arena`.
historyLine: ?*const fn (state: *anyopaque, doc: DocHandle, arena: Allocator) HistoryLine = null,
/// Undo or redo until the step `id` is the last one applied; 0 is before the first.
historyGoTo: ?*const fn (state: *anyopaque, doc: DocHandle, id: u64) anyerror!void = null,

pub const HistoryLine = struct {
    version: u64,
    entries: []const Entry,
    /// How many of `entries` are applied.
    at: usize,
    pub const Entry = struct {
        /// Stable for the step's life: fizzy keys who made it, and where to go, by it.
        id: u64,
        /// What it did, short: "Draw", "Typing", "Add Layer".
        title: []const u8,
    };
};
```

- **Steps are named by id, not position.** Going to a step survives the line changing under
  the drag, and the line growing when going back becomes steps (below).
- **`version` keeps it cheap.** Fizzy keeps the last line and asks again only when the
  version moved, and only while a timeline is showing. Nothing runs per frame for a document
  whose timeline is shut.
- **Who made each step is fizzy's, not the owner's.** When `documentEdited(doc)` arrives, fizzy
  asks for the line (its version moved) and records, for the new last step's id: the claim
  holder's label if the document is claimed, "a tape" while one plays, and otherwise the person.
  Owners keep nothing new. A step from before fizzy started (a reload) has no author.

Text implements it over `undo_stack` and `redo_stack`, with `Entry.id` as the step id and its
`EditKind` as the title. Pixi needs a step id on each `Change`, a counter in `History`, and takes
the title from the change type.

## The timeline

A strip along the bottom of the document's pane, opened per document (a command,
`fizzy.history`, and a button on the pane), in the glass of the demo player's scrubber:

- **Steps as ticks**, coloured by who made them: you in the theme's text colour, an agent in its
  highlight, a tape dimmer. A run of one author's steps reads as one stretch.
- **A knob at `at`.** Dragging it goes to the step under it: one `historyGoTo` a frame at most,
  to the latest target, so a fast drag over a thousand pixi steps costs one jump per frame, not
  one per step crossed.
- **Hovering a tick** shows its title and author. The activity feed's toasts are the same steps
  as they arrive.
- **A claimed document's timeline is read-only** while it is held: the person watches, and does
  not drag the agent's work away under it.

## Going back is a step

Today going back and editing clears redo. With a timeline that loses work you can see: drag back
over twenty steps to look at something, touch the canvas, and the twenty steps are gone. Worse
when they were an agent's two minutes of drawing.

So the line never loses anything, and stays one line. When you edit after going back, the owner
first records the jump back itself, as ordinary steps: each undone step's reverse, appended
after the last step there was. Then your edit goes on after them. Nothing is cleared. The work
you went back over is still on the line, before the jump, and dragging there brings it back
like any other point.

- **No branches.** No forks to draw, no branch to pick, no new hook members: `historyLine` and
  `historyGoTo` already say everything. The strip shows a jump's steps like any others, titled
  "Back to <step>" so a run of them reads as one move.
- **In each owner, one place:** where `append` clears redo today, it appends the redo entries'
  reverses instead, newest first, and then the new change. Pixi and text already apply each
  change both ways for undo and redo, so a reverse is a change they know how to make.
- **Only when you edit.** Dragging back and forth only moves `at`. The line grows when an edit
  lands after going back, by as many steps as were gone back over.
- **The cost is length.** The line grows by the steps gone back over, each time. A cap per
  document (a setting) drops the oldest steps when it is reached, as an undo history does.

## Recording a session (opt-in)

Separately from the timeline, a person can record their session as a tape, as an agent's
session already is (fizzyedit/agent#6): a snapshot of each document they touch, then each command
they run, spaced as they ran. It is for showing someone what happened, or replaying a bug. It is
not how you go back.

- **Off by default and clearly on when on:** a setting, and a dot on the infobar while recording.
  Off costs nothing: the recorder is a `documentEdited` and command listener that is not
  subscribed.
- **Commands replay; raw input mostly doesn't.** A step that wasn't a command (a stroke painted
  by hand) only replays from raw input in the same window and layout. The owner closes that gap
  by recording a hand-made step as the command that would make it (a finished pixi stroke as one
  `pixi.draw`), through a hook named when this step starts.
- **Privacy:** recording keeps what you typed. Recorded tapes stay in the profile's `tapes/`,
  and nothing leaves the machine unless the person shares the file.

## Steps

1. **sdk: the history hook, implemented by text.** `historyLine` and `historyGoTo` on the
   document vtable (the fingerprint moves), text's implementation, and a headless test: type,
   undo twice, go to a step by id, check the text.
2. **app: a document's timeline.** The strip, the command and the button. Who made each step,
   recorded on `documentEdited`. Read-only while the document is claimed.
3. **pixi: its history on the line.** Step ids on `Change`, titles from the change type. A store
   release after the SDK release that carries step 1.
4. **text, then pixi: going back is a step.** An edit after going back appends the reverses of
   the steps gone back over instead of clearing them, then the edit.
5. **app: record a session (opt-in),** generalizing the agent's `Recording` to the person's
   commands, with the setting and the infobar dot.
6. **pixi: a stroke records as a command,** so a recorded session replays hand-drawn work.

Steps 1 to 3 are the timeline. 4 makes it lose nothing. 5 and 6 are recording, and can wait.
