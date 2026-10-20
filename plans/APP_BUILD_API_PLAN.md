# App build API: the pieces `buildApp` is made of

Status: decided 2026-10-19, ready to build (fizzyedit/fizzy#527). Requested by Graphl, an app built on fizzy as a library.
Related: `WEB_EMBED_PLAN.md` (its web build options are this plan's `addWeb`),
`EMBEDDED_EDITOR_PLAN.md` (its `addEmbedded` is built on step 2 here), fizzyedit/plans#13 (apps
choose fizzy's plugins; done but its step 3), `LIB_CHECKPOINT.md`.

## The problem

`fizzy.buildApp(fizzy_dep, plugins)` runs `build/app.zig`'s `construct` on **fizzy's** builder
and returns nothing. Everything it makes (the exe, the packaged exe, the dylibs, the web build,
the `package` step, the tests) is a step of fizzy's builder, found by name
(`fizzy_dep.artifact(name)`, `fizzy_dep.namedLazyPath("package")`). An app that needs anything
different reaches into those steps and edits them. Graphl does that today for:

- **`rehome` (`build/plugins.zig`) copies only a plugin's root module.** It swaps the root's
  `dvui`/`core`/`fizzy_sdk`/`icons` and keeps every other import as is, so a submodule that
  imports dvui still has the native executable's dvui and target, and the copy loses the root's
  C sources, include paths and link objects. A plugin that brings a dvui library (Graphl's do)
  fails to build for the web.
- **The web build is closed.** `build/web.zig` fixes the wasm export list, installs fizzy's
  `web/index.html`, and puts `web`, `check-web` and the installs on fizzy's builder.
- **No dvui for another backend from the SDK package.** A plugin's tests, or a host, write
  `sdk.builder.dependency("dvui", .{ .backend = .testing })` by hand (example-app's replay app
  does exactly this).
- **A plugin can't reach the executable's `platform` module** (title-bar hints, window icon).

Graphl suggests building blocks in the style of velopack-zig 0.1.0 and sentry-zig: `addApp`,
`addWeb`, `addPackage`, `dvuiModule`, each returning artifacts and `LazyPath`s, with `buildApp`
a thin wrapper over them so example-app stays as it is.

## The rule

- **A block returns what it made; the caller installs it.** Each block takes the fizzy
  dependency and options and returns a struct of `*Compile`s, `*Module`s and `LazyPath`s. It
  adds no top-level step and installs nothing. `addPackageSteps` (#537) already works this way
  for packaging: the app's `b` gets the steps, fizzy's output is a `LazyPath`.
- **`buildApp` is the blocks composed the way fizzy composes them**, on fizzy's builder, with
  fizzy's steps. Fizzy's own `build.zig` keeps calling `construct` and gets byte-for-byte the
  same graph.
- **One wiring per executable kind, written once.** Today the editor module is wired three
  times: the exe (`build/exe.zig` `addFizzyExecutableForTarget`), the web (`build/web.zig`
  `addSteps`) and `test-integration` (`build/app.zig`, the `fizzy_test_module` block). Each
  block calls one function that wires the editor onto a given dvui and backend. That function is
  also what `EMBEDDED_EDITOR_PLAN.md` exposes.
- **Options stay `b.dependency` options** (identity, branding, layout, `dynamic-plugins`), read
  once by fizzy's `build()` into `deferred`. A block's own options are what a dependency option
  can't carry: modules, `LazyPath`s, lists.
- **Copy, don't configure.** A block is one file a reader can follow. An app that needs a
  different web page passes its own page; it does not get a flag per element of fizzy's.

## The API, when done

```zig
const fizzy = @import("fizzy");
const fizzy_dep = b.dependency("fizzy", .{ .target = target, .optimize = optimize,
    .@"defer-app" = true, .@"app-name" = @as([]const u8, "graphl") });
const plugins: []const fizzy.BundledPlugin = &.{ fizzy.plugins.workbench, fizzy.plugins.text,
    .{ .name = "graphl", .module = graphl_dep.module("plugin") } };

const app = try fizzy.addApp(fizzy_dep, .{ .plugins = plugins });
b.installArtifact(app.exe);                // also: app.packaged_exe, app.dylibs, app.modules

const web = try fizzy.addWeb(fizzy_dep, .{
    .plugins = plugins,
    .exports = &.{ "GraphlRunGraph" },     // beyond fizzy's own
    .page = b.path("web/index.html"),      // instead of fizzy's
});
b.step("web", "…").dependOn(&b.addInstallDirectory(.{ .source_dir = web.dir,
    .install_dir = .prefix, .install_subdir = "web" }).step);

try fizzy.addPackage(b, fizzy_dep, app, .{});   // `package`, `desktop`, `packageall` on b

// In the SDK package, for a plugin's tests or a host:
const d = fizzy_sdk.dvuiModule(b, sdk_dep, .{ .target = target, .optimize = optimize, .backend = .testing });
```

`App.modules` is `{ dvui, backend, platform, core, sdk, app }` as the exe has them, so an app
can build a tool or a test against the same framework its exe links.

## Steps

Each step is one fizzy PR, `build:` area, ≤ 800 lines outside tests and docs. Every step runs the
full gates (`scripts/check-fmt.sh`, `zig build`, `test`, `test-integration`, `check-web`,
`test-sdk-version`, `check-tests-reached`) and `scripts/check-examples.sh`. None moves the SDK
fingerprint.

**Order and what can start now.** 1, then 2 and 8; 3 and 6 are independent and can run beside
them; then 4 (needs 1, 2, 8), 5 (needs 4), 9 (needs WEB_EMBED_PLAN step 2), 10 last; 7 needs its
velopack-zig PR, which can start at once, then step 5. **Can start now:** 1, 3, 6, step 7's
velopack-zig PR, and 8 (rebased onto 1 if that lands first). Each step ends with a kickoff: Task,
Plan, Builds on, Workspace (a worktree at `~/dev/fizzy-<slug>`), Branch, Notes.

1. **`construct` returns what it made.** Split `build/app.zig` `construct` (≈ 1200 lines from
   `readConfig` to the end of the tests) into `native(…) → Native`, `web.build(…) → Web` and the
   test wiring, plus `install(…)`, which adds the steps and installs that `construct` adds today.
   `build/web.zig` `addSteps` splits the same way: `build(…) → Web { wasm, dir }`, where `dir` is a
   `WriteFiles` holding the stamped `index.html`, `web.js`, the worker, the font, the OAuth page
   and plugin pages, and `addSteps` installs `dir`. The dylib installs, repeated four times in
   `construct` and in `exe.addVelopackPackDirInstall`, become a loop over
   `FizzyExecutable.dylibs` (a slice of `{ id, *Compile }`). No public change.
   *Verify:* the gates; `zig build web` installs the same file list with the same bytes
   (`diff -r` against a build from `main`); `zig build --help` lists the same steps.

   *Kickoff.* Task: `construct` returns what it made. Plan: APP_BUILD_API_PLAN step 1. Builds on: nothing (can start now). Workspace: `~/dev/fizzy-construct-split`. Branch: `build/construct-split`. Notes: no public change; `diff -r` the web output against main.

2. **One wiring for the editor on any dvui.** Add `build/editor.zig` with
   `wire(b, .{ target, optimize, dvui, backend, platform, proxy?, plugins, build_opts, … })
   → Modules`: core, icons, sdk, the linked plugins (`plugins.link`), `bundled_plugins`, `app`,
   `app_layout`, nightwatch, singleton. The exe, the web build and `test-integration` call it.
   The test build then links its plugins through `plugins.link` like the others, which drops the
   test build's own list (fizzyedit/plans#13 left it, "since it tests fizzy itself": the list
   becomes `plugins.entries.all`, which is the same thing). `EMBEDDED_EDITOR_PLAN.md` step 4 is
   this function exposed.
   *Verify:* the gates; `check-tests-reached` still finds every test; integration test count
   unchanged.

   *Kickoff.* Task: one editor wiring (`build/editor.zig`). Plan: APP_BUILD_API_PLAN step 2. Builds on: step 1. Workspace: `~/dev/fizzy-editor-wire`. Branch: `build/editor-wire`. Notes: EMBEDDED_EDITOR_PLAN step 4 exposes this function.

3. **`rehome` copies the import graph.** In `build/plugins.zig`, `rehome` walks the plugin's
   module graph and copies every module that reaches `dvui`, `core`, `fizzy_sdk` or `icons`
   (memoised, so a shared submodule is copied once). Each copy takes the new target, optimize,
   `single_threaded` and `link_libc` and keeps the source module's C sources, include paths,
   system include and framework paths, link objects, linked libraries and C macros. Modules that
   reach none of the four are shared, not copied. A module with C sources that targets the web
   gets `link_libc = false`; if its C needs a libc, the build fails with the module's name.
   *Verify:* a fixture plugin under `tests/fixtures/rehome/` with a submodule that imports dvui
   and a C file, linked into the test build and the web build (`check-web`); example-app's
   `shader` plugin on the web; Graphl's plugin, built by its maintainer, without its patch.

   *Kickoff.* Task: `rehome` copies the import graph. Plan: APP_BUILD_API_PLAN step 3. Builds on: nothing (independent; can start now). Workspace: `~/dev/fizzy-rehome-graph`. Branch: `build/rehome-graph`. Notes: Graphl builds its plugin to confirm.

4. **`fizzy.addApp` and `fizzy.addWeb`.** Public in `build.zig`, over step 1's `native` and
   `web.build`. `addApp(fizzy_dep, .{ .plugins, .layout? }) -> App { exe, packaged_exe?, dylibs,
   modules }`. `addWeb(fizzy_dep, .{ .plugins, .exports = &.{}, .files = &.{} }) -> Web { wasm,
   dir }`: `exports` are appended to the list step 8 moved beside `web_main.zig`, `files` are
   copied into `dir`; fizzy's `web/index.html` is used as is until step 9 adds `page`. `buildApp`
   becomes `addApp` + `addWeb` + install on fizzy's builder, as today. Both blocks read
   `deferred`, so `defer-app` stays the switch. Document both in `docs/APPS.md` (new, public:
   building an app on fizzy, from `AGENTS.md` "Bundling plugins into an app" and the doc comments
   in `build.zig`).
   *Verify:* example-app unchanged and green; the blocks build is step 10.
   *Kickoff.* Task: public `addApp` and `addWeb` (exports, files). Plan: APP_BUILD_API_PLAN step 4. Builds on: steps 1, 2 and 8. Workspace: `~/dev/fizzy-add-app`. Branch: `build/add-app-web`. Notes: `buildApp` keeps working and is not deprecated until step 10.

5. **`fizzy.addPackage`.** `addPackage(b, fizzy_dep, app, .{})` takes the `App` from step 4, so
   the packaged exe and dylibs are the ones the app holds, not found by step name.
   `addPackageSteps` stays as `addPackage` for a `buildApp` app. Packaging bugs filed apart
   from #527 are fixed in their own PRs, before or after.
   *Verify:* `zig build package` from example-app produces the same installer as from `main`
   (contents listed with `vpk`'s output, not only a green step) on macOS; `packageall` in CI.
   *Kickoff.* Task: `addPackage` over the `App`. Plan: APP_BUILD_API_PLAN step 5. Builds on: step 4. Workspace: `~/dev/fizzy-add-package`. Branch: `build/add-package`. Notes: same installer as `main`; compare contents, not only a green step.

6. **`dvuiModule` in the SDK package.** `sdk/build.zig` gets `pub fn dvuiModule(b, sdk_dep,
   .{ target, optimize, backend: enum { testing, sdl3, web, custom } }) → { dep, dvui, backend? }`,
   from the one dvui pin, with the option sets fizzy's own builds use (`build/web.zig`'s and
   `build/exe.zig`'s `fizzyNativeDvui`; they call it, so the sets can't drift and the
   fingerprint stays one). `.custom` returns dvui with no backend, for a host that links its own
   (`EMBEDDED_EDITOR_PLAN.md`). example-app's replay app uses it (PR there). It reaches plugins
   with the next SDK release: build surface, not the boundary, so no fingerprint move.
   *Verify:* `test-sdk-version`; a plugin's `zig build test` on `.testing`; example-plugin.
   *Kickoff.* Task: `dvuiModule` in the SDK package. Plan: APP_BUILD_API_PLAN step 6. Builds on: nothing (independent; can start now). Workspace: `~/dev/fizzy-dvui-module`. Branch: `sdk/dvui-module`. Notes: build surface only, ships with the next SDK release; EMBEDDED_EDITOR_PLAN step 4 needs `.custom`.

7. **Velopack through velopack-zig's own build step, used directly.** The maintainer controls
   velopack-zig and wants it easy for other projects to use, so fizzy stops carrying a copy of
   its glue (`build/velopack.zig`, 292 lines, "mirrors velopack-zig's build.zig") and calls
   velopack-zig's step. The constraint is AGENTS.md's rule: the root zon keeps `velopack_zig`
   `.lazy = true` and no build script does `@import("velopack_zig")`, because a build-script
   import makes Zig fetch the package for every consumer. The rule is about the *import*, not
   about using velopack-zig, so it holds like this: fizzy resolves
   `b.lazyDependency("velopack_zig", .{})` only in `build/app.zig` (as today) and reaches
   velopack-zig through what its `build()` publishes, not through its helper source:
   `vz.artifact("velopack-pack")`, one executable wrapping the trim step, vpk, the dotnet tool
   restore and the mksquashfs/MSVC setup behind a documented command line (app dir, id, version,
   name, icon, channel, output), run with `addRunArtifact`. `build/velopack.zig` shrinks to that
   Run wiring (about 60 lines) plus the pure path logic (`resolveWindowsMsvcLibc`, which needs no
   dependency). The `plugins/image` acceptance test in AGENTS.md stays as is and must stay empty;
   AGENTS.md's sentence changes from "the helper surface is vendored" to "used through
   velopack-zig's published artifact". Projects without fizzy's laziness constraint keep using
   velopack-zig as a normal import (`addVelopackStep`), unchanged.
   **velopack-zig needs a change of its own (a PR in graphl-tech/velopack-zig, first):** its
   `build()` is a no-op that installs nothing today. It gains (a) the `velopack-pack` artifact
   with a stable CLI covering what fizzy's copy does (trim, name and icon carried through, MSVC
   libc handling for Windows cross builds), (b) a README section "Use from a build.zig" with the
   eager `@import` recipe and the lazy `lazyDependency` + artifact recipe, and (c) a tag, so
   fizzy's zon pins a release, not a commit. The fizzy PR follows that tag.
   *Verify:* the AGENTS.md acceptance test (empty `zig-pkg | grep -i velo`); `package` on macOS
   and Windows (VM) as in step 5, installer contents compared against `main`'s.
   *Kickoff.* Task: replace fizzy's vendored Velopack glue with velopack-zig's published pack
   artifact. Plan: APP_BUILD_API_PLAN step 7. Builds on: the velopack-zig PR and tag above (can
   start now), then step 5. Workspace: `~/dev/velopack-zig-pack` (that repo), then
   `~/dev/fizzy-velopack`. Branch: `pack-artifact` (velopack-zig), `build/velopack-zig-direct`
   (fizzy). Notes: two repos, two PRs; keep `.lazy = true`; keep the vendored file until the new
   path packages identically.

8. **`web`: the export list beside `web_main.zig`.** Move fizzy's wasm export list out of
   `build/web.zig` into a file next to `src/web_main.zig` (`export fn` and the list in one place;
   the build reads the list from there), so the list an app extends with `addWeb`'s `exports` is
   plainly the page's contract.
   *Verify:* `check-web`; `zig build web` output identical to `main` (`diff -r`).
   *Kickoff.* Task: put the web export list next to `web_main.zig`. Plan: APP_BUILD_API_PLAN
   step 8. Builds on: nothing hard (step 1 touches `build/web.zig` too; rebase onto it if it
   lands first). Workspace: `~/dev/fizzy-web-exports`. Branch: `web/export-list`. Notes: small;
   the build reads the list, it does not duplicate it.

9. **`addWeb`'s `page`.** After `WEB_EMBED_PLAN.md` step 2 (`startFizzy`), `addWeb` gains
   `page: ?LazyPath`, replacing `web/index.html` (stamped the same way: storage name and
   cache-buster), so an app's page is the small "import `fizzy.js`, call `startFizzy`" file and
   not a copy of fizzy's. The contract goes in `docs/APPS.md`.
   *Verify:* a page fixture served in `check-web`; example-app's blocks build (step 10) uses it.
   *Kickoff.* Task: `addWeb` custom page. Plan: APP_BUILD_API_PLAN step 9. Builds on: step 4 and
   WEB_EMBED_PLAN step 2. Workspace: `~/dev/fizzy-web-page`. Branch: `build/add-web-page`.
   Notes: do not start before WEB_EMBED step 2 merges.

10. **example-app on the blocks, and `buildApp` deprecated.** In fizzyedit/example-app, an
    `apps/blocks` build uses `addApp` + `addWeb` (an extra export and its own page) +
    `addPackage`; then the main example moves to the blocks. `buildApp` gets a deprecation doc
    comment and a line in `docs/APPS.md` ("use the blocks"), stays for one SDK release, and
    `check-examples.sh` fails if an example still calls it.
    *Verify:* example-app CI; `zig build package` identical to the `buildApp` build's.
    *Kickoff.* Task: move example-app to the blocks and deprecate `buildApp`. Plan:
    APP_BUILD_API_PLAN step 10. Builds on: steps 4, 5 and 9. Workspace: `~/dev/example-app-blocks`
    and `~/dev/fizzy-deprecate-buildapp`. Branch: `blocks` (example-app),
    `build/deprecate-buildapp` (fizzy). Notes: two repos, two PRs; remove `buildApp` in a later
    release, not here.

## What stays compatible

- **Fizzy's own build** keeps every step name and output; step 1 checks it by diff.
- **example-app** builds unchanged after every step but 10 (`check-examples.sh` in CI). Its `buildApp`
  call and `fizzy_dep.artifact(name)` keep working: `addApp` still names the exe after
  `app-name` on fizzy's builder.
- **Plugins** see nothing: no fingerprint move, no SDK release needed except to ship step 6.

## Out of scope, and where it goes

- **A plugin reaching `platform`.** A build-time import only works for a statically linked plugin
  and leaves dylibs and the web out. Title-bar hints and the window icon are host services: an
  `EditorAPI` member or a `window` service in the SDK, owned by the menu/title-bar issue Graphl
  refers to (not filed yet; to be filed with that seam). `App.modules.platform` is exposed for an
  app's own code meanwhile.

## Open questions

- **velopack-zig's published surface.** Is a `velopack-pack` executable with a CLI the right
  contract (fizzy stays lazy, AGENTS.md's rule unchanged), or would the maintainer rather have a
  tiny pure-Zig `velopack-zig-build` package with no dependencies of its own that fizzy imports
  eagerly? The second nearly removes fizzy's glue but changes AGENTS.md's rule.
- **Where velopack-zig lives.** The zon pins `graphl-tech/velopack-zig`; does it move to
  `fizzyedit` before it becomes the shared dependency?
- **Export list format.** A data file the build reads, or `export fn`s parsed out of
  `web_main.zig`'s source? The first is simpler; the second keeps one place to edit.
- **Removing `buildApp`.** Deprecated in step 10; in which SDK release is it removed?
