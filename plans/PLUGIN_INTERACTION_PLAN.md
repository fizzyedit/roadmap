# Plugins working together, never depending on each other

Status: in progress, 2026-10-13. Events: fizzyedit/fizzy#391. Load order, then trimming the boundary.

Plugins never import each other and never require each other to load (TOOLS_AND_TRUST_PLAN,
"works with, never requires"). They still need to work together: the chat draws through markdown,
markdown's code fences are coloured by the zig plugin, the agent drives whatever is installed. The
rule that makes that work without a dependency tree:

**Plugins meet on a name, never on each other.** A provider puts something under a name; a
consumer asks the host for the name; either may be missing, and the other degrades.

## What there is

| Mechanism | Shape | Example | Owned by |
|---|---|---|---|
| Keyed registries | A provider registers for a key; a consumer asks the host for the key. | `Host.treeSitterHighlightFor(".zig")`: markdown's fences coloured by the zig plugin. Preview panes, file types, icons. | fizzy (the SDK names each registry) |
| Services | A typed vtable by name and version. | `markdown.renderInline`, `files`, `plugins`. | fizzy's contracts only |
| Commands | An id, ZON arguments in, a ZON result out. `visibility = .internal` for plugin-to-plugin contracts (fizzyedit/fizzy#389). | `chat.post`, `<client>.chat_send`, `text.read`. | The plugin that registers them |
| Works with | Install-time: offered together, never required. | Agent works with chat. | TOOLS_AND_TRUST_PLAN step 7 |

## What is missing

1. **Events.** A plugin cannot react to another without polling it or the other calling it by
   name. Named events with a ZON payload: any plugin emits, any plugin subscribes by name or by
   prefix, and neither knows the other.
2. **Finding providers of a plugin-defined capability.** Keyed registries exist only where fizzy
   defines the key's contract (highlighting). For contracts a plugin defines (chat clients,
   formatters, "open with"), a consumer should ask "who provides `chat.client`?" instead of a
   handshake like `chat.register`.
