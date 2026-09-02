---
status: accepted
---

# External agents get peer-messaging tools through a Zed-hosted MCP server

Zed-defined tools only exist for the native agent. External agents such as Claude Code and Codex run over the Agent Client Protocol, which lets the client offer filesystem and terminal services but not arbitrary tools. We decided to give external agents the peer-messaging and sibling-thread tools by reviving the in-process MCP server in `crates/context_server/src/listener.rs`, running one instance per external session, and passing it in the session's MCP server list alongside the user's configured context servers. The tools call the same host implementation the native tools use.

## Considered options

- An ACP extension method. Rejected because it would require every external agent to learn a Zed-specific protocol, and the agent's model would have no tool to call.
- One global MCP server with the caller's thread passed as a tool parameter. Rejected because the model could fill it in wrongly or spoof another thread; a per-session server makes the caller's identity implicit.
- HTTP transport. Deferred. The listener speaks over a unix socket, and the `zed` CLI already has a hidden netcat-style socket bridge for askpass, so a stdio bridge is the smaller change and avoids opening a port.

## Consequences

- Each external session spawns one unix socket and one bridge process. The socket lives in a temp directory and is removed when the session ends.
- External agents apply their own permission rules to MCP tool calls, which route back through Zed's existing permission UI.
- A blocking wait primitive should not be exposed over MCP until client timeouts are understood.
