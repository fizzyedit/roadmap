Status: proposed

# Shader plugin: edit shaders, see them live, errors in the editor

An external store plugin, `fizzyedit/shader`, shaped like `fizzyedit/zig` (`plugin.zig`,
`plugin.zig.zon`, `build.zig`, `src/`, `queries/`). It opens `.glsl .frag .wgsl .hlsl .metal`
in the text editor with highlighting and a language server where one exists, and draws the shader
live in the editor's preview pane: every edit recompiles, a failed compile keeps the last good
picture and puts its errors in the text as diagnostics. A later step does the same for a Zig
game (hot-reload pane).

## What exists (the plugin builds on this)

- **Custom GPU programs are already a plugin-callable API.** `backend/src/glass/programs.zig`,
  re-exported as `core.programs` (`core/core.zig:60`). `Program.from(.{glsl,msl,spirv,dxil}, shape)`
  (`programs.zig:175-228`), `drawRect` (`:323`), `draw` (`:286`), `Hooks` published each frame by the
  host into the shared dvui window (`:106`, `publishHost :127`), so a plugin dylib or web side module
  calls the host's functions directly. The fragment shader only: the vertex shader is the default one.
  Binding contract: `vColor`/`vTextureCoord`, the draw's texture at slot 0 (`uSampler`), `uTex1`,
  `uTex2` (at most 2 extra), uniforms `uData[]` as vec4s (`backend/src/shaders/program_example.fragment.hlsl`;
  cap `GpuRenderer.uniform_cap`, `GpuRenderer.zig:51`; web cap 64, `WebBackend.zig:908`).
- **Native compile path.** `SDLBackend.program_api.createNative` (`SDLBackend.zig:3171`) ->
  `GpuRenderer.programCreate` (`GpuRenderer.zig:1251`) takes MSL, SPIR-V or DXIL by the device's
  `shader_format`. MSL compiles on a thread (`Compile`, `:130`); `programStatus` (`:1296`) is
  failed/compiling/ready, and `Program.ready` (`programs.zig:230`) keeps frames coming while compiling.
  Web: `program_api` (`WebBackend.zig:908`) hands GLSL ES to `web/web.js` (`programCreate`, `programLinked`),
  which logs link errors to the console only.
- **How the glass shader gets compiled.** One HLSL source; `scripts/compile-shaders.sh` runs
  shadercross (SPIR-V, DXIL, MSL) and spirv-cross (GLSL ES 300 for WebGL2); the Shaders workflow
  (`.github/workflows/shaders.yml`) rebuilds and byte-compares them. All offline. Nothing compiles
  source at runtime today: a plugin that edits shaders must.
- **Text editor hooks to ride.** `LanguageSupport.VTable` (`sdk/src/language.zig:42`):
  `treeSitterHighlight`, `supportsPreview`/`previewPane` (the text|preview split: `bytes` is the live
  document text, called every frame the pane is visible; `plugins/text/src/TextEditor.zig:173`),
  `documentOpened`, `hover`, `completion`, `signatureHelp`, `gotoDefinition`, `format`. The zig
  plugin is the template: `~/dev/fizzyedit/zig/plugin.zig` + `src/Highlight.zig` + `src/Lsp.zig`
  (zls over the shared `core.lsp.Client`, one `Client` instance per language).
- **Textures.** The plugin shares dvui with the host (render bridge, `sdk/src/render_bridge.zig`), so a
  `dvui.Texture` handed over by the host is drawable and bindable to a program (`DrawOptions.textures`).

## What does not exist

1. **Diagnostics.** Nothing in fizzy shows them. `core/lsp/Client.zig:1663` drops
   `textDocument/publishDiagnostics`; the text editor has no squiggle, gutter mark or problem list;
   `LanguageSupport` has no hook. Needed for both LSP errors and our own compile errors.
2. **Program lifetime.** `GpuRenderer.programs` (`:210`) is append-only and `Hooks` has no destroy.
   Hot reload makes a program per keystroke burst; ids and `SDL_GPUShader`s (and the pipeline cache
   keyed by program id, `pipeline :462`) would grow for the session.
3. **Which form to compile to.** The plugin cannot ask what the device wants (MSL / SPIR-V / DXIL / GLSL).
   `Source` carries all four forms; compiling all of them per edit is wasteful.
4. **Why a program failed.** `status` returns 0 and nothing else. The backend's `SDL_GetError` text
   (Metal's compile log, Vulkan's validation) and WebGL's `getShaderInfoLog` never reach the plugin.
