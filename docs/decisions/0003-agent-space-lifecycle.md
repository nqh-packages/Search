# ADR-0003: Agent Space lifecycle and human control

## Status
Accepted

## Date
2026-09-23

## Context
Agent Spaces need visible work without displacing the person's tabs. WebKit pages retain processes while live, but closing a page during an agent action, a draft, or active media can lose work. A persistent Space also needs different site-data retention from an ephemeral one. [ADR-0001](0001-agent-spaces.md) owns their isolation, not their lifetime.

## Decision

- Limit concurrently working agent Spaces to an adjustable default of eight. The person's tabs and parked Spaces do not consume slots. Creating or resuming work must acquire a slot; at capacity Search refuses the request without stopping another Space. Finishing a task parks it, and an idle task can park to release its slot.
- Show Spaces in a switcher in Search's existing window. The person can open any Space to watch its pages without selecting a tab on the agent's behalf or interrupting the agent. An explicit Take Control blocks subsequent agent page calls until Return Control. A call already in progress is not silently suspended.
- Sleep eligible idle pages before closing a Space. Check again after asynchronous page inspection and before discarding a live view: a new agent call or human interaction cancels that sleep attempt. Loading pages, unsaved input, active media, and downloads block sleep and closure. When Search cannot establish that a page is safe to sleep, it leaves the page live and retries later. Sleeping preserves a route back to the page, not its running JavaScript state.
- Close a Space after an adjustable inactivity interval, one hour by default, only when no work is in flight and the page safeguards allow it. Closing an ephemeral Space discards its site data. Closing a persistent Space releases its live views but keeps its website data for that owner. Revoking a client closes all its Spaces, working or parked. Search discards its ephemeral records and keeps its persistent records and website data visible only to the person until explicit removal. Removal releases the WebKit views before deleting the persistent store.
- Do not start downloads in agent Spaces in the first slice. A later download feature must track pending downloads before it can allow idle sleep or closure.

## Scope
This record owns task capacity, page sleep, inactivity closure, and the person's handoff controls. [ADR-0002](0002-agent-mcp-interface.md) owns the tool entry point and pairing; [ADR-0001](0001-agent-spaces.md) owns the website-store and owner boundary. It does not change sleep or download behavior in the person's ordinary tabs.

## Alternatives considered

- Count every parked Space against the working limit. A completed task would block new work despite having released its active slot.
- Close every Space exactly at the inactivity deadline. A timer would discard drafts and ongoing media or race an agent call.
- Keep every page live for review. Idle WebKit pages would retain processes indefinitely.
- Give the person control as soon as a Space is opened. Observation would interrupt work and make a casual review change agent behavior.

## Consequences
The working limit is not a memory ceiling. A parked Space with a draft or media may remain live beyond the close interval, and the person may need to clear its blocker or remove it. A resumed sleeping page reloads; callers must observe it again before acting. Persistent site data needs an explicit user-owned deletion path rather than timer-driven removal.
