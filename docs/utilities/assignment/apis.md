# Assignment API Reference

This page documents every public function, method, and type surface exposed by the **Assignment** library.

---

## Class: Assignment

The top-level module table. All scheduling functions are accessed directly on this table.

### Summary

**Pooled Family** — protect the server. These functions recycle execution resources, throttle heavy workloads automatically, and gracefully discard cancelled work before it ever runs.

| Name | Returns | Description |
|:---|:---|:---|
| **[Spawn](#spawn)** | `void` | Runs a function or thread immediately. |
| **[Defer](#defer)** | `Handle` | Queues work to run on the next available frame. Large backlogs are spread across frames automatically to protect frame time. |
| **[Delay](#delay)** | `Handle` | Schedules work to run after a set duration. Zero or negative durations queue on the next available frame. |

**Isolated Family** — exert absolute control. These functions bypass all throttles, give you a cancellation token (`thread`) to stop work at any time, and force the engine to prioritise your logic immediately.

| Name | Returns | Description |
|:---|:---|:---|
| **[SpawnIsolated](#spawnisolated)** | `thread` | Runs a function or thread immediately at full engine priority. Use for long-yielding work. |
| **[DeferIsolated](#deferisolated)** | `thread` | Queues work for the next available frame at full engine priority, bypassing frame-spread throttling. Use for long-yielding deferred work. |
| **[DelayIsolated](#delayisolated)** | `thread` | Schedules work after a set duration at full engine priority. Use for long-yielding delayed work. |

**Shared**

| Name | Returns | Description |
|:---|:---|:---|
| **[Wait](#wait)** | `number` | Pauses the current thread for a duration and returns the real elapsed time. |
| **[Repeat](#repeat)** | `Handle` | Calls a function repeatedly at a fixed interval, for a set number of iterations or indefinitely. |
| **[Cancel](#cancel)** | `void` | Stops a scheduled `Handle` or a cancellation `thread` from executing. |
| **[Migrate](#migrate)** | `void` | Hands all pending scheduled work to the engine's native scheduler. Server-only. |

---

#### Spawn

```luau
Assignment.Spawn(Function: ((...any) -> ()) | thread, ...: any): ()
```

Runs a function or resumes a thread immediately.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume immediately. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:** `void`

!!! info "Best For Short, Quick Work"
    The Pooled Family is designed for tasks that start and finish quickly, or yield only briefly. If your work will sleep for a long time — waiting on a DataStore, an HTTP request, or a multi-second loop — use `SpawnIsolated` instead. Long sleeps occupy the shared worker and force all other incoming short tasks to spin up their own resources rather than sharing the available one.

---

#### SpawnIsolated

```luau
Assignment.SpawnIsolated(Function: ((...any) -> ()) | thread, ...: any): thread
```

Runs a function or thread immediately at full engine priority, completely independent of the shared worker.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume immediately. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:**

| Type | Description |
|:---|:---|
| `thread` | The dedicated thread. Pass to `Assignment.Cancel` to stop it. |

!!! info "The Isolated Family — Absolute Control"
    `SpawnIsolated`, `DeferIsolated`, and `DelayIsolated` all give work its own dedicated engine-managed thread. Think of it as hiring a specialist who operates entirely independently — your regular team is never occupied by this job, no matter how long it runs.

    Use any of them when work will yield for a meaningful amount of time. The Pooled Family shares one reusable worker: a long-sleeping task keeps that worker busy for the entire sleep, leaving short incoming tasks without a free worker to reuse. Isolated threads sidestep this — the shared worker stays free at all times, and the engine manages the dedicated thread on its own.

!!! warning "Isolated Threads Trade Safety for Control"
    Isolated threads are not covered by Assignment's automatic throttling or cleanup. They run immediately at full engine priority, are never spread across frames, and are not automatically discarded if they go unused. Only use them when the work genuinely requires it.

---

#### Defer

```luau
Assignment.Defer(Function: ((...any) -> ()) | thread, ...: any): Handle
```

Queues work to run on the next available frame.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:**

| Type | Description |
|:---|:---|
| `Handle` | A cancellable handle. Call `Handle:Break()` or pass it to `Assignment.Cancel()` to discard the work before it runs. |

!!! tip "Automatic Frame Spreading"
    Unlike the engine's native defer, which processes everything queued in one frame all at once, Assignment meters deferred work across frames. Deferring a large batch of calls in one frame will not spike your frame time.

---

#### DeferIsolated

```luau
Assignment.DeferIsolated(Function: ((...any) -> ()) | thread, ...: any): thread
```

Queues work to run on the next available frame at full engine priority.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:**

| Type | Description |
|:---|:---|
| `thread` | The dedicated thread. Pass to `Assignment.Cancel` to stop it. |

!!! info "When to Use"
    Use `DeferIsolated` when the work you want to run next frame will then sleep for a long time — for example, a batch of DataStore writes or a chain of HTTP calls. Queueing that kind of work through the standard `Defer` path would occupy the shared pool for the entire duration of those yields. See `SpawnIsolated` for the full explanation.

---

#### Delay

```luau
Assignment.Delay(Duration: number, Function: ((...any) -> ()) | thread, ...: any): Handle
```

Schedules work to run after a set number of seconds.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Duration` | `number` | Seconds to wait before running. |
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:**

| Type | Description |
|:---|:---|
| `Handle` | A cancellable handle. Call `Handle:Break()` or pass it to `Assignment.Cancel()` to discard the work before it runs. |

!!! info "Zero and Negative Durations"
    Passing `0` or any negative number tells Assignment to run the work as soon as possible — on the next available frame, just like `Defer`. The returned `Handle` behaves identically regardless of which path was taken and is fully cancellable.

!!! tip "Timing Precision"
    Assignment fires delayed work as close to the requested time as the current frame rate allows. Very short durations may fire on the frame after the one they were scheduled in. For precise, immediate execution without any timing constraint, use `DelayIsolated`.

---

#### DelayIsolated

```luau
Assignment.DelayIsolated(Duration: number, Function: ((...any) -> ()) | thread, ...: any): thread
```

Schedules work to run after a set number of seconds at full engine priority.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Duration` | `number` | Seconds to wait before running. |
| `Function` | `((...any) -> ()) | thread` | The function to call or thread to resume. |
| `...` | `any` | Arguments forwarded to the function or thread. |

**Returns:**

| Type | Description |
|:---|:---|
| `thread` | The dedicated thread. Pass to `Assignment.Cancel` to stop it. |

!!! info "When to Use"
    Use `DelayIsolated` when the work that fires after the delay will then sleep for a long time — for example, an async multi-step workflow. Running that through the standard `Delay` path would occupy the shared pool for the duration of those yields. See `SpawnIsolated` for the full explanation.

---

#### Wait

```luau
Assignment.Wait(Duration: number?): number
```

Pauses the current thread for approximately the requested duration and returns how long it actually waited.

Passing zero, `nil`, or a negative value is valid: the thread pauses for the shortest possible time, resuming on the next frame.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Duration` | `number?` | Seconds to pause. |

**Returns:**

| Type | Description |
|:---|:---|
| `number` | The actual time elapsed since `Wait` was called, in seconds. |

!!! warning "This Function Yields"
    `Assignment.Wait` suspends the calling thread. Do not call it from a synchronous callback or any context that must return immediately, such as a signal handler.

!!! tip "The Returned Time Is Real, Not Requested"
    The return value reflects how long the thread actually slept, not the duration you asked for. Under heavy load the actual wait may exceed the requested duration. If you need precise elapsed time for animation or interpolation, measure with `os.clock()` directly.

!!! info "Zero, Nil, and Negative Values — Next Frame Yield"
    Passing `nil`, `0`, or any negative number is the lightest possible yield: the thread is paused for exactly one frame and then resumes. This is useful for letting other work run between steps of a long process without introducing a meaningful time delay.

---

#### Repeat

```luau
Assignment.Repeat(Count: number, Interval: number?, Function: (number, Handle, ...any) -> (), ...: any): Handle
```

Calls a function repeatedly at a fixed interval for a set number of iterations, or indefinitely when `Count` is `-1`. Each iteration receives the current iteration number (starting at 1), the loop's `Handle`, and any extra arguments you forward.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Count` | `number` | Number of times to call `Function`. Pass `-1` to loop forever. |
| `Interval` | `number?` | Seconds between calls. |
| `Function` | `(number, Handle, ...any) -> ()` | Called each iteration with the iteration number and the loop handle as the first two arguments. |
| `...` | `any` | Extra arguments forwarded to `Function` after the iteration number and handle. |

**Returns:**

| Type | Description |
|:---|:---|
| `Handle` | A cancellable handle. Call `Handle:Break()` or pass it to `Assignment.Cancel()` to discard the work before it runs or after execution when called mid-running. |

!!! warning "Yielding Inside the Callback Delays the Next Iteration"
    The loop pauses between iterations. If your callback also yields — for example, by calling `Assignment.Wait` itself — the next iteration is delayed by the full length of that yield on top of the configured interval.

!!! tip "Cancellation Is Immediate After the Current Callback"
    `Handle:Break()` is checked both before the callback runs and immediately after it returns. If you call `Break` from inside the callback, the loop exits before the next wait period — it does not wait out the remaining interval.

---

#### Cancel

```luau
Assignment.Cancel(Target: Handle | thread): ()
```

Stops a scheduled task from running. Accepts either a `Handle` returned from a pooled call or a `thread` token returned from an isolated call.

**Parameters:**

| Name | Type | Description |
|:---|:---|:---|
| `Target` | `Handle | thread` | The handle or thread to cancel. |

**Returns:** `void`

!!! tip "Cancel and Break Are Equivalent for Handles"
    `Assignment.Cancel(handle)` and `handle:Break()` do exactly the same thing when passed a `Handle`. Use `Cancel` when your code may hold either a `Handle` or a raw `thread` — it handles both uniformly, making it the right choice for generic cleanup systems.

---

#### Migrate

```luau
Assignment.Migrate(): ()
```

Signals Assignment to hand all pending scheduled work to the engine's native scheduler, preserving each task's remaining time. After migration, all Assignment functions continue to work as normal but route through the native scheduler.

This is called automatically when the server closes and requires no code from you. It is exposed as a manual call for controlled shutdown sequences.

**Parameters:** `void`

**Returns:** `void`

!!! note "Server Only"
    `Migrate` exists only on the server. The automatic shutdown binding is also server-only. Calling it on the client will error.

!!! warning "Cancellation Is No Longer Available After Migration"
    Once migration completes, Assignment's cancellation system is no longer active. Tasks that were not yet cancelled will run at their scheduled times and cannot be stopped by `Handle:Break()` or `Assignment.Cancel`.

---

## Class: Handle

A cancellation token returned by `Defer`, `Delay`, and `Repeat`. Holds a single cancellation flag and exposes one method.

### Summary

| Name | Returns | Description |
|:---|:---|:---|
| **[Break](#break)** | `void` | Marks this handle as cancelled so its scheduled work is silently discarded. |

---

#### Break

```luau
Handle:Break(): ()
```

Marks this handle as cancelled. When the scheduled time arrives, Assignment sees the cancellation flag and silently discards the associated work without running it. For `Repeat` loops, the flag is checked both before the callback runs and immediately after, so the loop exits as soon as you call `Break`.

**Parameters:** `void`

**Returns:** `void`

!!! tip "Safe to Call More Than Once"
    Calling `Break` on an already-cancelled handle does nothing. There is no error and no state change — it is safe to call defensively without checking first.