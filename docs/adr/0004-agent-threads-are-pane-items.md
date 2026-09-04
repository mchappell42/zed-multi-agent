# Agent threads are pane items

The agent panel showed exactly one thread at a time, so users could not compare two threads or watch one while working in another. We are replacing that single-slot view with a real `Pane` inside the agent dock whose tabs are agent threads, which makes a thread an ordinary workspace item that can also live in a centre pane. The single-slot enum it replaces existed only because there had never been more than one view to hold.

## Considered options

**Splitting the agent dock instead**, mirroring the terminal dock's pane group, was the obvious cheaper path and we rejected it. The terminal dock is bottom-docked and spans the window width, so splitting it yields usable panes; the agent panel is left/right only and cannot be bottom-docked, so splitting it yields two narrow columns of wrapped prose — and the sidebar is already consuming width on one side. It was also not actually cheaper: once threads are items in a real pane, centre panes accept them via ordinary tab drag with no new code, so restricting threads to the dock would have meant writing a predicate to *remove* a capability that arrives for free.

**A bespoke tab bar** over the existing single-slot view, as the debugger panel does, was rejected because tabs, preview-on-single-click, promote-on-double-click and collapse-when-empty are all already implemented inside `Pane`.

## Consequences

Threads reachable by centre panes means thread items must be registered as serializable; an unregistered item is dropped from a restored layout silently, with no error, so this is a correctness requirement rather than a nicety.

The agent panel now returns a pane from `Panel::pane()`, which it previously did not. This is what makes the dock a normal location for focus purposes, and it is load-bearing for the sidebar's three-state highlight: without it the workspace's focus machinery can never report the agent dock as the focused pane.

Agent terminals share the dock's tab bar and are items too. Keeping them special-cased would have preserved exactly the single-slot fork this decision removes.

Splitting inside the agent dock, including vertically in a side dock, is deliberately deferred rather than ruled out. Nothing here forecloses it — the item model and `Panel::pane()` are unchanged by adding a pane group later.
