# Travis build sequencer

Force `push` builds in Travis to run one at a time without setting a concurrency limit for all builds.

Useful in cases where `push` builds perform deployments that would otherwise interfere.

## Goals and approach

### One build at a time

When a build starts, it checks if it is the running (i.e. `started`) build with the earliest `started_at` time. If so, it proceeds. Otherwise, it exits by cancelling itself.

When a build observes itself to be first by this ordering, it will remain first until it exits -- and it will _always_ have been first from the perspective of any other running build. This lets us use "earliest-started running build" as a simple and stable master-election strategy.

### Newest build will eventually run

When a running build finishes (successfully or not), it looks at the most recently queued build (i.e. highest `ID`, since Travis assigns monotonically-increasing `ID`s) and restarts it if its state is `canceled`. Intervening builds are skipped for efficiency.

It is possible for a build to cancel itself at exactly the wrong moment: before the build it's waiting for has finished, but _after_ that build has looked for -- and not found -- a `canceled` build to restart.

The next queued build will run without incident, so this should only be a brief problem for active repositories. The `canceled` build can also be manually restarted. If we end up hitting this problem often, we can consider waiting for the newest build to _become_ `canceled` (as it must) so we can restart it.

### Builds run in queued order

Even if we're sure only one build can run at a time, it's conceivable that an older build will be delayed long enough for a newer build to run and finish. Since we use the Travis API to restart builds, there's also the possibility of a bug where we improperly restart an old build.

In order to prevent older builds from racing past newer ones, each build checks that it has a **higher `ID`** than any finished (`passed`, `failed`, or `errored`) build.

This still allows the most recently finished build to be restarted.
