# Assignment Best Practices

---

## 1. Always Store Handles When Cancellation Is Possible

Any scheduled call that might need to be stopped requires you to keep its `Handle`. Once you discard it, there is no way to cancel the work — Assignment has no lookup by function reference.

### The Problem

```luau
-- Handle is thrown away immediately; the callback has no off switch
Assignment.Delay(5, function()
    destroyTower(tower) -- fires even if the tower is already gone
end)

tower.Destroying:Connect(function()
    -- Nothing to cancel here — destroyTower will run against a dead object
end)
```

**Why this is wrong:** With no handle in scope, there is nothing to call `Break` on. The scheduled work runs unconditionally when its time comes, regardless of whether the target still exists.

### The Solution

```luau
local destroyHandle = Assignment.Delay(5, function()
    destroyTower(tower)
end)

tower.Destroying:Connect(function()
    destroyHandle:Break() -- tells Assignment to silently skip this when the time comes
end)
```

**Why this works:** `Handle:Break()` marks the work as cancelled. When the scheduled time arrives, Assignment sees the flag and discards the task without running it. Calling `Break` early costs nothing and is always safe.

---

## 2. Avoid Using the Native `task` Library Inside Assignment-Managed Code

Assignment tracks all of the work it schedules so it can cancel it, throttle it, and migrate it on shutdown. When you call `task.*` functions directly from inside an Assignment-managed thread, that work steps outside Assignment's awareness entirely. The two schedulers end up sharing the same thread without either having the complete picture, which produces subtle bugs that are hard to reproduce.

### The Problem

```luau
Assignment.Spawn(function()
    task.wait(2)           -- Assignment loses track of this thread for 2 seconds;
                           -- cancellation and shutdown migration won't reach it here
    doWork()

    task.spawn(doMoreWork) -- this new thread is invisible to Assignment;
                           -- it won't be throttled, cancelled, or migrated
end)
```

**Why this is wrong:** When `task.wait` is called inside a thread Assignment spawned, the engine takes ownership of that thread for the duration of the wait. During that window, `Assignment.Cancel` cannot reach it — it simply finds nothing to cancel. When the thread wakes up it continues on the engine's terms, not Assignment's. Any work spawned with `task.spawn` inside an Assignment thread is similarly invisible: it bypasses throttling, will not be cancelled when its parent handle is broken, and will not be handed off safely on server shutdown.

### The Solution

```luau
Assignment.Spawn(function()
    Assignment.Wait(2)       -- Assignment keeps full track of this thread
    doWork()

    Assignment.Spawn(doMoreWork) -- runs through Assignment; tracked, throttled, cancellable
end)
```

**Why this works:** `Assignment.Wait` keeps the thread fully within Assignment's scheduling. When the wait is over, Assignment wakes it up correctly with the real elapsed time. Any new work dispatched via `Assignment.Spawn` stays inside Assignment's awareness, so cancellation and shutdown work as expected.

!!! info "`Assignment.Wait` Is Always the Right Yield"
    `Assignment.Wait` is context-aware — it does the right thing whether you are inside a pooled thread or an isolated thread. You never need to think about which one to use. Just call `Assignment.Wait` everywhere inside Assignment-managed code.

---

## 3. Use Isolated Variants for Long-Yielding Work — Not as a General Default

The Isolated Family (`SpawnIsolated`, `DeferIsolated`, `DelayIsolated`) is designed for **absolute control**: it bypasses all of Assignment's throttles, gives you a cancellation token to stop work at any moment, and forces the engine to prioritise your logic immediately. These are powerful properties — but they come with trade-offs that make isolated variants the wrong default for most scheduling work.

### Why This Matters

Think of the Pooled Family as sharing a single diligent worker among all your short tasks. That worker picks up a job, finishes it, and immediately takes the next one. Everything flows smoothly because jobs are short.

Now imagine a job that makes the worker sleep for 30 seconds — waiting on a DataStore call, an HTTP response, or a loop body with a long interval. While that worker sleeps, every other job that comes in has to bring in its own independent resource to get done. If several long-sleeping jobs run at the same time, you end up with a crowd of independent resources all active simultaneously: each one costs memory, and as the crowd grows the engine has to do progressively more work to manage them all, which eventually shows up as CPU pressure and frame drops.

Isolated variants sidestep this entirely. Because they bypass the shared worker and run in their own dedicated engine-managed thread, the regular worker stays free at all times and short jobs are handled efficiently. The long-sleeping work runs independently without affecting anything else.

### The Problem — Long Work Through the Shared Pool

```luau
-- A 30-second loop keeps the shared worker occupied for the entire sleep.
-- Every Spawn or Delay call during those 30 seconds has to spin up a
-- temporary thread instead of reusing the shared one.
Assignment.Spawn(function()
    while active do
        Assignment.Wait(30)
        syncRemoteData()
    end
end)

-- A DataStore load can take several seconds — same problem.
Assignment.Spawn(function()
    local data = DataService:Load(player)
    applyData(player, data)
end)
```

**Why this is wrong:** Every second the shared worker spends sleeping on a long yield, incoming short work — UI updates, cooldown resets, status effects — has to bring in its own independent resource rather than using the available shared one. Under concurrent load this compounds: independent resources pile up, memory climbs, and the engine's housekeeping work intensifies to keep up.

### The Solution — Isolated Variants for Long-Yielding Tasks

