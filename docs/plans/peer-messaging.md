# Peer messaging between agent threads

Implementation plan for zed-industries/zed discussion #55122, as decided in the design session on 2026-09-01. Vocabulary is in `CONTEXT.md`. Decisions with lasting consequences are in `docs/adr/0001` through `0003`.

## Summary of the design

- A **peer** is any thread in the same project group. Native and external agents are both valid senders and receivers.
- A **peer message** is addressed by `ThreadId` and resolved to `acp::SessionId` at delivery.
- **Delivery mode** is queue by default, steer on request. A busy peer is never interrupted. An unloaded peer is loaded in the background and receives the message immediately.
- **Reply tracking** defaults on. A **reply** is the peer's complete next turn, delivered back as a peer message with tracking off. Replies carry stop reasons on error or cancel.
- The receiver sees a plain-text attribution header inside the user message. When tracking is on it also sees one line saying the final response will be returned.
- Pending deliveries and reply links persist in the sidebar thread metadata database.
- Replies never track. Every peer message carries a hop count, capped at 4.
- Gated by a new staff-on feature flag, `peer-messaging`.

Three PRs, each landing on `main` on its own:

1. Composer send with reply tracking.
2. Native agent tools.
3. MCP bridge for external agents.

Every PR that touches source must first prepend the review marker to `README.md` as required by `CLAUDE.md`.

## Existing seams this plan builds on

| Concern | Where | Note |
| --- | --- | --- |
| Sibling-thread host trait | `crates/agent/src/agent.rs:422` | `SiblingThreadHost`, installed by `AgentPanel` at `agent_panel.rs:4680` |
| Thread environment defaults | `crates/agent/src/thread.rs:774` | `ThreadEnvironment` default methods error; `NativeThreadEnvironment` forwards to the host |
| Sibling request and info | `crates/agent/src/thread.rs:823` | `SiblingThreadInfo` returns no identity today |
| Deliberate identity gap | `crates/agent_ui/src/agent_panel.rs:4884` | Comment explains why no session id is awaited |
| Sidebar identity | `crates/agent_ui/src/thread_metadata_store.rs:34` | `ThreadId`; session lookup via `entry_by_session` and `threads_by_session` |
| Project-group lookup | `thread_metadata_store.rs:643` | `entries_for_main_worktree_path` |
| Metadata persistence | `thread_metadata_store.rs:514` | `DbOperation` channel, `sidebar_threads_v2` table |
| Prompt submission | `crates/acp_thread/src/acp_thread.rs:3630` | `AcpThread::send`; `run_turn` cancels any in-flight turn |
| Turn completion | `acp_thread.rs:2169` | `AcpThreadEvent::Stopped(acp::StopReason)` |
| View-level queue | `crates/agent_ui/src/conversation_view/message_queue.rs` | Queue, steer, pause, resume, fast-track |
| View send path | `crates/agent_ui/src/conversation_view/thread_view.rs:1658` | `send_content` |
| Open or load a thread | `agent_panel.rs:1689` | `open_thread`, `retained_threads`, `conversation_view_for_id` |
| Mention completion | `crates/agent_ui/src/completion_provider.rs` | `MentionCompletion`, `completion_for_thread` at `:1810`, per-surface gating via `supports_context` |
| Mention kinds | `crates/acp_thread/src/mention.rs:20` | `MentionUri::Thread { id: acp::SessionId, .. }` |
| Tool trait | `crates/agent/src/thread.rs:5039` | `AgentTool`; doc comment on `Input` is the model-facing description |
| Tool gating | `crates/agent/src/tools.rs:241` | Flags filter model visibility, not registration |
| Feature flags | `crates/feature_flags/src/flags.rs:45` | `CreateThreadToolFeatureFlag` pattern |
| In-process MCP server | `crates/context_server/src/listener.rs` | Unused since the ACP migration; `add_tool`, unix socket |
| MCP servers per session | `crates/agent_servers/src/acp.rs:4397` | `mcp_servers_for_project` feeds new, load, and resume requests |
| Socket bridge precedent | `crates/cli/src/main.rs:154` | Hidden `--askpass` flag acts as netcat over a unix socket |

## PR 1: Composer send with reply tracking

Goal: type a message in any thread, route it to a peer, and get the reply back in the originating thread.

### 1.1 Feature flag

- Add `PeerMessagingFeatureFlag` in `crates/feature_flags/src/flags.rs`, name `peer-messaging`, staff-on, following `CreateThreadToolFeatureFlag`.

