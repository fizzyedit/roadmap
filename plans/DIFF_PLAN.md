# Diff plan: one fast diff library for text, images and structured documents

Status: proposed

Maintainer's idea: a very fast, efficient diff in fizzy that diffs text, images and pixi-type
(layered, structured) documents, "because the agents need it". This plan grounds that in what
exists, picks the algorithms, and cuts it into PRs. Line numbers are fizzyedit/fizzy `main` at
f9ba3f66c unless a repo is named.

## 1. Why now: the places that each hand-roll a piece of a diff

| Need | Where it is today | What it lacks |
|---|---|---|
| Agent writes whole text | `plugins/text/plugin.zig:659` `cmdWrite` trims common prefix/suffix (`std.mem.indexOfDiff`, a suffix loop) and replaces the middle as one edit | One hunk only: two distant edits become one big replacement, so carets, undo and other authors' edits in between are clobbered. No base version, so a write made against an older text lands over newer edits |
| Stale agent writes (MULTIPLAYER_PLAN step 1) | Not on `plans` main yet (referenced by the maintainer); the premise is "an edit made against an older version lands rebased" | Needs diff(base, theirs) replayed onto current = a three-way merge with byte ranges |
| Approval prompts with diff previews | `fizzyedit/chat` (in progress, `~/dev/fizzyedit/chat-approvals`) | Nothing to render: needs hunks of old/new lines (+ word spans) for text and rects for images |
| Suggest, then accept | Agents review and accept edits (CHAT_PLAN / AGENT_VIEW_PLAN) | A reviewable "pending change" is a ChangeSet; accept = apply, reject = drop |
| "What changed since I last looked" | `AGENT_VIEW_PLAN.md` :183 promises each tool answer carries the steps since the agent's last `step`; :190 refuses an `expect`-stale write | Steps say *that* something happened, not a compact *what*; an agent that missed N steps wants one net change, not N |
| History strip change regions | `app/HistoryFrames.zig:104` `contentOf` scans a thumbnail for pixels unlike the bottom-right pixel (a heuristic box, with an `edge_band` hack for scrollbars); #653/#656/#660 print column/row marks of changed content in the strip (`app/history_transport.zig`) | A real "which regions of the document differ between step A and B" instead of guessing from a screen thumbnail |
| Checkpoint deltas | `app/state_delta.zig:29` `encode`/`apply`: prefix/suffix trim, then fixed 64-byte `block` runs when lengths match, else the raw middle | Right idea, bytes only: a text insert that shifts the tail is stored raw. A line/tile-aware delta shrinks checkpoints (`app/Tracks.zig:13-23`) |
| Tape determinism | `sdk/replay/Stage.zig:68` `fingerprint` (one u64) and `sdk/replay/Player.zig:625` `verify`: on mismatch only logs "did not reach what playing did" and counts | A mismatch with no *where*. With a diff of the two captured states the report says "layer 3 tile (4,7) differs" |
| Screen diff for agents | `app/control/screen_diff.zig:123` keyed-node diff | Already the right output *shape* (added/removed/changed, compact); the new library returns the same family of shapes |
| Flight recorder bundle | RECORDER_PLAN (plans repo) | A bundle wants a minimal "state at crash vs last good checkpoint" |
| Shared history / tracks | `app/Tracks.zig` (base + checkpoints + `replayInto`); SHARED_HISTORY_PLAN | Merging two authors' tracks needs diff/rebase of document states, not only steps |
| Pixi | `fizzyedit/pixi` `src/internal/File.zig:33,241`: layers are an `internal.layers` list addressed by `selected_layer_index` | Identity by index: a reorder is indistinguishable from "everything changed". No ids to diff by |
| MCP / commands | `app/control/mcp.zig:207` `tools/list` = the app's commands; `text.read` (`plugins/text/plugin.zig:632`) | No command answers "what differs between X and Y" |

The common failure is the same: each site invented a private, weaker diff (prefix/suffix trim,
64-byte blocks, thumbnail contrast scan, a u64 fingerprint). One library replaces them.

## 2. Design

### 2.1 Where it lives and its constraints

