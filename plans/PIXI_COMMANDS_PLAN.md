# pixi, driven by commands

Status: in progress. A stack of draft PRs in fizzyedit/pixi, each based on the one before:
#12 draw and read, #13 layers, #14 colors and tools, #15 animations, #16 grid and origins,
#18 cells, #19 selection and transform, #20 export, project and view. Found alongside:
fizzyedit/agent#3 (the bridge's 64 KiB line limit broke enum arguments once pixi's command list
outgrew it).

Everything a person can do in pixi becomes a command with arguments and a result. A command can
be a menu row, a palette entry, a keybind, and a tool an agent calls through the agent plugin
(`AGENTS_PLAN.md`, "Every plugin, driven by its commands"), all at once. Pixi writes nothing for
agents. A person asking an agent in fizzy's chat to "draw me a sword in `items/sword.pixi`" is
the end-to-end case: a handful of calls, each one visible as it lands and undoable.

## Rules every command follows

- **A `path` names the document.** Empty means the active one. A command never acts on whatever
  happens to be focused when it runs, so an agent cannot race the person (the `text` commands,
  fizzy#333, do the same).
- **One call is one undo step.** Pixel edits go through a recorded edit
  (`commands.PixelEdit`) that becomes one `.pixels` history entry. A step that fails puts back
  everything the call did.
- **No dialogs.** A command that today opens a dialog (New File, Grid Layout, Export) gets a
  sibling that takes the dialog's fields as arguments. The dialog command stays for people.
- **Reading is commands too.** Each area that can be changed can also be read, so a caller
  checks its work as text before it reaches for a screenshot.
- **Pixels as text.** A region is `rows` (one character per pixel) plus a `legend`
  (`"#=#3b2a1a o=#e8b04a"`). `.` means transparent and a space leaves a pixel as it is.
  `pixi.read` writes this format and `pixi.draw`'s `paint` reads it, so a 32×32 sprite is one
  call of about 400 tokens rather than 1,024 calls.
- **No SDK change.** Arguments, results and `std.zon.parse` into tagged unions already exist in
  SDK 0.2.24, so each step is a pixi release with no fingerprint move.
- **Commands run inside the frame.** The agent plugin calls them from `beginFrame`, and the
  palette calls them inside the frame too, so pixi's helpers that use
  `dvui.currentWindow()` are safe.

## Steps (each its own pixi PR)

1. **Draw and read.** `pixi.create` (New File without the dialog), `pixi.describe` (size, grid,
   layers, animations, colors, palette), `pixi.read` (a region of a layer, or `"*"` for every
   visible layer as shown), and `pixi.draw`. `pixi.draw` takes a list of steps: `paint`,
   `fill`, `line`, `rect`, `ellipse` and `clear`. Verified in a sandbox fizzy through the agent
   plugin's MCP bridge: a 16×16 sword in one call; the read-back matches; one undo per call;
   rollback on a failing step.
2. **Layers.** Add, duplicate, delete, rename, move, show/hide, merge up/down, select, isolate
   (`File.createLayer`, `duplicateLayer`, `deleteLayer`, `mergeSelectedLayerUp/Down`, the
   explorer's inline rename/reorder/visibility). Visibility and collapse have no history today,
   though `layer_settings` exists for it. Decide whether the command and the eye button both
   start recording it.
3. **Colors, palette and tools.** Primary/secondary set and swap; list palettes (built-in and
   the user folder) and load one; the eyedropper at a point; tool, stroke size and selection
   mode (`Tools.set`, `setToolStrokeSize`). People mostly reach these through tools rather than
   commands, but a demo tape or an agent that "shows" its drawing uses them.
4. **Selection, transform, clipboard.** Select a rectangle, a color, rows of characters, all
   or none, adding or subtracting; select cells. A transform by arguments (move, flip, rotate
   by 90°, scale) applied on the CPU, because `Transform.accept` reads its result back from
   the GPU. Copy, and paste at a place.
5. **Sprites and grid.** Grid layout and slice with arguments (`File.applyGridLayout`,
   `applyGridSliceOnly`), resize by columns and rows, move a column, row or cells, and sprite
   origins.
6. **Animations.** Create from cells with a name and per-frame ms; rename, duplicate and
   delete; set frames; set frame ms; sort; play/pause; select. Animation reorder and frame-ms
   edits have no history today, and the commands should add it for both the command and the
   UI.
7. **Export and project.** Export a cell, a layer, everything, or an animation (GIF) to a path
   at a scale, without the dialog (`Export.exportCurrentSprite` and friends); pack; the
   project's outputs and export. "Everything" reads the GPU composite today; step 1's CPU
   composite can replace that read.
8. **View.** Zoom, pan, centre on a cell, select a cell or frame. This is for a demo, or an
   agent showing the person where it worked.

After the last step, every row of the inventory below has a command. A new UI action gets its
command in the same PR that adds it.

## The rule the steps settled on

**Whatever is saved with the sprite is an undo step, from the panel as much as from a command,
and marks the sprite unsaved.** The steps found four panel actions that changed the saved
sprite with no history and fixed them: layer visibility and packing fold (#13, through the
`layer_settings` entry that had an undo and no caller), animation reorder and frame duration
(#15). Selections, colors, tools and the view are not saved, so they stay out of history, as in
the panel.

The grid snapshot (`History.Change.GridLayout`) now carries every animation's frames (#18), so it
is the undo step for anything that rearranges the whole sheet.

## Left

- **The system clipboard by command.** `pixi.copy` and `pixi.paste` exist without arguments
  and act on the active sprite; a paste that takes a place, and copying between sprites, is
  `transformSelection` with `copy` within one sprite today.
- **The interactive transform with arguments.** `pixi.transform` opens the handles;
  `pixi.transformSelection` applies one on the CPU. Driving the handles themselves belongs with
  the live-tape work (AGENTS_PLAN, "Showing the work").
- **A selection in history.** Undo restores pixels and leaves the selection, as with the tool.
- **pixi's guide** for agents and people (AGENTS_PLAN, "Coaching").

## Inventory

The full list of what a person can do, with the function behind each action and its history
behaviour, was taken on 2026-10-13. Re-derive it from `plugin.zig`, `src/explorer/`,
`src/widgets/FileWidget.zig` and `src/internal/File.zig` when a step starts; the code moves.
The facts that shape the commands:

- Brush drawing depends on global tool state (`st.tools.stroke`, `offset_table`, `stroke_shape`).
  The commands draw through `PixelEdit` instead, so no tool state is touched.
- A pixel write to a layer that is not the selected one needs `layer.invalidate()`, the
  composite flags and `invalidateActiveLayerTransparencyMaskCache`. `PixelEdit.show` does all of
  this.
- UI paths that change state without history: layer visibility and collapse, animation
  reorder, frame ms, and selection, colors, tools, isolate and play (the last five are not edits).

## Follow-ups outside pixi

- **SDK: `Host.createDocument` names its owner.** Today it gives the document to the plugin
  whose New File dialog is pending (`pending_new_document_owner`), else to the first plugin that
  can create documents. Any plugin creating a document by command has to set that field first
  (`pixi.create` does). The fix moves the fingerprint, so it goes with the boundary-trimming
  batch.
- **Agent: `fizzy_commands` filtered by prefix**, so a caller lists `pixi.*` rather than every
  command.
- **pixi: a `zig build test` step.** `pixel_text.zig` is std-only and runs with
  `zig test src/commands/pixel_text.zig`, but pixi's build has no test step.
- **Rebase onto the save stack.** `pixi.create` writes the new file with `saveZip`, which
  becomes `writeNow` once `save-through-fizzy` (pixi#11) lands.
