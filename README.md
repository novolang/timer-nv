# timer-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Deadlines and intervals for a program that has one loop and many of
them.

One-shot and periodic timers live in a wheel the caller polls.  The
wheel answers two questions: how long the next poll should block, and
which timers came due.  So an editor's debounce, a cursor's blink and a
status bar's refresh all come out of the same loop, on one clock read,
with no thread and no runtime.

Monotonic and wall-clock deadlines are two types, because a timeout
computed on a clock that can be set backwards is a bug with a long
history.  Three lateness policies, because a program that is late has
three honest answers.  Deterministic jitter, a stopwatch, and a token
bucket.

## Adding it, and checking it

```bash
novo pkg add timer-nv       # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/tmrwheel_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: timer-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use std.time
use tmrclock
use tmrwheel

// A terminal program's pump, with three timers in it and one clock
// read per turn.  There is no sleep in this loop: the wait is the
// poll the program was going to do anyway, and the wheel only says
// how long it may last.
const BLINK: Int = 1
const STATUS: Int = 2
const REPARSE: Int = 3

fn pump(w0: TmrWheel, d0: TmrDebounce) -> Int [io, time]
    var w = w0
    var debounce = d0
    var running = true
    while running
        let now = tmrclock.mono_now()
        let wait = tmrwheel.timeout_ms(w, now)

        // …hand `wait` to a poll — ptypoll.wait_ready, termios-nv's
        // reader, io.wait_either — and read whatever arrived…
        // …and when the user typed, tell the debounce:
        let poked = tmrwheel.debounce_poke(debounce, w, now)
        debounce = poked.debounce
        w = poked.wheel

        let step = tmrwheel.fire_due(w, tmrclock.mono_now())
        w = step.wheel
        for entry in step.fired
            if entry.tag == BLINK
                running = true          // …flip the cursor…
            else if entry.tag == STATUS
                running = true          // …redraw the status bar…
            else if entry.tag == REPARSE
                running = true          // …the typing settled; re-parse…
    0

fn main() [io, time]
    let now = tmrclock.mono_now()
    var w = tmrwheel.wheel_new()

    let blink = tmrwheel.schedule_with(w, tmrclock.mono_add(now, Duration.of_ms(530)),
                                       TmrPeriodic(Duration.of_ms(530)), BLINK,
                                       TmrCoalesce, tmrwheel.no_jitter())
    w = blink.wheel

    let status = tmrwheel.schedule_with(w, tmrclock.mono_add(now, Duration.of_secs(1)),
                                        TmrPeriodic(Duration.of_secs(1)), STATUS,
                                        TmrSkip, tmrwheel.no_jitter())
    w = status.wheel

    println("${pump(w, tmrwheel.debounce_new(Duration.of_ms(300), REPARSE))}")
```

## The layer, and why

`host`, and **two functions in the whole package declare anything**.

| module | row | why |
| --- | --- | --- |
| `tmrclock.mono_now`, `.wall_now`, `.clock_offset` | `[time]` | they read a clock, and nothing else here does |
| everything else in `tmrclock` | `[]` | arithmetic on times the caller already has |
| all of `tmrwheel` | `[]` | `now` is an argument; the wheel is a comparison and an insertion |
| all of `tmrextra` | `[]` | a stopwatch and a bucket are two integers each |

**The wheel is `[]` and says so.**  A timer that fires by itself needs
a thread, a runtime or a signal; a timer a caller *polls* needs none of
those, and every terminal program already has a loop to poll it from.
So nothing here sleeps, nothing spawns, and nothing reads a clock
except the two functions whose whole job is to.

**`layer = "host"` and not `layer = "core"` with `host_modules`.**  The
narrower-layer-with-named-modules form is for a package whose *subject*
is the pure half — a matcher with a directory walker beside it.  This
package's subject is time, and a `core` package has no clock at all: a
`core` timer package with no way to read one would be published
half-built, and its consumer would have to find a clock somewhere else
and be told which unit it wanted. A consumer that wants only the wheel
gets exactly that by never calling the two `[time]` functions, and the
effect checker holds it to that on its *own* signatures — which is a
stronger guarantee than a manifest line.

## The load-bearing interface

**`timeout_ms` and `fire_due`, around a poll the program was going to
make anyway.**

```novo
let wait = tmrwheel.timeout_ms(w, now)   // how long to block
// …poll for input with `wait`…
let now = tmrclock.mono_now()            // ONE clock read
let step = tmrwheel.fire_due(w, now)     // what came due
```

`timeout_ms` returns `TMR_NO_TIMEOUT` (`-1`) when nothing is scheduled,
`0` when something is already due, and the remaining milliseconds
otherwise — which is exactly the convention `poll` takes, and exactly
the convention `ptypoll.wait_ready`, `termios-nv`'s reader and
`io.wait_either` take.  The number goes straight through with no
arithmetic in between, and that is the point: a pump that was going to
block gets its deadline for free, and every timer in the wheel costs no
wake-up of its own.  On a laptop that is the difference between a
program that lets the processor idle and one that does not.