`core/diff/` as `core.diff` (namespace `diff.zig` over `Text.zig`, `Image.zig`, `Merge.zig`,
`Unified.zig`, `ChangeSet.zig`). **Std-only** like `state_delta.zig` and `screen_diff.zig`, so
plugins (dylibs get `core` via `sdk/core_module.zig`), the web (wasm32, no threads) and a plain
Linux test root all use it. No dvui import. Allocation-light: every entry point takes a caller
allocator (the `host.arena()` for a one-shot answer) and a reusable `Scratch` struct, so a
repeated diff (the history strip, an agent polling) allocates nothing after the first call.
Inputs are slices; nothing assumes the whole document is resident twice. Large inputs are
processed in bounded slices so a frame's cost can be capped (`budget_ns` option; returns
`.partial` with a resume cursor, the same "spread over the frames after" promise
`app/Tracks.zig:19-21` already makes).

Optional costs nothing when unused: no global state, no hooks on the frame path; a diff runs
only when something asks (command, strip, merge).

### 2.2 Text: histogram line diff, then refinement

Recommendation: **line diff with the histogram algorithm** (git's `--histogram`, JGit; the
fast path of `imara-diff`), over interned line ids, with Myers (O(ND), linear space) as the
fallback when histogram finds no low-occurrence anchor.

Why not plain Myers: correct but gives worse hunks on code (it aligns on `}` and blank lines),
and an approval preview is read by a person. Why not patience: it needs unique lines and falls
back anyway; histogram generalises it (lowest-count anchor, not only unique) and is faster on
real source. Prior art: git xdiff histogram, JGit `HistogramDiff`, `imara-diff` (Rust, among the
fastest), `similar`, `diff-match-patch` (char-level, slower, hunks not tuned for code).
Implementation in Zig is roughly 600 lines. (Prior art above was written from memory, not re-fetched: check each citation, and the claims about speed and hunk quality, when step 1 starts.)

Pipeline, each stage skippable:
1. **Trim** common prefix/suffix with a SIMD `@Vector(32, u8)` compare (also the identical-input
   fast path: `.equal` in one pass at memory bandwidth).
2. **Tokenize into lines**: SIMD scan for `\n`, hash each line (`std.hash.Wyhash`), intern to
   dense u32 ids. A hash collision is resolved by comparing bytes only for lines the algorithm
   actually matches.
3. **Histogram diff** on the id arrays between the trimmed ends, recursing on the gaps; Myers
   when no anchor or the occurrence cap (64, git's default) is hit.
4. **Hunk shifting**: slide each hunk to the most readable boundary (git's indent heuristic:
   prefer ending on a blank line / lower indentation).
5. **Refinement** inside a replaced hunk pair: Myers on words (split at class boundaries:
   identifier / space / punctuation), chars only when a changed word is short. Output is byte
   ranges within the line, for highlighting in a preview.

Output: `[]Hunk { a_start, a_len, b_start, b_len }` in lines, plus `Edit { start, end,
replacement }` in **byte ranges**, convertible to LSP `Range { line, character }` through a
position-encoder callback (text already has `lineCharacterForByteOffset`; the library never
knows the document type). `Unified.write(w, a_name, b_name, hunks, context)` emits standard
unified diff (`@@ -a,l +b,l @@`), the format agents read best and `patch`/`git apply` accept, with
a `max_bytes` cap that ends with "... N more hunks" so it is sized for an agent's context.

### 2.3 Three-way merge / rebase (stale writes)

`Merge.three(base, ours, theirs)`: diff(base to ours) and diff(base to theirs) with the same line
ids, then walk both hunk lists (diff3): non-overlapping hunks merge; overlapping hunks with
identical replacement merge; any other overlap is a `Conflict { base, ours, theirs }` span.
`Merge.rebase(base, theirs, current) -> { edits, conflicts }` is the helper `cmdWrite` needs: an
agent that read version V and writes T gets T's changes *relative to V* applied to the current
text, as byte-range edits, each one `doc.replaceRange` inside one undo group, so other
authors' carets and anchors outside the hunks do not move. Conflicts are returned, never
silently resolved: the command fails with the conflicting hunks in unified form so the agent
can retry.

The base: `text.write` gains an optional `base` (a `version` that `text.read` returns). Decided:
the version resolves to base text from a per-connection copy of the last read (cheap, capped).
It does not go through `Tracks.replayInto`. Writes with no base keep today's behaviour (but
multi-hunk).

