# The dashboard, and fizzy's graphics for live data

Status: in progress. Step 1 done (fizzyedit/fizzy#345, #347); step 2 in review (#353, #355).

The profiler window (`src/editor/Profiler.zig`, "Toggle Profiler") answers one question well:
where did this frame's time go, by plugin and as a call tree. This plan grows it into the place
a person or an agent looks when they ask "what is this app doing right now, and what is it
costing?", and gives fizzy a small library of graphics for showing live numbers, which the
dashboard is the first user of and an agent the second.

## Principles

- **One model, two readers.** Everything the dashboard draws is data first: a section collects
  into a plain struct, the dashboard draws that struct, and the same struct serialises to ZON for
  an agent's query. A person and an agent never see different numbers, and nothing is scraped from
  a drawing.
- **Nothing costs a frame unless someone is looking.** The rule `core.profile` already keeps
  (`want`, recording only while the window is open or an agent asks) holds for every section. A
  closed dashboard with no agent connected is zero work. Counters that are free to keep (the
  backend's `Health`, integer increments) may run always; anything that sorts, allocates or walks
  a structure runs on request.
- **No privileged reads.** The dashboard reaches the app the way any plugin can: services and
  `core`. If a section needs something only fizzy knows, that is a missing service, added for
  every app on the framework (`fizzyedit/example-app` included), not a reach into `src/`.
- **Plugins bring their own sections.** Pixi's batch counts, atlas's graph size, the text
  plugin's language-server latency: an author shows what matters about their plugin in the same
  place, queryable the same way.

## What it shows

Each is a section: collected on request, drawn from the collected struct, answered as ZON.

| Section | What | Where the data is |
|---|---|---|
| **Frame** | fps, interval, work, submit, frame-time percentiles, presents and frames with no drawable, GPU work per frame, input rate; the last 240 frames as a graph | `core.profile` `FrameStats` + history; the backend's `Health` (#309) |
| **Plugins** | every plugin: id, version, link mode (static, dylib, store, development), state, load generation, failures and why, last reload's open and swap times, self time per frame and share, surfaces drawn, commands and services it registered | the `plugins` service (#339) grown with identity and registrations; `core.profile` owners |
| **Most expensive** | top scopes by self time, by worst single call, by calls per frame, over a chosen window, not only the last half second | `core.profile` entries, kept over a window |
| **Hitches** | the worst frames of the window, each with its call tree as it was in that frame, so a spike is explained rather than averaged away | `core.hitch` phases + a profile tree retained per hitch (new, bounded ring) |
| **Memory** | live bytes and allocation rate per owner (fizzy and each plugin), dvui's texture cache, the wasm heap on the web | new: a counting wrapper on each plugin's `host.allocator` (named seam below) |
| **Documents** | open documents, owner, dirty, size, kept across reloads | the document registry (`App`) |
| **Work** | loading jobs, watchers, child processes (language servers, agents), live tapes, agent connections | `app` job and watch lists, `core.lsp`, the `automation` service |
| **Log** | recent warnings and errors, by source, with counts | the planned `log` service (AGENTS_PLAN) |

**Most expensive** and **Hitches** answer the question the averages hide: the call that took
40 ms once, rather than the one that takes 0.2 ms every frame.

## The graphics library: `core.viz`

Immediate-mode dvui functions over slices the caller owns: no allocation per frame and no
retained state beyond dvui's own, drawn in fizzy's theme. They live in `core/` because an app and
every plugin draw with them.

- `spark` (a line in a run of text), `line` (series over time, several series, live-scrolling),
  `bars` (horizontal, with the share bars the profiler already draws), `histogram`,
  `stat` (a number, its unit, its trend), `gauge` (against a budget, such as 8.3 ms at 120 Hz),
  `timeline` (spans on rows: a frame's scopes as a flame chart, a build's stages),
  `table` (sortable columns with inline bars and sparks), `heat` (a grid of values).
- Each takes the data and a small options struct, and reports hover (the value under the
  pointer) so a caller can hold a frame still, as the profiler's graph does today.
- The profiler's graph and share bars (`drawGraph` and the table) are the first two, moved out of
  `src/editor/Profiler.zig` rather than written again.
- Over dvui's `PlotWidget` where it fits; our own where dvui's does not (spans, heat).

### Live graphics from an agent: views

An agent shows a number or a series without writing a plugin. A **view** is a declarative spec,
data an agent, a tape, or a plugin can produce:

```zig
.{ .id = "build-times", .title = "Build time per change", .kind = .line,
   .unit = "ms", .series = .{ .{ .name = "build", .points = .{ 1710, 520, 480 } } } }
```

- `fizzy_show` (the agent plugin) opens or replaces a view by id; `fizzy_update` appends points
  or changes values without resending the whole view, so a running measurement streams in.
- A view is a document (a `.view` file, `chat`'s pattern): it opens in a tab, splits beside the
  code, survives a reload, and a person can keep it. The chat plugin draws the same spec inline
  in a transcript, so an agent can answer with a chart.
- The kinds are `core.viz`'s, one to one. The spec is the library's declarative face, not a
  second library.
- A view can **bind** to a dashboard section instead of carrying data (`.source =
  "frame.work_ms"`), so an agent can put up "watch this while I change it" without polling.

## Where it lives

- **The data:** services, as above. New ones are `log` and per-owner memory. `plugins` grows its
  `Status` with identity and registrations (additive, no fingerprint move if added as new calls).
- **The graphics:** `core.viz`.
- **The dashboard:** a bundled plugin, `plugins/devtools`, built only on services and `core`, in
  the third-party shape. It replaces `src/editor/Profiler.zig`; "Toggle Profiler" opens it. Being
  a plugin is the proof that it needs no privileged access, and an app on fizzy bundles it or
  leaves it out (`bundledPluginsModule`).
- **Sections from plugins:** a surface with the `dashboard` keyword, which the devtools plugin
  declares a region to accept (`host.region`), plus a command `<id>.stats` that returns the
  section's struct as ZON. The command is how an agent queries it (commands already take arguments
  and return results), and the surface is how a person sees it. No new registry.
- **The agent's query:** `fizzy_stats { sections, window_ms, top }`, a thin tool over calling
  `devtools.stats` and each `<id>.stats`. It replaces `fizzy_profile`, whose report becomes the
  Frame, Plugins and Most expensive sections.

## Named seams

- **Per-plugin memory:** `Host` hands each plugin an allocator already (`host.allocator`). The
  seam is a counting wrapper the loader puts around it per plugin, compiled in always, counting
  only while asked (one branch on a flag). Measure the cost before turning it on by default.
- **Hitch trees:** `core.profile` keeps the last 240 frames' totals. The seam is a bounded ring
  of full trees for frames over budget, filled only while the dashboard or an agent is looking.
- **GPU timings per pass:** the backend's `Health` counts GPU work per frame. Per-pass timing
  (blur, glass, the plugin's own render passes) waits on timestamp queries in SDL_GPU; the section
  shows the column empty until then.
- **The web:** the same sections draw there. Memory reads the wasm heap; Work has no child
  processes.

## Steps

Each is one PR, useful without the next.

1. `core.viz`: `line`, `bars`, `stat`, `table`, moved and generalised from the profiler window,
   which is redrawn on them. No SDK move, reaching plugins with the next release. **Done** (#345): `line`,
   `bar`, `stat`, `Table`, `scale`. Then `Pie` (#347): the profiler shows where a frame went as
   a donut beside the table, with leaders, hover, easing and colours the table shares
   (`keyColor`, `Pie.colors`). Built by driving a sandbox through the agent plugin and judging
   screenshots, a round per change.
2. The `plugins` service reports identity and registrations; `core.profile` keeps scopes over a
   chosen window for **Most expensive**; the hitch ring. **In review:** #353 (`core/profile/Lookback.zig`:
   60 half-second windows and the newest 16 frames over budget, made on the first recorded frame
   and published under its own key so the profiler's `abi` stays; `report`'s `.over` and
   `.hitches`; the profiler window's "Last 30 s" and hitch chips) and #355 (`plugins` service v2,
   `list`: identity, link, version, registration counts, status).
3. `plugins/devtools`: the dashboard with Frame, Plugins, Most expensive and Hitches, replacing
   the profiler window. `devtools.stats` returns each section as ZON.
4. The agent plugin's `fizzy_stats`, replacing `fizzy_profile`.
5. Sections from plugins: the `dashboard` keyword and `<id>.stats`, with the text plugin's
   language-server latency as the first.
6. Views: the `.view` document, `fizzy_show` and `fizzy_update`, then inline views in the chat.
7. The `log` service and Memory, then Documents and Work.

## What performance work lacked (2026-10-10)

Found building the restart handover and the crash checkpoint (RESTART_AND_CRASHES_PLAN), where each
of these cost a round of guessing:

- **Frame causes.** fizzy draws only when something asks, and the bugs are a frame that never
  came (the checkpoint waited for an unrelated event) or frames that never stop (the handover
  refreshed every frame while it waited). dvui records each `refresh` call site, but only behind
  `FIZZY_LOG_REFRESH`, into the log. Keep it in a ring instead: each frame's cause (input, a timer,
  an animation, `refresh` from file:line), and the timers pending. That gives a profiler report
  section, a dashboard section, and the agent's `fizzy_frames`. **In review, fizzyedit/fizzy#364:** `FrameCauses`, the report's
  `.causes`; refresh places from dvui's own records (host and built-ins exactly, dylib plugins in
  their Debug builds), `Host.refresh` counted as one place (which plugin, a later SDK seam).
- **Spans on any thread, each occurrence kept.** The profiler times UI-thread scopes, averaged per
  window. Work off the UI thread (a checkpoint's write, a plugin reload's open, file loads, LSP
  calls) shows only as a worst frame. `core.profile.span(name)` would record each start and end
  with its thread, nothing unless recording. Queried by name, drawn as a timeline (the dashboard's
  `timeline`), exported as a Chrome trace for Perfetto. **In review, fizzyedit/fizzy#374:**
  `core/profile/Spans.zig` (std-only, own clock, a ring of 16k written lock-free from any thread),
  found from any image through an atomic set by the host and by `dvui_context.inject`
  (`core.profile.adopt`); the report's `.spans`, `core.profile.writeTrace` and the
  `fizzy.writeTrace` command; checkpoint writes, plugin reload opens, HTTP requests and the UI
  thread's frames timed.
- **Startup and handover phases.** Process start, window created, plugins loaded, first frame,
  window shown, a handover's ready and released, stamped by fizzy itself, so startup time and a
  restart's gap come from the app on every platform (measured from outside today, with a window
  sampler). Reported by the profiler and by `fizzy_build`. **In review, fizzyedit/fizzy#365** (stacked
  on #364): `core.profile.phase`, `.startup` on request, a startup line in the log. First finding:
  the window shows ~80 ms before its first frame.
- **Before and after in one call.** A `fizzy_run` step that profiles, runs steps, profiles again
  and answers the difference per scope and in the worst frame. Agent plugin only. **Built,
  fizzyedit/agent `fefae66`:** `compare`; and measuring no longer drives frames unless asked
  (`drive`), since the first frame-causes reading showed the agent's own profile kept fizzy awake.

## Open

- Whether views belong in the devtools plugin or in a plugin of their own: an app may want views
  (an agent's charts) without the dashboard.
- How long a window **Most expensive** keeps by default, and what it costs to keep it.
- Whether a plugin's section should draw its own widgets or only return the struct and let the
  dashboard draw it with `core.viz` (consistent look and no draw cost, but less freedom).