**A handler is a tag, not a closure.**  Every entry carries an `Int`
the caller chose, and `fire_due` hands those back for the caller to
match on.  A callback would have been shorter to write and would have
put the caller's state behind a capture the wheel cannot see, made the
wheel carry a function value per timer, and made a fired timer
impossible to assert on in a test.  An integer is something a test
compares.

**And the firing order is deterministic**: deadline order, with ties in
schedule order.  A program whose timers depend on each other gets the
same sequence every run, which is what makes a bug in one of them
reproducible.

## Two clocks, two types

A wall clock answers *what time is it* and is allowed to jump — NTP
steps it, daylight saving moves it, a user sets it, a virtual machine
resuming from a snapshot finds it hours out.  It can move **backwards**.

A monotonic clock answers *how long since some arbitrary moment*, never
jumps and never goes backwards.  Its origin is meaningless and
comparing one process's to another's is meaningless.

**A timeout on a wall clock is the bug.**  "Expire this lease in thirty
seconds", computed as `wall_now() + 30s`, expires *immediately* when
the clock steps forward and *never* when it steps backward.  It has
taken down distributed systems, and the fix is always the same: the
deadline was a duration, and durations are measured on the clock that
only measures durations.

So `TmrMonoTime` and `TmrWallTime` are different types, and
**`tmrwheel` accepts only the first**.  There is no overload of
`schedule` that takes a wall time.

An alarm that genuinely means a moment in the day — *run the backup at
02:00* — is converted with `tmrclock.wall_deadline_to_mono`, from a
`TmrClockOffset` that pairs the two clocks at one instant.  That
conversion is correct for as long as the two clocks keep the
relationship they had, and `tmrclock.clock_stepped` is how a program
finds out they have not.  It is a real cost and it is the caller's,
because only the caller knows whether its deadline meant a moment or a
delay.

`clock_stepped` takes a *tolerance* rather than comparing exactly,
because NTP's ordinary behaviour is to **slew** — to run the clock
slightly fast or slow until it agrees — and a slew is not a step.  A
program that rescheduled on every millisecond of drift would reschedule
constantly and for no reason.

## Three answers to being late

A loop that was busy for two seconds, with a timer that repeats every
hundred milliseconds, owes twenty firings.  There is no single right
thing to do about that, so the policy is the caller's:

| policy | what it does | who wants it |
| --- | --- | --- |
| `TmrCatchUp` | fires once per missed period, keeping the original grid | anything where each firing *means* something that must not be lost — a sampler, a frame counter, a retry budget |
| `TmrCoalesce` | fires once however many were missed, and reschedules from now | anything idempotent — a redraw, a status refresh, a cache sweep.  Twenty missed repaints are one repaint |
| `TmrSkip` | fires once, and reschedules on the **original** grid | a clock display.  It should tick on the second, and rescheduling from now drifts a little further off the second every time the loop is busy |

`TmrEntry.periods` says how many periods a firing covered, so a caller
under `TmrCoalesce` that is counting something still knows what it
missed.

## Jitter is deterministic on purpose

A hundred connections each refreshing every thirty seconds, scheduled
at the same moment, wake together every thirty seconds forever — the
classic name for it is a thundering herd.  Jitter spreads them.

The spread here comes from a **seed the wheel advances**, not from a
random number.  Three reasons: it keeps the whole module `[]`, it makes
a schedule reproducible in a test, and nothing about this problem needs
unpredictability — the goal is only that two timers disagree.  It is
not suitable for anything that needs to be unguessable, and this
package says so rather than leaving a reader to assume.

## Where this sits beside `core.rt.sleep_ms` and the async runtime

They solve a different problem, and a program usually wants one or the
other rather than both.

**`core.rt.sleep_ms(ms)`** is *one* deadline for *one* place in the
code.  Inside a task it parks the task and peers run meanwhile; at the
root it drives the scheduler until the deadline.  That is the right
tool when a piece of code wants to wait — a retry backoff, a poll
interval in a script.  It is the wrong tool for a program with five
independent deadlines, because a sleep can only be waiting for one of
them, and whichever one it picked is the one it cannot be interrupted
for.

**The async runtime** gives `await`, `all` and `select` over futures,
and the natural shape there is one task per timer: spawn a task that
sleeps and does its thing. That is a good design and it wants a
runtime, a task per timer, and the caller's state to be reachable from
each of them.

**This package** is for the program in between: one loop, many
deadlines, and no runtime.  A terminal program's pump is already
blocked in a `poll` on a pty and a keyboard; it does not want a task
per timer and it cannot afford to sleep, because sleeping is exactly
what it must not do while input might arrive.  `timeout_ms` gives its
existing wait a deadline, and `fire_due` tells it what happened.

