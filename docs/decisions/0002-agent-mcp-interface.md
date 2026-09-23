# ADR-0002: Local MCP interface for agent Spaces

## Status
Accepted

## Date
2026-09-23

## Context
An external agent needs to browse through Search's WebKit pages. MCP stdio gives that agent a local tool transport, but it does not authenticate the client. Search's Bench socket exists for testing ordinary tabs and cannot serve as the agent entry point under the [Space isolation boundary](0001-agent-spaces.md).

The browser should not run a resident model or a remote agent service merely to expose its pages.

## Decision

- Bundle a Search-specific MCP stdio helper with the app. The external client launches the helper when needed; Search itself owns WebKit pages and serves a separate, opt-in local socket. The helper translates tool calls but does not decide which client owns a Space.
- Target the `2026-07-28` MCP revision. Each request carries its protocol version and client capabilities in `_meta`; the helper implements `server/discover`. It does not require the retired `initialize`/`initialized` handshake. The first slice does not add a second, legacy protocol path.
- Pair clients through Search's UI. Search issues each client an opaque bearer credential, shows it once, keeps only its digest in the profile's private ledger, and accepts it until revocation. The app checks it before resolving a Space or page. The MCP `clientInfo` field and access to a same-user socket are not identities. If the ownership ledger cannot be read, Search refuses agent operations rather than starting with an empty client list. Revocation rejects later calls; [ADR-0003](0003-agent-space-lifecycle.md) owns the resulting Space cleanup.
- The first tool contract lets a client create, list, and finish its Spaces; open and navigate HTTP or HTTPS pages; read bounded page text; click or fill a CSS-selected element; evaluate JavaScript in an owned page; and capture an owned page image. Operations on an existing Space or page require its ID; create and list do not. A missing or foreign ID yields the same not-found result. No tool addresses the person's tabs, another client's pages, or the desktop screen.
- Keep the tool names and behavior Search-native. Ego Lite informs browsing workflows, not an executable or script-compatibility contract. A Search-specific agent skill can teach the shipped tool catalog after that catalog is reviewed.

## Scope
This record owns the external tool and authentication entry point. [ADR-0001](0001-agent-spaces.md) owns what data an authenticated client may reach; [ADR-0003](0003-agent-space-lifecycle.md) owns when a Space may run and when the person takes control. This interface does not change Bench's testing role or ordinary browsing.

## Alternatives considered

- Serve MCP through Bench. Bench can address ordinary tabs and uses their website store, so it cannot enforce this tool boundary.
- Trust a client name or the local user's socket permission. A same-user process can supply any MCP name or connect to a same-user socket. Search still needs a paired credential and owner checks.
- Ship the initialization-based `2025-11-25` revision first. It would make the initial public interface depend on a retired handshake instead of the selected current protocol.
- Support both legacy and modern revisions. It would expand the authentication and protocol test paths without a confirmed legacy-client requirement.
- Expose Chromium CDP or promise Ego script compatibility. Search uses WebKit, and those APIs would turn engine details into a false public contract.
- Add a resident model or remote HTTP service. Both add processes or network exposure without helping an external client use local pages.

## Consequences
Clients must pair in Search and configure the bundled helper with their credential. Anyone holding a client's credential can act on that client's signed-in Spaces until revocation, so the helper must not print it and Search must not store it in plaintext. The helper is an extra bundled executable, but it does not stay running when no client launches it. Clients that only speak the initialization-based revision cannot use this first interface; acceptance must use a client that sends per-request protocol metadata.

MCP [stdio transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio) does not authenticate a peer; Search must enforce the pairing and owner boundary.
