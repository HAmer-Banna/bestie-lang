# `bestie.framework.schedule`

Scheduled jobs for backend services — cron expressions, fixed intervals, retries, and graceful shutdown — in the spirit of Spring's `@Scheduled` and Quartz, built on fibers.

```toml
[dependencies]
"bestie/schedule" = "1.0"
```

```bestie
import bestie.framework.schedule
```

## Purpose

Almost every backend runs work on a clock: expiring sessions, sending digests, refreshing caches, compacting tables. `schedule` covers that with a few annotations and a small runtime, and it is a second example — after `bestie/web` — of a framework built entirely on processors (`core/metaprogramming.md`).

It is an **in-process** scheduler. It is not a distributed task queue like Celery or Sidekiq: those need a broker, a persistence model, and a worker fleet, which are infrastructure choices, not language ones. `schedule` defines the `JobStore` protocol (§7) so third-party packages can add Redis- or Postgres-backed coordination, and so a fleet of instances can run each job once per tick.

## Layering

| Depends on | For |
| ---------- | --- |
| `core` | Processors, `T ! E`, ownership |
| `bestie.lib.concurrency` | Fibers, `fiber.sleep`, cancellation |
| `bestie.lib.datetime` | `Instant`, `Duration`, `TimeZone` |

## 1. Declaring Jobs

```bestie
import bestie.framework.schedule

class Maintenance {
    val ref sessions: SessionStore
    val ref mailer:   Mailer

    @Every("5m")
    fun expireSessions(): void ! DbError {
        try sessions.deleteExpired(Instant.now())
    }

    @Scheduled(cron = "0 0 7 * * MON-FRI", zone = "Europe/Berlin")
    @Retry(times = 3, backoff = "30s")
    fun sendDigest(ctx: ptr<JobContext>): void ! MailError {
        try mailer.sendDailyDigest(since: ctx.val.previousRun ?: Instant.now() - Duration.new(86_400 as Seconds))
    }
}

fun main() {
    val own sessions = SessionStore.new(db: openDb())
    val own mailer   = Mailer.new(smtp: smtpConfig())
    val own jobs     = Maintenance.new(sessions: sessions, mailer: mailer)

    val own scheduler = Scheduler.new()
    scheduler.onFailure((run: ptr<const JobRun>) => log("job ${run.val.name} failed: ${run.val.error}"))
    scheduler.mount(jobs.address())
    scheduler.start()

    // … run bestie/web or other work here …

    scheduler.shutdown(grace: Duration.new(30 as Seconds))
}
```

A job is a method on an ordinary class. Its dependencies are `ref` fields, wired in `main` like a `bestie/web` controller.

## 2. Annotations

| Annotation | Target | Effect |
| ---------- | ------ | ------ |
| `@Every(period: str)` | method | Runs every `period`: `"500ms"`, `"30s"`, `"5m"`, `"2h"`, `"1d"` |
| `@Scheduled(cron: str, zone: str = "UTC")` | method | Runs on a cron schedule in the given time zone |
| `@InitialDelay(delay: str)` | method | Waits before the first run |
| `@Retry(times: int, backoff: str, maxBackoff: str = "")` | method | Retries a failed run with exponential backoff |
| `@Overlap(policy: Overlap)` | method | What to do if the previous run is still going (§4). Default `Skip` |
| `@Misfire(policy: Misfire)` | method | What to do with runs missed while the process was paused or busy (§4). Default `RunOnce` |
| `@Timeout(limit: str)` | method | Cancels a run that takes longer than `limit` |

**Checked at compile time**, at the annotation:

* The cron expression and every duration string are parsed by the processor. `@Every("5 min")` or a cron with 61 in the seconds field is a compile-time error that shows where the expression is wrong.
* The time zone name must be a known zone id.
* A job method takes no parameters, or one `ptr<JobContext>`, and returns `void` or `void ! E`.
* `@Every` and `@Scheduled` on the same method is an error.

### Cron Syntax

Six fields — `second minute hour day-of-month month day-of-week` — or five, with the seconds field omitted and taken as `0`. Fields accept numbers, ranges (`1-5`), lists (`1,15`), steps (`*/10`), and names (`MON`, `JAN`). The shorthands `@hourly`, `@daily`, `@weekly`, and `@monthly` are accepted.

## 3. Jobs Without Annotations

```bestie
scheduler.every(Duration.new(10 as Seconds), () => metrics.flush())
scheduler.cron("0 */15 * * * *", () => cache.refresh()) catch |e| panic(e.toStr())
scheduler.once(at: deadline, () => report.close())
```

A run-time cron string can be wrong, so `scheduler.cron` returns `void ! CronError`. The annotation form never fails at run time because its expression was checked when the program was compiled.

## 4. Execution Model

