# Control server: fizzy answers HTTP on a socket of its own

Status: in progress. Step 1 merged (fizzyedit/fizzy#490); steps 2–4 and 5a–5b in review (#499, #508, #509, #538, #541); 5c next, in fizzyedit/agent. Proposed 2026-10-17. Agreed with the maintainer: the server lives in `app/`, not in the
agent plugin.

Today the only way into a running fizzy from outside is the agent plugin's socket
(`fizzyedit/agent` `src/Server.zig`): a Unix socket, a thread per connection, requests handed to
the UI thread. Its wire is private — one ZON value per line — and the one client that speaks it is
`fizzy-mcp`, a separate binary. So scripting fizzy needs an agent, an MCP client, and a plugin
that most installs won't have.

This plan keeps that server's shape and changes three things:

- **The wire is HTTP/1.1**, so `curl --unix-socket` drives fizzy with no agent and no MCP tooling,
  and the API has an OpenAPI spec.
- **The app owns it**, in `app/control/`, so it is there in every fizzy and every app built on
  fizzy, not only where the agent plugin is installed.
- **The API is commands.** Every plugin's commands, with the typed parameters they already
  declare (AGENTS_PLAN milestone 1), are the routes. Nothing registers a route.

## Principles

- **A socket, not a port.** The server listens on a Unix socket in the profile directory, readable
  by this user only. Being able to open the socket is the authorization, as for `ssh-agent` or
  Docker. Loopback TCP is opt-in, for clients that can only dial a URL (step 6), and needs a token
  when it is on.
- **Nothing here is a back door.** A command over HTTP is the same command the palette runs, with
  the same arguments. An agent can do what a person at the keyboard can, and no more (the agent
  plugin's rule, kept).
- **Idle costs nothing.** The listening thread blocks in `accept`. A request costs one wake and
  is answered on the next frame. Nothing polls.
- **JSON outside, ZON inside.** Command arguments and results stay ZON across the plugin boundary
  (decided in AGENTS_PLAN). The server converts at its edge, once, in one file with tests. A
  client may send `Content-Type: application/zon` to skip the conversion.
- **Every seam names a non-agent consumer.** curl and shell scripts, CI and integration tests,
  editor integrations, a second launch forwarding its argv (`app/single_instance`), the chat panel
  followed from a phone.

## The API

```
GET  /                       fizzy's identity, version, SDK version, profile
GET  /commands               every registered command: id, title, owner, params
POST /commands/{id}          run a command; body = its arguments, response = its result
GET  /openapi.json           the spec, generated from the commands loaded right now
POST /sessions               open a session; returns its id
DEL  /sessions/{id}          close it: claims let go, its recording becomes a tape
GET  /events?topics=a,b      Server-Sent Events
POST /mcp                    MCP over Streamable HTTP (step 5)
```

- **Errors are HTTP's.** 404 for an unknown command, 400 for arguments that do not parse (with the
  parameter named), 409 for a command that is not enabled now (the reason from `runWith`), and 200
  with the result otherwise.
- **Sessions are explicit.** The agent plugin ties claims and recordings to a connection, but each
  curl call is a connection of its own. A call carrying `Fizzy-Session: <id>` belongs to that
  session. A call without one is a one-off: no claim, nothing recorded. A session left idle for
  ten minutes closes as if deleted.
- **Long calls stream.** A command whose answer takes frames (`agent.build`, `agent.run`, a tape)
  answers `text/event-stream` when asked with `Accept: text/event-stream` (progress events, then
  the result), and a single JSON body otherwise.
- **Finding the socket.** `<profile>/control.sock`, the default profile's path or `--profile`'s.
  `fizzy --control-path` prints it, so a script writes
  `curl --unix-socket "$(fizzy --control-path)" localhost/commands`.

## Steps

Each step is one PR. Steps 1–3 land dark behind `FIZZY_CONTROL=1`. Step 4 switches it on.

1. **`app/control`: the server and the commands routes.** Lift the agent plugin's `Server` into
   `app/control/Server.zig` (thread per connection, the UI-thread queue, inode-checked socket
   removal, the wake pipe) and put `std.http.Server` on each stream. Routes `/`, `/commands` and
   `/commands/{id}`, over `Host.callCommand`. Add `app/control/json.zig` (JSON ⇄ ZON, guided by each
   parameter's `settings.Kind`) with unit tests, plus an integration test that runs a command over
   the socket. On Windows, use AF_UNIX (Windows 10 1803+, and Windows' bundled curl supports
   `--unix-socket`) rather than a named-pipe server; check it in the Windows VM. Measure the round
   trip (request → next frame → answer) and record it.
2. **Events** (fizzyedit/fizzy#499). `/events` as SSE, relaying the host's existing event bus
   (`Host.emit` / `subscribe`): fizzy's document, folder and files events and every plugin's,
   filtered by `?topics=`. There is no new SDK seam, since the bus already carries plugins'
   topics, and no fingerprint move. The relay subscribes only while a stream is open. Still to
   come as events: `settled`, from the automation service, and plugins loaded or reloaded.
   Sessions moved to step 5: nothing holds state on the server until `agent.*` commands want
   claims and tapes, and a session with nothing behind it is a seam without a consumer.
3. **`/openapi.json`** (fizzyedit/fizzy#508). Generated per request from the command registry: a
   path per command, its body a JSON Schema of what the server enforces (each parameter's type and an
   enumeration's choices; a number's bounds are hints the server never checks, so they sit in the
   description), and `/events` with the declared events. No committed snapshot of fizzy's own
   spec: it would fail CI on every command description edit, and the live one is always current.
   A result type beside `Command.Params(…)` is the seam for typed results.
4. **On by default, and documented** (fizzyedit/fizzy#509). Drop the flag. Add a `control.enabled` setting (default on,
   native only; the web build has no socket) and `fizzy --control-path`. Write `docs/CONTROL.md`
   (the API, curl recipes, sessions, events). Add a test that the socket is created 0600 in a
   directory only the user can read.
5. **Commands that answer later, `/mcp`, and sessions.** 5a (fizzyedit/fizzy#538, `sdk`): a
   command may answer in a later frame (`Command.Call.later`, `Host.finishCall`,
   `callCommandAwaiting`), which the agent plugin's waiting tools need and any long command uses;
   `fizzy.screen` and `fizzy.settled` are the first. 5b (fizzyedit/fizzy#541): `/mcp`.
   Originally: `/mcp` on the same server, and sessions. MCP Streamable HTTP: `tools/list` = the
   commands, `tools/call` = `POST /commands/{id}`. Sessions arrive here (`/sessions`, the
   `Fizzy-Session` header, idle expiry), with the claims and tapes that give them something to
   hold. The agent plugin's tools move with it in step 5b:
   - **5a (fizzy).** The automation service's snapshot, settled and capture become built-in
     commands (`fizzy.screen`, `fizzy.settled`, `fizzy.capture`), since integration tests and
     curl want them too.
   - **5b (fizzyedit/agent).** What only an agent wants — `run`, `build`, `act`, `profile`,
     `tape` — becomes `agent.*` commands. The plugin's socket and ZON wire go. `fizzy-mcp` shrinks
     to a stdio ⇄ HTTP proxy onto the socket, for MCP clients that speak only stdio. Superseded
     (2026-10-19, AGENTS_ANYWHERE_PLAN step 2): the proxy is fizzy's own `fizzy mcp`, not a
     program the agent plugin ships, so the plugin store can release the agent plugin.
6. **Loopback TCP for URL-only clients (opt-in).** Most HTTP MCP clients take a URL and cannot
   dial a Unix socket. `control.tcp` (default off) listens on `127.0.0.1` with a random port and
   requires `Authorization: Bearer <token>`. The token is written beside the socket and is fresh
   each launch. The server refuses a `Host` other than `127.0.0.1:<port>` / `localhost:<port>`
   (DNS rebinding) and any request carrying an `Origin`. With it on, the chat plugin hands an
   agent `{type: "http", url, headers}` instead of spawning `fizzy-mcp`; with it off, chat keeps
   the proxy.
7. **Chat over the server (fizzyedit/chat).** The chat's actions become commands (`chat.list`,
   `chat.new`, `chat.send`, `chat.read`), and each conversation's transcript is an event topic
   (`chat.<id>`: messages, deltas, tool calls, permission requests). A terminal, a script or a phone
   can then follow a conversation the panel shows and post into it, and an agent in a terminal and
   one in the panel share one server. Builds on CHAT_PLAN's chat service (fizzyedit/fizzy#389).
8. **`single_instance` forwards over the control socket.** A second launch posts its argv to
   `fizzy.open` instead of using `dvui-singleton-app`'s own socket and named pipe. The early lock
   stays as it is; only the message path moves, retiring a second IPC channel.

## Open questions

- ~~**AF_UNIX on Windows through `std.Io.net`.**~~ Settled in step 1 (fizzyedit/fizzy#490):
  std offers it only to a target whose minimum is Windows 10 1803 (`has_unix_sockets`), so
  fizzy's `build.zig` raises the minimum. With it, the server's test passes in the Windows 11 VM,
  and no named pipe is needed. The socket is a reparse point there, so its inode is not checked.
  An app built on fizzy with its own target needs the same minimum.
- **Commands that take a document.** Many commands act on "the active document". Over HTTP, a
  caller should be able to name one (`?document=<path>`) without focusing it. That needs a
  per-call target through `runWith`, as text's by-path commands already do (fizzyedit/fizzy#333).
  Decide in step 1 whether this is a convention per command or a field of the call.
- **Many fizzys, one profile.** A restart's handover briefly runs two fizzys on one profile. The
  inode check in the lifted `Server` covers the socket. Session handover across a restart is not
  covered: sessions end at a restart, and a client opens a new one.

Related: AGENTS_PLAN (commands as the agent API, the agent plugin), CHAT_PLAN (the chat service),
TOOLS_AND_TRUST_PLAN (what an agent may run), AUTOMATION_PLAN (snapshot, settled, tapes).
