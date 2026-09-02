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
