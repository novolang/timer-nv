# Changelog

All notable changes to timer-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
- **Two `[time]` rows in the whole package**, both in `tmrclock`.
- **No dependencies.**  A duration is the standard library's
  `Duration`; humantime-nv is what a caller parses `"5m"` with, beside
  this package rather than under it.
- **No device claim.**  The wheel would build for one and the package
  does not say so, because a claim that is not built is not a claim.
