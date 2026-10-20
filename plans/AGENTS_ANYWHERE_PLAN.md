# Any agent, on the desktop and the web

Status: proposed, 2026-10-19 (roadmap#8); relay hosting decided 2026-10-19. Builds on CONTROL_SERVER_PLAN (steps 5b–7),
AGENT_VIEW_PLAN (#18), SHARED_HISTORY_PLAN (#14) and AGENTS_PLAN ("The web"). Multiplayer
across machines is its own plan (roadmap#9), on the same rooms.

**The goal, end to end:** on a phone, open fizzyed.it/app, open a sprite in pixi, chat with Claude,
and watch it draw. With the Mac open, `fizzy-link` runs there; with the Mac closed, the same agent
runs in a Claude Code cloud session on the person's own account (step 10d).

Any agent a person already runs and that speaks ACP (Claude, Gemini, Codex, Copilot, Cursor, or an
agent that wraps a model API) should be able to drive fizzy: the desktop app and the web app in a browser tab. It sees what it works on
without taking the person's view, and works beside the person, not instead of them.

Decided with the user, 2026-10-19 (revised 2026-10-19, ACP only):

- **Chat is ACP-only.** Chat talks to agents in the Agent Client Protocol and nothing else. A plain
  model is reached through an ACP agent that wraps a model API; fizzy has no model-API client.
- **The agent comes to fizzy.** On the desktop, ACP runs over stdio, as now. On the web, the same
  ACP JSON-RPC messages run over a WebSocket through the relay: the person's agent runs on their
  machine, already signed in, connects out to the relay, and the fizzy tab joins the room.
- **Fizzy stores no model keys, and the page holds none.** Each agent owns its login (ACP
  `authenticate`); chat shows the agent's sign-in choices and its permission requests. The only new
  sign-in is pairing a tab with an agent (a code, or OAuth to the relay).
- **Hosted agents are out of scope**: the plan covers agents the person runs themselves. No claude.ai,
  ChatGPT or Gemini connector against the relay.
- **fizzy's tools stay on the control server and MCP.** The agent is handed fizzy's tools as an MCP
  server in `session/new`: `fizzy mcp` (stdio) on the desktop, the room's MCP endpoint on the web.
- **A relay is the bridge to a browser tab**, because a page cannot be a server. It is a room
  server, so the same piece later carries people working together (#33). Fizzy runs one on
  Cloudflare Workers + Durable Objects, first at a free `workers.dev` address, later optionally
  at `relay.fizzyed.it`; the same protocol ships as a small
  self-hostable binary, and fizzy and `fizzy-link` take a relay URL setting. The relay sees only
  end-to-end-encrypted bytes (section 5), and keeps no conversation.
- **Fewer, bigger steps:** the model's thinking dominates latency, not the network.

## Principles

- **The app is the transport; plugins own the tools.** `/mcp` serves commands. Anything only an
  agent wants is an `agent.*` command the agent plugin registers.
- **One tool surface.** The same command list is served by the desktop's `/mcp`, `fizzy mcp`, the
  page's in-process MCP, `fizzy-link`'s local MCP (the web's route to it) and WebMCP, under names every client accepts.
- **One conversation protocol.** Chat speaks ACP over a transport (a child's stdio, or a
  WebSocket). Everything above the transport is the same on both platforms.
- **Fewer, bigger steps.** Tools take scripts (`agent.run`), answer with what changed and the
  screen after, and do whole operations, so a task is few turns.
- **Nothing costs a frame unless asked.** Looks and frames are drawn only when an agent asks.
- **Beside the person, not over them.** An agent has its own views, its steps carry its name
  (SHARED_HISTORY_PLAN), and the person sees who is connected and can revoke it.

## The design

### 1. One tool surface, in every build

- **Names:** `/mcp` maps each command id to a name every client accepts (`.` becomes `__`, other
  characters outside `[A-Za-z0-9_-]` become `_`, a short hash on a collision or past 64
  characters; the map is two-way and the command id stays in the description).
- **The agent's tools as commands:** act, run, build, capture, profile, tape, tape outcome, claim
  and release become `agent.*` commands, with authorship per call. All but `build` work on the web.
- **In-page MCP:** the `/mcp` dispatch (`tools/list`, `tools/call` over `Host.callCommandAwaiting`)
  is split from the control server, built for wasm32, and exposed to the page's JS as one async
  call, `fizzy.mcp(request) -> response`. The room and WebMCP sit on it.

### 2. Looks and frames, on request

AGENT_VIEW_PLAN's look (the document's owner draws it offscreen, at a region and scale the agent
picks, with the agent's own camera) on both platforms, through `agent.frame`: drawn only on
request, read back and encoded in wasm. The text snapshot (`fizzy.screen`) stays the cheap
default, and "what changed since you looked" comes with every answer.

### 3. Chat is an ACP client

Checked 2026-10-19 against `chat` (7f36c34) and the ACP spec (agentclientprotocol.com).

| ACP client duty | Chat today | Gap |
| --- | --- | --- |
| `initialize`, `session/new`, `resume`, `load`, `prompt`, `session/update`, `session/cancel` | done (`AcpClient.zig` 428-447, 505-535, 597-) | none |
| `session/request_permission` | done: `askPermission` (823-851) turns the agent's options into buttons on the tool line (`View.zig` 622-); `onAnswer` (316-320) sends `selected`; `cancel` (521-534) answers every open request `cancelled`; allow once/always and reject once/always are mapped | the person sees only the call's title; no detail to judge it by; nothing when the tab is hidden |
| `authMethods` in the `initialize` result | **missing**: `answered(.initialize)` (598-603) reads only `loadSession` and `resume` | show the agent's choices |
| `authenticate` (`methodId`) | **missing**: never sent. A refusal at `initialize` or `session/new` is printed as "The agent refused to start" (`refused`, 639-644), telling the person to use the agent's own CLI | send it; handle `auth_required` and retry |
| `terminal` auth methods (`clientCapabilities.auth.terminal`) | **missing**: not advertised | run the agent's login command interactively, then reconnect |
| `logout` | **missing** | offer it when `agentCapabilities.auth.logout` is set |
| `fs/read_text_file`, `fs/write_text_file` | done, through open documents (853-900) | none |
| `terminal/*` | not advertised (`.terminal = false`, 439), requests answered `method_not_found` | none for now; agents use their own shell |
| MCP servers in `session/new` | stdio only: `fizzy-mcp` from the agent plugin (`openSession`, 459-) | `fizzy mcp`; on the web a placeholder entry that `fizzy-link` fills with its own loopback HTTP server (or `fizzy-link mcp` over stdio), by the agent's `mcpCapabilities` |
| Transport | stdio child only (`Agent.zig`, spawn with `core.darwin_spawn`) | a seam for a WebSocket; chat's web build |

### 4. Sign-in and approvals: the agent's own

- Each agent owns its login. On `initialize`, chat reads `authMethods` (`id`, `name`,
  `description`, `type`). If a session start answers `auth_required`, chat lists the methods in the
  conversation as buttons.
- An `agent` method: chat sends `authenticate { methodId }`. The agent opens the browser itself
  (or whatever its flow is); chat says "Waiting for <agent> to sign in" and starts the session
  again on success. A `terminal` method (offered only if chat advertises `auth.terminal`): chat
  runs the agent's command with the method's `args` and `env` in a terminal pane, waits for exit
  zero, then reconnects. Chat never sees a token or a password.
- Approvals are `session/request_permission`: chat shows the call and the agent's own options
  (allow once, allow always, reject once, reject always) and answers `selected` or `cancelled`.
  What "always" remembers is the agent's business.
- On the web the same two flows run through the relay unchanged: they are ACP messages.

### 5. The web path: the person's agent through the relay

A **room** is one shared session: a fizzy (a browser tab, or later a desktop fizzy) and the agent
linked to it. Until multiplayer (#33) a room holds exactly one tab and one agent (`max_members`
is 2 and a room setting, so #33 raises it without a protocol change).

- **The link:** `fizzy-link -- <agent command>` runs on the person's machine. It starts the agent
  as chat does on the desktop (stdio, in the person's shell, already signed in), connects out to the
  relay over a WebSocket, and copies ACP lines both ways. It holds no model key and adds no agent
  logic, with one exception below (it fills in the MCP entry).
- **The tab** opens a WebSocket to its room. Chat's web transport sends and receives the same ACP
  JSON-RPC lines the desktop sends over stdio, through a `room` service the tab's fizzy owns
  (chat does no crypto and opens no socket of its own).
- **Pairing and the end-to-end key (decided: from the start).** The tab makes a pairing code of
  80 random bits (16 base32 characters, shown in groups, with a copy button; an invite is a link
  with the code in the fragment, which is never sent to a server). From the code both sides
  derive, with HKDF, a **room id** (sent to the relay) and a **room key** (never sent). The relay
  never learns the code, so it cannot read the room or guess its id. After joining, the two sides
  run an X25519 exchange sealed with the room key, so each session has its own keys (forward
  secrecy against a later leak of the code). Everything above the relay's tiny envelope is sealed
  with XChaCha20-Poly1305 (Zig `std.crypto`, both ends in the one `relay_wire` library, so the
  browser tab is Zig in wasm and needs no JS crypto).
- **Why now and not later:** the cost is small (a key derivation, one sealed frame type, a
  library both ends share) and the protocol is cheapest to get right before anything depends on
  it. Adding it later changes every frame and every client. It also lets the relay be run by
  fizzy with no promise to trust: it holds ciphertext it cannot open, so the privacy policy can say
  "we cannot read it", and a leak or subpoena of the relay yields nothing. What the relay does see
  is metadata: the room id, IPs, timing, message sizes, and the envelope's plain fields (below).
- **The pairing window:** the code is good for one join within **10 minutes** (a setting), then
  the room closes to new members. Joining proves the code by a sealed hello each side checks, so
  a wrong guess dies at the other end, and the relay counts failed joins and closes the room after
  five. Each side gets a **resume token** at join (random, kept by the relay as a hash) so a drop
  reconnects without the code. **Revoke** in the tab drops the link and invalidates its token; the
  tab can then open a new pairing window with a new code. Expiring and re-pairing is normal.
- **The envelope (the relay's whole vocabulary):** `{v, type, to, seq?}` plus an opaque sealed
  payload. Types: `hello`, `joined`, `data`, `revoke`, `ping`, `error`. The relay routes `data`
  to the room's other member and reads nothing inside it. #33 adds plain header fields the
  sequencer needs (peer, local id, document id as an opaque id, kind) and keeps the effect sealed.
  The wire is written once in the relay repo's `PROTOCOL.md` with shared test vectors.
- **fizzy's tools:** the tools reach the agent through **`fizzy-link`'s own MCP server**, not the
  relay's. The agent's MCP client talks to the link on loopback (HTTP with a token, or `fizzy-link
  mcp` over stdio for agents without http MCP); the link seals each call and sends it through the
  room to the tab, which serves it through the in-page MCP and answers back the same way. So the
  tool traffic is end to end as well, and the relay has no MCP code, no OAuth, and no bearer
  tokens to leak. The tab's `session/new` carries a placeholder MCP entry (`fizzy-link://mcp`);
  the link, which reads `initialize` to see the agent's `mcpCapabilities`, replaces that one entry
  with its loopback server (http) or its stdio command. That is the only message the link edits.
- **Where it runs:** a Cloudflare Worker on the free `workers.dev` address of the maintainer's own
  (new, free) Cloudflare account, e.g. `fizzy-relay.<subdomain>.workers.dev`; no DNS or domain is
  needed. `relay.fizzyed.it` is a later, optional step (below). The Worker that hands each connection to **one
  Durable Object per room**, which holds the room's two sockets using hibernatable WebSockets
  (ping/pong answered by the platform's auto-response, so an idle room is evicted from memory and
  costs nothing). It stores nothing about the conversation: room state is two socket attachments
  and a few hashed tokens, with an alarm that deletes the room at its TTL. Fizzy operates it.
  The same protocol is a small Zig binary (`relay-selfhost`) for people who run their own.
- **Abuse limits (defaults, settings of the deployment):** pairing window 10 min; room idle TTL 15
  min with no member connected, absolute life 12 h; one tab and one agent per room; frames of at
  most 1 MiB; 200 frames/s and 8 MiB/s per room with a burst of twice that, then the socket is
  closed with `error`; at most 5 open rooms and 20 room creations per minute per IP (counted by
  the Worker before it wakes an object); a connection that does not send `hello` in 10 s is
  dropped; five failed joins close a room. Numbers get tuned from the first weeks of use.
- **In the page:** an "agent connected" bubble says which agent (from the sealed hello), with
  Revoke. A connected tab keeps drawing, since browsers slow hidden tabs and an agent would stall.
- **Privacy:** the relay carries ciphertext and keeps nothing past the session. No model key is
  ever sent anywhere. The privacy policy says what the relay sees (metadata) and what it cannot.
- **Settings:** both fizzy and `fizzy-link` take a relay URL (default the `workers.dev`
  address, `wss://fizzy-relay.<subdomain>.workers.dev`, until a custom domain replaces it), so a
  self-hosted relay is a one-line change.

### The relay repo: `fizzyedit/relay`

New repo, **private until the first deploy works, then public under MIT** (the self-host binary
and the protocol are meant to be read and run). Layout:

- `PROTOCOL.md` and `protocol/vectors/`: the envelope, the join/resume/revoke flow, error codes,
  limits, and JSON test vectors (envelopes, key derivation, sealed frames).
- `worker/`: the Cloudflare Worker and the `Room` Durable Object, in **TypeScript**.
- `wire/`: `relay_wire`, a std-only Zig library (pairing derivation, seal/open, envelope
  encode/decode, vectors). `fizzy` and `fizzy-link` depend on it by URL, as the one copy of the
  protocol on the client side.
- `link/`: `fizzy-link`. `selfhost/`: `relay-selfhost`. Both Zig.
- `conformance/`: one protocol test suite that connects real WebSockets to *a relay URL* and
  checks it against `PROTOCOL.md`. CI runs it against the Worker (in workerd) and against the
  Zig binary, so the two cannot drift.

**TypeScript for the Worker (recommended), not Zig compiled to wasm.** A Durable Object class must
be a JavaScript class that the runtime instantiates, and the hibernatable-WebSocket API
(`acceptWebSocket`, `webSocketMessage`, attachments, alarms) is JS-only; wasm would be a thin
glue file calling into a module, adding a build and a boundary and buying nothing, since the relay
copies sealed bytes and has no logic worth sharing with Zig. TypeScript has the types for the
bindings, `wrangler`, and the best local testing: `@cloudflare/vitest-pool-workers` runs the
Worker and its Durable Objects in the real `workerd` runtime (Miniflare underneath), with
hibernation and alarms included, on Linux with no account. The Zig code stays where it earns its
keep: the client library and the self-host binary, which the browser and desktop need anyway.

**CI** (GitHub Actions, Linux): worker tests (`npm ci`, vitest in workerd), `zig build test` for
`wire/`, `link/` and `selfhost/`, the conformance suite against both servers on localhost, a
`wrangler deploy --dry-run`, and an `ci-ok` gate like fizzy's. **Deploy** is a job that runs
`wrangler deploy` only on pushes to `main`, after the tests, in a `production` environment, using
the repository secret `CLOUDFLARE_API_TOKEN` (and `CLOUDFLARE_ACCOUNT_ID`); with no secret set it
skips, so everything merges before the account exists.

**What the maintainer provides** (agents do all the rest): a free Cloudflare account (new, the
maintainer's own); that is enough to go live on `fizzy-relay.<subdomain>.workers.dev`. Later, an API
token scoped to Workers Scripts edit, saved as the repo secret `CLOUDFLARE_API_TOKEN` (with
`CLOUDFLARE_ACCOUNT_ID`) so CI can deploy; the `production` environment's approval rule if wanted;
and a look at the Workers plan's Durable Object limits before launch (the free plan allows
SQLite-backed objects; the paid plan is for headroom). Creating the private `fizzyedit/relay` repo
is a one-minute step the maintainer or an agent with org rights can do.

**Later, optional: `relay.fizzyed.it`.** A Workers custom domain needs the zone in the
maintainer's own Cloudflare account. `fizzyed.it` is registered at 101domains, whose included
Cloudflare is their partner setup with no dashboard login, so this means moving the domain's
nameservers from 101domains to Cloudflare. Cloudflare imports the existing records, and the
website's DNS would then live there (check the import against the old records before switching).
The token then also needs Workers Routes / custom domain edit on the zone. Nothing else changes:
the relay URL setting default moves to `wss://relay.fizzyed.it`.

### 6. Agents with no relay

- **WebMCP:** the page registers the same tools with the browser (`navigator.modelContext`, a W3C
  draft), for browser-built agents. Cheap once the in-page MCP exists; ships late, while its API
  moves.
- **Desktop agents:** `fizzy mcp`, the executable's stdio mode onto the local socket. Opt-in
  loopback HTTP with a token is CONTROL_SERVER_PLAN step 6. Chat hands its agents `fizzy mcp`.

### 7. Several agents at once: branches

One message can go to several ACP agents (each may wrap a different model); each answers in its
own branch.

- **A conversation holds branches,** one per recipient, each with its own transcript and its own
  agent session. A message is sent to one branch, to several, or to all.
- **Side by side:** read next to each other; any branch continues alone, or the next message goes
  to all again.
- **Each branch is its own author** (SHARED_HISTORY_PLAN), so history and undo say who did what.
- **Documents:** the same documents by default, claims keeping two agents out of one edit. Later,
  a branch can work on its own copy (HISTORY_TIMELINE_PLAN, #10). Group messaging is later.

### 8. Store releases

Agent and chat get the standard release workflow (with `web-wasm32` once their web builds
compile) and a registry entry each. Agent ships only its library.

## Trade-offs, accepted

- **A service to run:** uptime and some cost, mostly bandwidth. Self-hostable. Cloudflare is a
  single vendor; the protocol is plain WebSocket, so moving the hosted relay is a redeploy.
- **Metadata through the room** (IPs, timing, sizes), mitigated by end-to-end sealing, keeping
  nothing, a clear policy, and self-hosting.
- **A pairing code to copy** (16 characters, or a link), the price of a relay that cannot read.
- **The person runs a small program** (`fizzy-link`) beside their agent to use it from the web.
- **A connected tab must stay open,** and fizzy keeps it drawing while an agent is connected.
- **Agents that are only hosted** (a chat app with no local ACP agent) cannot join. Accepted.

## Steps (each its own PR, at most 800 lines)

Each step lists repo, branch and builds-on, then its kickoff. "Cloud" means it can run in a Linux
cloud session (tests only); "macOS" means it needs a Mac to see it run.

### 1. agent: its tools as `agent.*` commands (done, fizzyedit/agent#25)

### 2. fizzy: tool names every client accepts; the MCP dispatch split from the control server

- Repo: fizzy. Branch: `plans/agents-names-dispatch`. Builds on: 2a is fizzyedit/fizzy#638 (merged); 2b in progress: MCP tool names and the wasm-buildable dispatch.
- Start now: in progress. macOS.

```
Task: Finish 2b: map command ids to MCP tool names (`.` to `__`, hash on collision, two-way map), split the /mcp dispatch (tools/list, tools/call) from the control server so it builds for wasm32.
Plan: AGENTS_ANYWHERE_PLAN section 1; CONTROL_SERVER_PLAN 5b; fizzy docs/CONTROL.md
Builds on: fizzy#638
Workspace: ~/dev/fizzy-mcp-names
Branch: plans/agents-names-dispatch
Notes: Keep the command id in each tool description. No socket in the dispatch. Tests for the name map, including collisions and 64 chars.
```

### 3. fizzy: the in-page MCP (`fizzy.mcp` in the web build); `fizzy mcp` on the desktop

- Repo: fizzy. Branch: `plans/agents-fizzy-mcp`. Builds on: step 2.
- After 2. macOS (web build run in a browser); the stdio framing tests can run in the cloud.

```
Task: Expose the dispatch to page JS as one async `fizzy.mcp(request) -> response`; add the `fizzy mcp` stdio mode that forwards JSON-RPC lines to the local control socket.
Plan: AGENTS_ANYWHERE_PLAN sections 1 and 6
Builds on: step 2 (names + dispatch)
Workspace: ~/dev/fizzy-mcp-stdio
Branch: plans/agents-fizzy-mcp
Notes: Responses may be async (Host.callCommandAwaiting). `fizzy mcp` exits when stdin closes. Document in docs/CONTROL.md. Do not start fizzy if none is running (open question).
```

### 4. fizzy + owners: looks and frames on request; `agent.frame`

- Repo: fizzy (and agent, pixi, text owners). Branch: `plans/agents-frames`. Builds on: step 3; AGENT_VIEW_PLAN steps 2-4.
- After 3 and AGENT_VIEW_PLAN 2-4. macOS. May split per owner.

```
Task: Land `agent.frame`: a picture of a pane, window or look, drawn offscreen only on request, read back and encoded (WebGL framebuffer on the web).
Plan: AGENTS_ANYWHERE_PLAN section 2; AGENT_VIEW_PLAN
Builds on: step 3
Workspace: ~/dev/fizzy-frames
Branch: plans/agents-frames
Notes: Agent picks size and region. Small pictures; PNG vs WebP is open. Nothing drawn unless asked. One PR per owner if over 800 lines.
```

### 5. chat: agent sign-in (ACP `authMethods`, `authenticate`, terminal methods, logout)

- Repo: chat. Branch: `chat-acp-auth`. Builds on: none (chat main).
- Start now. Pure parts (acp.zig: method parsing, request writers, tests) can run in the cloud; seeing the buttons needs macOS.

```
Task: Make chat sign in to an agent through ACP: read `authMethods` from `initialize`, show choices when a session start fails with `auth_required`, send `authenticate`, retry; run `terminal` methods in a terminal pane (advertise `auth.terminal`); offer `logout`.
Plan: AGENTS_ANYWHERE_PLAN sections 3-4; ACP spec: protocol/authentication
Builds on: chat main (7f36c34)
Workspace: ~/dev/fizzyedit/chat-acp-auth
Branch: chat-acp-auth
Notes: Edit chat src/AcpClient.zig `answered(.initialize)` and `refused`, acp.zig, View.zig buttons (reuse the question UI, `chat.ask`). Chat never sees tokens. Terminal methods must not be passed to `authenticate`. Tests: method parsing, auth_required retry.
```

### 6. chat: approvals you can judge (`session/request_permission` polish)

- Repo: chat. Branch: `chat-acp-approvals`. Builds on: none (can follow 5 in the same area).
- Start now. Mostly macOS (UI); tests for option mapping can run in the cloud.

```
Task: Request handling exists. Add what is missing: show the call's detail (input, command, file paths, diff) beside Allow/Reject, clear open requests when the agent exits or the chat closes, and flag a pending approval when the chat is not in view.
Plan: AGENTS_ANYWHERE_PLAN section 4; ACP spec: protocol/tool-calls
Builds on: chat main
Workspace: ~/dev/fizzyedit/chat-acp-approvals
Branch: chat-acp-approvals
Notes: Today: AcpClient.zig askPermission/onAnswer/cancel, View.zig drawQuestion. Answer `selected` with the agent's optionId, `cancelled` on cancel. No auto-allow here; TOOLS_AND_TRUST_PLAN owns policy.
```

### 7. chat: a transport seam under the ACP client

- Repo: chat. Branch: `chat-acp-transport`. Builds on: none; touches the same files as 5 and 6, so land after them.
- After 5 and 6 (to avoid conflicts). macOS (spawns the agent).

```
Task: Split Agent.zig into a Transport (send a line, take lines, exit state) with the stdio child as the first implementation; AcpClient uses only the interface. No behavior change.
Plan: AGENTS_ANYWHERE_PLAN sections 3 and 5
Builds on: 5, 6
Workspace: ~/dev/fizzyedit/chat-acp-transport
Branch: chat-acp-transport
Notes: Keep core.darwin_spawn and the reader threads. The web transport arrives in step 12. Make the interface line-based so a WebSocket text frame is one line.
```

### 8a. relay (new repo): the protocol, the Worker and its rooms, tested locally

- Repo: fizzyedit/relay (new, private). Branch: `relay-rooms`. Builds on: none.
- Start now. Cloud (Linux, workerd/Miniflare; no Cloudflare account).

```
Task: New repo fizzyedit/relay. Write PROTOCOL.md (envelope hello/joined/data/revoke/ping/error; join, resume, revoke; error codes) and JSON vectors. Worker (TypeScript) + `Room` Durable Object with hibernatable WebSockets: room id from the URL, one tab + one link, pairing window 10 min then closed, five failed joins close it, resume tokens (stored hashed), revoke, alarm deletes at TTL. Routes `data` unread. Conformance suite (vitest, takes a relay URL) and a vitest-pool-workers run of it. CI: tests on Linux, ci-ok.
Plan: AGENTS_ANYWHERE_PLAN section 5, "The relay repo"
Builds on: none
Workspace: ~/dev/fizzyedit/relay
Branch: relay-rooms
Notes: Never parse or log payloads. Keep nothing but socket attachments and hashed tokens. Use the hibernation API and setWebSocketAutoResponse for ping. No wrangler login needed: test in workerd. MIT, README with the protocol overview. Leave abuse limits to 8b.
```

### 8b. relay: abuse limits, TTLs, hibernation

- Repo: relay. Branch: `relay-limits`. Builds on: 8a.
- After 8a. Cloud (workerd/Miniflare).

```
Task: Add the limits from the plan's section 5: frame max 1 MiB, per-room rate (200 frames/s, 8 MiB/s, burst x2) then close with `error`, hello within 10 s, idle TTL 15 min and 12 h life through alarms, and per-IP counts in the front Worker (5 open rooms, 20 creates/min; a counter Durable Object keyed by IP, not per-room state). All numbers in one config module, overridable by env vars.
Plan: AGENTS_ANYWHERE_PLAN section 5, abuse limits
Builds on: 8a
Workspace: ~/dev/fizzyedit/relay-limits
Branch: relay-limits
Notes: Tests in workerd with fake time (alarms) for each limit; assert a room evicted from memory still honors its TTL. Add each limit to PROTOCOL.md and its error code to the conformance suite. Document how to see counts without reading payloads (no logging of content).
```

### 8c. relay: deploy workflow and the relay URL

- Repo: relay. Branch: `relay-deploy`. Builds on: 8a.
- Code is cloud-able; **going live needs the maintainer's free Cloudflare account and, for CI deploys, a token.**

```
Task: wrangler.toml with the Durable Object (SQLite class) migration, the workers.dev name `fizzy-relay` for the first deploy (the custom domain relay.fizzyed.it is a later, optional step, documented but not enabled). A deploy job in CI: on push to main, after tests, environment `production`, `wrangler deploy` using secrets CLOUDFLARE_API_TOKEN and CLOUDFLARE_ACCOUNT_ID; skipped with a notice when the secret is unset. DEPLOY.md: exactly what the maintainer sets up (account, workers.dev subdomain, token scopes, secret, environment; the optional nameserver move for relay.fizzyed.it) and a smoke test script (pair two sockets).
Plan: AGENTS_ANYWHERE_PLAN, "The relay repo"
Builds on: 8a
Workspace: ~/dev/fizzyedit/relay-deploy
Branch: relay-deploy
Notes: Never commit a token. `wrangler deploy --dry-run` runs in PR CI. After the maintainer adds the secret, a re-run of the job on main deploys; then run the smoke test against the live URL and say so in the PR.
```

### 9. relay: `relay_wire`, the pairing, the key and the sealed frames (Zig)

- Repo: relay. Branch: `relay-wire`. Builds on: 8a (its vectors).
- After 8a. Cloud (Linux, `zig build test`).

```
Task: wire/: a std-only Zig library. Pairing code (80 bits, base32 groups), HKDF to room id and room key, the X25519 session exchange sealed with the room key, XChaCha20-Poly1305 seal/open with counters, envelope encode/decode, and the sealed hello (agent name). Passes protocol/vectors/ from 8a; add key-derivation and sealed-frame vectors there that the TypeScript conformance suite also checks.
Plan: AGENTS_ANYWHERE_PLAN section 5 (pairing and the end-to-end key)
Builds on: 8a
Workspace: ~/dev/fizzyedit/relay-wire
Branch: relay-wire
Notes: std only (it must build for wasm32 and every host target). Use Zig std.crypto; no custom primitives. Reject replayed and out-of-order counters. Tests: tamper, wrong code, replay, reordered frames. Document the threat model (relay sees metadata only) in PROTOCOL.md.
```

### 10a. relay: `fizzy-link`, the person's machine joins a room

- Repo: relay. Branch: `relay-link`. Builds on: 8a, 9.
- After 8a and 9. Cloud (Linux, tests with a fake ACP agent and a local relay); a Mac check with a real agent.

```
Task: link/: `fizzy-link --relay URL --code CODE -- <agent command>` starts the agent on stdio, pairs, and copies its ACP lines to and from the room, sealed with relay_wire, reconnecting with the resume token on drops. Relay URL also from FIZZY_RELAY_URL; default the workers.dev address. One static binary per OS (a release workflow can come later).
Plan: AGENTS_ANYWHERE_PLAN section 5
Builds on: 8a, 9
Workspace: ~/dev/fizzyedit/relay-link
Branch: relay-link
Notes: No agent logic and no keys. Agent resolves on the login shell PATH, as chat does. Exit when the agent exits or the tab revokes. WebSocket client: std.http upgrade and framing, kept in link/ unless fizzy's room client (step 11) wants it. Integration test: local relay (workerd or selfhost), fake agent, echo through a fake tab.
```

### 10b. relay: `fizzy-link` serves fizzy's tools to the agent (loopback MCP)

- Repo: relay. Branch: `relay-link-mcp`. Builds on: 10a.
- After 10a. Cloud (tests with a fake agent and fake tab).

```
Task: In the link: a local MCP server (Streamable HTTP on 127.0.0.1, random port, random bearer token; and `fizzy-link mcp` over stdio) whose tools/list and tools/call are sealed to the tab through the room and answered by it. The link watches the agent's `initialize` result for mcpCapabilities and rewrites the one placeholder entry (`fizzy-link://mcp`) in session/new, load and resume to the http server or the stdio command.
Plan: AGENTS_ANYWHERE_PLAN section 5 (fizzy's tools)
Builds on: 10a
Workspace: ~/dev/fizzyedit/relay-link-mcp
Branch: relay-link-mcp
Notes: This is the only message the link edits; everything else is copied byte for byte. Time out and answer an MCP error when the tab is gone. Bind loopback only; reject requests without the token. Tests: both paths, a tab that leaves mid-call, an agent without http support.
```

### 10c. relay: `relay-selfhost`, the same protocol in one Zig binary

- Repo: relay. Branch: `relay-selfhost`. Builds on: 8a (protocol), 8b (limits).
- After 8b. Cloud (Linux).

```
Task: selfhost/: a Zig server for the same protocol (rooms, pairing window, resume, revoke, all the limits, config by flags/env), one binary, TLS left to a reverse proxy. Passes the conformance suite from 8a/8b unchanged. README: run it, put Caddy/nginx in front, point fizzy and fizzy-link at it.
Plan: AGENTS_ANYWHERE_PLAN, "The relay repo"
Builds on: 8a, 8b
Workspace: ~/dev/fizzyedit/relay-selfhost
Branch: relay-selfhost
Notes: In memory only; no files written. Same limits and error codes as the Worker, from the same table in PROTOCOL.md. CI runs the conformance suite against it. If it passes 800 lines, split the WebSocket server (std.http upgrade) from the room logic.
```

### 10d. relay: the agent in a Claude Code cloud session ("Claude on my phone, Mac closed")

- Repo: relay. Branch: `relay-cloud-session`. Builds on: 10a, 10b, 8c (live relay to test).
- After 10a-c. Cloud for the scripts and docs; **the live run needs the maintainer** (Cloudflare account live, a Claude Code on the web account, the `fizzy-pm` environment).

```
Task: Let the agent run in a Claude Code cloud session on the person's own account, so it works with the Mac closed. The session runs fizzy-link with Claude Code's ACP adapter (`fizzy-link --relay URL --code CODE -- <adapter>`) and joins the room by pairing code. Ship cloud/: a start script (build or fetch fizzy-link, run it, print status), a session prompt that runs it, and CLOUD.md: how the person starts it (a routine, or `claude --cloud` / Claude Code on the web with that prompt), pairing from the phone (the tab shows the code; the person pastes it into the session, or the routine takes it as input), lifetime, and idle shutdown.
Plan: AGENTS_ANYWHERE_PLAN section 5, step 10d
Builds on: 10a, 10b
Workspace: ~/dev/fizzyedit/relay-cloud-session
Branch: relay-cloud-session
Notes: The person's own account, so inside "no hosted third-party agents". No keys in the repo or relay. Use cloud environment fizzy-pm (installs Zig and the dependency cache). The link exits on revoke or the room's idle TTL, ending the session. Test with a fake adapter; the real run needs the maintainer.
```

How it works: the cloud session is a Linux sandbox with outbound network, running `fizzy-link` and
the adapter as child processes, like a Mac does. It is the same pairing as on the desktop: the
tab (on the phone) shows the code, the session joins. Idle shutdown follows the room: the relay
deletes a room 15 minutes after the last member leaves, the link exits, and the session finishes;
the person starts a new one and pairs again. **Needs the maintainer:** a live relay, the
`fizzy-pm` environment kept up to date (Zig version, dependency cache), and a try with their
Claude Code on the web account on the phone.

### 11. fizzy (web, and desktop): join a room; pairing code, "agent connected" bubble with Revoke, drawing kept awake

- Repo: fizzy. Branch: `plans/agents-room`. Builds on: steps 3, 8a, 9.
- After 3, 8a and 9 (test live with 10a). macOS (browser run).

```
Task: Fizzy opens a room: WebSocket (JS in the web build, std.http upgrade on desktop), shows the pairing code and an invite link, seals with relay_wire (a zon dependency), serves the room's MCP calls through `fizzy.mcp`, exposes a `room` host service (send/receive ACP lines) for chat, shows who is connected with Revoke, and keeps frames running while connected. Relay URL is a setting (default the workers.dev address).
Plan: AGENTS_ANYWHERE_PLAN section 5
Builds on: 3, 8a, 9
Workspace: ~/dev/fizzy-room
Branch: plans/agents-room
Notes: Core only (no chat dependency). relay_wire is app-side only: no plugin build may pull it in (see AGENTS.md on plugin deps). Steps carry the agent's name (SHARED_HISTORY_PLAN). Verify against a local relay (workerd) with fizzy-link and a fake agent; say which browsers. Split native/web if over 800.
```

### 12. chat (web): the room transport, chat's web build, the MCP placeholder in `session/new`

- Repo: chat. Branch: `chat-web-room`. Builds on: steps 7, 10a, 10b, 11.
- After 7, 10b and 11. macOS (browser run).

```
Task: Compile chat for web-wasm32 (no process spawn); add a Transport that uses fizzy's `room` service (ACP lines in and out; no socket, no crypto in chat); send the placeholder MCP entry `fizzy-link://mcp` in session/new on the web so the link fills it in by the agent's mcpCapabilities.
Plan: AGENTS_ANYWHERE_PLAN sections 3 and 5
Builds on: 7, 10a, 10b, 11
Workspace: ~/dev/fizzyedit/chat-web-room
Branch: chat-web-room
Notes: Sign-in and approvals need nothing new: they are ACP. Show "waiting for an agent" with the pairing code until the link joins, and the agent's name once it does. No model key anywhere.
```

### 13. fizzy (web): WebMCP registration

- Repo: fizzy. Branch: `plans/agents-webmcp`. Builds on: step 3.
- Later; after 3. macOS.

```
Task: Register the in-page tools with `navigator.modelContext` when present.
Plan: AGENTS_ANYWHERE_PLAN section 6
Builds on: step 3
Workspace: ~/dev/fizzy-webmcp
Branch: plans/agents-webmcp
Notes: A draft API behind a flag; keep it behind a feature check and small.
```

### 14. chat + agent: `fizzy mcp` for chat's agents; agent drops its bridge and socket; loopback HTTP

- Repo: chat, agent, fizzy. Branch: `chat-fizzy-mcp / agent-drop-bridge`. Builds on: step 3.
- After 3. macOS. Two PRs (chat, agent); loopback HTTP is CONTROL_SERVER_PLAN 6.

```
Task: Chat's `session/new` hands agents `fizzy mcp` instead of `fizzy-mcp`; the agent plugin removes its bridge and socket.
Plan: AGENTS_ANYWHERE_PLAN section 6; CONTROL_SERVER_PLAN 6
Builds on: step 3
Workspace: ~/dev/fizzyedit/chat-fizzy-mcp
Branch: chat-fizzy-mcp
Notes: Change AcpClient.zig findMcp/openSession. Settings.mcp_command stays as an override. Do the agent PR after chat has shipped.
```

### 15. agent and chat on the store

- Repo: agent, chat, plugins. Branch: `release-store`. Builds on: 14 and chat's web build (step 12) for `web-wasm32`.
- After 12 and 14. Cloud for the workflow files; macOS to check a release.

```
Task: Standard release workflow and a registry entry for each; agent ships only its library.
Plan: AGENTS_ANYWHERE_PLAN section 8
Builds on: 12, 14
Workspace: ~/dev/fizzyedit/agent-chat-release
Branch: release-store
Notes: Follow plugin-build-action. One PR per repo.
```

### 16. chat: branches. One message to several ACP agents, side by side

- Repo: chat. Branch: `chat-branches`. Builds on: steps 7 and 5.
- After 7. macOS.

```
Task: A conversation holds branches, one per agent: send to one, several or all; each has its own session, transcript and author, shown side by side.
Plan: AGENTS_ANYWHERE_PLAN section 7; SHARED_HISTORY_PLAN
Builds on: 7, 5
Workspace: ~/dev/fizzyedit/chat-branches
Branch: chat-branches
Notes: Each recipient is an ACP agent (a model API is reached through an ACP agent that wraps it). Layout (tabs, columns, compare) is open; pick columns first.
```

### 17. Later: a branch working on its own copy of a document (with #10's history); group messaging

Start now: 2b (in progress), 5, 6, 8a. Right after: 7 (after 5 and 6), 8b, 8c, 9 and 10c (after 8a/8b),
10a and 10b (after 8a and 9), 10d (after 10a-c), 3 (after 2). Cloud-friendly (Linux, tests only): all of 8-10 and the
code of 8c, the parsing and release-workflow parts of 5, 6 and 15. **Everything in the relay repo
is testable locally with workerd/Miniflare and a local Zig relay, so none of steps 8a-10c waits for
Cloudflare.** Only **going live** (the deploy in 8c to the `workers.dev` address, the token
secret, and the optional `relay.fizzyed.it` domain move) needs the maintainer, as does the live run of 10d; fizzy's step 11 and chat's step 12 are developed and tested against a
local relay and need the live one only for the final check. Needs macOS: 2, 3, 4, 7, 11, 12, 13, 14,
16 and the visible parts of 5 and 6.

Steps 1-3 let a desktop agent drive fizzy over `fizzy mcp`. Steps 5-6 settle sign-in and approvals
on the desktop. Steps 8-12 put the person's own agent in a browser tab, end to end encrypted; 10d lets that agent
live in a cloud session when the Mac is closed.

## Later: people working together

Several people in one room, on different machines: a plan of its own (roadmap#9), on this
plan's rooms (the relay raises `max_members`, and adds the sequencer; its payloads stay sealed). Still to design: a room-ordered step log; rebasing concurrent steps or regional
claims; presence; joining mid-session from a track's checkpoint.

## Open

Decided 2026-10-19 and removed from here: where the relay runs (Cloudflare Workers + Durable
Objects, first at a free `workers.dev` address and optionally `relay.fizzyed.it` later, run by fizzy, plus a self-host Zig binary), end-to-end sealing from
the start, TypeScript for the Worker, the repo, and `fizzy-link` living in the relay repo.

- Pairing by code only at first, or OAuth to the relay too (the code needs no account; OAuth would
  add identity and would let one tab pair without copying).
- The code's shape: 16 base32 characters (80 bits) is the default; shorter would need a PAKE.
  Whether a QR code or a "pair" deep link is worth adding for phones.
- Abuse numbers (rates, rooms per IP, TTLs) are guesses until there is traffic; who watches them.
- Whether the repo's public date is the first working deploy or the first release of fizzy-link.
- How people get `fizzy-link` (a release binary per OS, the store, a one-line installer).
- Cloudflare plan and cost ceiling; what happens if the free allowance is exceeded (the relay
  refuses new rooms rather than failing open).
- Whether a tab may hold more than one agent at once (branches on the web), and how claims interact.
- Picture format and limits for frames.
- How branches are laid out.
- Whether `fizzy mcp` can start fizzy when none is running.
- Whether chat should advertise `terminal/*` so agents can run commands in a fizzy pane.
