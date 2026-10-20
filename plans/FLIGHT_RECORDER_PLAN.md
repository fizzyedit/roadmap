# The flight recorder: "Report a problem" with the last minute attached

Status: proposed.

Fizzy already keeps the last minute of what it was handed, in memory, for almost nothing (the
black box, `RECORDER_PLAN.md`, fizzyedit/fizzy#386). A maintainer cannot use that today: it
never leaves the person's machine, it holds what they typed, a replay needs the documents it
started from, and nothing says which build, plugins and settings it ran under. This plan is the
step from a recorder to a **report**: one file a person saves from Help > Report a Problem,
that a maintainer opens in fizzy and watches, rewinds and replays, without either side
setting anything up.

```
 ring (always on, memory) ─┐
 start state (layout, docs)├─▶ review (the person sees and chooses) ─▶ report.zip ─▶ attach / hand over
 versions, settings, logs ─┘                                                          │
                                                   maintainer: Open Report ◀──────────┘
                                                   (sandbox profile, start restored, replay, rewind)
```

It builds on and does not redo: the black box and captures (`sdk/replay/BlackBox.zig`,
`src/editor/captures.zig`, `RECORDER_PLAN.md` steps 1-3), the tape library (`sdk/tape`,
`sdk/replay`), demo automation (`app/automation`, `docs/AUTOMATION.md`, `AUTOMATION_PLAN.md`),
and the shared tracks (`app/Tracks.zig`, `SHARED_HISTORY_PLAN.md`, `RECORDING_PLAN.md`).

## What there is

- **The ring.** `replay.BlackBox.record` copies each frame's dvui events into fixed rings
  (`sdk/replay/BlackBox.zig:34`); measured 0.5 ns a frame with none, 5 ns with four, 1.3 MB,
  allocated on the first frame that records (`:5-12`). Fed once a frame from
  `src/Entry.zig:556-557`, gated by the `black_box` setting, default on (`app/settings/Settings.zig:119`).
  Each stretch is written by `ring.writeLast` (`sdk/tape/ring.zig:217`), the stream named in its header.
- **A capture** is `input.bin`, `report.zon` (profiler), `meta.zon` (build, plugins, window),
  written to `<config>/captures/<when>-<why>/` by `fizzy.capture` (`src/editor/captures.zig:25`,
  `src/editor/Keybinds.zig:182`). It holds **no start state** (`captures.zig:12-13`): replayable
  from a fresh start only. `fizzy --replay` (RECORDER step 3) is not built.
- **Start state is captured three ways already.** `KeptDocuments` (documents, place and state
  through the owners' `captureDocumentState`; `src/editor/KeptDocuments.zig`), written by the
  crash `Checkpoint` off the UI thread (`src/editor/Checkpoint.zig`); `layout.zon`; and
  `Tracks`: per document a base state, a checkpoint every 256 steps or 30 s, and every step with
  its time, in memory and kept in the profile when a document closes (`app/Tracks.zig:1-60`,
  `:93-96`, `track_file.zig`). `Tracks.replayInto` puts a document back at any step (`:719`).
- **Replay machinery.** `Player` seeks from snapshots it takes at calm moments, silently, with a
  fingerprint check (`docs/AUTOMATION.md` "Snapshots make the way back short"); `Stage.capture /
  restore / fingerprint`; optional owner hook `documentFingerprint`. `LiveDriver` plays input into
  the running app (`sdk/replay/LiveDriver.zig:104`).
- **Anchors.** `core.anchor.mark` names a widget only while someone asked this frame
  (`sdk/replay/anchor.zig:36-60`); names are listed in `docs/AUTOMATION.md`.
- **Settings and logs.** Settings fields can be `secret` (`sdk/src/settings.zig:80`); the output
  log is an in-memory ring of 2000 lines (`src/editor/OutputLog.zig`). dvui marks password entries
  with `role = .password_input` (dvui `TextEntryWidget.zig:123`).
- **Crash reports** (`RESTART_AND_CRASHES_PLAN.md` step 3) are specified to "hold no user data"
  and RECORDER step 5 has the crash handler write the ring beside them; the two collide, see
  Privacy.

## Principles

1. **Recording stays what it is: a copy into a fixed ring.** Everything this plan adds happens
   when the person asks for a report, or on an event that already happens (a layout change, a
   document step), never on a clock and never per frame.
2. **Nothing leaves the machine until the person has seen what is in it and clicked.** The
   review shows the contents, sizes and the typed text; a report is a file they hold.
3. **Anything written without a click is the redacted form.** Crash files, hitch captures.
4. **Two layers of replay, so a report is useful when one fails.** Frame-exact input replay
   (the ring) reproduces a UI bug in the same layout; the documents' tracks (effects) rewind a
   document's content on any machine, layout or build. A bundle carries both where chosen.
5. **A plugin publishes what it already has**: anchors (`core.anchor.mark`), its document hooks
   (`captureDocumentState`, `restoreDocumentState`, `documentFingerprint`), and a marker on a
   widget that holds a secret. No new recorder API per plugin.
6. **Plugins meet only at the SDK; shapes are copied.** The review dialog is ordinary code over
   the public dialog API, so an app built on fizzy can copy it or drop it.

## Cost: zero when idle, measured first

What recording costs is already a memcpy per frame with events and a branch without. What this
plan adds, and what each costs:

| Addition | When it runs | Cost target |
|---|---|---|
| Ring record (exists) | each frame that runs | < 1 µs, no allocation after the first frame |
| Secret check | only in frames that carry a text or key event, one `dvui.dataGet` | < 50 ns on those frames, 0 otherwise |
| Layout anchor | when the layout changes (a drag, a split), 1-4 KB of ZON, last two kept | < 0.2 ms per change, none idle |
| Document start state | none at record time: the report takes the nearest `Tracks` checkpoint before the ring's first frame, and the steps since | nothing idle; at report time a few ms, off the UI thread where the owner allows |
| Document with no track (image, chat, archive) | not kept; reopens from disk as saved, unless chosen | nothing |
| Report build and zip | on the click only; ring copy on the UI thread (< 1 ms), the rest on a worker | not in a frame |

An idle app records nothing: no frames run, no wake-up, the ring is simply not appended to. The
report says so (`ring_ends_ago_ms` in the manifest) so a maintainer knows when the minute ended.

**Measure first.** Step 1 lands before anything else: `zig build bench-blackbox` (ReleaseFast,
beside `bench-replay` and `bench-tracks`) for a frame with 0 / 4 / 64 / 512 events, text events,
the first-frame allocation, the secret check, and the heap in use with the recorder on and off;
plus tests that the second frame allocates nothing and that an idle app's frame counter does not
move because of the recorder. Each later step adds its row to the same bench and the PR quotes
the numbers. A number over target in a later step is that step's to fix or to drop. Timing is
reported, not gated in CI (shared runners are noisy); allocation and wake-ups are gated.

## Privacy: what a tape exposes

A tape is not only clicks.

| In the ring | Exposes |
|---|---|
| `text` events | every character typed or composed in the minute: documents, chat, search boxes, **password fields** |
| `key_down` / `key_up` / `key_repeat` with no ctrl/cmd | the same characters, as key codes. Redacting only `text` leaves the typing readable |
| shortcuts (ctrl/cmd chords) | commands used; little |
| pointer positions, wheel | what was looked at, in window pixels; little |
| window size and scale | display class |
| not in the ring | clipboard content (paste is a key chord; a replay pastes whatever the clipboard holds then), files dropped in (paths come as drop events, if any, not yet recorded) |

Around the ring: documents and tracks hold content outright (a text document's text, a pixi
document's pixels and every stroke); settings hold secrets (`secret` fields), account names,
recent folders and absolute paths; the log holds paths, file names and LSP stderr; `config_folder`
and anchors (`workbench.file:<abs path>`) hold the user name and file names.

**Redaction, by tier.** The review chooses; the default is the first.

1. **Shape** (default). The tape redacted: `text` events keep their length and whitespace and
   become `x` (newlines stay, so a text editor lays out the same), and printable keys with no
   command modifier become one key code of their class (letter, digit, punctuation) so typing
   looks the same and reads as nothing. Chords, arrows, Enter, Tab, Escape, pointer and wheel
   stay. No documents, no tracks. Settings with `secret` and paths removed. Log lines with the
   home folder as `~` and file names kept only when the person ticks "file names". Enough for a
   stutter, a layout or a rendering bug; too little for one that needs the document.
2. **With chosen documents.** The shape tape plus, for each document the person ticks (none by
   default; each shown with its owner and size), its start state and its track. Typed text
   stays redacted unless they also tick "typed text".
3. **Everything.** The raw tape, all open documents, settings with secrets still removed (a
   credential is never exported, at any tier), logs as they are.

**Secret entries.** Redaction by tier cannot save a password typed at tier 3, and a password
must never be kept in the ring at all. The seam: a widget that holds a secret publishes, while it
has focus, `core.secret.mark(wd)` (a `dvui.dataSet` of the frame time on the shared window, as
`core.anchor` does; the password entry and every `secret` settings field call it). `BlackBox.record`
reads it only in a frame with a text or key event, and while it is the frame's or the previous
frame's, **drops those events from the record** (a gap the replay shows as "typed text withheld").
dvui's own password entry sets `role = .password_input` already; fizzy's wrapper marks it. A
plugin with its own secret field marks it the same way (documented in `docs/PLUGINS.md`).
Over-recording a secret into memory is the failure to avoid, so this is step 2, before any file
holds a ring.

**Consent.** The ring is in memory, on by default, and nothing is written but by a click (the
existing rule, `Settings.zig:116-118`). Added: a one-time notice the first time the recorder
runs (an infobar toast with a link to the setting), `Forget the Last Minute`
(`fizzy.report.forget`, clears the ring and the layout anchors), the minute's length as a
setting (`black_box_seconds`, 15-300, default 60, applies to what is kept and what is written), and
a build switch for an app that wants none (`.@"black-box" = false`).

**Crash collision.** The crash handler cannot run the review. So what a crash writes is the
tier-1 form only (the ring through `tape.redact`, no documents), into `<config>/crashes/` with the
report string, and the next launch offers "Report this crash", which opens the same review
with that tape. RECORDER step 5 and RESTART step 3 are amended accordingly, in their plans, by
step 4 below.

**Commands from an agent.** `fizzy.report.save` takes a tier. A call that does not come from the
person (an agent's, `Command.Call`'s author; `SHARED_HISTORY_PLAN.md`) can ask for tier 1 and
nothing above it; a higher tier opens the review dialog for the person to confirm.

## The bundle

`fizzy-report-<when>.zip`. A zip because GitHub accepts it as an attachment (25 MB on free
plans; `.zip` is on its allowed list and a private extension is not), any machine opens it,
and fizzy's own `archive` plugin mounts it as a folder, so a maintainer can read a report
without a tool. The layout is the capture's, grown; `captures.zig` writes the same files to a
directory when asked for one locally, so there is one format and a directory form of it.

| Path | What |
|---|---|
| `manifest.zon` | `format` (integer), tier, `created`, `ring_seconds`, `ring_ends_ago_ms`, `frames`, the person's description, per-section byte counts and a hash of each file, which sections were redacted and how, `settings_changed_in_window` |
| `input.bin` | the ring (`tape.ring`, stream `dvui`), raw or redacted per tier |
| `versions.zon` | fizzy version, commit, optimize mode, OS, arch, GPU/backend, SDK version and fingerprint, each plugin's id, version, link (static or dynamic) and build id |
| `settings.zon` | the app's and each loaded plugin's settings, secrets and (tier 1) paths removed, with a list of what was removed |
| `report.zon` | the profiler's report over the stretch (exists) |
| `log.txt` | the output log, scrubbed per tier |
| `start/layout.zon` | the layout when the ring's first frame ran |
| `start/docs/<n>/` | per chosen document: `state.bin` (the owner's `captureDocumentState` at the ring's first frame) and `meta.zon` (owner, display name, size) |
| `tracks/<n>.track` | per chosen document: its track in the existing `track_file` format, for rewinding its content |
| `screen.png` | optional; the last frame, only if ticked (it shows content) |

Size: the ring is <= 1.3 MB and deflates well; documents are the rest. The review shows the total
and the report stays under a cap (default 20 MB): over it, the biggest documents are unticked
with an explanation. Versions of the format: a reader accepts the same and older `format`, and a
newer one is refused with the producing fizzy's version, as `track_file` does.

## Starting state: so that a replay lands

A frame-exact replay is only right from the state the ring began in. Three cases, in the order
the report uses them:

1. **Start state from the tracks (documents that record).** The ring's first frame has a time;
   per document `Tracks` has a checkpoint before it and the steps after. The report writes the
   checkpoint's whole state (`Track.wholeState`) as `state.bin` and the track, and the start is
   *that state at that time*: `replayInto` to the step count at the ring's first frame. No
   snapshot is taken at record time, so idle cost stays zero. A document with no steps in the
   whole session still has its base.
2. **Layout anchors.** `layout.zon` is small, so the host keeps its last two serialisations,
   taken on the frame a layout change lands, with the frame number. The one at or before the
   ring's first frame is the start layout. The window's size and scale are in the ring.
3. **Known start.** If the session is shorter than the ring (launched less than a minute ago), the
   report says `start = launch`, no state is written, and replay starts as a fresh profile does.
   Documents with no track and no ticked state open from disk "as saved" and the manifest says
   so; a replay that depends on them is flagged `start_incomplete` rather than silently
   wrong.

Not covered, said plainly: state no owner can capture (a terminal's session, a language
server's, network responses, a clipboard), other threads' timing (RECORDER's determinism
section), and a plugin whose version differs on the maintainer's side (the manifest says so and
the open step warns). The determinism check (`Stage.fingerprint`, `documentFingerprint`) at the
start and the end tells a maintainer whether the replay reproduced the run, and where it first
did not.

**Rewind** is the shared-history strip (`SHARED_HISTORY_PLAN.md`, `HISTORY_TIMELINE_PLAN.md`)
over the restored tracks, for content, and the demo transport over the `Player` for the UI
replay: the player takes snapshots at calm moments while a report plays, so scrubbing back
costs what it does for a demo (2.5 ms mean, `docs/AUTOMATION.md`).

## Anchors: how plugins publish what a replay needs

Frame-exact replay needs none: it feeds the same pixels into the same layout. Anchors matter
when the layout is not the same (another window size, a plugin that moved a button, a later build).
Resolving pointer events to anchors **at record time** would need `anchor.want()` every frame,
which turns on a string format and a tag per anchored widget, against the cost principle. So it is
done **at replay time**, which is free for the person: a maintainer replays the report frame-exact
with anchors wanted, and the host resolves each pointer event to the innermost named widget under
it (RECORDING_PLAN "Aiming without positions") while it goes, producing a portable tape
(RECORDER step 6) as a by-product. The conversion exposes a plugin's coverage:

- **Unnamed targets are reported per plugin**: "pixi: 14 clicks landed on no anchor, the widget
  ids are ...". A plugin author sees what to `core.anchor.mark`, from a user's real session
  rather than a guess.
- **Naming rules** for plugins (added to `docs/AUTOMATION.md` and `docs/PLUGINS.md`): `<plugin id>.<kind>[:<subject>]`;
  a subject that is a path or a title is redactable (the bundle tier 1 rewrites it to a stable
  hash so the same document keeps the same name across the tape); a widget that appears
  once per document is named by document, a repeated widget by index; never a pixel.
- **What else a plugin publishes** so that a replay lands, all optional and already in the SDK:
  `captureDocumentState` / `restoreDocumentState` (start state), `documentFingerprint` (the
  landing check), history steps through `Host.recordStep` (tracks), and `core.secret.mark`.
  A plugin with none of them still replays frame-exact; the manifest lists which of its
  documents were captured and which not.

A plugin-defined landmark ("the stroke ended", "the build finished") is not added now. If replays
keep missing on async work, a `Host.landmark(name)` that the ring stamps into the frame is the seam,
and it waits for a plugin that asks.

## Where reports go

No server is assumed, and fizzy ships none.

1. **A file the person attaches.** Save writes `<config>/reports/fizzy-report-<when>.zip` and
   reveals it (`reveal_path`); the dialog's buttons are Reveal, Copy Path and **Open Issue**, which opens
   the repository's new-issue page in the browser with the title and a body of the manifest's
   non-sensitive lines only (version, OS, plugin table, the person's description) in the query
   string; no tape, state or path ever goes in a URL. The person drags the zip into the issue.
   On the web there is no file system: the bundle is built in memory and handed to a browser
   download.
2. **Which repository.** The dialog asks "Which part is it?" with the active document's owner
   and each loaded plugin as choices (default: the focused surface's owner, else fizzy); the
   plugin's repository comes from its manifest/registry entry (`PLUGIN_MANIFEST_PLAN.md`),
   so a pixi problem opens `fizzyedit/pixi`. A report names every plugin either way.
3. **An upload an app configures.** A setting `report_endpoint` (empty, and absent unless the
   app sets it) makes a second button POST the same zip to a URL the app's owner runs. Fizzy
   builds the request and nothing else: no account, no default host. An app built on fizzy (the
   example app shows it) points it at its own. Not built until an app asks; the review and the
   zip are identical, so adding it is one button.

## Opening a report

`File > Open Report...` and `fizzy --report <zip>` (desktop; on the web `?report=<url>`):

1. Open the zip with the `archive` mount; read `manifest.zon`; refuse a newer `format`.
2. Start a **sandbox profile** (the existing `--profile` mechanism), so the maintainer's own
   session and settings are untouched; apply `settings.zon` over it, which the manifest says is
   what the reporter ran.
3. Compare `versions.zon` with the running build and say what differs (fizzy commit, each plugin);
   a plugin missing is offered from the store at the reported version.
4. Restore the start: layout, then each document through `restoreDocumentState` and its track
   (`pickUp`, `Plugin.restoreHistory`), then hand `input.bin` to RECORDER's replay (step 3) as a
   `Stage`-backed driver. Play, pause, scrub and the fingerprint check are the demo transport's.
   `--fast` and `--headless` are RECORDER's.
5. A tier-1 report has no documents: the dialog says "this report has no documents; the replay
   starts empty" and the replay still shows layout, timing and rendering.

Reports are data: opening one never runs its content as anything but input events, and
`manifest.zon` hashes are checked before a section is read.

## Steps

Each is a PR of at most 800 lines, useful without the next. Steps 1, 4 and 7 can run in the
cloud (Linux, tests only); 2 and 3 run their tests there but need macOS to measure (3 also for real
documents, 2 for a real password field); 5, 6 and 8 need macOS and the person's eye.

**1. Measure the recorder: `bench-blackbox` and the idle gates.**
Repo fizzyedit/fizzy, branch `replay/blackbox-bench`, builds on #386. `sdk/replay/bench` (as
`bench-replay`): frames of 0/4/64/512 events, text, first-frame allocation, heap with the
recorder on and off; tests: the second record allocates nothing, a stopped app's counters do not
move. Quotes the numbers in the PR. Cloud: yes (tests); the bench numbers are from macOS.
Kickoff — Task: add the recorder's bench and idle gates. Plan: plans/FLIGHT_RECORDER_PLAN.md
step 1 "Cost". Builds on: fizzyedit/fizzy#386. Workspace: fizzy worktree flight-recorder-1.
Branch: replay/blackbox-bench. Notes: ReleaseFast; no behaviour change; beside bench-replay and
bench-tracks; report numbers, gate only allocation and wake-ups; don't touch BlackBox.record
except to expose a test hook.

**2. Redaction and the secret seam.**
fizzyedit/fizzy, `replay/redact`, builds on 1. `tape.redact` (std-only, `sdk/tape/redact.zig`):
a function over `events` frames producing the tier-1 form (text to `x` keeping length and
newlines; printable no-modifier keys to a class code), with property tests that no typed byte
survives and that replayed layout (line lengths) is unchanged. `core.secret.mark` and the check in
`BlackBox.record` (frames with text/key events only), the password entry and `secret` settings
fields calling it; docs in `docs/PLUGINS.md`. Cloud: tests yes; a real password field needs the
person's eye.
Kickoff — Task: redact tapes and keep secrets out of the ring. Plan: plans/FLIGHT_RECORDER_PLAN.md
"Privacy" and step 2. Builds on: step 1. Workspace: worktree flight-recorder-2. Branch:
replay/redact. Notes: key codes leak typing as much as text events do, redact both; the secret
check must cost 0 in frames without text/key events, bench it; amend sdk docs for plugins with
their own secret field.

**3. Start state: layout anchors and the tracks slice.**
fizzyedit/fizzy, `app/start-state`, builds on 1. Host keeps the last two layout serialisations on
layout change; `Tracks.startAt(doc_id, t_ns)` returns the nearest checkpoint whole state and
the step count at a time (uses `Track.stateBefore`, `wholeState`, `app/Tracks.zig:162-190`); owners
with no track fall back to `captureDocumentState` at report time, flagged. Bench rows: layout
change cost, `startAt` on a large pixi/text track. Cloud: unit tests for the slice; macOS for real documents.
Kickoff — Task: the state a ring starts from. Plan: plans/FLIGHT_RECORDER_PLAN.md "Starting
state". Builds on: step 1; app/Tracks.zig. Workspace: worktree flight-recorder-3. Branch:
app/start-state. Notes: nothing is captured on a clock; layout snapshot on change only; test a
track that began after the ring's first frame (start = its base) and one that closed and was
picked up again; no UI.

**4. The bundle: zip writer and reader, manifest, scrubbing of secrets.**
fizzyedit/fizzy, `app/report-bundle`, builds on 2 and 3. A std-only stored/deflate zip writer and
reader (check `std.zip`, which reads only) in `sdk/tape/zip.zig` or `app/report/`; `manifest.zon`
with hashes and `format`; `app/report/Bundle.zig` assembling the files in the table from
`captures.zig`'s pieces (which now writes the same layout to a directory); settings export
without `secret` fields and with paths removed; log scrubbing (home to `~`); the report size cap.
Amends RECORDER step 5 and RESTART step 3 in their plans: crash files are tier 1. Cloud: yes.
Kickoff — Task: write and read a report bundle. Plan: plans/FLIGHT_RECORDER_PLAN.md "The bundle".
Builds on: steps 2-3, src/editor/captures.zig. Workspace: worktree flight-recorder-4. Branch:
app/report-bundle. Notes: round-trip and hash tests; a secret setting must never appear in any
file (test every settings field that is `secret`); zip must open in the archive plugin and
GitHub; one format with a directory form; also edit RECORDER_PLAN/RESTART plans in fizzyedit/plans.

**5. Help > Report a Problem: the review dialog.**
fizzyedit/fizzy, `app/report-dialog`, builds on 4. Command `fizzy.report` (menu, palette),
`fizzy.report.save` (tier argument; agent callers capped at tier 1), `fizzy.report.forget`; the
dialog: a description box, "Which part is it?", the tier choice, per-document tick list with
sizes, a view of the typed text the tape holds, total size, Save / Reveal / Copy Path / Open
Issue; first-run notice; `black_box_seconds` setting. Web: download. macOS and the person's eye.
Kickoff — Task: the Report a Problem flow. Plan: plans/FLIGHT_RECORDER_PLAN.md "Privacy",
"Where reports go". Builds on: step 4. Workspace: worktree flight-recorder-5. Branch:
app/report-dialog. Notes: the dialog is plain code over core.dialogs so apps can copy it; the
issue URL carries only version/OS/plugin table/description, never content or paths; ring copy on
the UI thread, zip on a worker; verify with fizzy_screen and a real save; web download untested
until the person tries it.

**6. Open a report.**
fizzyedit/fizzy, `app/report-open`, builds on 4 and RECORDER step 3. `File > Open Report...`,
`--report`, the sandbox profile, the version comparison, restore of layout, documents and tracks,
handing `input.bin` to the replay with the transport and fingerprint check; messages for tier-1
and `start_incomplete` reports. macOS and the person's eye.
Kickoff — Task: open a report and replay it with rewind. Plan: plans/FLIGHT_RECORDER_PLAN.md
"Opening a report". Builds on: step 4, RECORDER_PLAN step 3 (replay) if merged, else stub the
play step. Workspace: worktree flight-recorder-6. Branch: app/report-open. Notes: never touch
the maintainer's own profile; refuse a newer format; test with a real report from step 5 on a
document edit plus a click.

**7. Resolve at replay: portable tapes and plugin coverage.**
fizzyedit/fizzy, `replay/resolve`, builds on 6. A replay mode wanting anchors that resolves each
pointer event to the innermost anchor, writes a portable tape (RECORDER step 6) and a per-plugin
list of unnamed targets; the naming rules and the path-hash option in `docs/AUTOMATION.md` and
`docs/PLUGINS.md`; the `automation` service lists the coverage. Cloud: the resolver over a
fixture frame; yes.
Kickoff — Task: resolve a report's pointer events to anchors at replay time. Plan:
plans/FLIGHT_RECORDER_PLAN.md "Anchors". Builds on: step 6. Workspace: worktree
flight-recorder-7. Branch: replay/resolve. Notes: costs nothing while recording; the unnamed
targets list is the point, per plugin; do not add a plugin-side API; update docs/AUTOMATION.md.

**8. Plugin attribution and the optional upload.**
fizzyedit/fizzy, `app/report-routing`, builds on 5. The plugin picker's repository from the
manifest; a `report_endpoint` setting and the second button, only when set; the example app shows
it. Small. macOS.
Kickoff — Task: route reports to the right repo; optional upload. Plan:
plans/FLIGHT_RECORDER_PLAN.md "Where reports go". Builds on: step 5. Workspace: worktree
flight-recorder-8. Branch: app/report-routing. Notes: ship no default endpoint; the request is
the same zip; also edit fizzyedit/example-app (separate PR) so CI's example check keeps passing.

## Open questions

- **Default tier.** Shape is safe and often not enough to reproduce. Is a first-launch choice
  better than a default, or a "send shape first, ask for more" flow in the issue template?
- **Shape-preserving documents.** An owner hook that returns a document with the same structure
  and none of its content (a text file as lines of `x`, a pixi document as flat colour per
  layer) would make tier 1 replayable. Worth the SDK surface, or leave to owners later?
- **Clipboard and drops.** Paste is not in the tape, so a replay pastes the maintainer's
  clipboard. Record clipboard reads (redacted per tier) or flag the event?
- **Minute length vs privacy.** Is 60 s the right default, and may a build set a ceiling?
- **GitHub's size limit.** Cap at 20 MB, or split the documents into a second attachment?
- **Notice wording and the infobar indicator**: always show that the recorder is on, or only the first time?
- **Web.** Is `?report=<url>` worth it, given a report may hold documents behind a private link?
- **Retention.** How long `<config>/reports` keeps files (the `Tracks` kept cap is the model).
