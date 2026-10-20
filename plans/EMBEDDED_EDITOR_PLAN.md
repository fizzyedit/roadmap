# Embedded editor: fizzy as the editor, a host shows a remote view

Status: decided 2026-10-19, rewritten around out-of-process embedding (fizzyedit/fizzy#529). Requested by Graphl,
which runs inside Blender and Unreal through blender-dvui and unreal-dvui and keeps a second shell of about 1200 lines
for them. The first version of this plan (fizzy linked into the host as `Embedded`) is kept as a short fallback at the
end; it is not built now.
Updated 2026-10-19 with the maintainer's decisions on repos, installing, builds, sandboxes, apps built on fizzy and
quiet-to-normal (steps 15-18).
Updated 2026-10-19 (second pass) with the maintainer's decision after Graphl's author proposed per-embed shells: **one
process stays the design**, the protocol is a C API (`fizzy_remote.h`) with an in-process and a socket transport, a
signed standalone component is the fallback, and a pasteboard exchange format bridges apps (steps 19-22).
Related: `CONTROL_SERVER_PLAN.md` and `docs/CONTROL.md` (the socket), `POPOUT_WINDOWS_PLAN.md` and
`NATIVE_WINDOWS_PLAN.md` (fizzy's own extra OS windows), `UNIFIED_DRAG_PLAN.md` (step 11), `SHARED_HISTORY_PLAN.md`,
`WEB_EMBED_PLAN.md` (keys only while focused), fizzyedit/plans#35 (`--quiet`; fizzy#645 landed the flag, #655 is the
backend acting on it). `APP_BUILD_API_PLAN.md` step 2 (one editor wiring) and step 6 are **no longer needed for
embedding**: nothing links fizzy's editor into a host any more. They stand for their own reasons.

## The decision

An installed fizzy is required. The person never opens it for the host's sake. The running fizzy is *the* editor; a host
(Blender, Unreal, ...) shows a **remote view**: a fizzy window whose surface, input and focus live in the host, driven
over the control socket.

- **Auto-start.** When a host needs an embedded view and no fizzy runs, the host's adapter starts the installed one
  silently with `--quiet` (no window, no focus; on macOS no Dock icon until a real fizzy window opens) and connects.
  When the last remote view closes and no fizzy window is open, that background fizzy exits after a grace period. A
  later normal launch of fizzy joins the same running instance (`app/single_instance` forwards to it) and shows a
  window.
- **Standalone fallback, not a second design.** With no fizzy installed or running, the adapter loads a signed editor
  component and runs it in-process ("Standalone mode" below). One process remains the design and the default.

### Why one process

Decided 2026-10-19, after Graphl's author proposed a shell per embed (each embed its own editor). The running installed
fizzy is the editor; hosts show remote views of it. What that buys:

- **One live document everywhere**: one save, no conflicts between copies.
- **One history, authorship and tape** across every host.
- **Agents and MCP see every view**, and claims work everywhere.
- **Drags move live views** (scroll, selection, undo state), not copies.
- **Plugins, language servers, watchers, indexes (Atlas), fonts and GPU resources exist once.**
- **Settings, theme, keymap and plugin installs once**, one updater.
- **Sign-in once.**
- **No plugin code in the host process**: crash isolation, and permissions decided in one place.
- **Rooms and presence once**, later.

What it costs, said plainly:

- it needs an install (the standalone component below is the answer for a person who has none);
- the transport and its failure modes;
- about a frame of latency for draw lists, and a glass fallback in tier 3;
- sandbox grants (Flatpak, sandboxed macOS hosts);
- a hung fizzy hangs every host's fizzy panels (the host keeps running; the add-on offers a restart).

## One interface for every window: `ViewHost`

Inside fizzy, its own OS windows and remote views are **the same thing**. The editor talks to one *view host*
interface (a `{ctx, vtable}` seam, as `app/` takes everywhere): the surface it draws into, size and content scale,
input events in, focus, IME state and cursor out, and the frame (begin, draw, end, present). Two implementations:

- **local**: fizzy's main window and its pop-out windows (`Popout`), in-process, zero IPC. Today's code put behind the
  interface.
- **remote**: a host over the control socket.

The editor never special-cases which. Fizzy's own windows do **not** go through the socket: speed is fizzy's edge, and
the web build is single-process. The remote path is exercised by tests through the same interface (a loopback client
drives a remote `ViewHost` in the integration tests, on Linux in CI). Step 1 defines the interface by **refactoring the
existing window and pop-out code onto it before any remote implementation exists**, so the interface is shaped by two
real users and not by the remote one alone.

**Considered and not chosen: every fizzy window a separate client process, Chrome-style** (one editor process, each
window a client of it; remote and local would then be literally one path). Rejected: it puts a process round trip into
every frame of fizzy's own windows; the macOS window work (Liquid Glass, native full screen, live resize,
`displayLayer`, view drags across windows, `NATIVE_WINDOWS_PLAN.md`) is built on one process owning every window and
would be redone across a boundary; and the web build has one process and no sockets. A host's view is the one place
the boundary is unavoidable, so it is the only place that pays for it.

## The protocol

Same socket, one more route. The control server is HTTP over a Unix socket (`app/control/Server.zig`; `0700` folder,
`0600` socket, same user only; AF_UNIX on Windows 10 1803+). A remote view starts as `GET /remote` with
`Upgrade: fizzy-remote-view/1`, answered `101`, after which the connection carries **length-prefixed binary messages**
both ways until it closes. One authentication model, one place to find fizzy, nothing on the network. The messages are
a std-only library, `sdk/remote/` (as `sdk/tape/` is), so fizzy and every adapter (through a C ABI) share one
definition. It adds nothing under `sdk/src/**`, so the plugin fingerprint does not move.

- **Versioning.** The first message each way is `hello {protocol versions supported, app id, fizzy version, tiers
  offered / wanted, feature bits}`. The highest common version is used; none in common closes with a reason the host
  can show ("update fizzy" / "update the add-on"). Fields append; a message type unknown to the peer is skipped by its
  length unless marked required.
- **Views.** `create_view {view id, kind (editor | a named region | a document), size, scale, tier, native handle for
  tier 1}` / `view_created` / `close_view` / `view_closed {reason}`. Many views per connection. A new view opens on the
  last workspace, so a second host shows the same documents.
- **Size and scale.** `resize {view, w, h in physical pixels, scale, rect in the host window if known}`; the scale is
  the host's backing scale (Retina 2x), sent, never inferred.
- **Input, focus, IME.** Host to fizzy: pointer motion, buttons, wheel, touch/pen, key down/up (host key code, a
  layout-independent code, modifiers), text input, IME composition (marked text and range), `focus {view, focused}`,
  clipboard replies. Fizzy to host: `cursor {shape}`, `ime_rect {caret rect}` (so the host places its candidate
  window), `text_input_active`, clipboard requests, `wants_keys` (does the view take a chord or does the host keep
  it), `open_url`, and for step 11 `drag_begin`.