### 1.2 Domain types (new module `crates/agent_ui/src/peer_messaging.rs`)

- `PeerMessage { origin: ThreadId, target: ThreadId, body: String, track_reply: bool, delivery: DeliveryMode, hop: u8, in_reply_to: Option<PeerMessageId> }`.
- `DeliveryMode { Queue, Steer }`.
- `ReplyLink { peer: ThreadId, origin: ThreadId, message_id: PeerMessageId, hop: u8 }`.
- `render_header(message, origin_title) -> String` producing the attribution header and, when `track_reply` is on, the one-line return instruction. Keep this the single place the header text is built so PR 2 and PR 3 reuse it.
- A `PeerMessagingHost` entity that owns pending deliveries and reply links, resolves addresses through `ThreadMetadataStore`, and performs delivery. This is the implementation the `SiblingThreadHost` extension in PR 2 forwards to.

### 1.3 Persistence in `thread_metadata_store.rs`

- Two new tables, created in the same migration block as `sidebar_threads_v2`:
  - `peer_pending_deliveries(id, target_thread_id, origin_thread_id, body, track_reply, delivery_mode, hop, in_reply_to, created_at)`.
  - `peer_reply_links(message_id, peer_thread_id, origin_thread_id, hop, created_at)`.
- Extend `DbOperation` with upsert and delete variants for both, keeping the single async channel.
- Load both tables in the store's reload path and expose `pending_deliveries_for(ThreadId)` and `reply_link_for_peer(ThreadId)`.

### 1.4 Delivery

- `PeerMessagingHost::send(message, cx)`:
  1. Validate that origin and target share a project group via `entries_for_main_worktree_path`. Reject otherwise.
  2. Enforce the hop cap. Replies are created with the origin's hop plus one and `track_reply: false`.
  3. Persist as a pending delivery.
  4. If `track_reply`, persist a reply link.
  5. Attempt immediate delivery.
- Immediate delivery:
  - Find the live `ConversationView` via `AgentPanel::conversation_view_for_id`. If absent, call `open_thread` in background mode so the view exists without stealing focus. This is the load-then-send path; a new `AgentPanel` method that loads without activating is the likely addition.
  - Hand the message to the view's `MessageQueue` as a `QueueEntry` built from the rendered header plus body, with `steer` set from the delivery mode. The queue then applies its existing idle, pause, and steer rules, so a busy peer is never interrupted.
  - Remove the pending delivery when the queue accepts it.
- Drain on load: when a thread's view is created, ask the store for its pending deliveries and enqueue them in creation order.

### 1.5 Reply capture

- When a reply link exists for a thread, subscribe to its `AcpThread` for `AcpThreadEvent::Stopped`.
- On `Stopped`:
  - `EndTurn`: collect the final assistant message text from `entries()` and build a reply peer message.
  - Error or cancel: build a reply whose body states the stop reason.
  - A turn that stalls on a permission prompt never emits `Stopped`, so nothing is sent until the human acts.
- Send the reply through `PeerMessagingHost::send` with `in_reply_to` set. Delete the reply link.
- If the originating thread no longer exists in the store, log and delete the link.

### 1.6 Composer

- Add a "Send to thread" section to the `@` completion menu in `completion_provider.rs`, gated by the flag and by the same surface check used for thread mentions. Reuse `completion_for_thread` for the candidate list and show recent threads first.
- Selecting an entry inserts a routing crease. Represent it as a new `MentionUri::PeerThread { id: ThreadId, name }` in `mention.rs` so the crease, serialization, and icon machinery are reused. The crease carries the `track_reply` toggle, default on.
- In `ThreadView::send`, before the normal path: if the editor contains a routing crease, strip it, build a `PeerMessage` from the remaining text, call `PeerMessagingHost::send`, clear the editor, and show the ephemeral "Sent to «peer»" notice with a link that calls `open_thread`. The local agent never sees the message.
- Only one routing crease per message. A second one is rejected with an inline error.

### 1.7 Extend `create_thread` with the address

- Add `thread_id: ThreadId` to `SiblingThreadInfo` and to the `Success` variant of `CreateThreadToolOutput`. `ThreadId` is available synchronously when the thread entry is created, so this does not introduce the race described at `agent_panel.rs:4884`. Update that comment.
- This requires `crates/agent` to know the `ThreadId` type. Move `ThreadId` out of `agent_ui` into a crate both depend on, most plausibly `acp_thread` beside `MentionUri`, or re-export it from `agent`.