### 2.4 Images: tile hashes, then exact diff

For RGBA8 `[]const u8` with width, height, stride:
1. **Row fast path**: compare whole rows with `@Vector(64, u8)`; identical rows (the common
   case in sprite work) cost memory bandwidth only. Identical buffers return after one
   `std.mem.eql`.
2. **Tile signatures**: 32x32 default (16x16 for small canvases; power-of-two so index math is
   shifts). Hash each tile with a fast 64-bit hash over whole-row vector loads, no per-pixel
   work. `TileGrid { w_tiles, h_tiles, hashes: []u64 }`: 8 bytes per 4 KiB tile, so a 4K
   image's grid is about 16 KB and cheap to *keep* per step. That is what makes "diff against a
   past step" affordable: store grids, not images.
3. **Changed tile set** = grids differ (bitset), then **merge to rects**: run-length along rows,
   merge vertically adjacent identical runs (greedy rectangle cover, like the strip's column
   and row marks). Output `[]Rect` in pixels, capped (`max_rects`; beyond it, coalesce the
   remainder to its bounding box and say so).
4. **Exact refinement** (optional): inside each changed tile, scan rows to shrink the rect to
   the true bounding box and count changed pixels; optional `tolerance` (per-channel max
   delta) for lossy sources, off by default.
5. Different sizes: report `size_changed` and diff the overlap; no alignment search (this is not
   a perceptual or motion diff).

### 2.5 Structured documents: an owner hook returning a ChangeSet

`ChangeSet` is the one result type everything shares: a flat list of typed changes with a
path-like subject and the primitive result attached:

```
Change = union(enum) {
  added: Subject, removed: Subject, renamed: {id, from, to}, moved: {id, from_idx, to_idx},
  property: {subject, key, old, new},          // formatted values, small
  text: {subject, hunks},                      // 2.2
  pixels: {subject, rects, changed_px},        // 2.4
}
Subject = { kind: []const u8 ("layer","animation","frame",...), id: ?u64, name: []const u8 }
```

