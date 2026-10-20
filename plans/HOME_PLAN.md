# The home page: a split tree of tiles that plugins contribute

Status: proposed. Not yet tracked in an issue.

Idea (the maintainer's): each workspace has a customizable home page, built from small
self-contained **tiles** that plugins contribute. The page is a layout region accepting views
keyworded `home`; the user starts with one tile and splits it by dragging more tiles onto the drop
wheel. The home page is a normal split tree, not a grid system.

## What exists already (so the plan does not rebuild it)

- **The `home` keyword and a `Home` place.** `sdk.keywords.ide.home` = `{"home"}`
  (`sdk/src/keywords.zig:35`); the workbench declares a plugin region named `Home` with those
  keywords, `shows = .one`, `hide_when_empty`, `key = grouping` (`plugins/workbench/src/Workspace.zig:862-871`),
  only from the sole pane while it is empty (`:842`). A document opening puts it away, the last
  closing brings it back (`docs/PLUGINS.md:636-655`). With no `home` surface the workbench draws its
  own page (`drawHomePage`, `Workspace.zig:881`: logo, New Window/Document, Open Folder, recents).
  So: **no new keyword and no new region name.** A "tile" is a surface with `.keywords = ide.home`;
  a start page like `graphl.start` is a one-tile arrangement and keeps working.
- **Everything the arranging needs, for places the app draws.** Splits are a forest keyed by place
  name (`State.splits`, `app/layout/State.zig:~137`; `Region.splitOn`, `app/layout/Region.zig:1556`;
  `SplitTree.zig`), drops are `Drop.plan` + `ViewDrag` (`SPLITS.md` "The one rule"), the corner
  button/picker (#634, `Region.zig:cornerButton`), assignments by place name (`State.assign`,
  `State.zig:506`), shut-when-empty and minted-leaf rules (`SPLITS.md` "A place a split made shuts"),
  persistence in `layout.zon` (`State.zig:149`), uninstalled-plugin offers in the picker
  (`StoreOffer`, `State.zig:36`).
- **What is missing, and it is the one real gap.** A plugin-declared region is `manual_contents`
  (`Layout.zig:1164`): the app draws no corner button on it, no drop zones, and an edge drop "does
  nothing, the app having no way to split a region a plugin laid out" (`sdk/src/RegionSpec.zig:103`;
  `Region.zig:606-611`: the picker "lives on the shape region that hosts the plugin"). Home is such a
  region, so today it is one view you can only swap through Settings.

## Design

### 1. The Home place becomes an arranged place

The home stays declared by the workbench (only it knows the pane is empty), but the app gives it
what every place the app draws has: corner button, picker, drop zones, split forest. Its leaves are
ordinary minted places (`Home/r1`, ...) holding any surfaces; the origin keeps the name `Home`.
Alternatives rejected:

- *The shape declares Home in Main while no document is open.* Makes the shape (`src/editor/layout.zig`)
  swap Main's content on document count, leaves per-workspace emptiness (a ⌘N window) unanswerable
  from the shape, and every copied shape (`AGENTS.md` "Shipped shapes") would need to rewrite it.
- *A tile grid inside the workbench.* A second implementation of splits, drops and persistence, the
  exact duplication `WORKSPACES_PLAN.md` "Why" removed.

**SDK impact.** Keyword: none. The region needs one bit saying "the app arranges this one". Two ways:
(a) infer it in `beginPluginRegion` (`Layout.zig:~1140`) from `shows = .many` and no `on_drop` -
no SDK change, but the workbench's own document panes are `.many` too and differ only by having
`on_drop` (`Workspace.zig:130`), which is an implicit rule a third-party plugin will trip over;
(b) an additive `RegionSpec.arranged: bool = false` (fingerprint moves, recorded in
`sdk/src/version.zig`, PR labelled `sdk`, `sdk_version` untouched; the plugins repin at the next
release train, `CONTRIBUTING.md` "Changing the SDK"). **Decided: (b)** - the user prefers no
SDK change, but (a) is a policy hidden in a combination of fields. Step 1 builds the seam either
way behind this one decision (decided: (b), see Decisions 1). A tile itself needs no SDK change.

### 2. When it shows, and the fold