### 1.8 Tests

- `thread_metadata_store` tests for both tables, including reload.
- A GPUI test in `agent_ui` with two fake agent connections: send with tracking, assert the peer receives the header plus body as a user message, complete the peer's turn, assert the origin receives a reply peer message with `track_reply` off.
- Cases: busy peer queues; unloaded peer loads and receives; error stop reason becomes a reply; hop cap rejects; different project group rejects; origin deleted before reply drops with a log line.

## PR 2: Native agent tools

Goal: the native agent can list peers, message them, and act on replies.

### 2.1 Host extension

- Add to `SiblingThreadHost`:
  - `post_to_thread(&self, request: PeerMessageRequest, cx) -> Task<Result<PeerMessageReceipt>>`.
  - `list_threads(&self, cx) -> Result<Vec<PeerThreadSummary>>` returning address, title, agent id, and live status for every peer in the project group.
- Add matching default-error methods to `ThreadEnvironment` and forward them in `NativeThreadEnvironment`.
- `AgentPanel`'s host implementation forwards to `PeerMessagingHost` from PR 1. The caller's `ThreadId` is known to the host because each native thread's environment is created per session.

### 2.2 Tools

- `crates/agent/src/tools/post_to_thread_tool.rs`: input `{ thread_id, message, track_reply: default true, delivery: default queue }`. No authorization prompt, matching `create_thread`. Output is the receipt plus a short note that any reply will arrive as a later user message with an attribution header.
- `crates/agent/src/tools/list_threads_tool.rs`: no input, returns the summaries.
- Register both in `Thread::add_default_tools`, add to the `tools!` macro, and gate model visibility in `tools.rs` on `PeerMessagingFeatureFlag`.
- Update the `create_thread` tool description at `create_thread_tool.rs:20` to say the returned `thread_id` can be used with `post_to_thread`.

### 2.3 Tests

- Tool unit tests using a fake host.
- One end-to-end GPUI test: native thread calls `create_thread`, then `post_to_thread` with the returned address, and receives the reply.

## PR 3: MCP bridge for external agents

Goal: Claude Code and Codex sessions get the same four tools.

### 3.1 Revive the listener

- Confirm `context_server::listener::McpServer` still builds and speaks the current MCP version. Register four `McpServerTool` implementations that forward to `PeerMessagingHost` and the existing sibling host: `post_to_thread`, `list_threads`, `create_thread`, `list_agents_and_models`.
- Instantiate one `McpServer` per external session, owned by the session record in `crates/agent_servers/src/acp.rs`, so the calling thread's identity is fixed at construction and never taken from the model.

### 3.2 Stdio bridge

- Add a hidden `--mcp-bridge <socket>` flag to `crates/cli/src/main.rs`, copying the `--askpass` netcat behavior: connect to the socket, pump stdin to it and its output to stdout.
- Locate the `zed` CLI binary the same way askpass does.

### 3.3 Session wiring

- In `mcp_servers_for_project`, or a wrapper that takes the session, append an `acp::McpServer::Stdio` named `zed` whose command is the CLI binary with `--mcp-bridge <socket path>`.
- Apply to new, load, and resume requests so a resumed session regains the tools.
- Gate on `PeerMessagingFeatureFlag`.
- Tear down the server and socket when the session closes.

### 3.4 Tests

- Listener round-trip test: connect over the socket, `tools/list`, `tools/call` for `list_threads`.
- Manual verification with Claude Code: confirm the tool appears, that a tracked message reaches a peer, and that Claude Code's permission prompt for the MCP tool routes through Zed.

## Risks and open items

- **Loading a thread without focusing it.** `open_thread` currently activates the view. PR 1 needs a background variant. If that proves invasive, fall back to holding pending deliveries for unloaded threads in the same window until the human opens them, and revisit.
- **Sharing `ThreadId` across crates.** The crate move in 1.7 is the one structural change that touches unrelated code. Do it as the first commit of PR 1 so the diff is reviewable.
- **Queue ownership.** The `MessageQueue` requires a `MessageEditor` entity per entry. A peer message needs a lightweight editor or a queue entry variant without one. Prefer the variant to avoid creating editors for messages nobody edits.
- **External transcript ownership.** Zed does not persist external agents' transcripts. Reply capture reads the live `AcpThread` entries, which is fine while the session is open. A reply from a session that was closed and later resumed is out of scope.
- **Blocking wait.** `wait_for_peer_replies` is deferred until PR 2 has been used daily and MCP client timeouts are understood.
