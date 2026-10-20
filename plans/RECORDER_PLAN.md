# The black box: always recording, replayed on demand

Status: in progress. Step 1 in review, fizzyedit/fizzy#386.

Fizzy keeps the last minute of what happened to it, input by input and frame by frame, for
almost nothing, and can hand any stretch of it to an agent as a capture that replays exactly: the
same events into the same frames, from the same documents, on a clock that says the same times.
A person says "it stuttered just then"; the capture already exists. An agent changes the code,
replays the capture headless at many times real speed, and reads what got faster, what got
slower, and whether the stutter is gone, in one call, in a second or two.

It builds on what there is: the binary tape and its player (`sdk/tape`, `sdk/replay`; silent
frames, snapshots, the determinism hash), the profiler's lookback, frame causes and spans
(`core.profile`), `KeptDocuments` (what a restart and the crash checkpoint already capture and
reopen), and the agent plugin's `fizzy_run` and `fizzy_build`. AUTOMATION_PLAN's recorder (input
aimed at anchors, for authored demos) is the portable layer on top of this one (step 6).

## Principles

- **Frame-exact, then portable.** The black box keeps the events dvui handed each frame, as they
  were: positions in the window's pixels, modifiers, key down and up, text as it arrived. Replayed
  into the same frame numbers of a window the same size, from the same start, they reproduce the
  run. Resolving events to anchors (so a tape survives a layout change) is a conversion done
  afterwards, when someone wants a demo, never on the hot path.
- **Nothing costs a frame.** Recording is a copy of the frame's events into a fixed ring: no
  allocation, no lock (the UI thread is the only writer), no string formatting, no widget walk.
  Target: under 10 µs a frame, measured, and the ring a fixed couple of megabytes.
- **On by default, written only when asked.** The ring lives in memory and holds what was typed,
  so nothing of it reaches disk except a capture someone asked for (a command, an agent, a hitch
  capture they turned on). A setting turns the recorder off.
- **Replay is a run, not a movie.** A replay drives the real app, its plugins and its profiler;
  what comes back is the profiler's report of that run beside the recorded one, compared.
- **Every seam names its non-agent consumer.** A person files a stutter with a capture attached;
  a crash report carries the minute before it; a plugin author replays a user's capture against
  their fix.

## Not only fizzy's input: any stream, and rewinding

The same machinery is a game's replay and rewind, so it is built as two layers:

- **`tape.ring`**: a ring of any stream. `Ring(name, Frame, Record, caps)` keeps the last
  `caps.frames` frames, each naming its fixed `extern` records, with each record's payload in a byte
  ring. A frame needs only `t_ns`, `first` and `count`; everything else is the stream's (a dvui
  window's size, a game's tick and timestep). A record needs only `bytes_at` and `bytes_len`, and
  may say what a valid one is. Capacities are the stream's: a minute of fizzy is 1.3 MB, and a
  game can keep ten minutes of gamepad state. A file names its stream and record sizes, so a
  capture holding several (fizzy's input beside a game's) never mistakes one for another.
- **`tape.events`**: dvui's stream over it (`"dvui"`), which `replay.BlackBox` fills.

Going back is the ring plus the state it starts from, which is already a seam:
`replay.Stage`'s `capture`, `restore` and `fingerprint`. Rewinding means restoring the nearest kept
state, then feeding each frame's records from there, silently, to the frame asked for. The
fingerprint says whether the run came out the same. Demo seeking works this way today. Anchors
(step 2) are fizzy's states, and a game's snapshots of its world are its own. What a game engine
plugin brings:

- its input stream: its own `Ring`, recorded at its own tick and not once per dvui frame, so a
  fixed-timestep simulation replays tick for tick;
- its state: a `Stage` with `capture`/`restore` (cheap, in memory, every N ticks) and a
  `fingerprint` of its world;
- its clock: replay feeds ticks, not wall time, and the engine steps once per recorded frame.

Named seams for it, not built until a game plugin asks:

- **Streams in a capture.** A service (`recorder`) where a plugin registers its ring's
  `writeLast`, so `fizzy.capture` writes `<stream>.bin` beside `input.bin`.
- **A timeline.** Scrub and rewind UI over any ring plus a `Stage`, the demo transport bar
  generalized. It is the person-facing consumer of rewind in fizzy too: "go back ten seconds"
  in a document.

## What a capture is

A directory, `<config>/captures/<when>-<why>/`:

| File | What |
|---|---|
| `input.bin` | The frames and their events, from the anchor to the capture: per frame its number, its time (ns since the anchor), the window's size and scale when they changed, and its events (fixed records; text in a byte table). |
| `anchor/` | The app's state at the stretch's start: a `KeptDocuments` generation (every open document, its unsaved text, the session's layout) plus the settings. |
| `report.zon` | What the profiler saw over the stretch as recorded: `core.profile.report` with its lookback, hitches, frame causes and spans. |
| `meta.zon` | The build (version, commit, optimize mode), each plugin's id, version and build, the window, the reason (asked, hitch, crash), the span of time. |

Capturing is a copy of the ring's stretch and the anchor's files that already exist: a few
milliseconds, on the UI thread for the copy and a worker for the write.

## The pieces

- **The ring** (`sdk/replay/BlackBox.zig` over `tape.events`, std-only in `sdk/tape/ring.zig`
  and `sdk/tape/events.zig` so they test alone): frames and events in two fixed rings, a byte
  ring for payloads (text, a selection's offsets), each record numbered so a stretch is a range. Fed once a frame from `dvui.events()` right after
  `Window.begin`, the point the player injects at, so what is recorded is exactly what the app
  saw, including what a player or an agent's live tape injected. dvui-only, so any dvui app can
  keep one (the `replay` library's audience).
- **Anchors**: the state the ring's stretch starts from. A `KeptDocuments` capture taken when the
  recorder starts and again at idle moments (no input for half a second, at most every ten
  seconds), on the UI thread as the checkpoint takes its own and written by a worker; the last two
  are kept, so the ring always reaches back to one. Sharing the checkpoint's machinery means one
  way to capture state, already proven by restart and crash recovery.
- **Replay** (`fizzy --replay <capture> [--headless] [--fast]`): a fresh profile of its own, the
  anchor opened as a restart's session opens, the window sized as recorded, then each recorded
  frame's events injected into that frame, the clock reading the recorded times. `--fast` runs the
  frames back to back as silent frames (no vsync, no present): a minute of interaction in a couple
  of seconds. The profiler records throughout; the run ends with its report, the recorded one
  beside it, and the comparison: per scope, per plugin, the worst frames, whether each recorded
  hitch came back.
- **Determinism**, checked rather than hoped for: the replay compares the model's hash
  (`Stage`'s, as seeking does) at the end and at each anchor; a mismatch is reported with the
  first frame it differs at. Work on other threads (a file load, a language server, the network)
  finishes when it finishes; frame causes say where a replay waited on it.

## For agents

- `fizzy_capture { seconds, why }` → the capture's path and its report's summary.
- `fizzy_replay { capture, fast, headless }` → the run's report and the comparison with the
  capture's. After `fizzy_build`, the check that a change fixed the thing: build, reload, replay
  the capture, compare, one turn.
- Hitch captures announce themselves (an MCP notification), so an agent watching a person's
  session hears "frame 40 ms at 12:03:51, captured" with the call tree attached.

## Performance targets

| What | Target |
|---|---|
| Recording, a frame with no input | < 2 µs |
| Recording, a frame with input | < 10 µs |
| Ring | ≤ 2 MB for the last minute at 120 fps with steady input |
| Capture written | < 20 ms, off the UI thread except the ring copy (< 1 ms) |
| Replay `--fast` | ≥ 10× real time for an ordinary minute (frames ~1–3 ms) |
| Replay start (anchor opened, window up) | < 500 ms headless |

## Steps

Each a PR, useful without the next.

1. **The ring and `fizzy.capture`.** `Raw` records and rings (std-only, tested, benchmarked),
   `BlackBox` fed from the frame, a capture writing `input.bin`, `report.zon` and `meta.zon`; the
   setting. No anchor yet: a capture is replayable only from a fresh start.
2. **Anchors.** Idle-time `KeptDocuments` captures kept for the ring; a capture includes the one
   its stretch starts at.
3. **Replay.** `fizzy --replay` with `--headless` and `--fast`; the report and the comparison;
   the determinism hash.
4. **Agent tools.** `fizzy_capture` and `fizzy_replay` in the agent plugin; `fizzy_build` can
   replay as its check.
5. **Automatic captures.** On a hitch over a budget (bounded count, opt-in), and with a crash
   report (the crash handler writes the ring beside it); the MCP notification.
6. **Portable tapes.** Raw events resolved to anchors, AUTOMATION_PLAN's recorder: a capture made
   into a demo, or replayed across builds whose layout moved.

## Open

- Password fields: typed text into a password entry should never be kept. The ring cannot know
  where text goes when it records; dvui marking such events, or the entry itself, is the seam.
- How much state outside documents and layout a faithful replay needs (a plugin's own model, a
  terminal's session): each plugin's `captureDocumentState` covers its documents; other state is
  a plugin hook to come.
- Replaying on the web (no `--replay` process): an in-page replay over the same records.
