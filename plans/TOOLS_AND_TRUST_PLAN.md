# Tools and trust: plugins that need other plugins and programs, and who a person trusts

Status: proposed, 2026-10-10; extended 2026-10-13 (tools that update themselves, keeping tools
current, Claude as Claude Code's native build). Steps tracked in fizzyedit/fizzy#336.

Three needs arrived at once. The zig plugin wants `zls` (and, later, `zig` itself). The chat wants
an agent program, some of them npm packages that need Node, and it works best beside the agent
plugin. The agent plugin ships `fizzy-mcp`, which the store cannot install. Today each plugin finds
its programs alone (the zig plugin searches PATH, then the login shell's), nothing installs them,
and the store installs and updates plugins without a word about whose code they are.

Decided with the user (2026-10-10):

- **Nothing is hidden.** Every program fizzy installs, and every plugin it runs, is listed with
  where it came from, and the person can change any choice.
- **Installing a plugin for the first time asks the person to trust it**, saying plainly that
  they are trusting someone else's code. Trust lets its updates install automatically.
- **The cleanest, most configurable shape**, one each for plugin-to-plugin relations, programs,
  and trust, rather than a solver.

## Three mechanisms, kept apart

Plugins needing plugins and plugins needing programs look alike and are not. Mixing them is how
editors grow package solvers, so they stay separate, and trust is a third thing both rest on.

### 1. Plugin to plugin: "works with", never "requires"

Plugins meet only at the SDK and must already work when what they use is absent (a service lookup
returning null is a normal path). A dependency stays what that makes it: a suggestion, with a
reason.

- **In the identity manifest**, beside `min_sdk_version`:
  `.works_with = .{ .{ .id = "agent", .why = "gives the agent fizzy's own tools" } }`.
  Install metadata, like the version floor: nothing about capability (rule: `plugin.zig.zon` is
  identity only).
- **In the store**, installing a plugin lists what it works with, each ticked by default with its
  reason, and installs them together. Each still asks for trust in its own right (below).
- **At runtime**, a plugin that finds a partner missing says so in its own UI and offers its store
  page (`fizzy.store.show` with the id).
- **No load order, no versions, no solving.** A plugin never fails to load for another's absence.
- **The first case is the agent and the chat** (2026-10-13). The chat stays its own plugin: most
  apps built on fizzy will not want it, and it is where plugins working with plugins gets proven
  (`plans/CHAT_PLAN.md`: the chat is a conversation surface any plugin can use through its
  service). The agent plugin works without it (an agent in a terminal drives fizzy over MCP), and
  `works_with` the chat "to talk to agents inside fizzy". A person installs the agent, the store
  offers the chat ticked, and from there it is setup, not more installing.

### 2. Tools: programs fizzy manages for plugins

One manager in the framework (`app/`) for every external program a plugin runs: `zls`, `zig`,
Node, an npm package, an agent's binary, `fizzy-mcp`. Plugins say what they need; they never write
installers.

- **A tool spec is data**: an id, a name and homepage, the tools it needs in turn (`node` for an
  npm tool), and its sources:
  - `system`: on PATH or the login shell's (`core.shell_env`), as the zig plugin does now;
  - `release`: a URL and sha256 per target, and where the program is inside the archive;
  - `npm`: a package and version, run with the `node` tool;
  - `plugin`: shipped in a plugin's own release, which is how `fizzy-mcp` arrives;
  - `self`: a program that installs and updates itself (below), Claude Code being the first.
- **Specs live in the registry** (`fizzyedit/plugins`, a `tools/` directory beside `registry/`,
  through the same ingest and catalog export, hashes pinned), so a new `zls` needs no zig-plugin
  release. A plugin may register a spec itself for a tool the registry does not list.
- **Plugins ask through an SDK service**: `tools.need("zls", ">=0.16")` returns a ticket; its
  status is `resolving`, `ready` (an absolute path and the environment to run it with),
  `missing`, `declined` or `failed` with why. Asynchronous, as the automation service is (tickets,
  not callbacks, which dangle across a plugin's reload). On the web: `unavailable`, since a page
  cannot run programs.
- **Every tool is configurable in one place**, Settings > Tools: which plugins use it, where it
  resolves now and its version, its source and hash, and a mode per tool:
  - `auto` (the default): the system's if present and the version fits, else managed;
  - `system`, `managed`, or `path` (a file the person names);
  - plus a pinned version, Update, Remove, and Reveal. Existing installs (Homebrew, zvm, nvm) win
    by default.
- **Nothing installs unasked.** A first need shows what, from where, how big, and its hash: "Zig
  wants zls 0.16.0 from github.com/zigtools/zls (12 MB, sha256 …)". Install / Use my own / Not
  now, and "always allow tools this plugin asks for".
- **Installed tools live in `<config>/tools/<id>/<version>/`**, each with a receipt (source,
  hash, when), versions side by side, which leaves room for a zig per project later.
- **Running one takes its environment from the spec**: Node's directory on PATH, an agent's API
  key from fizzy's Secrets. Needs `core.darwin_spawn` to take extra variables (the seam the agents
  plan named), and a spawn helper beside it so every plugin spawns tools one way.
- **Agents are a thin layer on tools**: an agent is a tool, its launch arguments, how to talk to
  it, and its sign-in. Claude: Claude Code's native build, a `self` tool, spoken to directly over
  its stream-json protocol as VS Code's extension does (no Node; `plans/CHAT_PLAN.md`), with the
  `claude-agent-acp` npm tool as the ACP route where Node is present. Codex: the `codex-acp` release
  binary. The chat's agent picker lists agents; choosing one is `tools.need`. ACP's
  `authenticate` becomes a Sign in button; an API key goes to Secrets.

#### Tools that install and update themselves

Some programs ship their own installer and updater, and copying them would be wrong. Claude Code's
native build keeps each version in `~/.local/share/claude/versions/<version>` and links the current
one from `~/.local/bin/claude`; `claude install [stable|latest|<version>]` and `claude update`
fetch and switch, and the running program updates itself (seen on this machine 2026-10-13, with a
Homebrew copy beside it). Re-implementing that would mean following a download layout that is not
a public contract, fighting its own updater, and keeping a third 230 MB copy whose sign-in and
settings are shared through `~/.claude` anyway.

So a `self` source is a short recipe of steps fizzy knows how to run, never a shell script:

- **detect**: a command and how to read its version (`claude --version`), and which file answered;
- **bootstrap**: how to get it the first time: a verified download (`release`, URL and sha256 per
  target) and a command to run with it (`claude install stable`), as the vendor's own install
  script does;
- **check**: whether an update is available (a command, or the vendor's channel file);
- **update**: the command that updates it (`claude update`);
- **env**: variables to run it with, including the vendor's switch for its own updater when fizzy
  is the one scheduling updates.

Fizzy runs these as jobs with the same consent, progress and listing as every other tool; the
program keeps its own files. One already installed (Homebrew, the vendor's installer) is detected
and used, and keeps updating however the person set it up: fizzy reads its version and does not
touch it.

#### Keeping tools current

- **Each need names a floor**, the lowest version the plugin works with (`tools.need("zls",
  ">=0.16")`), and a spec names its **channel**: a pinned version, or how to find the latest
  (a release feed, a vendor's channel file).
- **Checks run in the background**, at launch and daily, through the store's verified downloader.
- **Managed tools** install a new version beside the old one, verified, and switch atomically;
  the previous one is kept for **rollback**, older ones removed once no need names them.
- **Self-updating tools** run their own update command on fizzy's schedule; rollback is the
  tool's (`claude install <version>`), offered as "pin this version".
- **Never under a running process.** Tools are started through the spawn helper (step 1), so
  fizzy knows which are running; an update waits until none is (for an agent: between
  conversations, never during one).
- **A tool that moves past what a plugin speaks** shows up as a failed version check at launch
  and in the plugin's own UI, with the pin offered. The plugin's CI runs against a tool's new
  releases (as SDK releases open repin PRs today), so the floor and the plugin move before people
  hit it.
- Settings > Tools shows each tool's channel, current and previous versions, last check, and
  Update now / Roll back / Pin.

### 3. Trust: the first install asks

A plugin is native code running with the person's permissions: it can read and change their
files, use the network, and run programs. Fizzy does not sandbox it and does not review it. The
person should know that, once, in words, before it runs.

- **The first install of a plugin shows a trust dialog**, before anything downloads:
  - its name, author, and source (the repository and the manifest URL the registry points at);
  - what trusting means: "This plugin runs as native code with your permissions. It can read and
    change your files, use the network, and run programs. Fizzy does not review or sandbox
    plugins. You are trusting its author.";
  - what it works with, and the tools it is known to ask for;
  - "Update automatically from this source", ticked by default;
  - **Trust and install** / Cancel.
- **Trust is recorded per plugin, bound to its source**:
  `.plugins.<id>.trust = .{ .author, .source, .since }` in `settings.zon`, beside `.enabled` and
  `.auto_update`. A trusted plugin's updates from the same source install automatically (the
  store's existing auto-update, now gated by trust).
- **A change of source asks again.** An update whose registry entry names a different author or
  manifest URL is held, and the store shows "this plugin's source changed", with the old and the
  new, to trust again or not. That catches a hijacked or transferred plugin.
- **A plugin that arrives outside the store** (a dylib copied in, a local build: today's
  "undecided" plugin) asks the same on first load, saying fizzy cannot tell who built it. A plugin
  author approves their own build once per id. A trust record written into `settings.zon` by hand
  counts, so a sandbox, a provisioning script or CI can pre-trust.
- **Built-in plugins are fizzy**, trusted with it.
- **Revoking** (Settings > Plugins) stops automatic updates and asks again before the next one; it
  does not uninstall.
- **Tools keep their own consent.** Trusting a plugin does not trust the programs it asks for
  (`zls` is zigtools', not the zig plugin's); "always allow tools this plugin asks for" is the
  person's shortcut, recorded per plugin.
- **The web** asks the same: a page loads plugins from URLs too.

### 4. Writes into other applications (integration plugins)

Added 2026-10-19 for `EMBEDDED_EDITOR_PLAN.md` (installing the Blender add-on, the Unreal plugin). Trust (section 3) says a
plugin is native code with the person's permissions; an **integration plugin** does something the person should be asked
about by name: it writes files into another application's folders and runs that application headless. It is declared,
shown and consented to per host application. It is *not* a sandbox: the declaration drives the store's display and the
host service's refusal, and a malicious plugin can still do anything (trust is the defence, as in section 3).

- **Declaration.** Identity in `plugin.zig.zon`, read by the build action into `manifest.json` beside `name`,
  `description` and `tags` (PLUGIN_MANIFEST_PLAN R17's route): `.external = .{ .{ .app = "blender", .writes = true,
  .runs = true, .folders = .{ "~/Library/Application Support/Blender/*", "%APPDATA%/Blender Foundation/Blender/*" } } }`. The
  declaration names the **folder patterns** as well as the app, so the consent dialog and the store list the places that will
  be written before any scan, and the service refuses a write outside them. `app` is a short id from a list the registry knows (`blender`, `unreal`), so the store can name it;
  an unknown id is shown verbatim. An app built on fizzy (Graphl, example-app) declares its own `app` the same way, with
  its own plugin source; an add-on put there by someone else (Blender's extension platform) is not in the manifest and is
  left alone. The registry aggregator copies the field into `registry.db` and the catalog.
- **Store display.** The plugin's card and detail page say "Installs into Blender; runs Blender in the background" with
  the same prominence as `works_with`; the trust dialog (section 3) gains a line for it. A release that adds an `app` the
  previous release did not declare is held for the person to confirm, as a source change is.
- **Consent, per host, first use.** The first `external_apps.install/run` for (plugin, app) shows a dialog: the app, the
  folders it will write (the found installs, chosen by the person), that it will run the app headless, and that fizzy can
  remove what it installed. Allow / Cancel. Recorded in `settings.zon` as `.plugins.<id>.external.<app> = .{ .since }`
  beside `.trust`, hand-writable for provisioning. Revoking removes the install through the manifest. Trust in the
  plugin does not imply this consent, and this consent does not imply tool consent.
- **Enforcement is the host service's.** The SDK `external_apps` service (EMBEDDED_EDITOR_PLAN step 12) refuses a call for an
  `app` the running plugin's identity did not declare or the person has not granted; writes go through it so every file
  is in the manifest (`<config>/integrations/<id>.zon`) and removal is exact. The SDK change moves the fingerprint; the
  zon field is additive.
- **Published as build targets.** The artifacts a declaration covers (the add-on zip per OS/arch, an Unreal plugin per
  engine version x platform) are built by the host repo's CI as gates and published through the store
  (`EMBEDDED_EDITOR_PLAN.md`, "Integrations are build targets").
- **Web.** No such service: an integration plugin does not load on the web.

### 5. Signed components loaded into a host process

Decided 2026-10-19 (`EMBEDDED_EDITOR_PLAN.md`, "Signing and verifying components"). An editor component downloaded for a
host's standalone mode is native code in *someone else's* process, so trust-on-first-install is not enough. It must be
signed (macOS Developer ID + notarization, Windows Authenticode, a detached minisign signature on Linux, verified against a public key built into the shell) by the app's publisher
in release CI, and the shell verifies the hash, the OS signature check and the signer identity built into it before
loading; a failure never loads. This is the first use of the "signature per release" idea under Open, scoped to
components; plugins keep the trust record of section 3.

## Steps (each its own PR)

1. `core.darwin_spawn` takes extra environment variables; one spawn helper for tools.
2. Trust: the dialog, the record, gated auto-update, the source-change hold, the outside-the-store
   case. No SDK change.
3. The tools manager in `app/` (sources including `self`, receipts, resolution, background
   checks, side-by-side versions and rollback, the store's verified downloader) and the SDK
   `tools` service (moves the fingerprint).
4. Settings > Tools, and the tool consent dialog.
5. `tools/` in the registry, exported with the catalog.
6. First consumers: the zig plugin gets `zls` through tools (managed); the agent plugin gets
   Claude Code (`self`) and Codex (managed), then Node and the npm agents; the agent plugin's `fizzy-mcp` as a `plugin`
   tool (or the agent plugin serving MCP over HTTP, which needs no program at all).
7. `works_with` in the manifest, shown and offered by the store.
8. The `.external` declaration in `plugin.zig.zon`, `manifest.json` and the registry, the store display, and the
   consent record, with folder patterns in the declaration. The service itself is EMBEDDED_EDITOR_PLAN step 12 (that PR carries this step's SDK half); the
   registry and `plugin-build-action` halves are PRs in `fizzyedit/plugins` and `fizzyedit/plugin-build-action`.

## Open

- Whether `zig` itself is a tool the zig plugin needs (a per-project version from
  `build.zig.zon`'s `minimum_zig_version`), or only `zls`.
- Signing: components are now signed (section 5); for store plugins the registry pins hashes, but nothing says the bytes are the author's. A signature per
  release, checked against a key recorded with trust, would make "trust this author" literal.
- Whether a Flatpak override (`flatpak override --user --filesystem=...`) is a write the `external_apps` declaration must name
  (`.runs`/`.writes` for `flatpak`?), or part of the host's own entry. Decided direction: recorded in the manifest and
  consented with the install (`EMBEDDED_EDITOR_PLAN.md` step 16); the field shape is open.
