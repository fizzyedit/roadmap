# fizzy roadmap

Fizzy's roadmap: the plans — designs, milestones, measurements in flight and reviews — kept in the open. The code, its contract and its public docs live in
[`fizzyedit/fizzy`](https://github.com/fizzyedit/fizzy); this repo is where the work is decided.
The plan for upstreaming fizzy's dvui work is not published; a plan that mentions it names it only in passing.
Plans were written in a private repo, so a mention of `fizzyedit/plans#N` in a plan or a fizzy PR may refer to an issue there that is not published here.

- **One plan, one file** in `plans/`, opening with a status line: `Status: proposed`,
  `in progress, fizzyedit/fizzy#N`, or `done`.
- **Changed by pull request**, one plan step per PR, the same as fizzy (`fizzy/CONTRIBUTING.md`).
  An implementation PR in fizzy says `Part of fizzyedit/plans#N` or names the plan file.
- **Paths are fizzy's.** `src/…`, `app/…`, `sdk/…`, `core/…` mean fizzy's tree; `fizzy/docs/…`
  and `fizzy/CLAUDE.md` are fizzy's public docs; a bare `AGENTS_PLAN.md` is a plan here.
  Issue and PR numbers are written `fizzyedit/fizzy#N`.
- **What stays public in fizzy:** the plugin contract (`docs/PLUGINS.md`), how automation works
  (`docs/AUTOMATION.md`), the forks fizzy builds on (`docs/DEPENDENCIES.md`), measurements already
  done, and release notes.

Moved from fizzy's `docs/` on 2026-10-09; their history up to then is in fizzy's.

| Plan | What |
|---|---|
| `APP_BUILD_API_PLAN.md` | `buildApp` as blocks an app composes: `addApp`, `addWeb`, `addPackage`, `dvuiModule` |
| `AGENTS_PLAN.md` | agent support as a plugin, the SDK seams it needs, fizzy building itself |
| `AGENT_VIEW_PLAN.md` | an agent's own view of a document: owner-drawn looks, working unseen, seeing each other's edits |
| `AUTOMATION_PLAN.md` | tapes, recording, seeking, `replay` for any dvui app |
| `CLOUD_FS_PLAN.md` | cloud drives as folders |
| `CONTROL_SERVER_PLAN.md` | HTTP on a Unix socket in `app/`: commands as the API, OpenAPI, events, MCP, curl without an agent |
| `EMBEDDED_EDITOR_PLAN.md` | fizzy's editor as a guest on a host's dvui backend (Blender, Unreal) |
| `MENUS_AND_TABS_PLAN.md` | menus and tabs |
| `NATIVE_WINDOWS_PLAN.md` | native windows, Liquid Glass, the backend owning its windows |
| `PLUGIN_MANIFEST_PLAN.md` | identity manifests, settings schemas, ZON user config |
| `POPOUT_WINDOWS_PLAN.md` | floats out into OS windows |
| `UNIFIED_DRAG_PLAN.md` | one drag for every carried thing |
| `WEB_EMBED_PLAN.md` | `fizzy.js`: fizzy on a host page's canvas, no globals patched |
| `WINDOWING_FOUNDATION_PLAN.md` | the windowing foundation |
| `WINDOWS_LINUX_GLASS_PLAN.md` | glass on Windows and Linux |
| `WORKSPACES_PLAN.md` | many workspaces, new windows, new documents |
| `ZIG_0_17_PLAN.md` | moving to Zig 0.17: the gate (ZLS), the order, each release-note migration |
| `LIB_CHECKPOINT.md` | fizzy as a library: what was done, what is next |
| `TRANSITIONS_HANDOFF.md` | screen transitions, cross-fade |
| `REVIEW_2026-09.md`, `REVIEW_NAYSAYER_2026-09-22.md` | architecture reviews |