```luau
-- The 30-second loop is isolated. The engine manages it independently.
-- The shared worker is never touched by this task.
Assignment.SpawnIsolated(function()
    while active do
        Assignment.Wait(30) -- works correctly in isolated threads too
        syncRemoteData()
    end
end)

-- The DataStore load runs in its own thread, keeping the pool free.
Assignment.SpawnIsolated(function()
    local data = DataService:Load(player)
    Assignment.Wait(0) -- yield one frame before continuing; lets other work run
    applyData(player, data)
end)

-- Short work still belongs on the standard pool.
Assignment.Spawn(function()
    Assignment.Wait(0.1)
    applyStatusEffect(player, "Stunned")
end)
```

**Why this works:** The long-sleeping threads are owned by the engine from the moment of creation. The shared worker is never involved with them. Short work always finds the shared worker free and ready, so no additional independent resources are created and memory stays stable.

!!! warning "Isolated Calls Return a Cancellation Token, Not a Handle"
    Isolated calls return a raw `thread` token instead of a `Handle`. You cancel them with `Assignment.Cancel(thread)`. There is no `Handle:Break()` equivalent. If you need to cancel work before it runs by holding a `Handle`, use a pooled variant instead.

!!! info "How to Decide"
    A useful rule of thumb: if the work calls into a slow external system (DataStore, HTTP, Messaging Service) or sleeps for more than a second, reach for an isolated variant. If the work is brief — a short calculation, a quick delay, a few frames of waiting — the Pooled Family handles it cleanly with less overhead and full cancellation via `Handle`.

---

## 4. Pass Arguments Directly — Avoid Wrapping Them in a Closure

When you schedule a call with `Delay`, `Defer`, or `Spawn`, you can pass arguments directly after the function. Assignment stores those values alongside the scheduled job and hands them back when the time comes. Wrapping them in a closure instead creates a new allocation every time you schedule work, which adds up under high call volume.

### The Problem

```luau
-- A new anonymous function is allocated for every scheduled call
Assignment.Delay(2, function()
    applyDamage(playerId, damage)
end)
```

**Why this is wrong:** At high call volume — hundreds of projectile impacts per second, for example — each closure is a small allocation. Those allocations accumulate and drive the garbage collector to run more often, showing up as periodic CPU spikes.

### The Solution

```luau
-- Arguments are stored directly alongside the job — no extra allocation
Assignment.Delay(2, applyDamage, playerId, damage)
```

**Why this works:** Assignment stores the arguments directly as part of the scheduled job entry and passes them straight to your function when it fires. No intermediate table or closure is created. For calls with more than four arguments, Assignment still handles it efficiently using a pooled scratch table that is recycled after each call rather than allocated fresh.

---

## 5. Pair Infinite `Repeat` Loops With Explicit Break Logic

`Repeat` with a count of `-1` runs forever. The only way to stop it is to call `Handle:Break()` or `Assignment.Cancel`. If no code path ever does that, the loop runs until the server shuts down — even if it is doing nothing useful.

### The Problem

```luau
-- This loop has no exit path — it will run for the lifetime of the server
Assignment.Repeat(-1, 1, function(i, _handle)
    if roundActive then
        updateScoreboard(i)
    end
    -- When roundActive is false this does nothing, but keeps looping regardless
end)
```

**Why this is wrong:** An idle loop that never stops still consumes scheduler time every iteration — it wakes up, runs, waits, and wakes up again indefinitely. Over a long session these costs accumulate quietly.

### The Solution

```luau
local scoreHandle = Assignment.Repeat(-1, 1, function(_i, handle)
    if not roundActive then
        handle:Break() -- stops the loop after this iteration
        return
    end
    updateScoreboard()
end)

-- Also stop it from outside when the game mode ends
onGameModeEnd:Connect(function()
    scoreHandle:Break()
end)
```

**Why this works:** `Break` is checked both before the callback runs and immediately after it returns. Calling it inside the callback stops the loop before the next wait period — no partial interval is observed after cancellation.

---

## 6. Cancel Player-Scoped Handles When Players Leave

Any work scheduled in response to a player joining — cooldowns, respawn timers, buff durations — must be cancelled when that player disconnects. If it is not, the work will fire against a player that no longer exists, producing errors.

### The Problem

```luau
Players.PlayerAdded:Connect(function(player)
    Assignment.Delay(300, function()
        grantDailyBonus(player) -- player may have left long before this fires
    end)
    -- Handle is discarded; no way to cancel it on disconnect
end)
```

**Why this is wrong:** The callback captures a reference to `player`. If that player disconnects before 300 seconds pass, the scheduled work still fires and tries to operate on a destroyed object.

### The Solution

```luau
local playerHandles: { [Player]: { Assignment.Handle } } = {}

Players.PlayerAdded:Connect(function(player)
    local bonusHandle = Assignment.Delay(300, grantDailyBonus, player)
    playerHandles[player] = playerHandles[player] or {}
    table.insert(playerHandles[player], bonusHandle)
end)

Players.PlayerRemoving:Connect(function(player)
    local handles = playerHandles[player]
    if handles then
        for _, handle in handles do
            handle:Break()
        end
        playerHandles[player] = nil
    end
end)
```

**Why this works:** Every handle is stored under the player's key. When the player leaves, all of their handles are broken before the player object is removed, ensuring none of the associated callbacks can fire against a destroyed instance.

---

!!! success "Pro Tip — Use `Assignment.Cancel` in Generic Cleanup Systems"
    `handle:Break()` and `Assignment.Cancel(handle)` do the same thing for a `Handle`. The advantage of `Assignment.Cancel` is that it also accepts a raw `thread` returned from isolated calls, stopping it cleanly in either case. If you are building a general-purpose cleanup registry that might hold either type, `Assignment.Cancel` is the single call that handles both.
