# THESIS — the north star

> **BrainKeep is an agentic architecture that enables efficient recall and builds itself
> with a human in the loop.**
> It is for builders who want to observe what is being built, and to make sure it stays
> human-readable.

Every feature, README line and diagram is checked against three claims and one principle:

1. **Efficient recall.** The right page reaches the agent without re-reading the corpus.
2. **It builds itself.** The Keepers maintain the Library and grow new agents.
3. **A human can read everything.** The human does not approve every change; nobody
   could. But any page, any agent, any change can be opened by a human who then says
   "ah, I understand what this is." Plain markdown, nothing hidden in an opaque store.
   The human steps in where it matters, not everywhere.

4. **Nothing fails silently.** Claim 3 applied to failures: every pending, dropped, stale
   or superseded thing is a file or a line a human can open.

If a change makes the Library less understandable to a human reader, it does not ship.

## Open problem

The Library is shelved with a Dewey-style class system (000–900). It works, but only its
author knows how to run it. BrainKeep has to make the shelves teachable: the Keepers file
pages themselves, and a human can read any shelf and understand why a page sits there.

*James, 2026-10-10.*