- **Frames.** Host to fizzy: `frame_request {view, seq, deadline_us}`. Fizzy to host: `frame {view, seq, ...}`: tier 3
  carries the draw list and texture updates; tier 2 a fence and the texture slot; tier 1 nothing per frame (the layer is
  live), only damage hints. Fizzy can push `frame_ready` when a view changed without being asked (animation, a load),
  so the host redraws and asks.
- **Keys only while focused** (carried over from the first plan and `WEB_EMBED_PLAN.md` step 3): a remote view's chords
  (`app/keymap`'s `editor_focused` context) apply only while the host says the view has focus. The host keeps every
  chord otherwise.

## Latency

A socket round trip is microseconds, so the cost is **pipelining**, not transport. Tier 3 and 2: the host's frame starts,
it sends `frame_request` (input events were already streaming and are applied first), fizzy renders on its UI thread
within a budget and answers before the host's deadline. A remote view is rendered when asked or when `frame_ready` was
pushed, never on its own timer, so an idle view costs nothing. If an answer is late the host draws the last frame it has
and takes the new one next frame: one frame of lag in the worst case, never a stall. Adapters draw the **pointer and the
text caret themselves** from the last `cursor` / `ime_rect` and the live pointer, so those are never behind. Fizzy
renders a remote view as `Popout` renders a float (its own band of the frame, `Window.renderCommands` replayed into a
target of its own): no second dvui window and no dvui change.

## Three render tiers, chosen per adapter

Fizzy speaks all three; `hello` says which an adapter wants and which fizzy can do on this OS.

1. **Native surface in the host window.** Fizzy renders into its own native surface and the OS composites it into the
   host window; no per-frame copy. Needs a native handle and a rect from the host.
   - macOS: fizzy makes a `CAContext` for the view's layer and sends its context id; the host makes a `CALayerHost` with
     it (CARemoteLayerServer, what Safari's and Chromium's WebContent processes use). Private API: fine for a
     developer-tool adapter, a risk for the Mac App Store, so tier 3 stays the fallback.
   - Windows: a child HWND of fizzy's, `SetParent`ed to the host's HWND (as VST/AU plugin UIs are hosted); the host
     sends the parent HWND and rect, fizzy moves it.
   - X11: reparent fizzy's window into the host's window id.
   - **Wayland: not in general.** A client cannot embed another process's surface (`wl_subsurface` needs the same client;
     xdg-foreign only relates top-levels, so it can keep a floating fizzy window above the host but not inside it). An X11
     host under XWayland gets the X11 path; the rest uses tier 2 (dmabuf) or 3.
2. **Shared GPU texture.** Fizzy renders into a texture the host imports, with a fence for same-frame display: IOSurface
   + `MTLSharedEvent` (macOS), DXGI shared handle + shared fence (Windows), dmabuf + explicit sync (Linux). Needs the host
   to expose its GPU API (Blender's `gpu` module cannot import foreign textures; Unreal's RHI can). Fizzy keeps a ring of
   3; `frame` names the slot and fence value.
3. **Draw lists over the socket.** Fizzy renders into a *recording* dvui backend (the spike's: vertices, indices, draw
   commands, texture create/destroy) and sends the list; the host draws it in its own frame with whatever it has
   (Blender's `gpu`, Unreal's canvas). Universal, no GPU sharing, works in a sandbox that allows only the socket.
   **Glass and blur fall back in that view** (no backdrop to sample): opaque panel fills in their place.

## The Blender spike (`~/dev/fizzyedit/blender-dvui-spike`)