3. **Lifetimes, written down.** Look up another plugin's service or command each time it is used,
   never keep the pointer, and treat absence as normal. Hot reload unregisters a plugin; quit
   tears plugins down last-first and withdraws each (fizzyedit/fizzy#390).
4. **Internal contracts listed for agents.** `fizzy_commands` shows internal commands and declared
   events with their arguments, so an agent uses a plugin's contract as another plugin would.

## Events (step 1)

- **Names**: `<emitter>.<event>`, the emitter's id first, like commands: `fizzy.document.saved`,
  `chat.turn_ended`, `zig.build_finished`. Fizzy's own under `fizzy.`.
- **Payload**: ZON, like command arguments: bytes across the boundary, types on each side. The host
  copies it at `emit`; a handler borrows it for the call.
- **Declared, optionally**: `Host.declareEvent(.{ .name, .owner, .description, .payload })`, a
  sentence and the payload's shape in words, so the palette's tooling, the devtools and agents can
  list what a plugin announces. Emitting an undeclared name works; declaring is how it is found.
- **Subscribing**: `Host.subscribe(.{ .event = "fizzy.document.saved", .owner = &plugin,
  .handler })`, or a prefix (`"fizzy.document.*"`), or everything (`"*"`, for an agent's bridge
  or the devtools). Removed with the plugin (`unregisterPlugin`, `withdrawPlugin`), so a handler
  never outlives its image.
- **Delivery**: queued, then delivered on the UI thread once a frame, before plugins' `beginFrame`,
  in the order emitted. Never inside `emit`: no re-entrancy (a handler that emits queues for the
  next delivery), no surprise timing, and `emit` is safe from any thread (a worker's build
  finished), waking the app (`Host.refresh`). A handler runs under the profiler as
  `<owner> event:<name>`.
- **Nothing costs a frame unused**: an event with no subscriber is not copied or queued, and an
  empty queue is one check a frame.
- **Non-agent consumers**: a git plugin refreshing status on `fizzy.document.saved`; a language
  plugin on `fizzy.file.renamed`; the chat's list on another plugin's `chat.*`; the recorder's hitch
  capture announcing itself. The agent plugin forwards subscribed events as MCP notifications.

First events fizzy emits: `fizzy.document.opened`, `fizzy.document.saved`, `fizzy.document.closed`
(`.{ .path }`), `fizzy.folder.opened` (`.{ .path }`).

## Trimming the boundary (reviewed 2026-10-13)

`Plugin.VTable` (~60 hooks) and `EditorAPI` (~80 functions), checked against every plugin in
fizzy and `fizzyedit` (pixi, zig, ghostty, atlas, drive, chat, agent). Kept: document ownership
and routing, drawing, the frame phases a frame needs in step (`beginFrame`, `endFrame`,
`drawOverlay`, `needsContinuousRepaint`), lifecycle, settings, regions, view drags, logging.

| Today | Used by | Becomes |
|---|---|---|
| Owner-run saving: `saveNeedsConfirmation`, `requestSaveConfirmation`, `showsSaveStatusIndicator`, `isDocumentSaving`, `saveDocumentAsync`, `timeSinceSaveCompleteNs`, `resetDocumentSaveUIState`; EditorAPI `setPendingCloseDocId`, `queueCloseAfterSave`, `trackQuitSaveInFlight`, `resumeSaveAllQuit`, `abortSaveAllQuit`, `requestWebSave`, `cancelPendingSaveDialog`, `showSaveDialog` | pixi (`saveDocumentAsync` also text) | Gone: fizzy saves every document through `documentBytes` (`DocumentIo`), as archive and text already do. Pixi's flatten confirmation becomes a question before the save. |
| `onFolderOpen`, `onFolderClose`, `folderPathsChanged`, `documentContentChanged`, `removeCanvasPane` | atlas, drive, pixi, zig | Events: `fizzy.folder.opened` / `.closed`, `fizzy.files.changed`, `fizzy.document.changed` (the path; read the text with `text.read`), `workbench.pane.closed`. |
| `undo`, `redo`, `canUndo`, `canRedo`, `revealPosition`, `requestNewDocumentDialog` | pixi, text, atlas | Commands: `<owner>.undo` / `.redo` with `isEnabled` (as copy and paste are), `text.reveal`, `<owner>.new { kind }`. |
| EditorAPI `setProjectFolder`, `closeProjectFolder`, `showOpenFolderDialog`, `showOpenFileDialog`, `openInFileBrowser`, `requestSaveAs` | workbench, archive, drive, pixi | fizzy's existing commands (`fizzy.openFolder`, `fizzy.saveAs`, …). |
| EditorAPI `explorerRect`, `explorerVirtualSize`, `explorerViewportWidth`, `explorerBranchIsOpen` / `setExplorerBranchOpen`, `drawWorkspaces`, `documentIsPreview` / `setDocumentPreview`, `setExplorerNewFilePath` | workbench, pixi, atlas | The workbench's own state and service: fizzy owns no explorer. |
| EditorAPI `getSecret` / `setSecret` | drive | A `secrets` service an app can supply (keychain, browser storage), like `files`. |
| `tickKeybinds`, `contributeKeybinds`, `prepareFrame`, `tickActiveDocument` | pixi, workbench | Keybinds bind to commands; the rest fold into `beginFrame`. |
| `contributeMenu`, `newDocumentKinds`, `recentFolderCount` / `recentFolderAt`, `isMacOS` | nobody, or a compile-time check | Deleted. |
| Document staging: `documentStackSize`, `documentStackAlign`, `documentIdFromBuffer`, `deinitDocumentBuffer`, `setDocumentGroupingOnBuffer`, `registerOpenDocument`, `documentPtr` | every document plugin | One hook: the plugin loads and owns its document, and hands fizzy a handle. |

Roughly `Plugin.VTable` ~60 → ~40 and `EditorAPI` ~80 → ~55. Each is a fingerprint-moving PR with
its store plugins' repins (pixi, atlas), in this order: pixi onto fizzy's saving; folder and file
events; undo, redo, reveal and new as commands; the workbench's state out of `EditorAPI`; the
unused deleted; document staging.

### Step 1 in detail: everything saves through fizzy

Found 2026-10-13: pixi already serializes (`documentBytes`, `documentWritten`), so plain Save,
Save All, the quit walk and Save As already go through fizzy's write (`DocumentIo`). Owner-run
saving survives only through the flatten-layers confirmation, the unsaved-changes dialog's Save
and Close (fixed for every plugin by fizzyedit/fizzy#393, no SDK change), and the cancel paths.

Still to decide and build, in one SDK release with pixi repinned right after:

- **`documentBytes` gets the target path**, so the owner encodes for it. Today pixi picks the
  format from the document's current path, so Save As from `x.pixi` to `y.png` writes zip bytes
  into `y.png`.
- **A question before a save**, declared by the owner and shown by fizzy wherever a save starts
  (Save, Save All, the quit walk, Save and Close, the web's download): pixi's "flatten layers?"
  becomes it, and `requestSaveConfirmation` and the seven EditorAPI save and quit calls go.
- **Encoding off the UI thread** where it can be: since saves moved to fizzy, a `.pixi` zip is
  compressed on the UI thread (it used to be pixi's worker). A snapshot on the UI thread and the
  encode on a fizzy worker; PNG and JPEG still read back from the GPU on the UI thread.
- **A save indicator fizzy draws** for every document (saving, then saved), replacing pixi's own.
- **What a flattened save means** (decided 2026-10-13): fizzy writes flat bytes and the document
  keeps its layers; the question is asked once per open document.

**Every save is a snapshot then an encode** (decided 2026-10-13), whatever the format:
`documentSnapshot(doc, target_path)` on the UI thread takes what the save needs and nothing more
(a text buffer's copy; a `.pixi`'s layers; a PNG's composite read back from the GPU), and
`encodeSnapshot(snapshot, target_path, writer)` runs on a fizzy worker with no dvui or GPU,
followed by the write (disk) or upload (a mount), also off the UI thread. They replace
`documentBytes`; the target path fixes Save As. The document stays editable while it saves; the
snapshot's version says whether the save left it clean. Budget: under ~2 ms of UI thread for an
ordinary document. Named seam, measured first: an asynchronous GPU readback in fizzy's backend
(asked one frame, pixels a frame or two later), so even a PNG's snapshot leaves the UI thread.

## Load order is never what decides anything

Today built-ins load in the build's order and user plugins in whatever order the OS lists the
plugins folder; a place (the rail, a sidebar) shows its views in registration order unless the
person has dragged it. So a plugin's hot reload, store update or fresh install moves its views to
the end, and a new machine can order them differently. Load order also silently picks the first
provider of a service name and the first highlighter of an extension.

Load order is an implementation detail, never shown:

1. **Deterministic anyway**: user plugins load sorted by id.
2. **Positions persist**: each place keeps the order of the views it has shown, not only after a
   drag; a new view goes at the end, a returning one back to its slot. An order still only sorts,
   never decides what a place holds.
3. **Real conflicts are the person's choice**, as file types are: two providers of one service
   name, two highlighters of one extension, resolved by their choice, then the only provider, then
   alphabetical; shown in Settings only when there is a conflict.
4. **Correctness never depends on order**: services and commands looked up at use, events
   delivered the frame after, quit tearing down last-first (fizzyedit/fizzy#390).

## Steps

1. **Events**: `Host.emit`, `subscribe`, `declareEvent`, the queue and its delivery, removal with
   the plugin, profiling, tests; fizzy's first four events. Moves the fingerprint.
2. **Events out to agents and the devtools**: `fizzy_events` (subscribe with a cursor) and MCP
   notifications in the agent plugin; an events view in the devtools.
3. **Capabilities**: `Host.provide(capability, key, command)` and `Host.providers(capability, key)`
   for plugin-defined contracts; the chat finds its clients through it, retiring `chat.register`.
4. **The rules in `docs/PLUGINS.md`**: meeting on names, lifetimes, internal commands and events as
   a plugin's published contract, versioned by the plugin.
5. **Internal contracts in `fizzy_commands`**, with declared events.
