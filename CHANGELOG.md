# Changelog

All notable changes to timer-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-10

The **implementation**.  Every body the 0.0.2 interface published is
written, `novo test --isolate` is green over four suites, and every
line under `src/` is covered by them, with no `cov: skip` marker
anywhere.

- **The wheel orders its entries by hand.**  `list.sort_by` with a key
  that builds a pair from runtime values orders wrongly on the compiled
  leg — silently, and one call site can be wrong while an identical one
  beside it is right — so `tmrwheel` keeps a private `earlier(a, b)`
  comparing deadline then identifier, and a private `inserted(xs, e)`
  that places one entry into an ordered list with it.  Firing order is
  the thing the suite is an oracle for, and it needed the pair rather
  than a sort key.  Filed as
  `list-sort-by-with-a-pair-valued-key-orders-wrong-on-the-compiled-leg`;
  put back when it closes.
- **`stopwatch_elapsed` reads `accumulated` into a name first.**  A
  method called on a struct field in the position a function answers
  from leaks one reference per call: `tmrextra_tests.nv` passed every
  verdict and the run still exited 23.  Filed as
  `a-method-called-on-a-struct-field-in-answer-position-leaks`; put back
  when it closes.
- **`TmrBucket.milli_per_second` holds millionths of a token per
  second, which is what its doc now says.**  The field's comment said
  thousandths, the same scale as the store it feeds, and a rate at that
  scale cannot refill anything below a thousand tokens a second; four
  an hour is 1111.1 millionths, rounded up so a bucket never refills
  slower than it says.
- **`TmrCoalesce` restarts a whole period at the moment it fired**, not
  on the deadline it missed: twenty missed repaints are one repaint,
  and the next repaint is one period after the one that happened.
  `TmrSkip` and `TmrCatchUp` stay on the grid they were scheduled on.

The new tests are in `tests/timer_edge_tests.nv` — a period that is not
positive, a cancel and a reschedule beside other timers, a bucket whose
capacity or rate is not positive, and a take of nothing — because the
API suites are the published oracle and are not edited.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tmrclock` — `TmrMonoTime` and `TmrWallTime` as two types, the three
  `[time]` functions that read a clock, the arithmetic that does not,
  and the conversion that turns a wall-clock alarm into a monotonic
  deadline together with the check that says when it has gone stale.
- `tmrwheel` — the wheel: one-shot and periodic timers, three lateness
  policies, deterministic jitter, tags instead of callbacks, and
  `timeout_ms` / `fire_due`, the two functions an event loop calls
  around the poll it was going to make anyway.  A debounce, because
  an editor, a search box and a file watcher are the same shape and
  are all written wrong in the same way.
- `tmrextra` — a stopwatch that can be paused without losing what it
  had, and a token bucket that refills by computation rather than by
  ticking.

### Known

- **`timeout_ms` and `fire_due` around an existing poll** is the
  load-bearing shape.  `TMR_NO_TIMEOUT` is `-1`, the same convention
  `poll`, `ptypoll.wait_ready` and `io.wait_either` take, so the number
  goes straight through.
- **A handler is a tag, not a closure.**  An integer is something a
  test compares; a capture is state the wheel cannot see.
- **Firing order is deterministic** — deadline order, ties in schedule
  order — so a program whose timers depend on each other is
  reproducible.
- **Monotonic and wall-clock deadlines are two types**, and the wheel
  takes only the first.  A timeout computed on a clock that can be set
  backwards fires immediately or never, which is a bug with a long
  history.
- **`clock_stepped` takes a tolerance**, because NTP slews as well as
  steps and a slew is not a step.
- **Three lateness policies**, because a program that is late has three
  honest answers and they belong to three different kinds of program.
- **Jitter is deterministic**, from a seed the wheel advances: it keeps
  the module `[]`, it makes a schedule reproducible, and nothing about
  a thundering herd needs unpredictability.
- **Nothing in this package can fail.**  There is no error type:
  cancelling an unknown identifier is a no-op, and being rate-limited
  is an answer with a duration rather than an error.
- **Nothing sleeps and nothing spawns.**  `core.rt.sleep_ms` is one
  deadline for one place; the async runtime is one task per timer; this
  is many deadlines in one loop with no runtime.
- **Three `[time]` rows in the whole package**, all in `tmrclock`.
- **No dependencies.**  A duration is the standard library's
  `Duration`; humantime-nv is what a caller parses `"5m"` with, beside
  this package rather than under it.
- **No device claim.**  The wheel would build for one and the package
  does not say so, because a claim that is not built is not a claim.