5. **Binding a sprite.** No service gives another plugin a texture of an open pixi document or its atlas.
6. **Offscreen targets for plugins.** Needed only for multipass/feedback and a render-scale slider: later.

## Design

### Pipeline

```
text bytes --(hash, debounce ~150 ms)--> job thread:
  wrap (prelude per language + Shadertoy `mainImage` adapter)
  front end -> SPIR-V          (glslang: GLSL, HLSL)       errors: file:line:col -> Diagnostic
  back end  -> device form     (SPIRV-Cross: MSL | GLSL ES 300; SPIR-V as is for Vulkan)
  -> core.programs.Program.create(...)   (async on the backend: Compile thread / link)
frame: previewPane draws newest READY program; the previous good one until then.
```

- **The plugin owns the compiler; the backend stays a loader.** The host gains no compiler and no
  size. A user who never installs the plugin pays nothing (AGENTS.md speed rule). The existing seam
  already takes finished MSL/SPIR-V/DXIL; we add three small hooks to it (below), not a front end.
- **Never crash the frame.** All compiling is off the UI thread (`core.work` thread task; on web a
  pump step, the browser compiles asynchronously via `KHR_parallel_shader_compile`). A failure only
  produces diagnostics and a status line; the pane keeps drawing the last good program, or a
  checkerboard if there never was one. Backend failures already degrade (`pipeline()` returning null
  skips the draw, `GpuRenderer.zig:462`).
- **Hangs.** A fragment loop that never ends can hang the GPU; nothing in-process can stop it.
  Mitigation: the plugin keeps a sentinel (`running <hash>`) in its profile dir, cleared after the
  first frame that presents; found at startup, the preview starts paused with a banner. Preview
  runs only while the pane is visible, and a "pause" toggle stops frames.
- **Cost when unused.** `previewPane` is not called unless the pane is open. The plugin scans the wrapped
  source for time/mouse/frame use: a static shader redraws on change only; an animated one calls
  `dvui.refresh` each frame (the same as the program example), paused when the window is unfocused.

### Languages and compilers

| Source | Front end | To MSL | To SPIR-V | To GLSL ES (web) | To DXIL |
|---|---|---|---|---|---|
| `.glsl/.frag` (Vulkan-flavoured, Shadertoy `mainImage`) | glslang | SPIRV-Cross | glslang | SPIRV-Cross | see below |
| `.hlsl` | glslang's HLSL front end (DXC later if needed) | SPIRV-Cross | glslang | SPIRV-Cross | see below |
| `.wgsl` | naga (Rust) via a tiny C shim, later step | naga msl-out | naga spv-out | naga glsl-out | see below |
| `.metal` | none: the text is the MSL (entry `main0`); macOS only | - | - | - | - |

Candidates, honestly: (sizes are estimates to measure in step 7, not facts)

| Library | Licence | Gives | Rough size in a dylib | Verdict |
|---|---|---|---|---|
| glslang | BSD-3 / Apache-2 (parts) | GLSL, HLSL -> SPIR-V, errors with lines | ~3-5 MB | yes: step 7 |
| SPIRV-Cross | Apache-2 | SPIR-V -> MSL / HLSL / GLSL ES | ~1-2 MB | yes; CI already uses it for the glass GLSL |
| naga | MIT / Apache-2 | WGSL, GLSL, SPIR-V in; MSL, HLSL, GLSL, SPIR-V, WGSL out | ~1.5-3 MB (Rust static lib) | yes for WGSL; needs a cargo step in plugin CI, so later |
| tint (Dawn) | BSD-3 | WGSL <-> SPIR-V/MSL/HLSL | heavy build tree | no: too much to vendor |
| shadercross (SDL_shadercross) | zlib | SPIR-V/HLSL -> MSL/DXIL | pulls DXC | no as a library; it is what CI uses |
| DXC | Apache-2 + LLVM exception | HLSL -> DXIL/SPIR-V | 20 MB+, a separate DLL | not bundled |
| Slang | Apache-2 + LLVM exception | everything | 30 MB+ | not now |

Zig builds C/C++ without a system toolchain, so glslang and SPIRV-Cross are `build.zig.zon`
dependencies built per target by `fizzyedit/plugin-build-action`, like any plugin's C code. The store
download grows by the compiler's size; the host does not.

