# Multiplayer: people and agents in the same documents at once

Status: proposed, 2026-10-19 (roadmap#9). Builds on `SHARED_HISTORY_PLAN.md` (#14),
`RECORDING_PLAN.md` (tracks), `AGENTS_ANYWHERE_PLAN.md` (#31, the relay and its rooms),
`AGENT_VIEW_PLAN.md` (#18) and `CONTROL_SERVER_PLAN.md`. Amends `SHARED_HISTORY_PLAN.md` on
undo, for documents in a room only (see "Amendments").

Several hands work in one document at the same time: two people on two machines, a person and
an agent beside them, or several of each. Each has their own view, their own cursor and their own
undo. Each sees who else is there and what they are doing. None of them overwrites another's work
by accident. This plan picks the model that keeps their copies the same, says what each document
owner must add so it can join, and gives the steps in order, starting from a slice that needs no
network.

## What there is

Fizzy already has most of the pieces of a replicated document. It has no way to merge two
concurrent changes.

- **Owners report every step as a forward effect.** `sdk/src/Step.zig:1-8`: a step is what the
  document became, in the owner's encoding, applied forward. An undo is reported as its reverse
  (`undone`), never run backwards. `Host.recordStep` (`sdk/src/Host.zig:1908`) carries each step.
  `Plugin.VTable.applyStep` (`sdk/src/Plugin.zig:271-276`) applies one, and refuses a step that does
  not fit. `step_format` (`Plugin.zig:39`) numbers the encoding. `documentFingerprint`
  (`Plugin.zig:258`) hashes what decides the next step. **This is the replication log already**,
  for one writer.
- **Tracks are snapshot plus log.** `app/Tracks.zig:1-40` keeps a base state, the steps, and
  checkpoints as deltas from a whole state. `replayInto` (`Tracks.zig:719`) rebuilds any moment.
  `pickUp` (`Tracks.zig:455`) puts a document's history back with `restoreHistory`
  (`Plugin.zig:283`). Late join needs exactly this.
- **Authors.** `Author` (`sdk/src/Author.zig`) is `{kind, plugin, session, label}`, where kind is
  `person`, `agent`, `tape` or `unattributed` (`Author.zig:15`). `Host.act`/`endAct`
  (`Host.zig:1942`) scope who is acting. The host keeps a step's author by step id, and only for
  steps that are not the person's (`noteAuthor`, `Host.zig:2054`; `theirs = who.kind != .person`
  at :2057). `mayMerge` (`Host.zig:2006`) stops an agent's typing from joining the person's run.
  There is exactly one `person`: the one at this machine's keyboard.
- **Every MCP client is its own author.** fizzyedit/fizzy#638: `initialize` opens a session, and
  each `tools/call` runs inside `host.act(.{ .kind = .agent, .plugin = "control", .session = id })`
  (`app/control/mcp.zig:33`, :209). #650 (open) makes a call that answers later keep its caller's
  author (`actAgain`).
- **Commands are the API.** `docs/CONTROL.md`: `POST /commands/{id}` and MCP `tools/call`. They
  are **intents, not effects**. They read the active document, the selection, the active layer and
  settings, so running a command on two machines does not give the same result. Steps are what
  replicate. Commands are what a person or an agent asks for.
- **Agents edit against what they last read, and nothing checks it.**
  - `text.write` (`plugins/text/plugin.zig:659`) takes the document's whole new text and diffs it
    against the current text by common prefix and suffix. An agent that read the file, then waited
    while the person typed on line 5, and then writes a change to line 50 takes back the person's
    line 5. The diff spans lines 5 to 50.
  - `text.insert` (`plugin.zig:578`) takes a line and column that may have moved since the read.
  - `text.replace` (`plugin.zig:599`) finds text and is safe: it refuses an ambiguous match.
  - Chat's ACP `fs/write_text_file` goes through `text.write`
    (`~/dev/fizzyedit/chat/src/AcpClient.zig:872-880`), so every ACP agent's file write has this
    problem today, with no network involved.
- **Text's steps are positional.** `textcore/Transaction.zig:1-40`: each edit is
  `{pos, removed, inserted}`, positions sequential within the transaction. `applyStepTo` checks
  `removed` against the text and fails with `StepDoesNotApply` (`Document.zig:636-660`). The buffer
  is a flat `std.ArrayList(u8)`. History is an undo stack (`textcore/History.zig`).
- **Pixi does not yet report steps** (`SHARED_HISTORY_PLAN.md` step 3, pixi half, open).
  - Its changes (`~/dev/fizzyedit/pixi/src/internal/History.zig:32-165`) name pixels by layer id
    (`Pixels.layer_id`), but layers, animations and reorders by **index** (`LayerName.index`,
    `AnimationFrames.index`, and so on).
  - Ids come from per-file counters (`File.zig:1544`, `layer_id_counter += 1`), so two machines
    would mint the same id.
  - It already waits out the person's stroke before anyone else's edit runs (`personBusy`,
    `afterPerson`, `pixi/src/commands.zig:84-104`). That is a local form of "don't interleave inside
    a gesture".
- **Claims** (`sdk/src/Claim.zig`; `Host.claimDocument`, `Host.zig:1812`) hold a whole document
  for one plugin. The person's input to it waits. Agents no longer claim by default.
- **Rooms are planned, not built.** `AGENTS_ANYWHERE_PLAN.md` steps 8-11 (repo `fizzyedit/relay`): an edge-hosted relay
  (Cloudflare Workers, one Durable Object per room, first at a free `workers.dev` address and later optionally `relay.fizzyed.it`, plus a self-host Zig
  binary of the same protocol) that copies sealed ACP and MCP frames between a browser tab and the
  person's machine. It is end-to-end encrypted from the start and holds two members per room. Fizzy has no WebSocket client today. The web build is deployed with Wrangler
  (`web/wrangler.toml`), so the relay can sit on the same platform.
- **A remote view is planned** (`EMBEDDED_EDITOR_PLAN.md`). A host shows fizzy's views over the
  control socket, by input in and frames or draw lists out (`sdk/remote/`). That is screen-sharing
  co-editing, Live Share-style. It is considered below.

## How interleaved edits go wrong

These are the failures the model must rule out. Each names the fizzy case it would hit.

1. **Lost update.** Two writers each read version N and write a whole value back, and the second
   write erases the first. This is `text.write` from a stale read (above), and any "set the layer
   name" racing with another.
2. **Shifted positions.** An edit is addressed by position, line number or index, and an earlier
   concurrent edit moved what it pointed at. This is text's `pos`, pixi's `LayerName.index` and
   `text.insert`'s line. Applied blindly, the edit lands in the wrong place. Checked, it fails
   (`StepDoesNotApply`).
3. **Interleaving.** Two people type at the same spot. A merge that orders concurrent characters
   one at a time produces "ebrgegasd" from "eggs" and "bread". Logoot and LSEQ do this, and RGA and
   YATA do it in some backward-typing cases. FugueMax provably does not (Kleppmann et al., PaPoC
   2019; Weidner and Kleppmann, *The Art of the Fugue*, TPDS 2025, Table I).
4. **Divergence.** Each replica applies the same steps in a different order, or applies a step
   that is not deterministic (float rounding, a setting, the plugin version). The copies drift
   apart silently.
5. **Intent violations.**
   - Someone deletes a layer while someone else draws on it.
   - Someone resizes the canvas under a stroke.
   - An agent's edit was correct for the text it read but is wrong for the text now there.
   Convergence alone does not fix these. Each needs a rule: tombstones, claims, or a review step.
6. **Undo that hurts someone else.** Linear undo of a shared history takes back another person's
   last step (Loro's undo docs give this as the reason undo must be local).

## Choosing the model

| | How it works | For fizzy |
| --- | --- | --- |
| **CRDT as the document** (Yjs/Yrs, Automerge, Loro) | Every replica holds the CRDT and merges without a server. | Each owner would keep a second copy of its document in the CRDT, beside its own buffer and history. That breaks "a document has one history, its owner's" (`SHARED_HISTORY_PLAN.md`, principle 1). It brings a Rust library into every build, wasm included, through `yffi`/`automerge-c` (no Zig CRDT exists). It does nothing for pixels. **No.** |
| **OT through a transforming server** (Google Docs, Jupiter) | The server transforms each op against those it has seen. | The relay would have to understand every owner's step format. Plugins are third-party and their bytes are opaque. **No.** |
| **State sync, last writer wins per property** (Figma) | The server orders changes, and each property's latest value wins. | Right for pixels and object properties. Wrong for text, where Figma itself keeps text whole. **Yes for structured and binary documents.** |
| **Ordered log, clients rebase** (Drawpile, tldraw sync, Replicache, Linear) | A sequencer only orders, and every client rebases its own pending changes onto the order. | The relay stays dumb and owner-agnostic. Owners keep their own history and apply steps forward, which they already do. The rebase is each owner's own function. **Yes, as the frame for everything.** |
| **Replicated computation** (Croquet) | Every peer runs the same deterministic program on ordered input. | Fizzy's commands and input are not deterministic across machines (active document, settings, frame timing, plugin versions). **No.** |
| **Remote view** (Live Share, `EMBEDDED_EDITOR_PLAN.md`) | One fizzy owns everything, and the others are views of it. | It needs zero owner work and works for every document. But every keystroke and frame crosses the network: tier-3 draw lists are about frame-sized per frame, and pointer latency is a round trip. **A fallback for owners that cannot merge**, not the main path (see "Every owner can join"). |

### Decision: a room-ordered step log, with each owner rebasing its own steps

- **The relay is a sequencer.** It numbers each step it receives (`seq`), keeps the log since the
  last snapshot, and sends every step to every member, the sender included. It never reads a step's
  effect. Because it doesn't, the effects could be end-to-end encrypted later without changing the
  model (see "Open").
- **A replica applies its own steps at once** (optimistic), and keeps them as *pending* until the
  relay sends them back numbered. A step from someone else that is numbered before a pending one is
  applied **under** the pending ones: rebase the remote step over them, or take them back, apply it,
  and put them back. This is Drawpile's "retcon" and tldraw's push/pull/rebase.
- **The owner rebases.** A new optional vtable entry, `rebaseStep`, transforms an effect over the
  effects numbered before it that it had not seen. It is pure and deterministic. Every replica
  computes the same result from the same log, so the copies converge without the relay knowing any
  format.
- **Each step names its base**, the last `seq` its author had applied. A slow author is just a
  replica with an old base. **An agent is a high-latency peer.** That is the whole of the "agent
  edits a stale view" answer, and it works on one machine with no room at all (step 1).

### Text: Fugue ordering, Eg-walker style, in textcore

- Text keeps its flat buffer and its own history. **No CRDT is held in steady state.**
- When a step must be rebased over concurrent ones, textcore replays just that window through a
  temporary sequence CRDT with FugueMax ordering, reads the transformed positions off it, and drops
  it. That is Eg-walker's design (Gentle and Kleppmann, EuroSys 2025): plain text in memory, fast
  load, a CRDT only during a merge, and a cost bounded by how much was concurrent.
- With a sequencer, the event graph is almost linear. It branches only where steps were
  concurrent, so a window is usually a few steps.
- Positions in the effect stay byte offsets. Inserts are whole UTF-8 runs, and the temporary CRDT
  orders runs, not characters.
- Ties between concurrent inserts at one spot follow Fugue: by left origin, then right origin, then
  the peer, so two people typing at one spot never interleave.
- Written in Zig in `textcore`, a few hundred lines plus a fuzzer. Not `yffi` or diamond-types:
  either would put Rust in the plugin build and keep a second copy of the text.

### Structured and binary documents (pixi): per-element last writer wins by sequence, with ids, tombstones and claims

- **Pixels: the later `seq` wins, per pixel.** A `Pixels` change is `{layer_id, indices, values}`.
  Two strokes that overlap keep both, and the later-numbered one is on top where they cross. The
  pixels are never merged. `rebaseStep` for pixels is the identity, and only the order matters.
  Optimistic display stays right: your pending stroke is numbered after anything already numbered,
  so it stays on top, and a remote step numbered earlier is painted underneath it.
- **Everything is named by id, never by index.**
  - Ids are minted per replica as `(peer, counter)`, as Figma embeds the client id in object ids.
  - Order (layers, animations, frames) is a fractional position per element, last writer wins, as
    Figma does.
  - A property (name, visibility, opacity, an animation's fps) is last writer wins by `seq`.
- **Deletes are tombstones.** A deleted layer stays in the file, hidden, until no pending step
  names it. A stroke on a layer someone deleted lands on the tombstone: nothing is lost, and undoing
  the delete brings it back with the stroke.
- **Whole-document structure needs a claim.** Resize, grid layout and cell reorders change what
  every index means. The owner refuses them in a room unless the actor holds a claim. A claim in a
  room is itself a numbered entry in the log, so every replica agrees who holds it. The person sees
  "Ana is resizing" and their strokes wait (`afterPerson`, as for a local gesture).
- Smaller claims (one layer, one animation) are an owner-defined `scope` on a claim. They are
  later, and only if last writer wins proves too coarse.

### Every owner can join, at the level it supports

| The owner has | In a room it is | |
| --- | --- | --- |
| `applyStep` + snapshot + `rebaseStep` | **co-edited**: everyone writes at once | text, pixi after steps 8-9 |
| `applyStep` + snapshot, no `rebaseStep` | **one writer at a time**: anyone may write, and the first local step takes a room claim, released when the actor goes idle. Others watch it live. | any owner that records steps |
| neither | **read-only** for guests, who see the host's saved bytes and reload on save | image, archive, chat |

No owner is special-cased. The level is read from which vtable entries are set. A remote view
(`EMBEDDED_EDITOR_PLAN.md`) is the later answer for the third row, once it exists.

## Undo with more than one person

- **In a room, Cmd-Z is local.** It takes back the last step made by *you or your own agents*,
  never another person's. That is what Yjs (`UndoManager.trackedOrigins`), Loro, Zed and Figma do,
  and Drawpile per user.
  - "Your own agents" keeps `SHARED_HISTORY_PLAN.md`'s principle 3, "an agent works for the person":
    agents your fizzy runs (author `agent` with your peer) are yours to take back.
- **Out-of-order undo needs a new owner entry, `takeBack(doc, id) -> effect`.** It gives a forward
  step that reverses entry `id` as the document stands now, and is numbered and rebased like any
  other step.
  - Text: the inverse of the entry's edits, rebased over everything after it. A deleted span that
    someone has since edited inside is restored around their edit, never over it.
  - Pixi: put back the old value only on pixels that still hold the value this step wrote. This is
    Figma's restore-the-property and Stewen and Kleppmann's restore operations (PaPoC 2024).
  - An owner without `takeBack` undoes only while your step is on top, and otherwise says "Ana has
    changed it since: fix it forward". That is today's `expect` refusal.
- **Redo** is `takeBack` of the take-back.
- **Alone, nothing changes.** No room means linear undo as `SHARED_HISTORY_PLAN.md` decided, with
  the label naming whose step it takes.

## Presence

- **Ephemeral, not in the log.** Presence travels as last-writer-wins state per member, as Yjs's
  awareness protocol does. The relay keeps the latest state per member and drops a member silent
  for 30 s.
- **What each member shares:** a name and colour, the document they are in, their selection or
  caret, their viewport in document coordinates, and the pointer while it is over the document.
  It is sent at most 20 times a second, and only when it changed.
- **The owner encodes and draws it.** Only the owner knows what a caret or a selected cell is.
  - `Host.setPresence(doc_id, bytes)`: the owner calls it when the local selection or view changes.
    It returns at once when there is no room.
  - `Host.peersOn(doc_id)`: the members on this document, with their author, colour, presence
    bytes and the `seq` it was taken at. The owner draws them in its own surface.
  - Text maps a caret through the steps since its `seq` (the same transform).
- **Agents have presence too.** An agent's presence is the region it is looking at
  (`AGENT_VIEW_PLAN.md`'s own camera, the `agent.frame` region) and "working on Draw". Its steps
  carry its name in the bubbles already (`SHARED_HISTORY_PLAN.md`).
- **Follow** (your view tracks another member's viewport) is a later step, on the same data.
- **Cost:** with no room, `setPresence` is one null check, and nothing is drawn for peers that are
  not there.

## Permissions and claims

- **A room has roles:** the host (whose files they are), editors, and viewers.
  - Viewers' steps are refused by the relay, and their fizzy greys out editing.
  - A role is per room. Per-document read-only is a flag on the room's document list.
- **An agent acts under its person's role.** A remote agent connects with its person's token and
  is an `agent` author with that peer. The person who started it can revoke it, as
  `AGENTS_ANYWHERE_PLAN.md` gives Revoke.
- **Claims become log entries in a room** (`claim`/`release`, numbered), so every member agrees on
  the holder. A claim ends when its holder leaves or goes idle (a room setting, 60 s default). The
  host can break any claim.
- **Moderation**, after Drawpile: the host can make anyone a viewer, kick them, or take the
  document back to before a member's steps. That last one is "Go back to before Claude's change",
  `SHARED_HISTORY_PLAN.md` step 6, by author.

## Joining late, and dropping off

- **Join: a snapshot, then the log.**
  - The room keeps, per document, the newest snapshot (`documentSnapshot` plus `encodeSnapshot`,
    the bytes a save writes) with the `seq` it stands at, and the steps after it.
  - A joiner opens the snapshot, applies the steps, then goes live. This is a track's base plus its
    steps (`Tracks.zig`).
  - The host uploads a new snapshot once the steps since the last weigh as much as the snapshot,
    which is the checkpoint rule from `RECORDING_PLAN.md`. The relay then drops the log before it.
- **History for the joiner:** they get the room's steps since the snapshot as their history
  (`restoreHistory`, attributed by author). Undo reaches back only to their own steps.
- **A short drop** (network, sleep) is the normal path. Pending steps wait, the client resends
  them with their base on reconnect, and they are rebased. The relay deduplicates by
  `(peer, local id)`.
- **A long offline stretch** (an afternoon on a train) is not the same thing. Rebasing hours of
  edits risks a confusing result. Past a limit (a setting, default 10 minutes or 500 steps), the
  offline work becomes a **branch** to review (see "Agents"), not an automatic merge.
- **Divergence check.** Every 64 `seq`, each member sends its `documentFingerprint`. A mismatch
  (a non-deterministic `applyStep`, a bug) makes that member re-join from the host's snapshot and
  logs both fingerprints for a bug report. The person sees "Resynced with Ana", not silently
  different pictures.

## Transport

- **A WebSocket to the relay, on every platform.** The relay is the room server from
  `AGENTS_ANYWHERE_PLAN.md` steps 8a-8b (a Durable Object per room, TypeScript, hosted at
  `relay.fizzyed.it`, and also `relay-selfhost`, step 10c there). This plan adds the sequencer,
  snapshot store and presence to it, to both implementations, behind the one conformance suite.
  - **Sealed, with a plain header.** The room key from pairing (invites carry the code in the
    link's fragment) seals every payload: steps' effects, snapshots, presence, claims. The relay
    reads only a small plain header per frame, `{type, peer, local_id, doc, base?}` (`doc` an opaque
    id, not a path), which is what numbering, dedupe, roles and rate limits need. So the relay
    stays owner-agnostic and cannot read an effect. It does see who sends how much and when.
    Snapshots it stores are ciphertext; the grace-period log is ciphertext.
  - **Members:** the room's `max_members` (2 for agent rooms) is raised per room; the host's
    invite is the only way in, and Revoke drops a member and their resume token.
  - **Roles are the relay's to enforce, not the key's:** every member holds the room key, so a
    viewer's steps are refused by the relay by token role, and a member's fizzy also ignores steps
    from a viewer. The key protects against the relay and the network, not against a guest.
  - The web has `WebSocket` in JS (`web/web.js` glue).
  - Native needs a small client: `std.http` upgrade plus framing, about 300 lines in
    `app/room/`.
- **WebRTC data channels, considered and not now.** Peer to peer still needs a signalling server.
  It needs TURN for the roughly 10-25% of networks that block it (vendor figures). It needs
  libwebrtc or a DTLS/SCTP stack in Zig on native. And peer to peer has no sequencer, so it would
  need a CRDT after all. The Yjs maintainers' own advice is a WebSocket relay over running TURN.
  It is a later option for the presence channel only, if latency demands it.
- **Same machine, no internet.** For development and the first slice, one fizzy is the sequencer
  over its control socket (`GET /room`, `Upgrade: fizzy-room/1`, the same pattern
  `EMBEDDED_EDITOR_PLAN.md` uses for `/remote`), and a second fizzy (`--profile <dir>`) joins it.
  It uses the same wire protocol and client as the relay. The relay is then just a remote
  sequencer.
- **The wire is a std-only library, `sdk/room/`**, as `sdk/tape/` and `sdk/remote/` are. It
  defines hello, join, snapshot, step, ack, presence, claim, fingerprint and leave messages. The
  relay (TypeScript on Workers) and the self-host binary (Zig) get a generated test vector set and
  run the relay repo's conformance suite, so the implementations cannot drift. Sealing is the
  relay repo's `relay_wire` Zig library, which `sdk/room/` builds on.

## Agents

- **Stale views: an edit carries the version it was made against.**
  - `text.read` answers `version` (the top step id, or the room `seq`). `text.write`,
    `text.insert` and `pixi.draw` take an optional `base`.
  - The owner rebuilds the text at `base` from its history, diffs the agent's whole new text
    against *that* (not against now), and rebases the diff over the steps since. The person's line
    5 survives the agent's line 50.
  - A write with no `base` behaves as today, but the answer warns when the document changed since
    the caller's last read (the control server knows the session's last read).
  - Chat passes `base` for ACP's `fs/write_text_file` from its last `fs/read_text_file` of that
    path.
- **Intent commands beat positional ones.** `text.replace` (find and replace, refusing ambiguity)
  is already robust to staleness. Tool descriptions steer agents to it, and the agent guide says
  so.
- **Rate limits.**
  - An agent's MCP call is one run (`Host.act`). Its steps are **squashed into one step per run**
    before they are sent to the room, so a 200-stroke `pixi.draw` script is one message.
  - The relay holds a token bucket per author (default 20 steps/s, burst 100). Over it, the steps
    queue on the agent's machine and the agent's call answers `.waiting`.
  - A person typing is never throttled.
- **Attribution across machines.**
  - `Author` gains a `peer`: the room's member number, 0 for this machine.
  - A remote person is `{kind = .peer}`. A remote agent is `{kind = .agent, peer = n}`.
  - The room keeps the author table, so every member labels a step the same.
  - Tracks record remote steps with their authors, so a session's tape shows every hand.
- **Review before it lands: suggestion branches.**
  - A **branch** is an author's steps held apart from the shared document, with their base. They
    are shown, not applied: text shows them as inline diff hunks, and pixi as a preview layer over
    the canvas, tinted with the agent's colour.
  - Accepting all or some (per hunk, per stroke run) rebases them and sends them as steps,
    attributed to the agent and accepted by the person. Rejecting drops them.
  - This is Patchwork's model (Ink & Switch) and Zed's keep/reject hunks.
  - A person turns it on per agent or per room ("agents suggest"). The default is off on your own
    machine (agents work beside you, as today) and on for remote agents.
  - The branch is host-kept steps plus an owner hook to draw them (`drawBranch`). An owner without
    it shows "3 suggested changes" and a diff of snapshots.
  - Several agents' branches side by side is `AGENTS_ANYWHERE_PLAN.md` section 7's "a branch on its
    own copy", built here.

## What the SDK adds

All of this goes in one PR (step 2), so the fingerprint moves once, then one SDK release. Every
entry is optional: an owner that sets none behaves as today and joins read-only.

| Addition | Where | Fingerprint |
| --- | --- | --- |
| `Plugin.VTable.rebaseStep(state, doc, step: Sequenced, over: []const Sequenced, arena) ![]const u8`, where `Sequenced = {effect, seq, base, peer}`. Pure: the same inputs give the same bytes. | `Plugin.zig` | moves |
| `Plugin.VTable.takeBack(state, doc, id, arena) ![]const u8`: a forward effect reversing entry `id` as the document stands | `Plugin.zig` | moves |
| `Plugin.VTable.drawBranch(state, doc, steps: []const []const u8, rect)` | `Plugin.zig` | moves |
| `Author.peer: u32 = 0` and `Author.Kind.peer`. `noteAuthor` keeps every author but the local person (`Host.zig:2057` becomes `!who.same(.person)`). | `Author.zig`, `Host.zig` | moves |
| `Host.Room` (`{ctx, vtable}` like `Recorder`): `wants(doc_id)`, `step(doc_id, Step)` for local steps, and the app's `applyRemote`. Applying a remote step runs `applyStep` inside `act(peer)`, and steps made under a remote author are not sent back. | `Host.zig` | moves |
| `Host.setPresence(doc_id, bytes)`, `Host.peersOn(doc_id)` | `Host.zig` | moves |
| `Claim.scope: []const u8 = ""` (owner-defined, empty for the whole document) | `Claim.zig` | moves; may come later, in its own `sdk` PR |
| `base` arguments on owner commands | each owner | no (commands) |

- **Cost alone:** `recordStep` gains a null check for `room`. `setPresence` returns at once. No
  allocation happens without a room. That meets "nothing optional costs a frame".
- **`step_format` becomes a hard gate.** A member whose owner reads a different format joins that
  document read-only and is told to update the plugin. A member missing the owner plugin is
  offered it from the store (`docs/PLUGINS.md` §6) before the document opens.

## What is shared, and what stays yours

| Shared in the room | Yours |
| --- | --- |
| the documents' contents and their step log | layout, panes, tabs, floats |
| who is here, and their presence | your selection, caret, viewport and tool |
| claims and roles | your undo stack (your steps and your agents') |
| the room's document list (the host's files) | settings, theme, keymap |

**The room's files are a mount.** Guests see the host's documents as a folder, `room://<name>/`,
through the workbench `files` service, as `archive` mounts a zip and `drive` mounts Google Drive.
Saving writes on the host's disk. A guest's "Save As" makes a local copy that leaves the room.

## Amendments

- **`SHARED_HISTORY_PLAN.md`:**
  - In a room, Cmd-Z is local (yours and your agents'), through `takeBack`. Alone, it stays linear.
  - Selective undo, which that plan left out, is built here as `takeBack`. Pixi's changes name
    layers and animations by id first (step 8), which that plan named as the prerequisite.
  - `Author` gains `peer`.
- **`AGENTS_ANYWHERE_PLAN.md`:** the relay's rooms (its steps 8-10c, repo `fizzyedit/relay`) gain
  the sequencer, snapshot store, presence and per-author rate limits (steps 6-7 here), in the
  Worker and in `relay-selfhost`. Section 7's "a branch on its own copy" is this
  plan's suggestion branches (step 11).
- **`AGENT_VIEW_PLAN.md`:** an agent's look region is its presence.

## Steps (each its own PR, at most 800 lines)

"Cloud" means it can run in a Linux cloud session (tests only). "macOS" means it needs a Mac and
someone looking at it.

The order delivers value early:

- Step 1 fixes agents clobbering the person's text, on one machine.
- Steps 2-4 give two fizzy instances on one machine co-editing text, with carets.
- Step 5 makes undo local.
- Steps 6-7 put the room over the internet and the web.
- Steps 8-10 bring pixi in.
- Steps 11-12 are agent review and remote agents.

### 1. text: an edit made against an older version lands rebased

- Repo: fizzy. Branch: `text/edit-from-base`. Builds on: none.
- Cloud: yes, tests and a fuzzer.

```
Task: text.read answers `version` (top step id); text.write and text.insert take optional `base`. text.write rebuilds the text at `base` from History (undo on a copy), diffs the new text against that, and rebases the edits over the transactions since. Add textcore/Merge.zig: rebase a transaction over concurrent ones with Fugue ordering (Eg-walker style: a temporary list CRDT for the window only). No base keeps today's behaviour.
Plan: plans/MULTIPLAYER_PLAN.md, "Text" and "Agents"
Builds on: none
Workspace: ~/dev/fizzy-edit-from-base
Branch: text/edit-from-base
Notes: Fuzz test: random concurrent transactions converge in any delivery order and never interleave runs typed at one spot. Test the stale text.write case (person edits line 5, agent writes from an older read to line 50: both survive). Byte offsets, whole UTF-8 runs. Chat's fs/write_text_file passing base is a follow-up PR in chat.
```

### 2. sdk: the room seam (rebaseStep, takeBack, drawBranch, peer authors, presence)

- Repo: fizzy. Branch: `sdk/room-seam`. Builds on: 1.
- Cloud: yes. Labelled `sdk`. The fingerprint moves once, here.

```
Task: Add the SDK entries in MULTIPLAYER_PLAN's "What the SDK adds": Plugin.VTable.rebaseStep/takeBack/drawBranch, Author.peer + Kind.peer (noteAuthor keeps every author but the local person), Host.Room seam beside Recorder (local steps out, applyRemote in under act(peer), remote steps never resent), Host.setPresence/peersOn. Text implements rebaseStep over step 1's Merge, and takeBack. Record the new fingerprint; leave sdk_version alone.
Plan: plans/MULTIPLAYER_PLAN.md, "What the SDK adds"
Builds on: step 1
Workspace: ~/dev/fizzy-room-seam
Branch: sdk/room-seam
Notes: With no room: one null check in recordStep, setPresence returns, no allocation. Host tests: a remote step applied under a peer is attributed and not echoed; mayMerge never joins a peer's run. Claim.scope can wait for its own PR. Run all gates incl. test-sdk-version, check-web.
```

### 3. app: two fizzy instances on one machine co-edit a text document

- Repo: fizzy. Branch: `app/room-loopback`. Builds on: 2.
- Cloud: the integration test, yes. macOS for seeing it.
- May split into 3a (`sdk/room/` wire library) and 3b (`app/room/` client and sequencer).

```
Task: sdk/room/ (std-only wire: hello, join, snapshot, step, ack, presence, claim, fingerprint, leave) and app/room/: a sequencer on the control socket (GET /room, Upgrade fizzy-room/1) and a client. `fizzy.room.host` shares open documents; a second fizzy (--profile dir) runs `fizzy.room.join` with the socket path. Join = snapshot + steps; pending steps rebased under remote ones; fingerprint every 64 seq, resync on mismatch. Behind FIZZY_ROOMS=1.
Plan: plans/MULTIPLAYER_PLAN.md, "Decision", "Joining late", "Transport"
Builds on: step 2
Workspace: ~/dev/fizzy-room-loopback
Branch: app/room-loopback
Notes: Integration test: two Editors in one process over a loopback pair type at the same spot concurrently and converge. Owners without rebaseStep: single writer via a room claim. Guests' docs as a room:// mount through workbench files. Say what was seen on macOS with two sandboxes.
```

### 4. text + app: presence (who is here, their carets and selections)

- Repo: fizzy. Branch: `text/room-presence`. Builds on: 3.
- macOS (look). The mapping tests can run in the cloud.

```
Task: Text calls host.setPresence on caret/selection/viewport change and draws peersOn carets and selections in the member's colour with a name flag, mapping each through steps since its seq. App: a "who is here" row in the infobar (name, colour, document) and presence over the room wire, at most 20/s, dropped after 30 s silent.
Plan: plans/MULTIPLAYER_PLAN.md, "Presence"
Builds on: step 3
Workspace: ~/dev/fizzy-room-presence
Branch: text/room-presence
Notes: Nothing drawn or sent with no room (measure with the profiler). Agents' presence (look region) comes with AGENT_VIEW_PLAN's agent.frame. Follow mode is later. Screenshot of two sandboxes in the PR.
```

### 5. app: in a room, undo is your own

- Repo: fizzy. Branch: `app/room-local-undo`. Builds on: 2, 3.
- Cloud: yes.

```
Task: In a room, fizzy.undo/redo take back the last step by you or your own agents (author peer == 0) through the owner's takeBack, numbered and rebased like any step. Label "Undo your Typing"; another's step is never taken. Owners without takeBack undo only when your step is on top, else refuse "changed since: fix it forward". Alone: unchanged (linear, SHARED_HISTORY_PLAN).
Plan: plans/MULTIPLAYER_PLAN.md, "Undo with more than one person"
Builds on: steps 2, 3
Workspace: ~/dev/fizzy-room-undo
Branch: app/room-local-undo
Notes: Test: A types, B types inside A's run, A undoes: B's text stays. Redo is takeBack of the take-back. Amend SHARED_HISTORY_PLAN in the plans repo in the same session (separate PR).
```

### 6. relay: the room sequencer, snapshots and presence

- Repo: fizzyedit/relay. Branch: `relay-sequencer`. Builds on: `AGENTS_ANYWHERE_PLAN.md` steps 8a,
  8b and 9, and step 3's wire.
- Cloud: yes (workerd/Miniflare; no Cloudflare account). May split Worker / `relay-selfhost`.

```
Task: Add to the relay's room (Durable Object, then relay-selfhost): raise max_members per room; number steps (seq) from the plain header, echo to all incl. sender, dedupe by (peer, local_id), keep the newest sealed snapshot per doc and the sealed steps after it (drop older once a snapshot lands), presence fan-out (latest per member, drop after 30 s), roles (host/editor/viewer: viewers' steps refused), per-author token bucket (20/s, burst 100). Never opens a payload.
Plan: plans/MULTIPLAYER_PLAN.md, "Transport", "Permissions", "Joining late"
Builds on: AGENTS_ANYWHERE 8a, 8b, 9; fizzy sdk/room wire (step 3)
Workspace: ~/dev/fizzyedit/relay-sequencer
Branch: relay-sequencer
Notes: Test against sdk/room's vectors in the conformance suite, both servers. Storage cap per room (ciphertext). Log kept only while the room is open plus a grace period (privacy policy). Header fields are the only plain ones; add them to PROTOCOL.md.
```

### 7. fizzy: join a room over the relay, native and web

- Repo: fizzy. Branch: `app/room-relay`. Builds on: 3, 6, and AGENTS_ANYWHERE 11 (the room client and sealing).
- macOS and the web in a browser. Linux and Windows via CI build.

```
Task: The room client over a WebSocket: native (std.http upgrade + framing in app/room/) and web (JS WebSocket in web.js). Share a room: an invite link with a code; roles. Joining asks for missing owner plugins from the store; a step_format mismatch opens that document read-only. Reconnect resends pending steps with their base; past 10 min or 500 steps offline, the work is kept as a branch instead.
Plan: plans/MULTIPLAYER_PLAN.md, "Transport", "Joining late"
Builds on: steps 3, 6
Workspace: ~/dev/fizzy-room-relay
Branch: app/room-relay
Notes: Relay URL is a setting (shared with AGENTS_ANYWHERE; default wss://relay.fizzyed.it); reuse step 11 there for the socket and sealing. Keep frames running while a tab is in a room. Split native/web PRs if over 800 lines. Verify: macOS desktop + Firefox tab co-editing a text document.
```

### 8. pixi: everything named by id, ids minted per replica

- Repo: fizzyedit/pixi. Branch: `history/ids-not-indices`. Builds on: none. It can start now.
- Cloud: yes.

```
Task: History changes name layers, animations and frames by id, never index (LayerName, LayerSettings, AnimationFrames, AnimationName, ...: src/internal/History.zig). Ids become (peer, counter) (File.zig layer_id_counter/anim_id_counter), peer 0 alone. Order is a fractional position per element. Deleted layers and animations are tombstones until no history names them.
Plan: plans/MULTIPLAYER_PLAN.md, "Structured and binary documents"
Builds on: none (also SHARED_HISTORY_PLAN's selective-undo prerequisite)
Workspace: ~/dev/fizzyedit/pixi-ids
Branch: history/ids-not-indices
Notes: The file format keeps reading old .pixi files (index order -> positions). Tests: undo after a reorder hits the right layer. No behaviour change alone.
```

### 9. pixi: steps it reports, applies and rebases (last writer wins by seq)

- Repo: fizzyedit/pixi. Branch: `history/room-steps`. Builds on: 8, step 2's SDK release, and
  `SHARED_HISTORY_PLAN.md` step 3 (pixi's `recordStep` and `applyStep`) if that has not landed.
- Cloud: tests. macOS to see two sandboxes draw.

```
Task: Pixi's rebaseStep (pixels: identity, the later seq wins per pixel; properties: last writer wins; structure by id, a stroke on a tombstone lands there), takeBack (restore old values only where the pixel still holds this step's value), and resize/grid/cell reorders refused in a room without a claim. Presence: pointer, selection, viewport; draw members' pointers and selections.
Plan: plans/MULTIPLAYER_PLAN.md, "Structured and binary documents", "Undo"
Builds on: step 8; fizzy SDK release with step 2
Workspace: ~/dev/fizzyedit/pixi-room
Branch: history/room-steps
Notes: Pending local strokes stay on top; remote earlier-seq pixels paint under them. A 512x512, 8-layer sprite with two people drawing: rebase under 2 ms per frame (bench). Repin first.
```

### 10. sdk + app: scoped claims in a room

- Repo: fizzy. Branch: `sdk/claim-scope`. Builds on: 3. Labelled `sdk` (the fingerprint moves).
- Cloud: yes.

```
Task: Claim.scope (owner-defined bytes, empty = whole document). In a room, claim/release are numbered log entries so every member agrees on the holder; a claim ends when its holder leaves or is idle 60 s; the host can break one. Panes say "Ana is resizing" and the person's edits in that scope wait.
Plan: plans/MULTIPLAYER_PLAN.md, "Permissions and claims"
Builds on: step 3
Workspace: ~/dev/fizzy-claim-scope
Branch: sdk/claim-scope
Notes: Pixi scopes (a layer) are a pixi PR after this, only if per-pixel last writer wins proves too coarse. Ride the next SDK release with step 2 if both are ready.
```

### 11. app + text: an agent suggests, and the person accepts

- Repo: fizzy. Branch: `app/suggestion-branches`. Builds on: 2 (`drawBranch`), 5.
- macOS (look). Logic in the cloud.

```
Task: A branch: an author's steps held by the host with their base, shown not applied. Setting "agents suggest" (per agent, per room; on for remote agents). Text's drawBranch shows inline hunks with Keep/Reject per hunk; accepting rebases and sends them as the agent's steps, accepted by the person. Owners without drawBranch show a count and a snapshot diff.
Plan: plans/MULTIPLAYER_PLAN.md, "Agents"
Builds on: steps 2, 5
Workspace: ~/dev/fizzy-suggest
Branch: app/suggestion-branches
Notes: Works with no room (an agent on your machine). Pixi's preview layer is a pixi PR after this. Several agents' branches side by side is AGENTS_ANYWHERE section 7.
```

### 12. agent and chat: edits with a base, runs squashed, remote agents as members

- Repos: fizzyedit/chat, then fizzyedit/agent. Branches: `chat/write-base`, `agent/room-member`.
  Builds on: 1 (chat), 7 (agent).
- Cloud: yes.

```
Task: Chat passes base (from the last fs/read_text_file of that path) on fs/write_text_file -> text.write. Agent: a remote agent in a room is an agent author with its person's peer; its run is squashed into one step before sending; a .waiting answer when the relay throttles; its look region is its presence.
Plan: plans/MULTIPLAYER_PLAN.md, "Agents"
Builds on: step 1 (chat); step 7 (agent)
Workspace: ~/dev/fizzyedit/chat-write-base, ~/dev/fizzyedit/agent-room
Branch: chat/write-base, agent/room-member
Notes: Two PRs, one per repo. Chat's can land as soon as step 1 merges. Repin to the SDK release carrying step 2 first.
```

## Open

- **Text merge: our own Zig (recommended) or `yffi`/diamond-types over a C ABI?** Ours keeps one
  copy of the text and no Rust in the build. It costs a careful few hundred lines and a fuzzer.
- **Is undo in a room local?** Recommended: yes, yours and your agents'. That amends
  `SHARED_HISTORY_PLAN.md`'s linear Cmd-Z, for rooms only.
- **Decided with the maintainer, 2026-10-19 (`AGENTS_ANYWHERE_PLAN.md`):** the relay is hosted by
  fizzy on Cloudflare Workers + Durable Objects (`relay.fizzyed.it`), with a self-host Zig binary,
  and payloads are end-to-end sealed from the start. What is left open here: whether guests'
  invites should expire sooner than an agent's pairing window (10 min), and whether presence
  (pointer positions) should be sealed too, since a plain presence frame would leak where people
  look (recommended: sealed, at the cost of the relay not dropping silent members itself, so it
  uses the plain header's timestamp).
- **Saving.** The host's disk only, or may an editor save (it writes through the host)?
- **LAN without the relay** (TCP on the local network) is left out. Is it wanted?
- **Remote view as the fallback** for owners with no steps (`EMBEDDED_EDITOR_PLAN.md`), or
  read-only is enough?

## Sources

- Kleppmann et al., *Interleaving anomalies in collaborative text editors*, PaPoC 2019:
  martin.kleppmann.com/2019/03/25/papoc-interleaving-anomalies.html
- Weidner and Kleppmann, *The Art of the Fugue*, arXiv 2305.00583 (TPDS 2025)
- Gentle and Kleppmann, *Collaborative Text Editing with Eg-walker*, EuroSys 2025: arXiv 2409.14252
- Figma, *How Figma's multiplayer technology works*:
  figma.com/blog/how-figmas-multiplayer-technology-works/
- Drawpile, *How Drawpile works*: github.com/drawpile/Drawpile/wiki/How-Drawpile-works
- Replicache, *How it works*: doc.replicache.dev/concepts/how-it-works. tldraw sync:
  tldraw.dev/docs/sync
- Zed, *CRDTs*: zed.dev/blog/crdts. Loro undo: loro.dev/docs/advanced/undo. Yjs UndoManager and
  awareness: docs.yjs.dev
- Stewen and Kleppmann, *Undo and redo support for replicated registers*, PaPoC 2024: arXiv
  2404.11308
- Ink & Switch, *Patchwork*: inkandswitch.com/patchwork/notebook/07
- VS Code Live Share: learn.microsoft.com/en-us/visualstudio/liveshare/overview/features
- Yjs on TURN versus a relay: discuss.yjs.dev/t/signalling-server-on-different-network/133