Exactly as today: the Home place is declared only while the workspace's pane is empty, so
"no open documents in this workspace" stays the workbench's definition (`Workspace.zig:842`). With
workspaces (`WORKSPACES_PLAN.md`, behind `FIZZY_WORKSPACES=1`, `src/editor/Workspaces.zig`) it is
per workspace for free: an emptied workspace in Main or a ⌘N window shows its own home.

New: the **fold**. Today a document opening just stops drawing the page. The pane already keeps a
`closing_snapshot` texture for the reverse case (`Workspace.zig:60-64`); the home reuses the idea:
on the first document, the last frame of the home is held and slides/cross-fades out
(`core.anim.slide`/`CrossFade`, the motion level and speed settings apply) while the document
fades in; on the last close the home returns the same way. Arrangement is not touched by either: it
is state in `State`, not in the drawing.

### 3. Per-workspace arrangement, saved

The place name is the key of everything saved (assignment, forest, extents), so each workspace's
home gets its own name: `Home` for the first workspace, `Home 2`, ... by workspace ordinal (the
stable ordinal arrives with saved workspaces, `WORKSPACES_PLAN.md` step 4; until then a single
`Home` is shared, which is also the right behaviour with `FIZZY_WORKSPACES` off). The widget `key`
stays the grouping. All of it rides `layout.zon` (assignments, splits, extents) with no new file and
no schema change, so an older fizzy reads past it as it does floats. A workspace that goes away
drops its home with `State.forgetPlace` (`State.zig:628`). A new ⌘N window starts as a copy of the
first workspace's arrangement (`State.renamePlace`, `:363`, is the nearest primitive; a copy needs a
small `State.cloneTree`); decided (Decisions 3).

### 4. First run, and "Customize"

With no saved state the Home place is assigned one tile (`Recents`) and nothing else: "start with a
single view and split it". The default is written once by the workbench through
`host.assignSurfaces("Home", &.{"workbench.home.recents"})`, the same call an app uses to pick a
start page (`docs/PLUGINS.md:652`), so it respects a user's later choice, including *empty*
(`Region.zig:~395`, a cleared place is an answer). The picker's "reset" for Home re-runs the default.
A user adds a tile by the corner button -> picker (lists every `home` surface of the loaded plugins
under "made for this place", `Picker.zig`, plus store offers for uninstalled ones), or by dragging a
tile from another leaf's corner onto the wheel; an empty leaf shows the hatch and picker as any
empty place. Any surface can be docked in a Home leaf - Explorer, a chat - by the same rule; `home`
only decides what is *offered* and what lands there by keyword.

### 5. Sizes and narrow windows

`Surface` has no size hint (`sdk/src/Surface.zig`). v1 needs none: Home refuses a split that would
leave a leaf under a floor (200x140 pt, a framework constant beside `Region.InitOptions.collapse_below`,
`Region.zig:269`), and a tile scrolls itself inside what it gets (`scrolls_itself`,
`Surface.zig`). An additive `Surface.min_size` (fingerprint) is a later seam if tiles need it; named
here so it is not invented ad hoc.

