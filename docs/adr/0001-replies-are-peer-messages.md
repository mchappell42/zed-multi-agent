---
status: accepted
---

# Replies are peer messages

When a thread sends a peer message with reply tracking on, the peer's next completed turn must reach the originating thread. We decided that a reply is itself a peer message, sent from the peer back to the originating thread with tracking off, rather than a separate event or entry kind. This keeps one delivery path, one attribution format, and one set of queue and steer semantics, and it means the originating agent actually processes the reply, which is what delegation needs. It also rules out reply loops by construction, since a reply can never request a reply.

## Considered options

- A distinct read-only reply entry in the originating thread. Rejected because the originating agent would never see it, so the human would still have to relay it by hand.
- Appending the reply to the originating agent's context without running a turn. Rejected because the Agent Client Protocol only adds context by running a prompt, so this cannot work for external agents.

## Consequences

- A reply causes a turn in the originating thread, subject to that thread's queue. If the human has paused the queue, the reply waits.
- Errors and cancellations in the peer produce a reply that states the stop reason. A peer stalled on a permission prompt produces no reply until the human acts.
- A peer message carries a hop count so that agent-driven ping-pong through tracked tool calls is capped.