**DXIL on Windows (open).** No small DXIL producer exists (it is signed by `dxil.dll` and made by DXC).
Options, in order: (a) pass SPIR-V and have the host open its device on Vulkan when a shader plugin is
active (SDL_GPU supports Vulkan on Windows), costing a device choice the host must expose; (b) check
whether SDL_GPU's D3D12 driver accepts DXBC from SPIRV-Cross HLSL + `d3dcompiler_47.dll` (present on
every Windows) - SDL_GPU lists DXBC as a D3D12 format; to be verified in step 1 before relying on it;
(c) load a system `dxcompiler.dll` if present. Plan: ship macOS (MSL) and Linux (SPIR-V) first; Windows
follows the answer to this question (`Source` would gain `dxbc` only if (b) holds: an addition at the end).

**Web.** The web build is WebGL2/WebGL1 (`web/web.js`), not WebGPU, so a WGSL file has no native
meaning there and WebGPU is out of scope (it would be a backend of its own). On web the browser *is*
the GLSL ES compiler and reports line numbers (`getShaderInfoLog`): for GLSL the plugin passes source
against the web prelude (`VARYING`, `TEX`, `FRAG_COLOR`, `uData`, `programs.zig` header) and needs no
compiler at all. HLSL/WGSL on web would need naga or glslang built to wasm (~1-3 MB, loaded lazily as
part of the side module); deferred until the desktop path ships. Plugins run as wasm side modules there
(`web/AGENTS.md`), so `has_lsp`-style gating applies: no language servers, no tree-sitter (`build/web.zig:46`).

### Inputs (Shadertoy-style, packed by the plugin)

The plugin owns the wrapper; the host only knows `uData` vec4s. A per-language prelude declares the
names, so user code reads like Shadertoy:

```
uData[0] = (iResolution.xy, iTime, iTimeDelta)
uData[1] = iMouse   (xy current px, zw click px, sign = held, as Shadertoy)
uData[2] = (iFrame, iDate.x(sec of day), pointer-in-pane flag, scale)
uData[3..] = user `uniform` values from the pane's panel (floats/vec3 colour), by `// @uniform name: range`
```

`#define iTime uData[0].z` and kin (GLSL), `static` accessors (HLSL/WGSL/MSL preludes). Adapter: a file with
`void mainImage(out vec4, in vec2)` is wrapped with `main()`; a file with its own `main` (raw mode) gets
only the prelude. `fragCoord` is derived from `vTextureCoord * iResolution` with y flipped (dvui uv is
top-left origin, Shadertoy bottom-left). Prelude lines are subtracted from error lines. Mouse comes
from dvui events inside the pane rect. Time is `dvui.frameTimeNS` (pausable, scrubbable, resettable).
Up to 3 channels: `iChannel0` = `uSampler` (the draw's own texture), `iChannel1/2` = `uTex1/2`
(the backend's cap of 2 extra textures, `programs.zig:98`).

### Textures: pixi sprites and atlases through a service

A new SDK service, `"imagesource"` (std + dvui only, in `sdk/src/services/`, the shape of `wikilink.zig`
and `workbench.zig`): `Api { list() []Entry{id, name, kind: sprite|atlas|file, w, h}, texture(id) ?dvui.Texture,
generation(id) u64, region(id, name) ?Rect }`, borrowed per frame, `service_version` for growth. pixi
registers it (its flattened open document / atlas and each sprite's rect); the shader plugin
looks it up with `getServiceTyped`, never imports pixi, and without it offers only files from the open
folder (decoded by dvui). Channel choice is a header pragma, `// @channel1 pixi:open` /
`// @channel1 file:assets/noise.png`, so the file is the source of truth and the preview reloads on
an edit of it; the pane has a picker that writes the pragma. A new service does not move the SDK
fingerprint (it is versioned by name and `service_version`, `sdk/src/Host.zig:1406-1444`). Texture
sampling (nearest for pixel art, wrap vs clamp) is the backend's default now; per-channel sampler state
is an open question and would be a `Hooks` addition.

### Diagnostics (the editor seam)

New service `"diagnostics"`, provided by the `text` plugin (the owner of the editor), consumed by any
language plugin: `publish(path, source_id, []Diagnostic{ start_line, start_col, end_line, end_col,
severity, message, code })` and `clear(path, source_id)`. Text draws squiggles under the range, a gutter
mark, the message on hover, and next/previous-problem navigation. A service, not a `LanguageSupport`
hook: `LanguageSupport.VTable` is in `sdk_boundary_types` (`sdk/src/dylib.zig:113`), so a field there would
move the fingerprint and every plugin would need a rebuild; a service does not, and a missing provider
(no text plugin) degrades to the plugin's own error list under the preview. `core.lsp.Client` also
starts keeping `publishDiagnostics` per URI and exposes `diagnosticsFor(uri)`; every language plugin,
the zig one included, then publishes its server's diagnostics with a few lines. The shader plugin
merges two sources: the compiler's (`source_id = "shader.compile"`, immediate, from our front end) and
the server's.

### Language servers

`glsl_analyzer` (nolanwhite/glsl_analyzer, written in Zig, MIT), `wgsl-analyzer` (Rust, MIT/Apache) and
the multi-language `shader-language-server` (antaalt, MIT; GLSL/HLSL/WGSL) are the candidates, to be
confirmed in step 9. Metal has no server worth using: highlight and our compile diagnostics only. Same
discovery as `zig/src/Lsp.zig` (PATH, then login shell), one `core.lsp.Client` per language; the
server is a prerequisite the user installs, not bundled, and its absence shows a one-line hint, not an
error. LSP is native only.

### Highlighting

Tree-sitter grammars (GLSL, WGSL, HLSL; C++-based queries for Metal) compiled into the plugin dylib
(`extern fn tree_sitter_glsl()`, as `zig/src/Highlight.zig:12-14` does for the grammar the host links).
The zig plugin relies on a grammar the host links; these do not exist in the host, so they are built into
the plugin and must match the host runtime's tree-sitter ABI (checked in step 6). No highlighting on web.

### Later: Zig game hot-reload pane

Same plugin, another mode: a `zig build` watcher rebuilds a game as a dylib, and a pane shows the running
game (frame into a texture via an offscreen target, or the game's own window). Needs the plugin to own
an offscreen render target and a process/dylib lifecycle: a plan of its own once steps 1-8 land; it
rides the same compile-diagnostics path (zig errors to the diagnostics service).

## SDK / host seams missing, and fingerprint impact

| Seam | Where | Fingerprint? |
|---|---|---|
| `Hooks.destroy(id)`, free list in `GpuRenderer.programs`, pipeline cache purge | `programs.zig`, `GpuRenderer.zig`, `SDLBackend.zig`, `WebBackend.zig`, `web.js` | no: `Hooks` is not an SDK boundary type; bump `programs.abi_version` |
| `Hooks.formats()` bitmask of accepted forms (msl/spirv/dxil/glsl) and `Hooks.lastError(id)` text | same | no (same) |
| `diagnostics` service + `core.lsp.Client` diagnostics store | `sdk/src/services/`, `core/lsp/Client.zig`, `plugins/text` | no (service); `core` is in the SDK package, so it rides an SDK release |
| `imagesource` service | `sdk/src/services/`, pixi | no |
| Per-channel sampler state, offscreen target for plugins | `Hooks` / dvui | no, later |
| Anything on `LanguageSupport.VTable` (e.g. a diagnostics poll hook) | `sdk/src/language.zig` | **yes**: avoided on purpose |

`Hooks` growth: add the new fields at the end and have `programs.hooks()` accept a smaller
`size`, so an old plugin on a new host keeps its programs. A bare `abi_version` bump would silently
disable every plugin's programs (`programs.zig:169`). A plugin that "owns a GPU program" is thus
already supported; what is missing is its lifecycle (destroy) and feedback (formats, errors).

## Steps

Each is one PR, <= 800 lines of diff outside tests and docs. "Cloud" = Linux, tests only. The rest
need macOS/GPU or the user's eye.

1. **core: program lifecycle, formats, last error.** Repo `fizzy`, branch `core/program-lifecycle`. Builds on: nothing.
   `Hooks.destroy`, `formats`, `lastError`; free list; pipeline purge; web.js `deleteProgram` and
   info-log capture; `size`-tolerant `hooks()`. Also answer the DXBC-on-D3D12 question. Needs GPU (macOS Metal; web in the browser). Std tests for the free list: cloud.
   *Kickoff.* Task: let a plugin destroy a program and read why one failed. Plan: SHADER_PLUGIN_PLAN step 1. Builds on: nothing. Workspace: `~/dev/fizzy-program-lifecycle`. Branch: `core/program-lifecycle`. Notes: append fields to `Hooks`, keep old readers working; `GpuRenderer.pipeline` caches by program id, purge on destroy; capture SDL_GetError in `Compile.run`; do not touch `sdk_boundary_types`.
2. **core: LSP diagnostics store.** Repo `fizzy`, branch `core/lsp-diagnostics`. Builds on: nothing. `Client` keeps `publishDiagnostics` per URI, `diagnosticsFor(uri)`, thread-safe, `refresh()` on change; fixture-based tests. Cloud.
   *Kickoff.* Task: keep publishDiagnostics in `core.lsp.Client`. Plan: SHADER_PLUGIN_PLAN step 2. Builds on: nothing. Workspace: `~/dev/fizzy-lsp-diag`. Branch: `core/lsp-diagnostics`. Notes: parse at `Client.zig:1663`; ranges are UTF-16 LSP positions, convert with `UriUtil`; free on didClose/restart; tests on canned frames, no server.
3. **sdk+text: `diagnostics` service and squiggles.** Repo `fizzy`, branch `text/diagnostics`. Builds on: 2 (for zls), not required for the service. Service type, text provider, squiggles, gutter, hover, next-problem command; zig plugin follow-up is a separate PR in `fizzyedit/zig`. Needs the user's eye (label `sdk` only if a boundary type changes; it should not).
   *Kickoff.* Task: draw diagnostics in the text editor via a service. Plan: SHADER_PLUGIN_PLAN step 3. Builds on: 2. Workspace: `~/dev/fizzy-text-diag`. Branch: `text/diagnostics`. Notes: service, not a VTable field (fingerprint); degrade if absent; squiggle cost only when a path has diagnostics; look at it in a sandbox with `fizzy_screen`.
4. **sdk: `imagesource` service definition.** Repo `fizzy`, branch `sdk/imagesource-service`. Builds on: nothing. Types, version, registration helper, tests with a fake provider. Cloud.
   *Kickoff.* Task: define the imagesource service. Plan: SHADER_PLUGIN_PLAN step 4. Builds on: nothing. Workspace: `~/dev/fizzy-imagesource`. Branch: `sdk/imagesource-service`. Notes: model on `services/wikilink.zig`; borrowed textures valid for one frame; `generation` for change detection; nothing from pixi here.
5. **pixi: provide `imagesource`.** Repo `fizzyedit/pixi`, branch `pixi/imagesource`. Builds on: 4 released in an SDK. Flattened open doc, atlas, per-sprite rect; no copy when a texture already exists. Needs macOS and the user's eye.
   *Kickoff.* Task: pixi offers its documents and atlas as textures. Plan: SHADER_PLUGIN_PLAN step 5. Builds on: 4. Workspace: `~/dev/pixi-imagesource`. Branch: `pixi/imagesource`. Notes: re-pin the SDK; do not read pixi from the shader plugin; bump `generation` only on a changed composite; measure with `fizzy_profile` that an idle doc costs nothing.
6. **shader: scaffold and highlighting.** Repo `fizzyedit/shader` (new), branch `shader/scaffold`. Builds on: nothing. `plugin.zig`, zon, ICON, `register`, extensions, GLSL/WGSL/HLSL grammars + queries, CI via plugin-build-action, entry in the store registry repo. Check the tree-sitter ABI. Cloud for build; the look needs the user.
   *Kickoff.* Task: new plugin repo with highlighting. Plan: SHADER_PLUGIN_PLAN step 6. Builds on: nothing. Workspace: `~/dev/fizzyedit/shader`. Branch: `shader/scaffold`. Notes: copy `~/dev/fizzyedit/zig` layout; grammars as zig deps compiled into the dylib; `has_lsp`-style wasm gating; Metal reuses a C++ query set.
7. **shader: compile pipeline.** Repo `fizzyedit/shader`, branch `shader/compile`. Builds on: 6. `Wrap.zig` (preludes, `mainImage`), `Compile.zig` (glslang + SPIRV-Cross deps, job thread, line mapping, Diagnostics out), tests: golden SPIR-V/MSL for sample shaders and error line mapping. Measure dylib size and per-edit latency. Cloud for tests.
   *Kickoff.* Task: GLSL/HLSL source to SPIR-V/MSL/GLSL ES with mapped errors. Plan: SHADER_PLUGIN_PLAN step 7. Builds on: 6. Workspace: `~/dev/fizzyedit/shader-compile`. Branch: `shader/compile`. Notes: pure functions of (text, language, target form); errors subtract prelude lines; record sizes and licences in the README; no UI here.
8. **shader: live preview pane.** Repo `fizzyedit/shader`, branch `shader/preview`. Builds on: 1, 3, 7. `supportsPreview`/`previewPane`, `Program` swap with last-good, destroy of replaced ones, inputs, pause/reset/time scrub, sentinel for hangs, diagnostics publish. macOS first. Needs macOS/GPU and the user's eye.
   *Kickoff.* Task: draw the shader in the preview split with hot reload and inline errors. Plan: SHADER_PLUGIN_PLAN step 8. Builds on: 1, 3, 7. Workspace: `~/dev/fizzyedit/shader-preview`. Branch: `shader/preview`. Notes: debounce ~150 ms; never block a frame; keep the last good program; animate only if the source uses time or mouse; verify idle frames with `fizzy_profile`.
9. **shader: language servers.** Repo `fizzyedit/shader`, branch `shader/lsp`. Builds on: 2, 3, 6. One `core.lsp.Client` per language, discovery as the zig plugin, hover/completion/definition, diagnostics merged with compile errors. Needs the servers installed: macOS/Linux by the user.
   *Kickoff.* Task: glsl_analyzer / wgsl-analyzer / shader-language-server support. Plan: SHADER_PLUGIN_PLAN step 9. Builds on: 2, 3, 6. Workspace: `~/dev/fizzyedit/shader-lsp`. Branch: `shader/lsp`. Notes: confirm which servers still exist and their licences first; absent server is a hint, not an error; copy `zig/src/Lsp.zig`; native only.
10. **shader: textures and uniform panel.** Repo `fizzyedit/shader`, branch `shader/channels`. Builds on: 5, 8. `// @channel` pragmas, file textures, pixi channel through `imagesource`, `// @uniform` sliders. Needs the user's eye.
    *Kickoff.* Task: bind sprites, atlases and files as iChannels and tune uniforms. Plan: SHADER_PLUGIN_PLAN step 10. Builds on: 5, 8. Workspace: `~/dev/fizzyedit/shader-channels`. Branch: `shader/channels`. Notes: max 2 extra textures; pixel-art sampling is an open backend question; absent pixi falls back to files.
11. **shader: WGSL, HLSL and Metal paths.** Repo `fizzyedit/shader`, branch `shader/wgsl`. Builds on: 8. naga shim (cargo in CI), HLSL front end settings, `.metal` direct, Windows per the DXIL answer. Cloud for compile tests, GPU per OS for the rest.
    *Kickoff.* Task: compile WGSL and HLSL and run Metal source directly. Plan: SHADER_PLUGIN_PLAN step 11. Builds on: 8. Workspace: `~/dev/fizzyedit/shader-wgsl`. Branch: `shader/wgsl`. Notes: naga as a static lib with a C ABI shim; cargo step in plugin-build-action may need a change in `fizzyedit/plugin-build-action`; settle the Windows DXIL option from step 1.
12. **shader: web.** Repo `fizzyedit/shader`, branch `shader/web`. Builds on: 1, 8. GLSL through the browser's compiler, `getShaderInfoLog` to diagnostics, side module build. Needs a browser and the user's eye.
    *Kickoff.* Task: run the shader plugin on the web for GLSL. Plan: SHADER_PLUGIN_PLAN step 12. Builds on: 1, 8. Workspace: `~/dev/fizzyedit/shader-web`. Branch: `shader/web`. Notes: web prelude (`VARYING`, `TEX`, `FRAG_COLOR`); no LSP or tree-sitter; WebGL1 may lack high-precision fragments (`programs.zig` header).
13. **Zig game hot-reload pane (separate plan).** Written after step 8; needs an offscreen render target for plugins and a rebuild/dylib lifecycle.

## Open questions

- Windows DXIL: Vulkan device choice, DXBC via `d3dcompiler_47`, or a system `dxcompiler.dll`?
- Plugin bundle size is acceptable at roughly 5-8 MB (glslang + SPIRV-Cross, then naga)? Measure in step 7.
- Is `shader-language-server` or a per-language server the one to recommend? Which still work with current specs?
- Per-channel sampler state (nearest, wrap) and an offscreen target: extend `Hooks` now or later?
- Should the preview also be a standalone Surface (a dockable pane, not only the text|preview split) so one pane can follow the active document?
- Diagnostics as a service in `text` or a bundled-neutral location, if a second editor plugin ever appears?
- Raw `.frag` files with no Shadertoy convention: how much of the contract do we infer vs require in a header comment?
