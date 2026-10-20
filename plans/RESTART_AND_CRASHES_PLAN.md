# Restarts that keep everything, and crash reports

Status: in progress. Steps 1, 2, 4 and 5 done (fizzyedit/fizzy#346, #348, #354, #356, #361; agent `ddcaa46`); step 3 continues (#351 merged; the plugin panic message and the launch notice next).

fizzy restarts for several reasons: an update was downloaded, a setting only applies at launch,
and (for anyone working on fizzy itself) the binary was rebuilt. It also crashes sometimes. Today
every one of these costs the person their session. A restart asks about unsaved documents first,
and "Quit Without Saving" or "Save All" closes those documents, so they are gone from
`layout.zon`. A crash loses everything typed since the last save and says nothing about why.

This plan makes a restart lose nothing, makes a crash lose at most a few seconds and say whose
code crashed, and gives agents working on fizzy the same loop plugins have (build, and the running
app picks it up). It replaces the `fizzy_host` tool AGENTS_PLAN once proposed. The restart,
session and crash pieces live in fizzy, where people benefit from them too. The agent bridge
keeps only what has to run outside fizzy.

## What exists

- `app/restart.zig`: one path for every restart. It posts the ordinary quit and, after teardown,
  starts the new version (through Velopack when an update waits) or this executable again.
- `src/editor/KeptDocuments.zig` (fizzyedit/fizzy#334): a plugin's open documents carried across
  its reload. It captures each document's place and its state (`captureDocumentState`: contents,
  caret, scroll, dirty), then loads it back from state with no disk read and restores it. A new
  build that cannot read old state opens the file as saved, and the unsaved contents go to a
  recovery file the person is told about.
- `layout.zon` keeps every tab, documents included (`<owner>.doc:<path>` assignments). At launch
  the workbench opens each one through `host.openFile` (`workbench_layout.zig`).
- The single-instance socket (`app/single_instance`), and the backend's window geometry
  persistence.

## Step 1: a restart keeps open documents

When a restart is under way, the quit asks nothing. Every document's state is written to
`<config>/session/`, and the tabs stay in `layout.zon`. At launch, a document the session holds
opens from its kept state instead of the disk, with unsaved edits, caret and scroll as they were.

- **Disk changed under a clean document** (an agent edited the file, or `git pull` ran): a clean
  document's state is used only when the file's size and modification time still match. Otherwise
  it opens from disk. A dirty document always comes back as it was, since those edits are the
  person's.
- **Cannot capture:** if a dirty document's owner has no state hook, the restart falls back to
  today's question.
- **Read once:** the session is deleted once read, so a later crash never applies stale state.
- **Untitled documents** come back too. Their state fills them, as on a plugin reload.
- **The restart keeps the arguments it was started with** (`--profile` above all). Today
  `relaunchSelf` drops them, so a restarted sandbox comes back on the person's own profile.

## Step 2: restart into a rebuilt binary

**In review, fizzyedit/fizzy#348:** `app/watch/ExecutableWatcher.zig` and the `restart_on_rebuild`
setting (on where the build is not Velopack's). Measured on macOS: the new process 301 ms after
the install's rename, documents back and answering at 1.2 s. On Windows, installing over a running
executable fails, so the install has to rename it aside first.

A setting, on by default in a source build: fizzy watches its own executable as it watches plugin
binaries, and restarts into the new build keeping everything (step 1).

**Windows, merged, fizzyedit/fizzy#354:** the build renames a running `fizzy.exe` aside before
installing (Windows allows a rename, not an overwrite), and the next launch deletes the leftovers.
Verified in the Windows 11 VM.

**The handover, merged, fizzyedit/fizzy#356:** as below, through `<config>/handover/{ready,released}`.
Measured: both windows on screen together for ~30 ms (macOS) and ~85 ms (Windows), never neither; a
new build that never comes up leaves the old instance running after 30 s. Needed the agent plugin
to stop safely while another instance holds its socket path (agent `8475e12`).

Then the handover: the new process starts hidden and loads its plugins while the old one keeps
drawing. Over the single-instance socket the old one hands over the session and exits, and the
new window shows in the same rect. The restart looks like a plugin swap, not a window closing.

## Step 3: crash reports

Modelled on bun.report (Bun's crash reporter), with plugins as a first-class part:

- **Capture, in-process, without allocating:** fizzy's own panic and segfault handlers walk the
  stack. Each frame becomes (module, offset): a fixed module table, updated on every plugin load
  and reload, maps an address to fizzy or a plugin's id, version and build id (Mach-O `LC_UUID`,
  ELF build-id, PDB GUID). The report goes to a file opened in advance in
  `<config>/crashes/`, and to stderr, with a ring of the last log lines.
- **The report string:** about 150 bytes. It holds the version, the platform, the SDK
  fingerprint, the loaded plugins, risky features in use (floats, glass, a live tape), the crash
  kind, and the frames as module index plus VLQ-encoded offset. It is paste-able and holds no user
  data.
- **Symbols:** Debug and source builds resolve the frames in-process (Zig's own DWARF reader).
  Release builds keep their debug info as a release asset, by build id, for fizzy and for every
  plugin (plugin-build-action uploads it beside the binaries). Store plugins publish theirs the
  same way.
- **On the next launch:** fizzy says it crashed and whose code it was. With the person's OK it
  fetches that build's symbols and resolves the report locally. Nothing leaves the machine unless
  the person chooses to file it, and "Report" opens a prefilled issue on the right repo: pixi's
  for a pixi crash.
- **Safe mode:** a plugin that crashed fizzy is offered to be left off for the next launch.
- **Frame pointers** stay on in release builds of fizzy and plugins, so the walk is reliable.
  Measure the cost before deciding.
- **The web:** a wasm trap reaches JavaScript with a stack. Same report string, later.

## Step 4: a periodic session checkpoint

**Merged, fizzyedit/fizzy#361:** `src/editor/Checkpoint.zig`. Looked at once a second (dirty flags
only), written about a second after input and at least every 30 s while anything is unsaved,
captured on the UI thread and written on a thread of its own to `checkpoint.new/`, which replaces
`checkpoint/` once whole. A dvui timer wakes an idle app when a look is due. Measured: 5.3 MB
checkpointed with the worst frame at 4.0 ms against a 3.6 ms average. The crash notice reads
`KeptDocuments.recoveredFromCheckpoint()`. Written in place by generation, never swapping the
folder under the settings watcher (which ignores the app's state folders), after Linux CI caught
inotify's NOENT on a folder renamed away.

Step 1's session written every few seconds while something is dirty, off the UI thread. After a
crash, the next launch offers it back. This is also where a "hot exit" setting goes (quit never
asks; everything is there next launch), since it is the same file.

## Step 5: the agent's side

**Built, fizzyedit/agent `ddcaa46`:** `fizzy_build` given an app's repo (no `plugin.zig.zon`) runs in
the bridge (`bridge/app_build.zig`), which outlives the restart. It answers compile errors as data
and leaves fizzy alone when the build fails. Otherwise it waits for the old fizzy's connection to
break and a new `connection.zon` token to answer, then runs the check. Measured on macOS: errors
in 7 s; a real rebuild had fizzy gone 291 ms after the build and the new build answering at 1.2 s,
its unsaved document intact. When fizzy quits and does not come back, the report says so; the
crash report (step 3) is what it will attach.

- `fizzy_build` accepts fizzy's own repo. It builds, waits for the running fizzy to restart into
  the new binary (step 2), and runs the check.
- When fizzy is gone, the bridge answers with the newest crash report in the profile, already
  resolved in a Debug build, rather than "connection refused". It can launch fizzy when it is not
  running.

## Open

- ~~Show a window after its first frame is drawn~~: in review, fizzyedit/fizzy#378 (macOS transparent,
  Windows DWM-cloaked until the first frame is presented; a handover signals ready once on screen).
- nightwatch upstream: skip a folder that is gone by the time its watch is added (NOENT), rather
  than logging an error.
- A "hot exit" setting: quit never asks; the checkpoint brings everything back.

- Whether the session belongs in `app/` from the start (the framework every app on fizzy gets)
  or moves there once crash checkpoints need it. Step 1 builds on `KeptDocuments`, which lives in
  `src/editor` today.
- A hosted resolver for report strings (bun.report's shape), if pasting into a local fizzy proves
  too awkward.
