# ADR-0001: Isolated agent Spaces

## Status
Accepted

## Date
2026-09-23

## Context
Search keeps a person's tabs, history, passwords, and website data in one browsing session. Its opt-in Bench testing socket can address ordinary tabs, and Bench pages use the ordinary website store. An agent interface built on that socket would inherit access to the person's session.

Agents need their own browsing state. Sharing one agent profile across clients would also let one client read another's signed-in sites.

## Decision

- Each agent task uses a named Space. Every new Space chooses its own ephemeral or persistent `WKWebsiteDataStore`. Pages within that Space share its store; no agent page uses Search's ordinary store.
- Agent pages do not enter the person's tab list, session, history, password autofill, or extension context. No agent operation may enumerate or address an ordinary tab.
- Search checks the authenticated owner on every Space and page lookup. A client sees and resumes only its own Spaces; the person using Search can see them all. Neither a guessed identifier nor access to a local socket grants another client access.
- Bench remains an opt-in testing interface for ordinary browsing. It does not implement the agent boundary.

## Scope
This record owns the browsing-data and owner-lookup boundary. The local MCP interface is specified in [ADR-0002](0002-agent-mcp-interface.md). Working slots, idle closure, and human handoff are specified in [ADR-0003](0003-agent-space-lifecycle.md). Those interfaces must preserve this boundary; neither changes the person's ordinary browsing or Bench policy.

## Alternatives considered

- Share the person's website store and tabs with agents. It would simplify sign-in but expose the person's browsing state.
- Give all agents one separate website store. It would protect the person but still expose one client's signed-in sites to another.
- Reuse Bench with checks on selected commands. Its existing tab lookup and default website store leave too many paths to the person's session; it is not an ownership boundary.

## Consequences
A client can use sites it signs into within its own Space. Keeping data stores separate is not enough on its own: every Space and page lookup must also check the owner, including after a page is reloaded or a Space is resumed.

WebKit documents [persistent profile stores on macOS 14](https://webkit.org/blog/14423/building-profiles-with-new-webkit-api/).
