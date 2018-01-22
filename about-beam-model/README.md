

### ParDo ###



### GroupByKey ###


Windowing data

=> GroupByKeyAndWindow

Support unaligned windowing:

- unaligned is the general case. Aligned windowing is the special form.
- Two related operations: AssignWindows, MergeWindows.

#### Assignwindows ####

creates a new copy of the element in each of the windows to which it has been assigned.

#### MergeWindows ####


### The Second Problem: When to output result for a window? ###

Trigger: provide multiple answers (or panes) for any given window.




- Windowing determines **where in event time** data are grouped together for processing.
- Triggering determines **when in processing time** the results of groupings are emitted as panes.