There is deliberately no `sleep` in this package.  A library that slept
would be a library a program cannot interrupt, and a program with a
timer wheel already has something better to wait on.

A program that has a runtime *and* wants many deadlines in one place
can use both: the wheel is `[]`, so it composes inside a task exactly
as it does inside a loop.

## Nothing here can fail

There is no error type in this package, and that is a claim rather than
an omission.  Scheduling cannot fail; cancelling an identifier the
wheel does not have is a no-op, which includes one that has already
fired — a caller racing a one-shot against a cancel should not have to
know which won.  `bucket_take` answers `allowed: false` with a duration
rather than an error, because being rate-limited is not a malfunction.

## What this does not do, on purpose

- **It does not sleep.**  See above.
- **It does not spawn.**  No thread, no task, no signal.
- **It does not parse durations.**  `interval = "5m"` in a
  configuration file is humantime-nv's, and its answer is
  `Duration.of_secs` away from what `schedule_after` takes.  A program
  that builds its intervals in code should not carry a duration parser
  it never calls, which is why that is a dependency of the program and
  not of this package.
- **It does not do calendars.**  "The first Monday of the month" is
  calendar-nv's arithmetic and cron-nv's schedule; when they have
  answered, the result is one `TmrWallTime` and this package takes it
  from there.
- **It does not format a duration.**  `Duration.to_str` is in the
  standard library, and humantime-nv is what a person reads.
- **No device claim.**  The package is `host`, because two of its
  functions read a clock.  The wheel itself would build for a device
  and a later release may say so; it is not claimed here, because a
  claim that is not built is not a claim.

## The reference implementation

Tokio's `time` module for the wheel, the interval policies and the
names (`MissedTickBehavior` is `TmrPolicy` with two of the three), and
Python's `asyncio` event loop for the `call_later` / `call_at` split
and for what a loop's `_run_once` actually needs from a timer heap.
`governor` and Go's `rate` package are the token bucket.

Three things change in the port.  Tokio's timers are futures that fire
by themselves, driven by the runtime's own timer thread — correct there
and unavailable here, so every timer is polled and `timeout_ms` is the
seam. Its `MissedTickBehavior` has `Burst`, `Delay` and `Skip` where
this has `TmrCatchUp`, `TmrCoalesce` and `TmrSkip`, renamed because
"burst" and "delay" describe the mechanism and the new names describe
the decision.  And `asyncio` keys its timer heap on the loop's own
monotonic clock and offers no wall-clock deadline at all, which is
right for a server and wrong for anything with an alarm in it — hence
`TmrWallTime` and the conversion that has to be redone.

## Status

| item | implemented |
| --- | --- |
| `tmrclock` — `TmrMonoTime`, `TmrWallTime`, `TmrClockOffset` | types only |
| `tmrclock.mono_now`, `.wall_now`, `.clock_offset` | no |
| `tmrclock.mono_of_ms`, `.mono_ms`, `.wall_of_unix_ms`, `.wall_unix_ms` | no |
| `tmrclock.mono_add`, `.wall_add`, `.mono_since`, `.wall_since`, `.mono_after`, `.wall_after` | no |
| `tmrclock.wall_deadline_to_mono`, `.clock_step`, `.clock_stepped` | no |
| `tmrwheel` — `TmrId`, `TmrKind`, `TmrPolicy`, `TmrJitter`, `TmrEntry`, `TmrWheel`, `TmrScheduled`, `TmrFired`, `TmrDebounce`, `TmrDebounced` | types only |
| `tmrwheel.TMR_NO_TIMEOUT` | yes — it is a constant |
| `tmrwheel.wheel_new`, `.no_jitter`, `.jitter` | no |
| `tmrwheel.schedule`, `.schedule_after`, `.schedule_with`, `.cancel`, `.reschedule`, `.clear` | no |
| `tmrwheel.next_due`, `.timeout_ms`, `.fire_due` | no |
| `tmrwheel.scheduled_count`, `.is_idle`, `.entry_of`, `.is_scheduled`, `.entries_tagged`, `.cancel_tagged` | no |
| `tmrwheel.debounce_new`, `.debounce_poke`, `.debounce_cancel`, `.debounce_is_pending` | no |
| `tmrextra` — `TmrStopwatch`, `TmrBucket`, `TmrTake` | types only |
| `tmrextra.stopwatch_new`, `.stopwatch_start`, `.stopwatch_stop`, `.stopwatch_reset`, `.stopwatch_elapsed`, `.stopwatch_is_running` | no |
| `tmrextra.bucket_new`, `.bucket_per_interval`, `.bucket_take`, `.bucket_tokens`, `.bucket_wait_for`, `.bucket_refill`, `.bucket_drain` | no |

No dependencies.  A duration is the standard library's `Duration`,
which a package may not redeclare, so there is nothing to depend on for
the one type that crosses this surface.
