# ADR-0001: Isolated agent Spaces

## Status
Accepted

## Date
2026-09-23

## Context
Search keeps a person's tabs, history, passwords, and website data in one browsing session. Its opt-in Bench socket is a testing interface that also ships in the app: it can address ordinary tabs and its pages use the ordinary website store. It cannot be the access boundary for an external agent.

Agents need visible, independently controlled browsing without taking over the person's window. Search should remain quiet when no agent is working. The agent interface is local MCP; Ego Lite is a reference for browsing workflows and human handoff, not a protocol or engine dependency.

## Decision

- A Search-specific MCP stdio helper runs only when an external client launches it. It talks to a separate, opt-in local Search socket. Bench keeps its existing opt-in testing role and availability; it does not serve MCP or access agent Spaces. The app, not the helper or a self-reported MCP client name, enforces access. Search pairs each client through its own UI and assigns an opaque credential; the app scopes every Space lookup and page operation to that credential's owner. The user can see all Spaces, but clients cannot inspect or resume one another's Spaces.
- Each task explicitly creates or resumes an agent-owned Space. A new Space chooses either an ephemeral `WKWebsiteDataStore.nonPersistent()` or its own persistent `WKWebsiteDataStore(forIdentifier:)`. Its pages share that Space's store, never Search's user store. Agent pages do not enter the person's tab list, session, history, keychain autofill, or extension context. No MCP tool enumerates or operates on the person's tabs. Arbitrary page JavaScript is allowed only on pages owned by the calling client.
- A Space switcher in the existing window keeps the person's browsing separate. Opening an active Space lets the user watch without interrupting the agent. Explicit Take Control blocks subsequent agent page actions until control returns. The bridge does not select a Space in the user's window on an agent's behalf.
- Settings defaults to eight concurrently working agent Spaces. It is adjustable and does not count the person's browsing or parked Spaces. Work on a parked Space must acquire a slot; at capacity, the request fails without interrupting another task. Search sleeps eligible idle agent pages with the existing tab-sleep safeguards before closing the Space; it does not suspend in-flight agent work or discard unsaved input, active media, or downloads. An adjustable inactivity interval defaults to one hour: after that interval with no agent work or human interaction, Search closes the Space unless a safeguard still applies. Ephemeral website data is discarded; persistent website data remains available to that owner when the Space is resumed.
- Agent tools target Search-native outcomes such as navigation, observation, actions, screenshots, and handoff. They do not expose Chromium CDP or promise drop-in compatibility with Ego scripts. A Search-specific agent skill documents the shipped MCP tools, rather than modifying Ego's skill.

## Scope
This record owns the agent Space isolation, ownership, lifecycle, and local control boundary. The tool catalog and reviewable workflows must obey it. It does not change ordinary browsing, Bench's existing availability, or Search's website data and password policies for the person's own tabs.

## Alternatives considered

- Reuse Bench: fewer files, but its same-user socket can read ordinary tabs and uses the ordinary website store. Separate access checks on some commands would not fix the shared ownership model.
- Share one agent profile with the person's session, or across clients: convenient sign-ins, but it defeats Space isolation and owner-only access.
- A resident model or remote HTTP agent service: unnecessary process, credential, and network cost for a local browser whose agents already run elsewhere.
- Keep every agent page live: preserves JavaScript runtimes, but idle WebKit pages retain processes. Sleeping restores navigation and interaction state, not a running page program.

## Consequences

A client that holds its pairing credential can act in its own isolated Spaces, including signed-in sites it visits there. A credential identifies a paired client, not a cryptographically verified agent executable. Revoking it blocks further tool calls and closes that client's active Spaces; its persistent website data remains visible to the user, inaccessible to other clients, until the user explicitly removes it. Even with eight working Spaces, active pages can consume substantial memory, so the limit is not a fixed memory ceiling. Closing an ephemeral Space loses its site state; persistent stores need explicit user-owned removal rather than deletion when the inactivity timer fires.

WebKit documents [persistent profile stores on macOS 14](https://webkit.org/blog/14423/building-profiles-with-new-webkit-api/). MCP [stdio](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports#stdio) provides local JSON-RPC transport, not authenticated client identity; Search must enforce ownership independently.
