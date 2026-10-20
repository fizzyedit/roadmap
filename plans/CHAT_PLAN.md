# Chat: conversations for any plugin, and a chat worth reading

Status: in progress, 2026-10-13. Step 1: fizzyedit/fizzy#389. Follows the first real Claude run (AGENTS_PLAN, "The chat plugin").

The first real conversation with Claude inside fizzy worked: the agent took fizzy's tools, read the
screen, opened the explorer when asked. It also showed what the chat is not yet:

- **Tangled.** The chat plugin *is* the ACP client, so every chat is a chat with an agent. A plugin
  that wants a conversation with anything else (a collaborator, a service, its own help) has no way
  to open one, and if it did, its chats would sit in the same list, in the same folder, as the
  agent's.
- **Hard to read.** What was sent and what came back look alike. Every `fizzy_screen` result is
  dumped whole into the transcript. The adapter's own `ToolSearch` steps show as tool calls, named
  `mcp__fizzy__fizzy_screen`. A permission request is a card of its own after the call it is
  about. Messages are plain text: no markdown, code, images or links. Opening the sidebar threw
  the conversation back to its top.

## Principles

- **Chat is to conversations what the workbench is to files.** It owns the tabs, the transcripts,
  the list and how a conversation is drawn, and knows nothing of who is on the other end. Its
  service holds only what it is the authority on: conversations.
- **A conversation belongs to one client.** The plugin that opened it is the only one called for
  it, and its chats are stored, listed and named apart from every other plugin's.
- **Rich content is ordinary.** Markdown, code with its highlighting, images, links to files, diffs:
  any client can post them, and they draw the way fizzy draws them everywhere else.
- **Nothing costs a frame it does not need.** A long chat draws only what is on screen, with each
  message's layout kept until its width or its content changes. Streaming text costs the chunk, not
  the transcript.

## The contract: commands, not the SDK (decided 2026-10-13)