A recording dvui backend (`backend/src/backend.zig`), a C ABI (`src/lib.zig`: create/resize/events/frame and buffer
accessors) and a Python add-on (`addon/`) drawing the lists with Blender's `gpu` module; ported to fizzy's dvui and
Zig 0.17, headless check passing in Blender 5.2.2 on macOS. It is **tier 3 already**. What changes: the recording
backend moves into fizzy (it is tier 3's encoder, step 6); the C ABI shrinks to "protocol client + decode + input in",
so the add-on carries no dvui, no Zig version to match and no fonts; the Python draw code stays. Known defect: Retina
scale is left at 1x; step 7 sends the real `scale` in `resize` and honours it when drawing.

## The remote-view API: `fizzy_remote.h`, two transports

Decided 2026-10-19. The protocol of the next section is defined once, as a **C API** (`fizzy_remote.h`, generated from
`sdk/remote/`): create a session, create/resize/close views, push input, request a frame, receive frames, cursor, IME and
clipboard events, `drag_begin`. Two transports carry the same calls:

- **in-process**: function calls into an editor component loaded in the same process (standalone mode);
- **socket**: the `GET /remote` upgrade of the control server (shared, live).

The adapter (the "shell": Graphl's Blender add-on, `fizzyedit/blender`, the Unreal plugin) is written against the C API only
and **picks the transport at runtime**: fizzy installed and running (or startable quietly) -> socket; otherwise -> load the
editor component and run it in-process. An adapter learns which it has only from the `hello` reply
(`mode: shared | standalone`), which it shows.

## Standalone mode: the updateable editor component

Replaces the old in-process fallback (`Instance` / `addEmbedded`, kept below as history) as the fallback.

- **The component** is a **signed dylib per OS/arch** (`fizzy_editor.{dylib,so,dll}`) exporting the same C API, built from
  fizzy's editor with the compiled-in plugins and no window chrome. It is **not bundled in the add-on**: the shell downloads
  it on first use and updates it from **the app's own release feed** (GitHub Releases for fizzy, Graphl's own for Graphl),
  published with each app release and checked by the shell (hash-pinned, versioned against the protocol), into the host's
  per-user data folder. Hosts with **strict library validation** (hardened runtime, no unsigned/foreign-signer libraries)
  get one-process mode only: standalone is not available there. The Blender spike loaded an unsigned library fine, so
  Blender likely does not enforce it; verify per host in its adapter step.
- **The panel says so**: "standalone - not connected to fizzy", with a link to install fizzy, and it states that **documents
  here are copies** (own history, own settings, no agents or drags with the real fizzy). Moving content between standalone
  views and fizzy goes through the exchange format below.
- **Reconnecting when fizzy appears.** The panel offers to reconnect: unsaved edits are handed to fizzy as one step
  ("edited in <host>, standalone") and the panel switches to a live view. If the hand-over cannot apply cleanly, it falls
  back to the normal "changed on disk" reload/conflict prompt.
- **Costs** it accepts: a second editor in the host's process, its own plugin set (compiled in), a host crash from a fizzy
  bug. That is why it is the fallback.

## Exchange format: the bridge between apps

Decided 2026-10-19. Drag and copy/paste between *processes that do not share a document* (fizzy and a non-fizzy app, another
fizzy-based app, a separate profile, a standalone shell) go through one **fizzy exchange format** on the OS pasteboard
(NSPasteboard / OLE / Wayland and X11 selections), shared by drag and the clipboard:

- A custom type (`it.fizzyed.exchange.v1`) carrying a versioned envelope: document content, view state, layers, selections,
  with the producing plugin id and app identity.
- **Fallbacks other apps can read**, always written beside it: plain text, an image (PNG) and file references, so a paste
  into Finder, a browser or a text editor works.
- A fizzy that understands the envelope (and has the plugin) restores it fully; one that does not takes the fallbacks.
- **Base: Graphl's existing cross-app serialization** (it already copy/pastes across Blender, Unreal, web and desktop) if it
  is suitable (typed payload, versioned, fallbacks). Otherwise fizzy defines the format and Graphl gets it through the
  shared framework. Step 0 obtains and assesses Graphl's format before step 21 commits to either.
- Between views of the same shared fizzy nothing is serialized: that stays a live, in-process drag (step 11).

## Signing and verifying components

Decided 2026-10-19. A downloaded component runs in someone else's process, so it must be signed:

- **Who signs**: the app's publisher (fizzyedit for fizzy; Graphl for its own component), in release CI: macOS Developer ID
  signature plus notarization, Windows Authenticode, and on Linux a **detached minisign signature** published with the component and verified against a public key built into
  the add-on/shell, since the OS has no gate. The hash goes in the release feed beside the artifact.
- **How the shell verifies** before loading: the hash from the feed; then the OS check (`codesign --verify` / `spctl`,
  `WinVerifyTrust`) *and* the signer matching the identity the shell was built for (team id / certificate subject / key
  recorded in the shell). A mismatch is never loaded; the panel says why.
- The shell is distributed by the same publisher (for Blender through its extension platform); it carries the trusted
  signer, not the component. See also `TOOLS_AND_TRUST_PLAN.md` section 5.

## Repos and names

- **fizzyedit/fizzy**: `sdk/remote/` (protocol and client library, std-only, shipped in the SDK tarball), `app/remote/`
  (server side), `ViewHost`, the recording backend, the background-exit rules, the `external_apps` service. The client C
  ABI (`fizzy_remote.h`) is built by release CI and attached to each `sdk-v*` release.
- **fizzyedit/blender** (decided 2026-10-19): **one repo holds everything Blender needs**: the add-on (`fizzy_view`), its
  native client library build, the integration plugin that installs it, and CI. It replaces the proposed `blender`
  and `blender-integration`; `blender-dvui` stays Graphl's.
- **fizzyedit/unreal** (proposed, same shape: the Unreal plugin, its integration plugin, CI). Replaces the proposed `unreal-fizzy`.

## What still applies from the first plan

- **Keys only while focused**: kept, on the remote `ViewHost` (protocol section).
- **The name `fizzy.Embedded`**: reserved for the in-process fallback below, so a later reader finds it; the
  out-of-process feature is a *remote view* in code (`app/remote/`, `RemoteViewHost`, `sdk/remote/`).
- **`{ctx, vtable}` seams** (`DialogDirs`, `auto_update.Hooks`): the style `ViewHost` takes.
- **Open questions** about finding the installed fizzy, which fizzy, and dirty documents in a drag: restated below, with
  simpler answers (one process owns the document; no temp copy).
- **No longer applies**: splitting `Entry` into `Instance`, `chrome = .none`, `addEmbedded`, `Options.config / dialogs /
  on_quit`, `Embedded.panic / logFn`, link-only plugin builds for hosts, the proxy render bridge for dylibs in the host,
  the optimize-mode gate and fingerprint move of the old step 10, "one editor per process", the single-instance
  exemption. Fizzy keeps its own dialogs, config, crash handling and plugins.

## Installing adapters: integration plugins

Decided 2026-10-19 (maintainer). A host adapter is not a second install the person does by hand: fizzy installs it. Each
host gets an **integration plugin** in fizzy's plugin store (`blender`, later `unreal`), a normal store plugin with no
document or surface of its own. Enabling it in fizzy finds the host's installs and puts the adapter in them; disabling
removes it. The integration plugin and the adapter live in **one repo** per host (`fizzyedit/blender`) and ship together:
the plugin embeds the adapter build it was released with. Only the install step is new: the protocol, the client and the
adapters are as above.

- **Blender: finding installs.** macOS `/Applications/Blender*.app`, `~/Applications`, and the per-version config dirs
  `~/Library/Application Support/Blender/<ver>/`; Windows `%APPDATA%\Blender Foundation\Blender\<ver>` (and the
  executables under `%ProgramFiles%\Blender Foundation`); Linux `~/.config/blender/<ver>`, plus Snap and Flatpak (below).
  One entry per version, with the executable if found (config dir alone means "Blender has run, no binary seen": offer,
  but cannot enable headless). The person chooses which versions; the default is all found.
- **Blender: installing. Blender >= 4.2 only.** The add-on is a Blender *extension*, installed with Blender's own command:
  `blender --command extension install-file --repo user_default --enable <zip>` (headless: `--background`, never the
  window, never `--factory-startup`, which would save factory prefs over theirs). People who install from inside Blender
  use Blender's extension platform with the same zip (a `blender_manifest.toml`); same protocol, same fizzy, and it tells
  them fizzy must be installed (the client's "not installed" error). Versions below 4.2 are not supported. A timeout and
  a kill on the child; its output goes to fizzy's log. **Manifest ownership:** an add-on already installed from the
  extension platform (not by this plugin, so not in the manifest) is left alone, never replaced or removed; the plugin
  says "already installed from Blender" and does not update it.
- **A running Blender: copy now, enable on its next start.** Detect it (process list; Blender's per-version lock/session
  files). Do not run a second headless Blender against a live session's prefs (they would be overwritten when it quits).
  Instead put the extension files in place and record "pending enable"; the add-on's registration, when Blender next
  starts, enables itself once (and the plugin's next scan, finding that Blender closed, runs the headless enable if the
  add-on has not). The UI says "installed; takes effect the next time Blender starts".
- **Builds.** Blender's add-on is Python plus a C-ABI library (`fizzy_remote`) loaded with ctypes, so **one build per
  OS/arch serves every Blender version**; only Python or `gpu`-module API drift may need per-version code paths in the
  Python, never per-version binaries. **Unreal is the opposite:** a C++ plugin is binary-tied to an engine version, so CI
  builds it **prebuilt per engine version x platform for the last ~3 engine versions**, and the integration installs the
  matching one. Building on the person's machine (their UnrealBuildTool) is a **fallback only**, for an engine version
  outside the prebuilt set, with the person's consent and a clear "this takes a while" progress.
- **Linux: Snap and Flatpak.** Blender's Snap uses classic confinement, so it works like a native install (same paths
  logic, socket reachable, fizzy startable). Flatpak Blender (`org.blender.Blender`) cannot see fizzy's socket or start
  fizzy. With consent the integration runs `flatpak override --user --filesystem=<fizzy socket dir> org.blender.Blender`
  (or the **narrowest grant that works**, to be verified: step 16), recorded in the manifest and reverted on removal. And
  **on Linux fizzy registers a systemd user socket unit**, so that connecting to the socket starts fizzy (`--quiet`):
  that is also the Linux auto-start path for *any* host, which needs no `exec` from the host's sandbox. If neither is
  possible (no systemd, an override refused), the add-on shows a clear message saying why and what to do. It never starts
  a private second fizzy.
- **Updates.** The store updates the integration plugin; its update hook re-runs the install for every recorded Blender
  (a running one: copy now, enable on its next start). Protocol skew is reported by whichever side is old: the add-on says
  "update fizzy", fizzy says "update the integration" (a remote `hello` with an adapter version lower than the protocol
  floor puts a badge on the integration in the store and in Settings).
- **Removal.** The integration keeps a manifest of everything it put anywhere (files, the extension it registered, the
  prefs it changed, a Flatpak override) in fizzy's config (`<config>/integrations/<plugin-id>.zon`: host app, version,
  path, hash, time). Disabling or uninstalling the plugin removes exactly what the manifest lists, disables the add-on
  headless (or marks it for removal at Blender's next start if running), and drops the manifest. A file whose hash changed
  since install (the person edited it) is left and reported.
- **Unreal.** Install into the engine's `Plugins/` per engine version found through the Epic launcher's
  `LauncherInstalled.dat` (macOS `/Users/Shared/Epic Games/`, Windows `%ProgramData%\Epic\`), or a per-project copy (the
  person picks a `.uproject`; the plugin goes into its `Plugins/`), using the prebuilt binary for that engine version
  (Builds, above).
- **The permission.** Writing into another application's folders and running it headless is a new kind of permission, a
  store plugin declares it (the app *and* the folder patterns) and fizzy asks consent, per host. The contract is in
  `TOOLS_AND_TRUST_PLAN.md`, "Writes into other applications". The SDK service the integration plugin calls is step 12.

## Integrations are build targets

Decided 2026-10-19. An integration is built the way the web build is: **a build target that produces per-host
artifacts**, which CI builds and tests as gates and the store publishes.

- `zig build blender` produces the add-on zip per OS/arch (the extension, with the client library for that target);
  `zig build unreal -Dunreal-version=5.x` produces the plugin for that engine version and platform. They are steps of
  `fizzyedit/blender` / `fizzyedit/unreal` builds, and fizzy's own tree exposes the client library they link.
- **Gates.** `zig build check-blender` runs the add-on's headless check against a real headless Blender (Linux CI: 4.2 and
  the current 5.x); Unreal's gate builds each of the ~3 supported engine versions and runs the automation test where an
  engine is available. Like `check-web`, a PR that breaks an integration fails CI.
- **Publish.** The release workflow attaches the artifacts to the repo release, `plugin-build-action` packages the
  integration plugin with the adapter it ships, and the store lists it; the extension platform gets the same zip.

## Apps built on fizzy get integrations too

Decided 2026-10-19. Graphl and example-app are fizzy-based applications, not fizzy; an app built on fizzy does **nothing
extra** for integrations. They are a **build option of the framework**:

```zig
fizzy.buildApp(dep, plugins, .{ .app_name = "Graphl", .app_bundle_id = "...", .integrations = .{ .blender, .unreal } })
```

- **App-branded adapters, built from generic code.** The build produces the Blender add-on zip and the Unreal plugin (per
  engine version) from the generic `fizzyedit/blender` / `fizzyedit/unreal` code, with the app's name, icon and id filled in
  at build time, and the signed standalone component.
- **A reusable app CI workflow** (like `plugin-build-action`, for apps) builds every target, signs and publishes to the
  app's release feed.
- **Runtime pieces live in `app/`** so every app gets them: remote views, quiet start, quiet-to-normal, single instance,
  the exchange format, and the installer with consent and an "Integrations" settings page. Discovery takes the app identity
  (bundle id, install path hints, app name, socket location), never "Fizzy"; `hello` reports it so skew is per app.
- **What an app still supplies:** signing identities as CI secrets, and a release location.
- An add-on installed from elsewhere (Blender's extension platform) is left alone (manifest ownership, above).
- **Later:** one add-on listing every fizzy-based app installed (a chooser in the add-on's panel).

## Quiet to normal

Decided 2026-10-19. A quiet instance (started by a host, no window, no Dock icon) and a normal fizzy are one process in
different states:

- **A normal launch while a quiet instance serves remote views forwards to it** (`app/single_instance`) and the quiet
  instance **becomes normal**: it leaves quiet (macOS: `.regular` activation policy, the Dock icon appears), opens a main
  window with the person's layout, and takes focus. Remote views continue uninterrupted: same process, same documents, no
  reconnect.
- **Closing the last main window while remote views exist returns to background** (the Dock icon goes); with no views the
  normal quit rules apply.
- **Quit (Cmd+Q) while hosts hold views:** ask ("Blender is using fizzy: close its views?") or hide; never silently kill a
  host's views. Which is the default is open (below).
- **A newer build launched over an older quiet instance takes over through the restart handover** (fizzyedit/fizzy#644);
  adapters reconnect with backoff (step 5), so a host sees a short gap, not an error.

## Steps

Each step is one PR, <= 800 lines of diff outside tests and docs, with the gates of `CONTRIBUTING.md`. **Cloud-able**
means a Linux session can build and verify it (unit and integration tests, no screen); **macOS-needed** means someone
must see or run it on a Mac. Steps 1-3 are (a), 4-5 are (b), 6-7 are (c), 8 is (d), 9 is (e), 10 is (f), 11 is (g), 12-14 are (h), installing adapters, 15-18 are (i), the 2026-10-19 decisions.

**Step 0 (before 21): obtain and assess Graphl's cross-app serialization.** Get the format from Graphl's author and judge
it against the exchange format's needs (typed payload, versioned, fallbacks). Adopt it as the base if suitable; otherwise
fizzy defines one and Graphl gets it through the shared framework. Not a PR; its finding goes into step 21's kickoff.

**Can start now:** 0, 1, and 2 (std-only, builds on nothing). 3 needs 1 and 2. 4 needs fizzy#655 merged. 5 needs only 2 and
can run beside 3.

1. **`app`: a `ViewHost` interface, fizzy's windows behind it.** New `app/view/ViewHost.zig`: the surface a frame
   draws into, size and scale, input in, focus, IME and cursor out, begin/end/present. Refactor the main window and
   `Popout`'s windows (`src/editor/Popout.zig`, `backend/src/viewports.zig`) to implement the local host; the editor,
   `Layout` and `Floats` talk to `ViewHost`, never to SDL or a viewport directly. No behaviour change, no remote code.
   *Verify:* gates; pop-out demos and integration tests unchanged; `core.profile` frame time before and after; on macOS
   float a view out, drag it, full screen, live resize by hand. Cloud-able for tests; **macOS-needed** for the hand check.
   *Kickoff.* Task: put fizzy's main and pop-out windows behind a `ViewHost` interface. Plan: EMBEDDED_EDITOR_PLAN step
   1. Builds on: nothing. Workspace: `~/dev/fizzy-viewhost`. Branch: `app/viewhost`. Notes: read
   `POPOUT_WINDOWS_PLAN.md` and `NATIVE_WINDOWS_PLAN.md`. Pure refactor; split main window / pop-out if over 800 lines.
   Two real users shape it; add no remote-only members. No per-widget cost in the frame.

2. **`sdk`: the remote-view protocol library (`sdk/remote/`).** Std-only: message types, length-prefixed
   encode/decode, `hello` and version negotiation, a connection state machine (views, seqs), a loopback pair for tests.
   No fizzy, dvui or sockets in it (the transport is a reader/writer pair). *Verify:* unit tests: round trip of every
   message, version-mismatch reasons, unknown optional/required types, truncated and oversize frames refused.
   **Cloud-able.**
   *Kickoff.* Task: write `sdk/remote/`, the std-only remote-view protocol. Plan: EMBEDDED_EDITOR_PLAN step 2. Builds
   on: nothing. Workspace: `~/dev/fizzy-remote-proto`. Branch: `sdk/remote-protocol`. Notes: like `sdk/tape/`; touch
   nothing in `sdk/src/**` (no fingerprint move). Wire its tests into `build/` and `check-tests-reached`. Append-only
   versions; document the encoding in `sdk/remote/README.md`.

3. **`app`: `GET /remote` and a `RemoteViewHost`.** The control server upgrades `GET /remote` and hands the stream to
   `app/remote/`; `RemoteViewHost` implements step 1's interface for a view (a window like a pop-out, never shown by the
   OS); input and focus feed dvui events, chords only while focused; a view closing closes its window. Tier 3 with a
   stub encoder (a solid fill) so the loop runs end to end. `fizzy.remote.views` lists open views. *Verify:* an
   integration test (Linux CI) drives the real socket with the loopback client: create a view, send
   motion/click/key/text, ask a frame, drop the connection mid-frame (fizzy keeps running, the window goes), a second
   client; one editor-facing test body runs against the local and the remote host. **Cloud-able.**
   *Kickoff.* Task: serve the remote-view protocol on the control socket; implement `RemoteViewHost`. Plan:
   EMBEDDED_EDITOR_PLAN step 3. Builds on: steps 1 and 2. Workspace: `~/dev/fizzy-remote-host`. Branch:
   `app/remote-view-host`. Notes: no thread per view; frames come from the UI thread on request. Socket stays `0600`,
   same user. Add the route to `docs/CONTROL.md`.

4. **`app`: a quiet fizzy exits when its last view closes.** A fizzy started `--quiet` (fizzy#645, acting on it #655)
   counts remote views and open windows; at zero for a grace period (30 s, `FIZZY_QUIET_GRACE_S`) it exits cleanly. It
   never exits with a dirty document: it shows a window instead. Becoming normal, and returning to background, are
   step 15's state machine; this step is the quiet-state exit only. On macOS no Dock icon while quiet (activation policy
   `.accessory`). *Verify:* unit tests of the state machine, and an
   integration test with the loopback client: last view closes, exit after the grace; a reconnect inside it cancels;
   a second `fizzy` launch forwards and keeps it alive; by hand on macOS the Dock icon. Cloud-able for the logic;
   **macOS-needed** for the Dock.
   *Kickoff.* Task: background-exit rules for a quiet fizzy serving remote views. Plan: EMBEDDED_EDITOR_PLAN step 4.
   Builds on: step 3 and fizzy#655. Workspace: `~/dev/fizzy-quiet-exit`. Branch: `app/quiet-exit`. Notes: never exit with
   a dirty document; what ends quiet is step 15. Test with `--profile`.

5. **`sdk`: the client library finds, starts and connects to fizzy.** `sdk/remote/client.zig` and the C ABI
   (`fizzy_remote.h`). Find the installed fizzy per OS (macOS: `/Applications/Fizzy.app`, `~/Applications`, then Launch
   Services by bundle id; Windows: the Velopack install under `%LOCALAPPDATA%` and the App Paths key; Linux: `$PATH`,
   the AppImage and `~/.local` places; `FIZZY_EXE` and `FIZZY_PROFILE` override), get the socket from `fizzy
   --control-path`, connect; if nothing listens, launch with `--quiet` (macOS: `open -g -j`, never taking focus) and
   retry with backoff to a deadline. Distinct errors for not installed / too old (protocol) / timed out. *Verify:* tests
   with a fake fizzy executable that listens on a test socket (found, not found, slow start, crash during start; Linux
   CI); real-install discovery by hand on a Mac and a Windows VM. Cloud-able for the library; **macOS/Windows-needed**
   for real installs.
   *Kickoff.* Task: client library that finds, quietly starts and connects to the installed fizzy. Plan:
   EMBEDDED_EDITOR_PLAN step 5. Builds on: step 2. Workspace: `~/dev/fizzy-remote-client`. Branch: `sdk/remote-client`.
   Notes: take the app identity (bundle id, name, install paths) as input so apps built on fizzy reuse it (step 18); on
   Linux try the systemd socket (step 16) before launching. A sandboxed host that can reach neither (Flatpak without the
   override, sandboxed macOS app): detect it and return a distinct error the add-on shows; never start a private second
   fizzy. Reconnect with backoff after a drop. Never take focus.

6. **`app`: tier 3, the draw-list encoder in fizzy.** Move the spike's recording backend into fizzy
   (`app/remote/RecordBackend.zig`: vertices, indices, commands, texture updates; no GPU) and have `RemoteViewHost`
   render a view through it and send `frame` with the list. Textures are created and destroyed by message and cached per
   view, so steady state sends only changes; fonts rasterise in fizzy. Glass and blur draw as opaque fills in a tier-3
   view. *Verify:* integration test: a loopback client decodes a frame of the real editor, replays it onto dvui's
   testing backend and compares with the in-process render within tolerance; record bytes per frame for a typical and an
   idle frame in the PR (idle must be about zero) and keep a benchmark. **Cloud-able.**
   *Kickoff.* Task: tier 3 draw lists from a remote view. Plan: EMBEDDED_EDITOR_PLAN step 6. Builds on: step 3.
   Workspace: `~/dev/fizzy-remote-drawlist`. Branch: `app/remote-drawlist`. Notes: source is
   `~/dev/fizzyedit/blender-dvui-spike/backend/src/backend.zig` (already on fizzy's dvui and Zig 0.17). Idle cost must be
   zero. Fingerprint untouched.

7. **`blender`: the Blender add-on on the remote view.** New repo `fizzyedit/blender` from the spike (it will also hold the
   integration plugin, step 13). The add-on (a Blender >= 4.2 extension, `blender_manifest.toml`) starts fizzy through the
   client library loaded with ctypes, makes a view per area, forwards events and focus, draws lists with `gpu`, draws
   pointer and caret itself, and **fixes Retina**: the real `scale` goes in `resize` and the draw code honours it (no more
   1x). One native library build per OS/arch serves every Blender version. CI: the spike's headless check against a fizzy
   built at a pinned commit, Blender 4.2 and 5.x on Linux; a zip per release. *Verify:* headless check in CI; by hand on a
   Retina Mac for scale, IME, focus hand-off, quitting Blender (background fizzy exits after the grace) and killing fizzy
   (the add-on offers a restart; Blender survives). Cloud-able for CI; **macOS-needed** for the rest.
   *Kickoff.* Task: Blender add-on on the remote view; fix Retina scale. Plan: EMBEDDED_EDITOR_PLAN step 7. Builds on:
   steps 5 and 6. Workspace: `~/dev/blender`. Branch: `init` (new repo `fizzyedit/blender`). Notes: Blender >= 4.2 only; one
   repo for add-on, client build, integration plugin (step 13) and CI; start from `~/dev/fizzyedit/blender-dvui-spike`, no
   zig-pkg/zig-out/.zig-cache, add a `.gitignore` with them and `.DS_Store`. No dvui or Zig at runtime. Keep per-Blender
   `gpu` API differences in Python only.

8. **`app`: tier 1 on macOS.** Fizzy renders a remote view into a `CAContext` layer and sends its context id (new
   `native_surface` messages: create, rect/clip, visibility, z-order); the host makes a `CALayerHost`. Input still
   arrives through the host as events. Falls back to tier 3 if the private classes are missing. *Verify:* a small AppKit
   test host (in `tests/` or example-app) shows the view through resize, scroll, moving between windows, Retina, dark and
   light; Blender by hand. **macOS-needed.**
   *Kickoff.* Task: tier 1 native surface for macOS remote views. Plan: EMBEDDED_EDITOR_PLAN step 8. Builds on: steps 3
   and 7. Workspace: `~/dev/fizzy-remote-calayer`. Branch: `macos/remote-calayerhost`. Notes: the classes are private:
   look them up at runtime and degrade. Read `docs/MACOS_LIVE_RESIZE.md` first. Windows `SetParent` and X11 reparent are
   follow-ups on the same messages.

9. **`app`: tier 2, shared GPU texture with a fence.** IOSurface + `MTLSharedEvent` first (macOS); DXGI shared handle and
   dmabuf as their own PRs (`windows/`, `linux/`). `hello` carries the formats the host can import; fizzy keeps a ring of
   3 and `frame` names slot and fence value. *Verify:* a test host drawing the texture with a frame counter in the view
   (no tearing, no wrong-slot read under a stalled host). **macOS-needed**; the Windows and Linux parts need their
   platforms.
   *Kickoff.* Task: tier 2 shared-texture frames with a fence, macOS first. Plan: EMBEDDED_EDITOR_PLAN step 9. Builds on:
   step 3. Workspace: `~/dev/fizzy-remote-iosurface`. Branch: `macos/remote-iosurface`. Notes: `Popout` already renders to
   targets: render a remote view to an IOSurface-backed one. A stalled host must never block fizzy's UI thread.

10. **`unreal`: the Unreal adapter.** New repo `fizzyedit/unreal`: an Unreal plugin (C++) linking `fizzy_remote`, a Slate
    widget per view, tier 3 first, tier 2 on the RHI later. Input and IME through Slate. The plugin is binary-tied to an
    engine version, so CI builds it prebuilt per engine version x platform for the last ~3 engine versions (step 17).
    *Verify:* editor automation test on Windows and Linux; by hand in the editor. Needs Unreal: **not cloud-able**.
    *Kickoff.* Task: Unreal plugin hosting a fizzy remote view. Plan: EMBEDDED_EDITOR_PLAN step 10. Builds on: steps 5 and
    6. Workspace: `~/dev/unreal`. Branch: `init` (new repo `fizzyedit/unreal`). Notes: coordinate with Graphl's maintainer
    on `unreal-dvui`'s Slate input layer; keep the engine version a build parameter (`-Dunreal-version`), not a fork.

11. **`app`: cross-app drag of files, tabs and views.** Between fizzy windows and a remote view it is in-process for
    fizzy (`UNIFIED_DRAG_PLAN.md`'s `ViewDrag` payloads; a remote view is one more drop target). With the host: `drag_begin`
    (payload description and preview image) from fizzy, `drag_enter / move / drop` from the host; files travel as paths,
    documents as ids (one model: no copy, no temp file for a dirty document). *Verify:* integration test dragging a tab
    into and out of a remote view with the loopback client; by hand across Blender and a fizzy window. Cloud-able for
    the logic; **macOS-needed** for the gesture.
    *Kickoff.* Task: cross-app drag through the remote view. Plan: EMBEDDED_EDITOR_PLAN step 11. Builds on: step 3 and the
    unified drag work. Workspace: `~/dev/fizzy-remote-drag`. Branch: `app/remote-drag`. Notes: read `UNIFIED_DRAG_PLAN.md`
    first.

12. **`sdk`: the `external_apps` host service (consent, install, manifest).** A service on `Host` for a plugin that puts
    files into, or runs, another application. `find(app_id)` lists installs (finders are per host, supplied by the
    integration, the service owns none); `install(.{ app, version, files, dest })` and `run(.{ app, version, args,
    timeout })` refuse unless the plugin declared that `app_id` in its identity (`plugin.zig.zon`) and the person granted
    it; the first call per (plugin, app) shows the consent dialog (what, where, "runs it headless"); every file written is
    recorded in the manifest `<config>/integrations/<id>.zon` with hash; `uninstallAll(plugin)` removes by manifest;
    a grant lives in `settings.zon` (`.plugins.<id>.external`); the declaration includes folder patterns as well as the app
    (consent lists them before any scan). Also the identity field and the store/registry plumbing
    (`TOOLS_AND_TRUST_PLAN.md`). **Moves the fingerprint** (a new service type and a zon field): record it, label `sdk`,
    leave `sdk_version` to the release PR. *Verify:* unit and integration tests on Linux with a fake app folder and a fake
    executable (declared vs undeclared refused; consent once per app; manifest written, removal exact, edited file kept;
    grant survives restart); `test-sdk-version`; dialog by hand. Cloud-able for the logic; **macOS-needed** for the dialog look.
    *Kickoff.* Task: `external_apps` service: consent, install into and run another app, manifest, removal. Plan:
    EMBEDDED_EDITOR_PLAN step 12 and TOOLS_AND_TRUST_PLAN "Writes into other applications". Builds on: nothing (trust
    dialog, TOOLS_AND_TRUST step 2, is a soft dependency: share its dialog). Workspace: `~/dev/fizzy-external-apps`.
    Branch: `sdk/external-apps`. Notes: the declaration is for consent, not a sandbox; say so in the dialog. Add the zon
    field to `docs/PLUGINS.md` s2.2 and s6. Sketch the finder interface with step 13's needs; add no Blender-specific code. Install/run take the app identity from
    the declaration (apps built on fizzy, step 18); `install` supports a recorded Flatpak override for step 16.

13. **`blender`: the Blender integration plugin (same repo as step 7).** A store plugin in `fizzyedit/blender`
    (`plugin.zig.zon` declares `.external = .{ .{ .app = "blender", ... } }` with folder patterns): finders for the paths
    above, version picker UI (a Settings pane), install through `blender --command extension install-file` (>= 4.2 only),
    headless enable with `save_userpref`, a running Blender gets copy-now/enable-on-next-start, manifest ownership (an
    add-on from the extension platform is left alone), update hook re-installing into every recorded version, removal
    through the manifest; embeds the add-on built by the same repo. *Verify:* Linux CI with a fake Blender (a script
    standing in for the executable) and fixture config dirs for every path pattern, plus a real headless Blender 4.2 and
    5.x job on Linux (install-file, enable, `save_userpref`, add-on listed enabled, disable removes; a running-Blender
    case with a stub lock file); by hand on macOS with a real Blender (never opens a window, a running one is copied and
    enables on next start, uninstall leaves the config as found). Cloud-able for CI; **macOS-needed** for the hand check;
    Windows and Linux paths by fixture, then a VM.
    *Kickoff.* Task: Blender integration plugin in fizzyedit/blender: find installs, install via Blender's extension
    command, enable, update, remove. Plan: EMBEDDED_EDITOR_PLAN step 13. Builds on: steps 7 and 12. Workspace:
    `~/dev/blender` (plugin dir `integration/`). Branch: `blender/integration-plugin`. Notes: use `FIZZY_PROFILE` for any
    plugin build; run `blender` only with `--background` or `--command`, never `--factory-startup`; a running Blender is
    never touched except copying files; Flatpak/Snap are step 16; leave extension-platform installs alone.

14. **Unreal integration plugin (later).** Same shape as step 13 for `fizzyedit/unreal` (same repo): engines from
    `LauncherInstalled.dat` (and a manual engine path), install into the engine's `Plugins/` or a per-project copy, using
    the prebuilt artifact for that engine version from step 17; building with the person's UnrealBuildTool only as a
    consented fallback. *Verify:* fixture `LauncherInstalled.dat` tests on Linux; real engine by hand on Windows/macOS.
    **Not cloud-able** past the fixtures; do not start before step 10.
    *Kickoff.* Task: Unreal integration plugin (engine and per-project install, prebuilt per engine version). Plan:
    EMBEDDED_EDITOR_PLAN step 14. Builds on: steps 10, 12 and 17. Workspace: `~/dev/unreal`. Branch:
    `unreal/integration-plugin`. Notes: read step 13's PR first; the build-from-source fallback needs explicit consent.

15. **`app`: quiet becomes normal, and back.** The state machine of "Quiet to normal": a normal launch forwarded to a
    quiet instance serving views makes it normal (Regular activation policy, Dock icon, main window with the person's
    layout, focus) with views uninterrupted; closing the last main window while views exist returns to background; Cmd+Q
    while hosts hold views asks or hides, never silently closes views; a newer build over an older quiet one takes over
    through the restart handover (fizzy#644) with adapters reconnecting. *Verify:* unit tests of the transitions;
    integration test with the loopback client (views survive quiet -> normal -> background, a document edited in a view
    shows in the window); by hand on macOS: Dock icon, focus, Cmd+Q dialog, handover with Blender running. Cloud-able for
    the logic; **macOS-needed** for the rest.
    *Kickoff.* Task: quiet <-> normal transitions with remote views alive. Plan: EMBEDDED_EDITOR_PLAN step 15. Builds on:
    steps 3 and 4 and fizzy#644. Workspace: `~/dev/fizzy-quiet-normal`. Branch: `app/quiet-to-normal`. Notes: same
    process and documents throughout; Cmd+Q never kills a host's views silently; read `macos-window-transition` tests
    first; test with `--profile`.

16. **`linux`: systemd socket activation and the Flatpak grant.** Fizzy registers a systemd user socket unit
    (`fizzy.socket` + `fizzy.service` starting `fizzy --quiet`) so connecting to the control socket starts fizzy, and
    removes it with the install; this is the Linux auto-start path for any host. For Flatpak Blender the integration
    (step 13) runs `flatpak override --user --filesystem=<socket dir> org.blender.Blender` with consent, or the narrowest
    grant that works (measured here: the socket folder alone, or a socket-activation path only); the add-on shows a clear
    message when neither is possible. Snap (classic) needs nothing. *Verify:* Linux CI for the unit file and activation
    (a client connects, fizzy starts quiet, serves); by hand with real Flatpak and Snap Blender on Linux (record which
    grant is minimal in the PR). Cloud-able for activation; **Linux-hardware-needed** for Flatpak/Snap.
    *Kickoff.* Task: systemd user socket activation for fizzy and the Flatpak override for Blender. Plan:
    EMBEDDED_EDITOR_PLAN step 16. Builds on: steps 4 and 5. Workspace: `~/dev/fizzy-socket-activation`. Branch:
    `linux/socket-activation`. Notes: socket stays `0600`, same user; activation must not pop a window or take focus;
    no systemd -> clear message, never a private second fizzy; verify the narrowest working `--filesystem` grant before
    documenting it.

17. **`build`: integrations as build targets.** `zig build blender` (the add-on zip per OS/arch), `zig build unreal
    -Dunreal-version=...`, `zig build check-blender` (headless Blender on Linux, 4.2 and 5.x) wired into CI as gates, and
    release publishing of the artifacts for the store (prebuilt Unreal per engine version x platform for the last ~3
    versions). Lives in the host repos on top of fizzy's exported client library; example-app proves an app can use it.
    *Verify:* the CI jobs run and fail on a broken add-on; `check-blender` green on Linux; artifact names and hashes in
    the release. Cloud-able (Blender); Unreal engine builds **not cloud-able**.
    *Kickoff.* Task: build targets and CI gates for host integrations. Plan: EMBEDDED_EDITOR_PLAN step 17. Builds on:
    steps 7 and 13 (Blender); step 10 (Unreal). Workspace: `~/dev/blender` / `~/dev/unreal`. Branch: `ci/integration-targets`.
    Notes: model on `check-web`; Blender add-on is one build per OS/arch, Unreal one per engine version x platform; building
    Unreal on the person's machine is a fallback only.

18. **`app`: integrations as a `buildApp` option.** Add `.integrations` (and `.app_name`, `.app_bundle_id`, icon) to the
    framework's build options: it builds app-branded Blender add-on and Unreal plugins (per engine version) from the generic
    `fizzyedit/blender` / `fizzyedit/unreal` code with the identity filled in, and the signed standalone component; add the
    reusable app CI workflow (build every target, sign, publish to the app's feed). Runtime pieces (remote views, quiet
    start, quiet-to-normal, single instance, exchange format, installer + consent + "Integrations" settings page) live in
    `app/`; discovery and `hello` carry the app identity. *Verify:* example-app builds with
    `.integrations = .{ .blender, .unreal }` and yields branded artifacts; an integration test runs a fake app with a
    different identity through find/start/connect; `scripts/check-examples.sh`. Cloud-able.
    *Kickoff.* Task: integrations as a build option of the framework, so Graphl/example-app need nothing extra. Plan:
    EMBEDDED_EDITOR_PLAN step 18. Builds on: steps 5, 12, 15 and 17. Workspace: `~/dev/fizzy-app-integrations`. Branch:
    `app/integrations-build-option`. Notes: no fizzy-only code in `app/`/`sdk/`; an app supplies only signing identities
    (CI secrets) and a release location; land the matching example-app PR; an add-on from the extension platform is left
    alone.

19. **`sdk`: `fizzy_remote.h`, the C API over two transports.** Define the C API from `sdk/remote/` and implement the socket
    transport (step 5's client) and an in-process transport behind it, selected at `fizzy_remote_open`. A loopback in-process
    host (the tests' remote `ViewHost`) proves both. *Verify:* one adapter test suite run against both transports with the
    same results; header checked in and diffed in CI. Cloud-able.
    *Kickoff.* Task: C API `fizzy_remote.h` with in-process and socket transports. Plan: EMBEDDED_EDITOR_PLAN step 19. Builds
    on: steps 2, 3 and 5. Workspace: `~/dev/fizzy-remote-capi`. Branch: `sdk/remote-capi`. Notes: std-only in `sdk/remote/`,
    no `sdk/src/**` change (fingerprint untouched); `hello` reports `mode`; ABI versioned like the protocol; ship the header
    in the SDK tarball and release CI.

20. **`app`: the editor component and runtime selection (standalone mode).** Build `fizzy_editor` as a dylib exporting the
    C API with compiled-in plugins and no chrome (`zig build component`, per OS/arch), and the shell-side loader: socket if
    fizzy is installed/running, else download, verify (step 22), load, run in-process. The panel states "standalone - not
    connected to fizzy" and that documents are copies. *Verify:* integration test runs the component in-process through the
    loopback host; by hand in Blender on a Mac with fizzy uninstalled. Cloud-able for the component; **macOS-needed** for the
    host.
    *Kickoff.* Task: updateable editor component for standalone mode. Plan: EMBEDDED_EDITOR_PLAN step 20. Builds on: step 19.
    Workspace: `~/dev/fizzy-component`. Branch: `app/editor-component`. Notes: not bundled in the add-on; downloaded into the
    per-user data folder via the release feed with a hash pin; no fizzy-only code in `app/`, take the app identity (step 18);
    nothing loads unsigned (step 22); update only while no view is open.

21. **`app`: the exchange format on the pasteboard.** `app/exchange/`: the envelope (`it.fizzyed.exchange.v1`), writers and
    readers per OS pasteboard, the plain-text/PNG/file fallbacks, shared by clipboard and drag; plugins supply content
    through a service. Base it on Graphl's serialization if step 0 finds it suitable. *Verify:* round-trip tests per envelope; paste
    into a plain text editor and Finder by hand; copy between fizzy, a separate profile and a standalone shell. Cloud-able
    for the codec; **macOS/Windows-needed** for pasteboards.
    *Kickoff.* Task: fizzy exchange format on the OS pasteboard for drag and copy/paste. Plan: EMBEDDED_EDITOR_PLAN step 21.
    Builds on: step 11 for drags. Workspace: `~/dev/fizzy-exchange`. Branch: `app/exchange-format`. Notes: step 0's assessment
    of Graphl's cross-app serialization decides base or own format; always write the fallbacks beside the custom type; version the
    envelope; copies only, no live state across processes.

22. **`ci`: sign and verify components.** Release CI signs the editor component (macOS Developer ID + notarization, Windows
    Authenticode, Linux detached minisign signature) and publishes hash and signature in the feed; the shell verifies hash, OS check
    and signer identity before load ("Signing and verifying components"). *Verify:* a tampered, an unsigned and a
    wrong-signer component each refuse to load with a clear message; a notarized load works on a clean Mac. **Needs the
    maintainer's certificates and macOS/Windows**; the verification logic is cloud-able.
    *Kickoff.* Task: signing and load-time verification for downloaded components. Plan: EMBEDDED_EDITOR_PLAN step 22 and
    TOOLS_AND_TRUST_PLAN section 5. Builds on: step 20. Workspace: `~/dev/fizzy-component-signing`. Branch:
    `ci/component-signing`. Notes: never load on a failed check; the signer identity is built into the shell; apps built on
    fizzy sign with their own identity (CI secrets) through the app CI workflow; Linux uses a minisign detached signature checked against a key built into the shell; never ask for secrets in chat.

## Superseded: fizzy in the host's process via `Instance`

Replaced on 2026-10-19 by standalone mode (a signed editor component behind the C API). Kept so a later reader finds why.

If a host cannot reach a socket or spawn a process, or must have fizzy inside its own frame loop, the first design
applies: `Entry` split into the process part and `src/Instance.zig` (allocator, `*dvui.Window`, options with
`{ctx, vtable}` seams for config folder, dialogs and quit; `chrome = .none`; link-only plugins; one editor per process;
no single-instance lock), `fizzy.addEmbedded(dep, .{ .backend, .plugins })` over `APP_BUILD_API_PLAN.md` steps 2 and 6
(the host's backend under fizzy's one dvui pin), and a public `fizzy.Embedded.init / frame / deinit` with optional
`Embedded.panic / logFn`. It is second because it costs a second editor with its own documents and history, plugins
limited to compiled-in ones (or dylibs with a proxy render bridge and matching optimize modes), and a host crash from a
fizzy bug. `ViewHost` (step 1) survives into it: an in-process host would be a third implementation. Revisit only when a
real host needs it.

## Open questions

- **Which fizzy.** A host connects to the default profile's fizzy (or the configured app identity's); an adapter setting
  for `FIZZY_PROFILE` is proposed; several fizzys at once is not supported.
- **Tier 1 and the Mac App Store.** CALayerHost is private API; a Store-distributed host stays on tier 2 or 3.
- **Quiet fizzy and updates.** A dirty document at grace-exit shows a window instead of exiting; the handover (#644) must
  not pop a window while quiet. Right default?
- **Grace period.** 30 s proposed; shorter restarts fizzy each time a person closes and reopens the area.
- **Cmd+Q with views.** Ask ("Blender is using fizzy: close its views?") or hide: which is the default, and is it a
  setting?
- **Look per view.** A remote view shares fizzy's settings. Should a host override theme or scale per view?
- **Permissions.** The socket is same-user with no token, and a remote view can run any command. Fine for local developer
  tools; revisit if an adapter ever runs untrusted code.
- **Flatpak's narrowest grant.** Is `--filesystem=<socket dir>` the minimum, or does `--socket`/`--talk-name` (a D-Bus
  activation) work without a filesystem hole? Step 16 measures it.
- **systemd user units.** Installed by the package, by the app on first run, or by the integration? Which unit when a
  distro has no systemd user session (message only)?
- **Extension platform vs our install.** Blender's platform may update the add-on independently; when both exist, is the
  "left alone, say so" rule enough, or should the integration offer to take over (remove theirs)?
- **Python/`gpu` drift.** How many Blender versions get a code path before dropping the oldest ( >= 4.2 is the floor)?
- **Unreal engine window.** "Last ~3 engine versions" as a rolling set: who bumps it, and what happens to a person on an
  older engine (consented source build only)?
- **One add-on for every fizzy-based app.** The chooser's shape and who owns the add-on when several apps are installed.
- **Apps may also ship as store plugins.** Apps built on fizzy may also ship their editor as a fizzy store plugin (e.g. Graphl inside fizzy) - the app's choice.
