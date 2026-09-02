---
status: accepted
---

# The public address of a peer is the sidebar thread identity

A thread has two identities. `ThreadId` is assigned by the sidebar metadata store at creation. The agent session identity, `acp::SessionId`, does not exist until the thread's first turn starts, and `create_thread` deliberately returns neither. We decided that peer messages are addressed by `ThreadId` everywhere they are composed, listed, or stored, and are resolved to a session identity only at delivery time through the metadata store's session lookup. This is the only choice that lets an agent message a thread it just created without racing the session start.

## Considered options

- Address by session identity, which thread mentions, `spawn_agent`, and live status already use. Rejected because a freshly created thread has no session identity, so the caller would have to poll or wait.
- Make `create_thread` wait for the session to exist. Rejected because it turns a fire-and-forget tool into a blocking one and still leaves drafts unaddressable.

## Consequences

- `SiblingThreadInfo` and the `create_thread` tool output gain the new thread's `ThreadId`.
- Delivery to a draft thread must start the session, which happens naturally because delivery submits a prompt.