* `scheduler.start()` starts one fiber that sleeps until the next due run (`fiber.sleep`). It does not poll.
* Each run executes in its own fiber, so a slow job never delays another job's start.
* **Intervals** (`@Every`) are measured on the monotonic clock, from the start of one run to the start of the next. Changing the system time does not move them.
* **Cron** schedules follow the wall clock in their zone. When a daylight-saving change skips a time, that run happens at the first valid time after it. When a time repeats, the run happens once, at its first occurrence.

**Overlap** — the previous run of the same job is still going:

| `Overlap` | Effect |
| --------- | ------ |
| `Skip` (default) | This run is skipped and reported |
| `Queue` | This run starts as soon as the previous one ends; at most one is queued |
| `Allow` | Runs concurrently. The job's state must then be safe to share between fibers |

**Misfire** — a run's time passed while the process was suspended or the scheduler was stopped:

| `Misfire` | Effect |
| --------- | ------ |
| `RunOnce` (default) | Run once now, however many were missed |
| `Skip` | Wait for the next scheduled time |
| `CatchUp` | Run every missed time, in order |

**`JobContext`** gives a run its scheduled time, the actual start time, `previousRun: Instant ?` (absent on the first run), the attempt number under `@Retry`, and the `CancellationToken` that `@Timeout` and `shutdown` use.

## 5. Failures

* A run that returns an error is retried according to `@Retry`, with the attempt number in `JobContext`.
* After the last attempt, the scheduler calls the `onFailure` handler with a `JobRun` describing the job, the error, and the attempts, then waits for the next scheduled time. One failing run never stops the schedule.
* A **panic** in a job ends the process, as everywhere in Bestie (`core/exceptions.md`). Jobs that call unreliable code belong in a supervised worker process (`vision.md` E4).

## 6. Shutdown

`scheduler.shutdown(grace)` stops starting new runs, signals every running job's `CancellationToken`, and waits up to `grace` for them to return. Jobs still running after `grace` have their fibers cancelled. `bestie/web`'s `SIGTERM` handling can call it, so a deploy finishes in-flight jobs before the process exits.

## 7. `JobStore` — Persistence and Coordination

By default the scheduler keeps its state in memory: last-run times are lost on restart, and every process runs every job. A `JobStore` changes both:

```bestie
protocol JobStore {
    // Claim the run of `job` at `scheduledAt` for this instance, for at most `lease`.
    // Returns false if another instance already holds it.
    fun tryAcquire(job: str, scheduledAt: Instant, lease: Duration): bool ! StoreError
    fun complete(job: str, scheduledAt: Instant, outcome: RunOutcome): void ! StoreError
    fun lastRun(job: str): Instant ? ! StoreError
}

val own scheduler = Scheduler.new(store: PostgresJobStore.new(pool: pool))
```

* With a shared store, several instances of a service run each job **once per scheduled time**: the instance whose `tryAcquire` succeeds runs it, and the others skip. No leader election is needed.
* A lease that expires without `complete` — the instance died — makes the run available again, so `Misfire` applies to it.
* `lastRun` survives restarts, so `previousRun` and `Misfire.CatchUp` work across deploys.
* `bestie/schedule` ships only `MemoryStore`. Store implementations for Redis, PostgreSQL, and others are third-party packages, written against this protocol.

## 8. Testing

`Scheduler.new(clock: TestClock.new(start))` replaces real time. `clock.advance(Duration.new(300 as Seconds))` runs every job that became due, in order, in the test's fiber, so scheduled behaviour is tested deterministically and instantly with `bestie.lib.test`.

## 9. What Is Generated

For `Maintenance`, the processor emits a constant job table and a `Scheduler.mount` overload:

```bestie
// @generated by bestie.framework.schedule for Maintenance

fun Scheduler.mount(c: ptr<Maintenance>) {
    this.add(JobSpec.every("Maintenance.expireSessions", Duration.new(300 as Seconds), Overlap.Skip, Misfire.RunOnce),
             c, Maintenance_expireSessions)
    this.add(JobSpec.cron("Maintenance.sendDigest", CRON_0, zone("Europe/Berlin"), Overlap.Skip, Misfire.RunOnce,
                          retry: RetryPolicy.new(times: 3, backoff: Duration.new(30 as Seconds))),
             c, Maintenance_sendDigest)
}

const CRON_0 = CronSchedule.parsed(…)    // the expression, already parsed into bit sets
```

Cron expressions are parsed at compile time into bit sets, so finding the next run time is a few bit operations with no string parsing at run time.

## 10. Non-Goals

* No distributed task queue, broker, or worker fleet — third-party packages can build one on `JobStore`
* No job persistence format or admin UI in the framework
* No hidden thread pool — runs are fibers, and their number is bounded by the jobs you declare
* No catching panics — a panic ends the process, as everywhere in Bestie
