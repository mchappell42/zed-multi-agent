# Agent Thread Messaging

Vocabulary for messages exchanged between agent threads in the Threads Sidebar. Threads are lateral: there is no orchestrator, and messaging never implies a parent/child relationship.

## Language

**Project group**:
A main git repository together with its linked worktrees, as grouped in the Threads Sidebar.
_Avoid_: Project, workspace, window

**Peer**:
Any other thread in the same project group, regardless of which agent backs it.
_Avoid_: Sibling, child, subagent (those name the `create_thread` and `spawn_agent` relationships, not messaging)

**Pending delivery**:
A peer message that has been sent but not yet handed to its peer, because the peer is busy or not loaded.
_Avoid_: Queued message, inbox, mailbox

**Peer message**:
A message composed in one thread and routed to a peer instead of to the local agent.
_Avoid_: Cross-thread message, forwarded message, handoff

**Originating thread**:
The thread in which a peer message was composed.
_Avoid_: Parent, sender thread, source

**Reply**:
The complete final text of the peer's next turn after it receives a peer message, captured when that turn ends.
_Avoid_: Response, result, output

**Reply tracking**:
The opt-in choice, made when a peer message is sent, to have the reply appear in the originating thread.
_Avoid_: Subscription, piping

**Delivery mode**:
How a peer message is handed to a busy peer. `Queue` waits until the peer is idle. `Steer` delivers at the peer's next turn boundary without discarding its work. Queue is the default.
_Avoid_: Interrupt, send now, priority

**Thread address**:
The stable identity used to name a peer in a peer message. It is the sidebar's thread identity, which exists from creation, and is resolved to the agent session identity only at delivery time.
_Avoid_: Session id (as a public address), thread handle

### Threads

**Thread**:
One agent conversation, identified by a stable id that outlives any view of it.
_Avoid_: chat, session

**Conversation**:
A root thread together with the subagent threads it spawned. A conversation is what occupies a single tab; a thread is what the sidebar lists.
_Avoid_: using interchangeably with Thread

**Agent**:
The program that runs a thread. An agent is a participant, never a conversation — "three agents open" is always wrong; say "three threads open".
_Avoid_: bot, model, assistant

**Draft**:
A thread that has never sent a message and so has no agent session behind it yet.

**Retained thread**:
A thread that is live and still running but has no tab anywhere. Retained threads keep producing status the sidebar displays.
_Avoid_: background thread, parked thread

**Archived thread**:
A thread the user has explicitly filed away. Orthogonal to whether it is open — an archived thread can be opened, and a closed thread is not archived.

### Where a thread appears

**Sidebar**:
The window-level registry of every thread, across every project held by the window. It is a list of threads that exist, not a container of threads that are open — a row is a pointer, and dragging one out never removes it.
_Avoid_: thread list panel, history panel, thread dock

**Agent Panel**:
The dock, belonging to a single project, that holds thread tabs. One project, one agent panel.
_Avoid_: assistant panel

**Pane**:
A tab container. Both the agent panel and the centre area hold panes, and a thread tab means the same thing in either.

**Workspace**:
One project's content — its panes, docks and status bar. A window holds several workspaces but displays one at a time, which is why the sidebar spans projects and a pane never does.

### Thread view state

**Thread view state**:
Where a thread is relative to the user's attention, as one of Focused, Open or Closed. Exactly one thread is Focused at a time.
_Avoid_: selected, active — both were previously used for two different ideas at once

**Focused**:
The thread the user is looking at right now. Survives the user clicking away into an editor: it is released only when a different thread is focused.

**Open**:
The thread has a live view somewhere the user is not currently looking — another pane, another tab, or the dock while focus is elsewhere.

**Closed**:
The thread has no live view anywhere. Says nothing about whether it is running or archived; a retained thread is Closed.

**Keyboard cursor**:
The sidebar row the keyboard is on. Independent of thread view state — the cursor can sit on a Closed row.
_Avoid_: selection, focus