Narrow regions: an arranged region narrower than 600 px of its own width (not the window's; Decisions 6) lays its leaves out as **one column in tree order** inside a
single scroll area, each at its natural height with the floor as minimum - the same idea as roadmap#11's
FieldRow stacking (rows that fit any width), applied to leaves. The arrangement is unchanged and
returns on a wide window. It is a pure function from (tree, width) to a list, so it is unit-testable
on Linux. Drag/split are disabled while stacked (the wheel on a 360-pt column is not usable).

### 6. Web, and zero cost when unused

- Web: the same code path (no desktop-only API); the fold uses the same CrossFade as the rest. Tiles
  must not assume dylibs: starter tiles ship static in their plugins, which already link on wasm.
- Nothing costs a frame unless the home shows: Home is declared only from an empty pane, so with a
  document open there is no region, no tile `draw`, no snapshot (the `closing_snapshot` is freed after
  `snapshot_idle_frames`, `Workspace.zig:64`). A tile that wants data asks a service **when drawn**
  (recents reads `State`/`Recents`, devtools only turns profiling on while its tile is visible, the
  `core.profile` `want` rule from `DASHBOARD_PLAN.md`). No tile may poll or hold a watcher while hidden;
  a tile hearing of new data uses `dvui.refresh`, not a timer. Step 4's test counts draws with a
  document open (zero).

## Starter tiles (each owned by its plugin; ids `<plugin>.home.<name>`)

| Tile | Owner | What | Notes |
|---|---|---|---|
| `workbench.home.start` | workbench | logo, New Window, New Document, Open Folder/Files | today's `drawHomePage` (`Workspace.zig:881`) minus recents |
| `workbench.home.recents` | workbench | recent folders and files, grouped by day | `app/Recents.zig`; the default first tile |
| `guides.home.tours` | `plugins/guides` | tours to start/continue | no new data |
| `fizzy.home.store` | fizzy (`src/editor`) | highlights from the plugin store with Install | the store is `app/store`, never a dylib, so fizzy registers it |
| `devtools.home.perf` | `plugins/devtools` | fps/frame-time spark + hitches count | cheap `Health` counters only |
| `chat.home.recent` | `fizzyedit/chat` | recent chats (`chat.chats`) | own repo |
| `agent.home.activity` | `fizzyedit/agent` | running/finished agent sessions, connections | own repo |
| `pixi.home.sprites` | `fizzyedit/pixi` | recent sprites/projects with thumbnails | own repo |

## Steps (one PR each; Part of the tracking issue, to be opened in fizzyedit/plans)

Cloud = Linux, tests only. Eye = needs macOS/GPU and the maintainer's eye.

1. **layout: a plugin region the app arranges** - repo fizzy, branch `layout/arranged-plugin-region`,
   builds on main. Add `RegionSpec.arranged` (decided (b); `sdk` label; align with `layout/tabs-when-many`); the app draws corner button, drops and split forest for such a
   region; unit tests for the drop/assign of a plugin-region origin; `docs/PLUGINS.md` and
   `RegionSpec.zig` comment. **macOS/eye** (tests cloud).
   Kickoff - Task: make a `host.region` place arrangeable like a shape place. Plan: plans/HOME_PLAN.md
   §1. Builds on: main (#634 corner merged). Workspace: `jj workspace add --name home1 -r main@origin
   ../fizzy-home1`. Branch: layout/arranged-plugin-region. Notes: spike first - prove a minted leaf
   under a `manual_contents` origin draws and persists (`Region.zig:606`, `Layout.zig:1164`). Record
   the fingerprint; label sdk; no `sdk_version` bump. Run test, test-integration,
   check-web, test-sdk-version. Say how it was seen.
2. **workbench: the home page is an arranged place** - repo fizzy, branch `workbench/home-arranged`,
   builds on 1. Home declares `arranged`, `shows = .many`, name per workspace ordinal, default
   assignment (Recents placeholder using the existing page). Round-trip test of assignment+forest
   through `layout.zon`. **cloud** for tests, **eye** for the look.
   Kickoff - Task: Home place arranged, per-workspace, defaulted. Plan: HOME_PLAN §1, §3, §4. Builds
   on: layout/arranged-plugin-region. Workspace: `home2`. Branch: workbench/home-arranged. Notes:
   `Workspace.zig:862`; key stays grouping, name carries the ordinal; `State.forgetPlace` when a
   workspace goes; no `FIZZY_WORKSPACES` dependency (single `Home` without it).
3. **workbench: home page split into start and recents tiles** - fizzy, `workbench/home-tiles`,
   builds on 2. Moves `drawHomePage` into two `home` surfaces; the empty-Home fallback stays so a
   build without tiles looks as today. **eye**; a unit test that both register with `ide.home`.
   Kickoff - Task: tiles from drawHomePage. Plan: §4, tiles table. Builds on: workbench/home-arranged.
   Workspace: `home3`. Branch: workbench/home-tiles. Notes: keep command-driven buttons (New Window by
   command id, shown only if it exists, `WORKSPACES_PLAN.md`); each tile scrolls itself.
4. **workbench: the home folds away and back** - fizzy, `workbench/home-fold`, builds on 2.
   Snapshot/cross-fade on the person's own first document / last close (never on session restore; Decisions 5); arrangement untouched; integration test
   (cloud): a draw count of zero tiles with a document open, and the forest equal before and after an
   open/close. **macOS/eye** for the motion at both motion levels.
   Kickoff - Task: fold animation. Plan: §2, §6. Builds on: workbench/home-arranged. Workspace:
   `home4`. Branch: workbench/home-fold. Notes: reuse `closing_snapshot` (`Workspace.zig:60`); respect
   the motion setting; do not keep a snapshot past `snapshot_idle_frames`; nothing runs while hidden.
5. **layout: an arranged place stacks when narrow; tile floors** - fizzy, `layout/arranged-stack`,
   builds on 1. Pure (tree, region width) -> column function, stacking below 600 px of the region's own width, + the 200x140 floor on splits; may measure comparable apps and adjust with reasons (Decisions 6).
   **cloud** (function tests), **eye** on a phone/web (`docs` phone recipe).
   Kickoff - Task: narrow stacking and floors. Plan: §5. Builds on: layout/arranged-plugin-region.
   Workspace: `home5`. Branch: layout/arranged-stack. Notes: model on roadmap#11's FieldRow; disable
   split/drag while stacked; arrangement must survive a resize round trip; check-web.
6. **devtools / guides / store tiles** - fizzy, three PRs `devtools/home-tile`, `guides/home-tile`,
   `store/home-tile`, each builds on 3. Static plugin tiles; devtools turns profiling on only while
   drawn. **eye** each; devtools tile checked for zero cost when hidden (cloud test of `want`).
   Kickoff (each) - Task: one starter tile. Plan: tiles table. Builds on: workbench/home-tiles.
   Workspace: `home6a/b/c`. Branch: as above. Notes: id `<plugin>.home.<name>`, keywords
   `sdk.keywords.ide.home`, `scrolls_itself`; no data held while hidden; plugin builds use `FIZZY_PROFILE`.
7. **chat, agent, pixi tiles** - repos fizzyedit/chat, fizzyedit/agent, fizzyedit/pixi, branches
   `chat/home-tile`, `agent/home-tile`, `pixi/home-tile`; build on 3 and on the SDK release that holds
   the fingerprint from step 1 (repin PR first, `plugin-repin-automation`). **eye** (macOS); the
   maintainer tags the plugin releases.
   Kickoff - Task: one tile in a plugin repo. Plan: tiles table. Builds on: repinned SDK + fizzy 3.
   Workspace: the repo's own. Branch: as above. Notes: keep the tile free of imports from other
   plugins; read via services (`chat`, `agent` control/activity); `FIZZY_PROFILE` for plugin builds.
8. **docs and demo** - fizzy, `docs/home-page`, builds on 2, 3. `docs/PLUGINS.md` §3.1.2 (tiles,
   arranged home, per-workspace), `app/layout/SPLITS.md` (plugin region splits), a demo tape arranging
   a home (`docs/AUTOMATION.md`), and the example-app note that a shape needs no home code.
   **cloud** (docs) + **eye** for the tape.
   Kickoff - Task: document the home page and record a demo. Plan: whole. Builds on: 2, 3. Workspace:
   `home8`. Branch: docs/home-page. Notes: pitfalls go to `plugins/workbench` or `app/layout` AGENTS.md.

## Decisions (maintainer's answers to the earlier open questions)

No open questions remain.

1. **`RegionSpec.arranged: bool = false`, explicit** (option b). It moves the fingerprint: the
   step 1 PR is labelled `sdk`, records the fingerprint, and leaves `sdk_version` alone. Do not
   infer arrangeability from `shows = .many`: a separate in-flight PR (branch
   `layout/tabs-when-many`) removes the region "single vs multiple" mode (tabs show only with 2+
   views unless the region owns selection). Step 1 aligns with that change and rebases on it if it
   lands first.
2. **Home leaves accept any surface.** The picker groups `home`-keyworded views in their own
   section.
3. **A new ⌘N window's home starts as a copy of the first workspace's home arrangement.**
4. **One shared `Home` is acceptable** until saved workspaces land.
5. **Session restore with documents open: Home simply does not show**, the document opens, no
   fold animation. Home animates only in response to the person's own action (opening the first
   or closing the last document).
6. **Tile floor 200x140 pt; stacking below 600 px of the region's own width** (not the window's).
   Step 5's worker may measure comparable apps (VS Code ~220 px editor group, iOS small widget
   ~160 pt, Grafana/Home Assistant ~600/1024 breakpoints) and adjust, giving reasons.
