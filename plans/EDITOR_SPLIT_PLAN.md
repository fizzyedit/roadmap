# Editor.zig, split by what it does

Status: proposed, 2026-10-13.

`src/editor/Editor.zig` is 6,365 lines and about 250 functions. Much of it is several jobs that
already keep to themselves: a section marker or a run of functions each. Split along those lines,
each piece can be read, tested and reviewed alone, and `Editor.zig` keeps what only it is: the
struct, its lifecycle, and the frame.

## What is in it (lines, roughly)

| Concern | Lines | What |
|---|---|---|
| `EditorAPI` glue | ~650 | about 80 one- to three-line `fizzy*` adapters, and the region adapters |
| Plugin lifecycle | ~1,500 | user and web plugin loading, install, enable, update, uninstall, the store's `PluginManager`, rebuilt binaries reconciled |
| File-type ownership | ~350 | the extension cache, conflicts, the dialog's rows |
| Documents | ~900 | open paths, previews, loading jobs, document surfaces, close and detach |
| Saving and quitting | ~550 | Save, Save All, Save As, the quit walk, closes after a save, save toasts, the web's save |
| Settings and appearance | ~700 | composing and merging `settings.zon`, reconciling outside edits, window ratios, themes, fonts |
| Layout persistence, native menu queue | ~250 | |
| The project folder | ~150 | open and close, the watcher sinks |
| What stays | ~1,500–2,000 | fields, `init` / `postInit` / `deinit`, `tick` (550 alone), the document verbs |

## How

- **A concern with state of its own becomes a struct**, its state moving with it (the repo's
  file-is-a-struct rule): `Saving.zig` (closes after a save, the quit walk's lists,
  `save_answered`, `save_question`; some of these live on `App` today), `PluginLifecycle.zig`
  (loading jobs, the web plugin queue), `Documents.zig` (loading jobs, previews). The Editor holds
  `editor.saving`, `editor.plugins`, `editor.documents`.
- **Functions over the Editor with no state** go in lowercase namespaces taking `*Editor`:
  `editor_api.zig`, `file_types.zig`, `settings_io.zig`. Call sites keep working through
  declaration aliases in `Editor.zig` (`pub const save = Saving.save;` keeps `editor.save()`), to be
  changed at the call site later or never.
- `tick` may become `frame.zig`: the frame loop is its own thing.
- **One PR per piece, no behaviour change**: a moved function lands as it was, reviewed with
  `git diff --color-moved`. None moves the plugin boundary's fingerprint, so none waits on or holds
  up an SDK release.

## Steps

1. `editor_api.zig`: the largest and most mechanical.
2. `Saving.zig`, after fizzyedit/fizzy#395 and #396 land (they rewrite that code; moving it first
   would mean rebasing them over a move).
3. Plugin lifecycle.
4. Documents.
5. Settings and appearance.
6. File types; layout persistence into `layout.zig`; the native menu queue into `Menu.zig`; the
   project folder.
7. `tick` into `frame.zig`, if what is left still reads as two things.