Stable ids are the hard prerequisite: pixi addresses layers by index today
(`pixi/src/internal/File.zig:33,241`). Decided: the `.pixi` format gets stable layer and
animation ids, not unique names. That is the same work as MULTIPLAYER_PLAN step 8 ("pixi:
everything named by id, ids minted per replica", branch `history/ids-not-indices`), which owns
it; this plan's step 6 builds on it and does not mint ids itself. The diff service matches by
id, then name, then index (old files read as index order, as step 8 says).

The seam, smallest first:
- **A `diff` service (step 4), no SDK change. Decided: the service is the seam, owner-registered.** A plugin registers a service (`host.registerService`,
  as workbench does with `files`); the `fizzy.diff` command looks up the document's owner and
  calls `service.diff(a_state, b_state)`. States are the bytes `captureDocumentState` already
  returns (`sdk/src/Plugin.zig:247`), the same bytes tracks and tape snapshots hold. So **any
  document diffs at byte level for free through `core.diff` with no owner code**, and an owner
  that implements the service gets typed changes. A service is outside the boundary
  fingerprint: no `recorded_sdk_shape_fingerprint` move, no SDK release, no repin.
- **No vtable hook is planned (decided).** Should the service ever prove awkward, the fallback is:
  `diffDocumentStates: ?*const fn (state, a: []const u8, b: []const u8, out: *ChangeSetBuilder) anyerror!void`
  on the plugin vtable. That moves the shape fingerprint (record it, label the PR `sdk`, leave
  `sdk_version`; a release PR bumps it per CONTRIBUTING "Changing the SDK") and means a repin.
  Promote when a second owner uses it ("Build the seam the next plugin needs"). `ChangeSetBuilder`
  lives in `core.diff`, which plugins already build in.
- Text and image owners need no hook: text diffs its own bytes via 2.2; an image's state is
  pixels via 2.4.

### 2.6 Agent and app surface

- Command `fizzy.diff` (internal visibility, so MCP `tools/list` at `app/control/mcp.zig:207`
  exposes it and the OpenAPI routes get it): args `path`, `from`, `to`, each `"disk"`, `"doc"`
  (open state, unsaved edits included), `"step:<id>"` or `"version:<n>"`. Output `{ summary,
  changes, unified?, rects? }`; `format: summary|unified|full`, `max_bytes` (default 8 KB), so
  the answer is sized for context and truncates with counts, never silently.
- "What changed since I last looked": AGENT_VIEW_PLAN's answers gain `changes_since: <step>`,
  produced by diffing the net states (net-of-N-steps beats listing N).
- Chat approvals and suggest/accept consume `ChangeSet` directly (this plan ships the data,
  chat draws it). Accept = `Merge.rebase` onto current.
- The history strip replaces `contentOf`'s heuristic with step-to-step `TileGrid` diffs for
  owners that expose pixels, keeping thumbnails as fallback.
- Tape replay: `Player.verify` (`sdk/replay/Player.zig:625`) logs the first differing subject
  from a diff of captured states, via an optional `Stage` hook that costs nothing unless a
  mismatch happens.
- Checkpoints: `state_delta.encode` uses `core.diff` text/tile mode when it is smaller (format
  version bump); shrinks `Tracks` memory. Optional, last.
- Flight recorder bundle: includes a `fizzy.diff`-shaped summary of "state at crash vs last
  checkpoint" once step 4 exists (a one-line follow-up in the RECORDER_PLAN, not its own step).

## 3. Performance budget (CONTRIBUTING: measure, keep a benchmark)

`zig build bench-diff` (like `bench-tracks`), numbers recorded in each PR. Starting targets,
revised by measurement on an M-series Mac and the Linux CI box:

| Case | Target |
|---|---|
| 1 MB text, identical | < 0.2 ms (memory bandwidth) |
| 1 MB text (~25k lines), one edited line | < 2 ms |
| 1 MB text, 1% of lines edited, scattered | < 15 ms |
| 1 MB text, every line changed | < 60 ms, bounded by a D cap that falls back to "replace" |
| 4K RGBA (33 MB) identical | < 4 ms |
| 4K RGBA, one small stroke | < 8 ms including the grid |
| 4K RGBA, grid vs a kept grid | < 1 ms |
| Sprite 2048x2048 x 50 layers, one layer touched | < 10 ms (identical layers hit the eql path) |
| Three-way merge, 1 MB, 20 hunks per side | < 25 ms |
| Allocation | zero after warmup with a reused `Scratch` |

Anything over a frame uses `budget_ns`. Tests: property tests (apply(diff(a,b), a) == b over
random edits; non-overlapping edits commute under merge); fuzz the line tokenizer on
CRLF/no trailing newline/binary.

## 4. Steps (one PR each, 800 lines or fewer outside tests and docs)

Cloud = Linux, `zig build test` only, no window. Local = needs macOS/GPU/the user's eye.

1. **core: text line diff, unified output and bench** (cloud). Repo fizzy, branch
   `core/diff-text`. Builds on: main. Trim, line interning, histogram + Myers, hunk shifting,
   byte-range edits, LSP converter callback, unified writer with cap, property tests, text rows
   of `bench-diff`.
   Kickoff: Task: add std-only core.diff text diffing. Plan: plans/DIFF_PLAN.md 2.1-2.2, 3.
   Builds on: main. Workspace: jj workspace fizzy-diff-text. Branch: core/diff-text. Notes: files
   core/diff.zig + core/diff/{Text,Unified}.zig; test root in build/app.zig like state_delta
   (check-tests-reached); @Vector with scalar fallback; do not touch cmdWrite yet; bench numbers
   in the PR body.
2. **core: word/char refinement and three-way merge** (cloud). Branch `core/diff-merge`.
   Builds on 1. `Merge.three`, `rebase`, conflict spans, word spans.
   Kickoff: Task: add Merge (diff3) and refinement to core.diff. Plan: DIFF_PLAN 2.2 stage 5,
   2.3. Builds on: core/diff-text. Workspace: fizzy-diff-merge. Branch: core/diff-merge. Notes:
   conflicts are returned, never auto-resolved; properties: non-overlapping edits commute,
   rebase(base,base,cur)==cur, identical edits merge; no document types in core.
3. **core: image tile-grid diff** (cloud). Branch `core/diff-image`. Builds on: main (parallel
   with 1 and 2). `TileGrid`, changed set, rect merge, exact refine, tolerance, row fast path;
   4K and 50-layer bench rows.
   Kickoff: Task: add core.diff.Image (RGBA8 tile hashes to rects). Plan: DIFF_PLAN 2.4, 3.
   Builds on: main. Workspace: fizzy-diff-image. Branch: core/diff-image. Notes: stride aware;
   32x32 default, 16 option; TileGrid keepable (8 B/tile); max_rects coalesce; own test root in
   build/app.zig; bench numbers in the PR.
4. **app: `ChangeSet`, the `diff` service and `fizzy.diff` command/MCP tool** (cloud for the pure
   part; local to see the MCP tool through fizzy-sandbox). Branch `app/diff-command`. Builds on
   1, 3 (2 optional). Byte-level fallback over `captureDocumentState` bytes, text and image
   owners diffed directly, from/to resolution (`disk`, `doc`, `step:`), summary/unified/full,
   size caps.
   Kickoff: Task: ship fizzy.diff (command = MCP tool) over ChangeSet. Plan: DIFF_PLAN 2.5-2.6.
   Builds on: core/diff-text, core/diff-image. Workspace: fizzy-diff-cmd. Branch:
   app/diff-command. Notes: register in the command registry (mcp.zig lists it); no SDK
   fingerprint move (owner hook is a service), confirm with `zig build test-sdk-version`;
   include an MCP call transcript in the PR.
5. **text: `text.write` rebases a stale `base`; `text.read` returns `version`** (cloud for
   logic with a headless doc test; local for feel). Branch `text/write-rebase`. Builds on 2, 4.
   Replaces the prefix/suffix trim at `plugins/text/plugin.zig:659` with hunk edits in one
   undo group; conflicts fail with unified hunks.
   Kickoff: Task: text.write applies diff hunks and rebases a stale base (MULTIPLAYER_PLAN
   step 1). Plan: DIFF_PLAN 2.3. Builds on: core/diff-merge. Workspace: fizzy-text-rebase.
   Branch: text/write-rebase. Notes: no base = today's behaviour but multi-hunk; text.read keeps
   a per-connection copy (Host.act author), capped; test: two authors' edits in different
   regions both survive.
6. **pixi: a `diff` service** (local: macOS and the user's eye; repo fizzyedit/pixi). Branch
   `diff/service`. Builds on 3, 4, and MULTIPLAYER_PLAN step 8 (`history/ids-not-indices`). The
   ids are that step's work, one piece of work with two consumers: step 8 there mints and
   persists layer and animation ids in the `.pixi` format; this step uses them. Do not build ids
   twice; if this step is ready first, start step 8 there. The service produces
   added/removed/renamed/moved/property/pixels changes on core.diff.
   Kickoff: Task: pixi diffs its documents into a ChangeSet. Plan: DIFF_PLAN 2.5. Builds on:
   app/diff-command, history/ids-not-indices (MULTIPLAYER_PLAN step 8). Workspace: pixi-diff.
   Branch: diff/service. Notes: ~/dev/fizzyedit/pixi, FIZZY_PROFILE for any build; match by id,
   then name, then index; test reorder is not "all changed"; 50-layer bench; old files without
   ids still load (step 8 handles the migration). Released through the normal repin cadence, no
   SDK change.
Order: 1 and 3 in parallel, then 2, then 4, then 5; 6/7/8 after 4; 9 last.

## 5. Open questions

Decided with the maintainer, no longer open: stale writes use a per-connection copy of the last
read as the base (2.3); structured diff is an owner-registered `diff` service outside the
boundary fingerprint, with no vtable hook (2.5); pixi gets stable layer and animation ids in
the `.pixi` format, not unique names, shared with MULTIPLAYER_PLAN step 8 (2.5, step 6).

1. Tile size default (32 vs 16), and whether `tolerance` is needed before an agent asks.
2. Default position encoding for the LSP-range converter (UTF-16 as LSP defaults, UTF-8 where
   `core.lsp` negotiates it), and char-level refinement for wide/CJK text.
3. Is a track-format bump (step 9) worth the checkpoint savings, or do we leave `state_delta`
   alone?
