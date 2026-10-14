# timer-nv

A program with one loop and several deadlines — a cursor that blinks, a
status bar that refreshes, a re-parse that waits for the typing to stop
— needs somewhere to keep them all. This package is a **timer wheel**: a
collection of deadlines that the program polls rather than one that
fires by itself. It answers two questions each time round the loop, how
long the next wait may last and which timers have come due, so every
timer shares one clock reading and costs no wake-up of its own. The
design follows [Tokio](https://docs.rs/tokio/latest/tokio/time/)'s
`time` module and Python's
[asyncio](https://docs.python.org/3/library/asyncio-eventloop.html)
event loop, with the firing moved from the runtime to the caller.

## What it is

A **timer** here is a deadline, a tag and a rule for what happens when
the deadline passes. The tag is an `Int` the caller chose; it is what
the wheel hands back, and the caller matches on it. A **one-shot** timer
fires once and is removed. A **periodic** timer carries a period and is
rescheduled each time it fires.

A **wheel** is the collection of them. It is a value: every call that
changes one returns a new wheel, and the caller keeps it. Scheduling
answers an identifier, and the identifier is meaningful only in the
wheel that issued it.

The wheel does not fire anything by itself. A program calls
`tmrwheel.timeout_ms` to find out how long it may wait, waits in
whatever it was already waiting in — a `poll`, a terminal read, a socket
select — and then calls `tmrwheel.fire_due` with a fresh clock reading
to collect what came due. Nothing in this package sleeps, and nothing
starts a thread or a task.

Two clocks are involved and they are two different types. A **wall
clock** answers what time it is, in milliseconds since 1970-01-01
00:00:00 UTC. It is allowed to jump in either direction: NTP steps it, a
user sets it, a virtual machine resuming from a snapshot finds it hours
out. A **monotonic clock** answers how long since some arbitrary moment
in this process. Its origin means nothing, it never jumps and it never
runs backwards. `TmrWallTime` and `TmrMonoTime` are those two, and the
wheel accepts only `TmrMonoTime`.

Three functions read a clock. Everything else in the package —
scheduling, polling, the policies, the jitter, the stopwatch, the token
bucket — takes the current time as an argument and declares no effects.
That is what lets a test drive a whole hour of a schedule without
waiting, and it is why a caller that never calls those three functions
is charged no effects by using this package.

## Install

```
novo pkg add timer-nv
```

## Example

```novo
use std.time
use tmrclock
use tmrwheel

// The caller's own tags, handed back by `fire_due`.
const BLINK: Int = 1
const STATUS: Int = 2

fn main() [io, time]
    // One clock reading to build the schedule from.
    let start = tmrclock.mono_now()
    var w = tmrwheel.wheel_new()

    // A cursor that blinks twice a second. `TmrCoalesce` because a
    // missed blink is not worth catching up on.
    let blink = tmrwheel.schedule_with(w, tmrclock.mono_add(start, Duration.of_ms(530)),
                                       TmrPeriodic(Duration.of_ms(530)), BLINK,
                                       TmrCoalesce, tmrwheel.no_jitter())
    w = blink.wheel

    // A status bar on the second. `TmrSkip` keeps it on the original
    // grid, so it does not drift when the loop is busy.
    let status = tmrwheel.schedule_with(w, tmrclock.mono_add(start, Duration.of_secs(1)),
                                        TmrPeriodic(Duration.of_secs(1)), STATUS,
                                        TmrSkip, tmrwheel.no_jitter())
    w = status.wheel

    while true
        // How long the next wait may last: -1 for "no deadline",
        // 0 for "something is already due", milliseconds otherwise.
        let wait = tmrwheel.timeout_ms(w, tmrclock.mono_now())

        // …hand `wait` to the poll this program was going to make
        // anyway, and read whatever arrived…

        // One clock reading, and everything due at it.
        let step = tmrwheel.fire_due(w, tmrclock.mono_now())
        w = step.wheel
        for entry in step.fired
            if entry.tag == BLINK
                println("blink")
            else if entry.tag == STATUS
                println("status")
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `tmrclock` | The two clocks: a monotonic time, a wall time, the pair of them read together, the three functions that read a clock, the arithmetic that does not, and the conversion from a wall-clock deadline to a monotonic one with the check that says when it has gone stale. |
| `tmrwheel` | The wheel: one-shot and periodic timers, the three lateness policies, the deterministic spread, scheduling and cancelling, the two functions an event loop calls, the readers over what is scheduled, and a debounce built from them. |
| `tmrextra` | Two things built on the same clock: a stopwatch that can be paused without losing what it had, and a token bucket that refills by calculation rather than by ticking. |

## How to choose an entry point

**`tmrwheel.schedule_after` is the ordinary call.** Almost every
deadline is "in so long" rather than "at that moment", so it takes the
current time and a duration.

**`tmrwheel.schedule` takes the deadline itself**, for a caller that has
already computed one.

**`tmrwheel.schedule_with` spells out the policy and the spread.** Take
it for a periodic timer, where the lateness policy is a decision only
the caller can make, and wherever jitter is wanted.

**`tmrwheel.timeout_ms` and `.fire_due` are the pair a loop calls.**
`timeout_ms` before the wait, `fire_due` after it, with one clock
reading each. `tmrwheel.next_due` answers the deadline itself for a
caller that needs it rather than a millisecond count.

**`tmrwheel.debounce_new` and `.debounce_poke` are a wheel entry that
replaces itself.** Take them when something should happen once the
events stop rather than on each event: a re-parse after typing, a search
after a query settles, a rebuild after a burst of file writes.

**`tmrclock.mono_now` is the clock a deadline is measured on** and
`tmrclock.wall_now` is the one a person means. See rule 1.

## The rules a user needs

1. **A timeout belongs on the monotonic clock, and the wheel accepts
   nothing else.** A lease computed as `wall_now() + 30s` expires
   immediately when the system clock steps forward and never when it
   steps backward. `TmrMonoTime` and `TmrWallTime` are two types so
   that the mistake does not compile, and there is no overload of
   `schedule` that takes a wall time.
2. **An alarm that means a moment in the day is converted once, and the
   conversion goes stale.** `tmrclock.clock_offset` reads both clocks
   together, and `tmrclock.wall_deadline_to_mono` turns a wall-clock
   deadline into a monotonic one against that pair. The answer is
   correct for as long as the two clocks keep the relationship they had.
   `tmrclock.clock_step` says how far they have moved apart since, and
   `tmrclock.clock_stepped` compares that against a tolerance the caller
   gives. The tolerance is not optional: NTP's ordinary behaviour is to
   **slew**, to run the clock slightly fast or slow until it agrees, and
   a program that rescheduled on every millisecond of drift would
   reschedule constantly and for no reason.
3. **`timeout_ms` uses `poll`'s convention.** `TMR_NO_TIMEOUT` is `-1`
   and means nothing is scheduled, so block indefinitely. `0` means
   something is already due. Anything else is milliseconds. It is the
   same convention `ptypoll.wait_ready` and `io.wait_either` take, so
   the number goes straight through with no arithmetic in between.
4. **A handler is a tag, not a function.** Every timer carries an `Int`
   the caller chose, and `fire_due` hands the entries back for the
   caller to match on. Nothing in the wheel calls anything.
5. **Firing order is deterministic: deadline order, and ties in schedule
   order.** A program whose timers depend on one another gets the same
   sequence every run.
6. **A periodic timer that is late has three honest answers, and the
   caller picks one.** A loop busy for two seconds with a
   hundred-millisecond timer owes twenty firings.

   | `TmrPolicy` | What it does | Who wants it |
   | --- | --- | --- |
   | `TmrCatchUp` | fires once per missed period, keeping the original grid | anything where each firing means something that must not be lost: a sampler, a frame counter, a retry budget |
   | `TmrCoalesce` | fires once however many were missed, and reschedules from now | anything idempotent: a redraw, a status refresh, a cache sweep |
   | `TmrSkip` | fires once, and reschedules on the **original** grid | a clock display, which should tick on the second and would otherwise drift a little further off it every time the loop was busy |

7. **`TmrEntry.periods` says how many periods a firing covered.** It is
   1 for a one-shot and for a periodic timer that was on time, and more
   under `TmrCoalesce`, so a caller that is counting something still
   knows what it missed.
8. **The jitter is deterministic and must not be used as randomness.**
   The spread comes from a seed the wheel advances, which keeps the
   module free of effects and makes a schedule reproducible in a test.
   It solves the case where a hundred connections refreshing every
   thirty seconds wake together — the classic name for that is a
   thundering herd — and it solves nothing that needs to be unguessable.
9. **A debounce must cancel before it reschedules, and
   `debounce_poke` does.** Scheduling a new timer on every event without
   cancelling the last one turns a debounce into a queue: a fast typist
   gets one re-parse per keystroke and all of them run.
10. **Nothing in this package can fail.** There is no error type.
    Cancelling an identifier the wheel does not have is a no-op, which
    includes one that has already fired, so a caller racing a cancel
    against a one-shot need not know which won. Being rate-limited is an
    answer with a duration in it and not an error.
11. **Only three functions read a clock**: `tmrclock.mono_now`,
    `.wall_now` and `.clock_offset`, each declaring `[time]`. Everything
    else takes `now` as an argument and declares nothing. A consumer
    that never calls those three is charged no effects, and the effect
    checker holds it to that on its own signatures.
12. **`clock_offset` reads both clocks in one call**, so a step
    happening between two separate reads cannot put the pair out by
    exactly the amount being measured.
13. **The token bucket refills by calculation, not by ticking.**
    `bucket_take` works out how much has refilled since the last call
    from the `now` it was handed. A bucket nobody touches for an hour
    costs nothing and is correct when it is next asked. A refused take
    spends nothing, so a caller that keeps asking does not drain it.
14. **A bucket holds thousandths of a token internally**, so that a rate
    slower than one per second is expressible: four an hour is
    `bucket_per_interval(4, Duration.of_secs(3600), now)`.
    `bucket_tokens` answers whole tokens. A new bucket starts full.
15. **Starting an already-running stopwatch does nothing**, rather than
    resetting its start and losing the elapsed time.

## Where this sits beside sleeping and the async runtime

`core.rt.sleep_ms` is one deadline for one place in the code. It is the
right tool when a piece of code wants to wait: a retry backoff, a poll
interval in a script. It is the wrong tool for a program with five
independent deadlines, because a sleep can only be waiting for one of
them, and that is the one it cannot be interrupted for.

The async runtime gives `await`, `all` and `select` over futures, and
the natural shape there is one task per timer. It wants a runtime, a
task per timer, and the caller's state reachable from each of them.

This package is for the program in between: one loop, many deadlines and
no runtime. A terminal program's pump is already blocked in a `poll` on
a pseudoterminal and a keyboard, and sleeping is exactly what it must
not do while input might arrive. A program that has a runtime and also
wants many deadlines in one place can use both, because the wheel
declares no effects and composes inside a task exactly as it does inside
a loop.

## What is not included

- **Sleeping.** A library that slept would be a library a program cannot
  interrupt.
- **Threads, tasks and signals.** Nothing here spawns.
- **Parsing a duration.** `interval = "5m"` in a configuration file is
  [humantime-nv](https://novo-lang.org/packages/humantime-nv)'s work,
  and its answer is one `Duration.of_secs` away from what
  `schedule_after` takes. A program that builds its intervals in code
  should not carry a duration parser it never calls.
- **Formatting a duration.** `Duration.to_str` is in the standard
  library.
- **Calendars.** "The first Monday of the month" is
  [calendar-nv](https://novo-lang.org/packages/calendar-nv)'s arithmetic
  and [cron-nv](https://novo-lang.org/packages/cron-nv)'s schedule. When
  they have answered, the result is one `TmrWallTime` and this package
  takes it from there.
- **Running on a microcontroller.** No such claim is made. The package
  is a host one because three of its functions read a clock, and the
  wheel itself is not built for a device today.

## Related packages

- [humantime-nv](https://novo-lang.org/packages/humantime-nv) turns
  `"5m"` from a configuration file into a length of time. It is a
  dependency of the program, not of this package.
- [cron-nv](https://novo-lang.org/packages/cron-nv) answers when a
  crontab expression fires next, in a time zone. That answer is a moment
  in the day, so it reaches this package as a `TmrWallTime` through
  `tmrclock.wall_deadline_to_mono`.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is civil
  date arithmetic. This package does none.
- [chrono-nv](https://novo-lang.org/packages/chrono-nv) reads the clock
  and formats an instant for a person. This package reads the clock only
  to measure with.
- `std.time` in the standard library owns `Duration`, which is the type
  every interval and every tolerance here is written in. A package may
  not redeclare it, which is why this package has no dependencies at
  all.

## Tests

```bash
novo test --isolate tests/tmrwheel_tests.nv  # 20 tests: scheduling, polling, the policies, the debounce
novo test --isolate tests/tmrextra_tests.nv  # 14 tests: the stopwatch and the token bucket
novo test --isolate tests/tmrclock_tests.nv  # 12 tests: the two clocks and the conversion between them
novo test --isolate tests/timer_edge_tests.nv # 7 tests: the boundaries the API suites leave out
```

Tokio's `time` module is the reference for the wheel and the interval
policies, and its `MissedTickBehavior` is `TmrPolicy` with `Burst`,
`Delay` and `Skip` renamed. Python's `asyncio` event loop is the
reference for what a loop's single pass needs from a timer collection.
`governor` and Go's `rate` package are the reference for the token
bucket.

One test reads a clock, and only to check that the three reading
functions can be called. Every other assertion builds its times with
`tmrclock.mono_of_ms`, which is what lets the suite drive a busy loop,
twenty missed periods and an NTP step without waiting for any of them.
`tmrwheel_tests.nv` asserts the three lateness policies against the same
missed-period case, that `timeout_ms` answers `-1`, `0` and a count in
the three situations that produce them, that firing order is by deadline
with ties in schedule order, that the same seed gives the same jittered
schedule, and that a debounce poked repeatedly keeps moving its one
timer. `tmrclock_tests.nv` asserts that a clock set forward or back by
an hour is a step and that a slew is not.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
