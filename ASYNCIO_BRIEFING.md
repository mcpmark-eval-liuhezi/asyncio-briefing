# Asyncio Brown-Bag Briefing: Off Legacy Event-Loop Code, Onto Modern asyncio

**Prepared for:** team brown-bag session, week of Mon 2026-09-21 – Sun 2026-09-27
**Sources:** Python 3.14.7 official documentation (docs.python.org, retrieved 2026-09-17), open issues in `python/cpython` on GitHub (retrieved 2026-09-17), Google Calendar (retrieved 2026-09-17).

---

## Part 1 — Current APIs

Per the official docs (Runners / Event loop / Coroutines and tasks pages, Python 3.14.7), application developers should use the **high-level** asyncio functions and "should rarely need to reference the loop object or call its methods." The loop is managed for you.

### The entry points

**`asyncio.run(coro, *, debug=None, loop_factory=None)`**
- Executes the awaitable in a new event loop and returns the result; manages the loop, finalizes async generators, and closes the executor.
- Cannot be called when another asyncio event loop is running in the same thread.
- Should be used as the **main entry point** and ideally called **once**.
- Added in 3.7; `loop_factory` added in 3.12; in 3.14 `coro` can be any awaitable object.
- Docs recommend `loop_factory` (instead of policies) to configure the event loop, e.g. passing `asyncio.EventLoop` to run asyncio without the policy system.

**`asyncio.Runner(*, debug=None, loop_factory=None)`** (context manager, added in 3.11)
- Use when **several top-level async functions** must run in the **same event loop and `contextvars.Context`**.
- `Runner.run(coro, *, context=None)` — runs the awaitable in the embedded loop (coroutines are wrapped in a Task); `context` allows a custom `contextvars.Context`.
- `Runner.close()` — finalizes async generators, shuts down the default executor, closes the loop, releases the context.
- `Runner.get_loop()` — returns the associated loop.
- Also installs a `SIGINT` handler: Ctrl-C cancels the main task so `try/finally` cleanup runs, then raises `KeyboardInterrupt`.

**Inside running code (no loop objects needed):**
- `await` coroutines directly.
- `asyncio.create_task(coro)` — schedule a coroutine to run concurrently as a Task.
- `asyncio.TaskGroup` (3.11) — the modern alternative to `create_task` for structured concurrency: `async with asyncio.TaskGroup() as tg: tg.create_task(...)`.

**Loop access, when genuinely needed (libraries/frameworks only):**
- `asyncio.get_running_loop()` — returns the running loop in the current OS thread; raises `RuntimeError` if none. **Preferred** inside coroutines and callbacks.
- `asyncio.get_event_loop()` — from a coroutine/callback returns the running loop; otherwise (3.14+) raises `RuntimeError` if there is no current event loop. Avoid in new code.
- `asyncio.set_event_loop(loop)`, `asyncio.new_event_loop()` — low-level, only for embedding/embedded-loop scenarios.

### Minimal runnable example (from the official docs, Runners page)

```python
import asyncio

async def main():
    await asyncio.sleep(1)
    print('hello')

asyncio.run(main())
```

The `asyncio.Runner` equivalent, for multiple calls in one loop/context:

```python
with asyncio.Runner() as runner:
    runner.run(main())
```

### Deprecated or removed — stop using these

**Removed: the `loop=` parameter.** Most asyncio APIs no longer accept a `loop` keyword (deprecated since 3.8, removed in 3.10). Every place our legacy code passes a loop object into `asyncio.sleep()`, `asyncio.gather()`, `asyncio.wait()`, etc. is dead code on modern Python. Use `asyncio.get_running_loop()` inside the coroutine if you truly need the loop.

**Deprecated since 3.14, removal in 3.16: the entire policy system.** Per the official docs ("Policies" page: *"Policies are deprecated and will be removed in Python 3.16"*):

- `asyncio.get_event_loop_policy()`
- `asyncio.set_event_loop_policy()`
- `asyncio.AbstractEventLoopPolicy`
- `asyncio.DefaultEventLoopPolicy`
- `asyncio.WindowsSelectorEventLoopPolicy`
- `asyncio.WindowsProactorEventLoopPolicy`

**Migration:** choose the loop implementation with `loop_factory` passed to `asyncio.run()` or `asyncio.Runner` (e.g. `asyncio.run(main(), loop_factory=asyncio.EventLoop)`), not by installing a policy.

**Also deprecated (3.14): `asyncio.get_event_loop()` raises `RuntimeError` when there is no current loop**, and its behavior is documented as complex; prefer `get_running_loop()` in coroutines/callbacks and `asyncio.run()` at the top level.

---

## Part 2 — Gotchas: open issues in `python/cpython`

Searches performed on 2026-09-17 (queries: `is:open asyncio event loop`; `is:open asyncio.run coroutine`; `is:open asyncio loop parameter deprecated`). Issues found, reported exactly as returned:

| # | Title |
|---|-------|
| [#157301](https://github.com/python/cpython/issues/157301) | asyncio event loop hangs if an eager task fails to start |
| [#157299](https://github.com/python/cpython/issues/157299) | asyncio eager tasks have no awaited_by edge during their first step |
| [#157507](https://github.com/python/cpython/issues/157507) | `asyncio.print_call_graph()` truncates the stack at a gen-based coroutine |
| [#157044](https://github.com/python/cpython/issues/157044) | `asyncio.print_call_graph()` loses the call stack at `aiter(callable, stop_value)` |

Relevance notes:
- **#157301** is the most relevant to our migration: an eagerly-started task whose context setup fails can **hang the whole event loop** instead of raising a clean `TypeError` — a sharp edge if we adopt `eager_start`/eager task factories while modernizing.
- **#157299, #157507, #157044** are diagnostics/observability issues (await-graph reporting) — worth knowing if we lean on `asyncio.print_call_graph()` for debugging the migrated code.
- The search `is:open asyncio loop parameter deprecated` returned only one unrelated issue (#134082, about docstrings) — no open issue specifically about the removed `loop=` parameter.

---

## Part 3 — Scheduling: busy slots for the week of Mon 2026-09-21 – Sun 2026-09-27

Committed events on the calendar for that week (times as returned by Google Calendar; the events are stored as recurring "Kids Dropoff" instances at 11:30–12:00 UTC, shown as 19:30–20:00 +08:00 / 07:30–08:00 America/New_York):

| Day | Time (UTC) | Event |
|-----|------------|-------|
| Mon 2026-09-21 | 11:30 – 12:00 | Kids Dropoff |
| Wed 2026-09-23 | 11:30 – 12:00 | Kids Dropoff |
| Fri 2026-09-25 | 11:30 – 12:00 | Kids Dropoff |

Note: the recurring series also has an instance on Mon 2026-09-28 (11:30–12:00 UTC), which falls in the *following* week.

**Everything else that week is free** — no other confirmed events, so the brown-bag can be scheduled around the three 30-minute dropoff windows above.