The chat's contract with its clients is the chat plugin's, not fizzy's: most apps will not ship
the chat, and a contract in the SDK would make every change to it an SDK release and a repin of
every plugin. So it is commands, ZON in and out, `visibility = .internal` (`Command.visibility`,
fizzyedit/fizzy#389: called by id, never listed for a person). Bytes cross the boundary, and each
side keeps its own types (the agents plan's rule for command arguments).

**What a client calls** (the chat's commands):

| Command | Arguments → result |
|---|---|
| `chat.register` | `{ client, title, version }`: the client's chats come alive and get a group in the list. `version` is the contract version the client speaks. |
| `chat.unregister` | `{ client }`: its chats turn read-only, saying which plugin they need. |
| `chat.open` | `{ client, title }` → `{ chat }`: a new conversation in a tab. |
| `chat.post` | `{ chat, author, blocks }` → `{ message }`: `author` is `.other` or `.note`; the person's messages come only from their typing. |
| `chat.append_text` | `{ chat, message, kind, text }`: streams `.text` or `.thought`, extending the last block. |
| `chat.append` | `{ chat, message, block }` |
| `chat.tool` / `chat.update_tool` | `{ chat, message, kind, title, status, content }` → `{ tool }`; `{ chat, tool, title?, status?, content? }` |
| `chat.ask` | `{ chat, message, text, options, tool? }` → `{ question }` |
| `chat.set_plan`, `chat.set_busy`, `chat.set_title`, `chat.set_resume`, `chat.focus` | `{ chat, … }` |

**What the chat calls on a client** (by convention, `<client>.<name>`, internal; one the client
did not register is a capability it does not have):

| Command | Arguments |
|---|---|
| `<client>.chat_send` | `{ chat, text, images? }`: the person sent a message. |
| `<client>.chat_answer` | `{ chat, question, option }` |
| `<client>.chat_cancel` | `{ chat }`: stop the turn. |
| `<client>.chat_closed` | `{ chat }`: its tab closed, or the chat plugin is going away. |
| `<client>.chat_resume` | `{ chat, state }` → `{ resumed }`: a chat reopened, with what `set_resume` saved. |
| `<client>.chat_new` | `{}`: New from the client's group in the list. |

**Blocks** (ZON unions): `.{ .text = "markdown" }`, `.{ .thought = "…" }`,
`.{ .code = .{ .language, .text } }`, `.{ .image = .{ .path, .mime, .alt } }` (a file the chat
copies beside the transcript, so no image bytes cross as ZON), `.{ .file = .{ .path, .line,
.end_line, .title } }`, `.{ .diff = .{ .path, .old, .new } }`, `.{ .plan = .{ .{ .text, .status } } }`.

**Cost**: a ZON encode and parse per call, microseconds a streamed chunk at tens of calls a
second. The contract is checked when a call is made rather than when plugins build; the chat
answers a malformed call with why, and `chat.register` carries the version.

**Kept apart**: a chat's transcript lives in `<config>/chats/<client>/`, its header naming the
client; its images beside it by content hash. The list groups chats under their client's title.
Only the chat's own client is called for it.

**Other callers**: a chat with people, a plugin's own help, a scripted client that tests the UI
with no agent (`chat.sample` in Debug builds), and an agent itself, which can open a chat to show
the person what it did.

## The look

- **Bubbles.** The person's messages sit on the right in rounded bubbles in the accent tint, at most
  about three quarters of the width. The other side's sit on the left in rounded bubbles in the
  surface tint, their markdown drawn in full. Notes sit centered, small and quiet.
- **Markdown inline**, through the markdown service. It gains an inline render: no scroll area of
  its own, laid out at a given width, its height kept until the width or bytes change. The other
  consumer is hover docs, which today need their own scroll area.
- **Code blocks**: monospace, highlighted through `Host.treeSitterHighlightFor` (as markdown's
  fences are), with Copy, and Open in a new tab.
- **Images**: inline, fitted to the bubble; click opens them in a tab (whichever plugin owns the
  type). Pasting or dropping an image into the input attaches it.
- **Files**: a path is a link that opens it at its line.
- **Tool calls**: one compact line each, inside the turn that made them. An icon by kind (read,
  edit, run, search, fetch, think), a readable title (`mcp__fizzy__fizzy_screen` → "Fizzy screen"),
  and a status (working, done, failed). Output is collapsed behind a disclosure. Diffs stay open,
  since they are the point. A permission request is buttons on the call it is about (Allow, Always,
  Deny); once answered it becomes a small label on that line.
- **Thinking**: one quiet line, "Thought for 4 s", that opens to show the thoughts.
- **Plans**: a checklist card, updated in place as the plan's entries change.
- **While a turn runs**: a typing indicator in a bubble on the left, and Stop where Send was.
- **Scrolling** is anchored to a message, not a pixel offset. At the bottom it follows new content;
  scrolled up, it keeps the message being read in place through reflows (the sidebar opening, the
  window resizing, a tool's output expanding).

What a client shows is its choice: chat draws what it is given. The agent plugin is where
`ToolSearch` gets left out and tool names made readable.

## Performance

- The transcript is a virtualized list. Each message keeps its laid-out height, keyed by width and
  a content version; only messages in view (plus a screen above and below) are laid out. Text is
  drawn directly (`renderText` and clip, as `core.viz.Table` does), not as a widget per line.
- Streaming bumps one message's version. A frame of a turn re-lays only that message.
- Budget, measured with the profiler: a 1,000-message chat under 1 ms a frame while scrolling, and
  under 0.3 ms idle.

## The agent plugin's side

The ACP client moves out of chat into the agent plugin, beside the MCP server it already runs. It
registers as the client `agent`, and opens a chat per conversation. It maps ACP to the service:
message chunks to `append`, tool calls to `tool`, permission requests to `ask`, `fs/*` through open
documents as today, images and resources to blocks. It drops the adapter's internal steps and makes
tool names readable. Agent profiles (several command lines, one picked per conversation) live
here. And since Claude's adapter advertises `mcpCapabilities.http`, the agent plugin can serve
fizzy's tools over localhost HTTP, so ACP agents no longer need `fizzy-mcp` beside it (which the
store cannot ship); `fizzy-mcp` stays for agents in a terminal.

## Without the chat: Claude in a terminal

The chat is optional (it `works_with` the agent plugin; `plans/TOOLS_AND_TRUST_PLAN.md`). With
only the agent plugin, Claude Code run in a terminal inside fizzy (ghostty) is the same agent:
the same program, model, sign-in and history, in Claude Code's own interface. What it lacks is
fizzy's tools, and what keeps it safe around open documents:

- **Connect Claude Code to fizzy**: the agent plugin offers, once and with the person's go-ahead,
  `claude mcp add --scope user fizzy <fizzy-mcp>`. Any Claude Code then drives the running
  fizzy: in ghostty, in another terminal, in VS Code. Shown in the agent plugin's settings with a
  Disconnect.
- **Server instructions**: `fizzy-mcp` sends MCP `instructions` on `initialize`: for a file open
  in fizzy, read and write through `fizzy_call` with `text.read`/`text.write`, so unsaved edits
  are read and an edit is one undoable change rather than a write to disk under an open buffer.
- **Named seam, not built**: environment for terminals started inside fizzy, contributed by
  plugins (the agent's: which fizzy `fizzy-mcp` talks to) and applied by the terminal plugin.
  Built when a second consumer needs it; the connect step needs none.

What the chat still adds: edits through open documents by default, permissions and diffs on the
tool call, conversations as tabs restored with the session, markdown, code, images and file links,
and other plugins taking part through the service.

## Getting the agent: nothing to install by hand

The first run needed Node (`npx` fetches `claude-agent-acp`, a Node package, which drives Claude
Code underneath) and a `claude /login` in a terminal. A person should need neither.

What VS Code does (seen on this machine, Cursor's Claude Code extension 2.1.295): the extension is
published per platform (`anthropic.claude-code-…-darwin-arm64`) and carries Claude Code's native
binary inside it (`resources/native-binary/claude`, a few hundred MB). It runs that binary with
`--input-format stream-json --output-format stream-json --permission-prompt-tool=stdio` and talks
to it directly: no Node, no ACP. Sign-in is the binary's own (`claude auth login` / `status`),
surfaced in the extension's UI.

The agent plugin does the same, per agent:

- **Claude**: Claude Code's native build, as a tool that installs and updates itself
  (`plans/TOOLS_AND_TRUST_PLAN.md`, "Tools that install and update themselves"). One already
  installed is used as it is. Otherwise, with the person's go-ahead, fizzy bootstraps it the way
  Anthropic's install script does (a verified download, then `claude install stable`), and runs
  `claude update` on its own schedule, never during a conversation. The plugin speaks its
  stream-json protocol directly (what the Agent SDK does). Sign-in is a "Sign in to Claude" button
  in the chat that runs the program's own login and its browser step; `claude auth status` says
  when it is done. Not bundled: too big to carry, and it updates on its own schedule.
- **ACP agents** stay the general route: Codex (native `codex-acp`), Gemini CLI, Copilot, Goose,
  and others, each with how to get it. Native ones are downloaded the same way; Node ones are
  offered when Node is there (a managed Node runtime is the named seam if that proves common).
- The protocol under Claude is the Agent SDK's, less of a public contract than ACP, so the plugin
  pins the binary version it speaks to and moves both together.

## Also found: what the agent could not see

- **Selected state.** The snapshot does not say which rail tab, tab or option is selected. dvui's
  frame dump has roles and names but no state; adding `selected`/`checked`/`expanded` is a patch on
  the fizzy-dev dvui stack (as the names were), read by `replay.Snapshot`. A screen reader needs it
  too.
- **Settled after motion.** `fizzy_act` reported while the sidebar was still opening (39 px wide).
  An act's snapshot should wait until animations started by its own input have finished
  (`dvui.animationsRunning`, or fizzy's motion clock idle), bounded.

## Steps

Each a PR, useful without the next.

1. **SDK**: the markdown service's inline render (v2) and `Command.visibility`
   (fizzyedit/fizzy#389). The chat service itself left the SDK for commands (above).
2. **Chat, the surface**: the commands implemented, transcripts per client, the list grouped by
   client, the new model drawn: bubbles, markdown, code, images, files, tool lines, inline
   permissions, thinking, plans, the anchored virtualized scroll. A scripted test client and a demo
   tape. The ACP code stays until step 3 so the chat keeps working.
3. **Agent, the client**: ACP moved in, registered as `agent`, `agent.newChat`, readable tools,
   internal steps left out; the chat's ACP code removed.
4. **Claude without Node**: Claude Code as a `self` tool (`plans/TOOLS_AND_TRUST_PLAN.md`:
   detected, or bootstrapped with the person's go-ahead, updated by its own updater on fizzy's
   schedule), its stream-json protocol spoken directly, sign-in from the chat. Needs that plan's
   steps 1 and 3.
5. **MCP over HTTP** from the agent plugin.
   With it: **Connect Claude Code to fizzy** and the server's instructions (above), so a terminal
   Claude works without the chat.
6. **Snapshot state and settling** (fizzy, independent): selected/checked/expanded in the frame
   dump and snapshot; `fizzy_act` waits for its animations.
7. **Terminals and profiles**: `terminal/*` through the terminal plugin; agent profiles;
   `session/load` through `resume`.
