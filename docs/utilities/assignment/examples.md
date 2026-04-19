# Assignment Examples

Practical, scenario-based examples for every public API in Assignment. Each section demonstrates realistic usage patterns including argument forwarding, cancellation, and edge cases.

---

## Spawn

**Pooled Family.** Runs work immediately. Recycled execution and automatic cleanup. Best for short, self-contained tasks.

**Scenario A — Fire-and-forget callback**
```luau
-- Run a non-blocking save without stalling the current frame.
-- Execution continues on the next line immediately.
Assignment.Spawn(function()
    DataService:SavePlayerData(player)
end)
```

**Scenario B — Resume an existing thread with arguments**
```luau
local pendingThread = coroutine.create(function(value: number)
    print("Resumed with:", value) --> "Resumed with: 42"
end)

Assignment.Spawn(pendingThread, 42)
```

**Scenario C — Brief yield inside a pooled call**
```luau
-- A short yield is fine here — the work finishes quickly
-- and the shared worker is free again as soon as it does.
Assignment.Spawn(function()
    Assignment.Wait(0.1)
    applyStatusEffect(player, "Stunned")
end)
```

---

## SpawnIsolated

**Isolated Family.** Runs work immediately at full engine priority in a dedicated thread. Bypasses all throttling. Returns a `thread` cancellation token. Use when work will sleep for a long time or involve repeated slow yields.

**Scenario A — Long-polling loop with multi-second intervals**
```luau
-- This loop sleeps for 30 seconds each iteration.
-- Running it through Spawn would keep the shared worker occupied
-- for that entire sleep, leaving all other short incoming work without
-- a free worker to reuse. SpawnIsolated gives this task its own
-- dedicated engine thread so the shared worker is never involved.
local serviceThread = Assignment.SpawnIsolated(function()
    while gameActive do
        Assignment.Wait(30)
        syncLeaderboardData()
    end
end)

onGameEnd:Connect(function()
    Assignment.Cancel(serviceThread)
end)
```

**Scenario B — Multi-step workflow with slow async calls**
```luau
-- Each DataStore call may take several seconds to complete.
-- Isolating this work keeps the shared worker free the entire time
-- so short tasks elsewhere in your code are never delayed.
Assignment.SpawnIsolated(function()
    local profileData = DataService:LoadProfile(player)
    Assignment.Wait(0) -- yield one frame between steps; lets other work breathe
    local inventoryData = DataService:LoadInventory(player)
    Assignment.Wait(0)
    applyLoadedData(player, profileData, inventoryData)
end)
```

**Scenario C — Isolated thread with external cancellation**
```luau
local monitorThread = Assignment.SpawnIsolated(function()
    while true do
        Assignment.Wait(5)
        checkServerHealth()
    end
end)

onServerShutdown:Connect(function()
    Assignment.Cancel(monitorThread)
end)
```

---

## Defer

**Pooled Family.** Queues work for the next available frame. Large backlogs are spread across frames automatically — no single frame is overwhelmed. Full `Handle`-based cancellation.

**Scenario A — Defer heavy setup out of module load**
```luau
-- Avoid blocking the current frame with expensive setup.
-- This runs on the very next frame after this code executes.
Assignment.Defer(function()
    buildLookupTable()
    cachePlayerData()
end)
```

**Scenario B — Cancel a deferred call before it fires**
```luau
local handle = Assignment.Defer(function()
    sendWelcomeNotification(player)
end)

-- Player disconnected before the next frame
if not player:IsDescendantOf(game) then
    handle:Break()
end
```

**Scenario C — Defer a thread with arguments**
```luau
local handshakeThread = coroutine.create(function(sessionId: string, token: string)
    validateHandshake(sessionId, token)
end)

local handle = Assignment.Defer(handshakeThread, sessionId, authToken)
```

**Scenario D — Spreading a large batch of work across frames**
```luau
-- Deferring many calls at once lets Assignment spread them across frames
-- rather than processing everything in one spike.
for _, system in ipairs(systemsToInit) do
    Assignment.Defer(system.Initialize, system)
end
```

---

## DeferIsolated

**Isolated Family.** Queues work for the next available frame at full engine priority. Bypasses frame-spread throttling. Returns a `thread` cancellation token. Use when the deferred work will then sleep for a long time.

**Scenario A — Deferred batch of slow async writes**
```luau
-- Each DataStore write may yield for several seconds.
-- DeferIsolated queues the batch to the next frame but gives it
-- a dedicated engine thread, so the shared worker is never occupied
-- during any of those writes.
Assignment.DeferIsolated(function()
    for _, record in ipairs(pendingRecords) do
        DataService:Write(record)
        Assignment.Wait(0) -- breathe between writes
    end
end)
```

**Scenario B — Deferred isolated call with cancellation**
```luau
local thread = Assignment.DeferIsolated(function()
    local result = HttpService:GetAsync(endpoint)
    processResult(result)
end)

-- If the request is no longer needed before the next frame fires
if requestAborted then
    Assignment.Cancel(thread)
end
```

---

## Delay

**Pooled Family.** Schedules work to run after a set duration. Cancelled work is silently discarded before it ever runs. Zero and negative durations queue on the next available frame, same as `Defer`.

**Scenario A — Ability cooldown reset**
```luau
setAbilityEnabled(player, "Dash", false)

local cooldownHandle = Assignment.Delay(5, function()
    setAbilityEnabled(player, "Dash", true)
end)

playerCooldowns[player] = cooldownHandle
```

**Scenario B — Cancel a delayed effect when conditions change**
```luau
local explosionHandle = Assignment.Delay(3, triggerExplosion, bombPosition)

onBombDefused:Connect(function()
    explosionHandle:Break()
    -- triggerExplosion will never run
end)
```

**Scenario C — Resume a thread after a delay with arguments**
```luau
local resultThread = coroutine.create(function(hitPosition: Vector3)
    spawnImpactEffect(hitPosition)
end)

Assignment.Delay(0.2, resultThread, hitPos)
```

**Scenario D — Passing zero or a negative duration**
```luau
-- Delay(0, ...) runs on the next available frame — same as Defer.
-- Useful when you want "as soon as possible" with a cancellable Handle.
local handle = Assignment.Delay(0, function()
    applyImmediateEffect(player)
end)

-- Negative values behave identically.
local handle2 = Assignment.Delay(-99, function()
    applyImmediateEffect(player)
end)

-- Both can be cancelled before the next frame fires.
handle:Break()
handle2:Break()
```

---

## DelayIsolated

**Isolated Family.** Schedules work after a set duration at full engine priority in a dedicated thread. Bypasses all throttling. Returns a `thread` cancellation token. Use when the work that fires after the delay will sleep for a long time.

**Scenario A — Delayed multi-step async operation**
```luau
-- Each step involves a slow async call that may take several seconds.
-- Running this through the standard Delay would keep the shared worker
-- occupied for every one of those yields. DelayIsolated gives this
-- work its own dedicated engine thread so the shared worker stays free.
local operationThread = Assignment.DelayIsolated(10, function()
    local snapshot = DataService:LoadSnapshot(serverId)
    Assignment.Wait(0) -- breathe between steps
    ReplicationService:Push(snapshot)
end)

onShutdown:Connect(function()
    Assignment.Cancel(operationThread)
end)
```

**Scenario B — Delayed spawn with async setup and cancellation**
```luau
local spawnThread = Assignment.DelayIsolated(5, function()
    local config = ConfigService:Fetch(entityType) -- slow async call
    spawnEntity(spawnPoint, config)
end)

onZoneCleared:Connect(function()
    Assignment.Cancel(spawnThread)
end)
```

---

## Wait

Pauses the current thread for a duration and returns the actual time elapsed. Works correctly whether called from a pooled or isolated thread — no special handling needed.

**Scenario A — Timed loop using elapsed time for smooth interpolation**
```luau
local startTime = os.clock()
while true do
    Assignment.Wait(0.05)
    local t = math.min((os.clock() - startTime) / 2, 1)
    setAlpha(lerp(0, 1, t))
    if t >= 1 then break end
end
```

**Scenario B — Passing nil — one-frame yield**
```luau
-- Assignment.Wait() with no argument pauses for exactly one frame.
-- Useful for letting other queued work run before continuing.
Assignment.Spawn(function()
    startPhaseOne()
    Assignment.Wait()   -- yields one frame
    startPhaseTwo()
end)
```

**Scenario C — Passing zero — identical to nil**
```luau
-- Assignment.Wait(0) behaves the same as Assignment.Wait(nil).
-- Both yield for one frame. Use whichever reads more clearly.
Assignment.Spawn(function()
    for _, step in ipairs(initSteps) do
        step()
        Assignment.Wait(0) -- breathe between steps
    end
end)
```

**Scenario D — Passing a negative value**
```luau
-- Negative durations are treated the same as zero — one-frame yield, no error.
Assignment.Spawn(function()
    Assignment.Wait(-99) -- same as Wait(0)
    continueWork()
end)
```

**Scenario E — Called from inside an isolated thread**
```luau
-- Assignment.Wait detects the context automatically.
-- Inside an isolated thread it defers to the engine's native wait.
-- No special handling needed at the call site.
Assignment.SpawnIsolated(function()
    while active do
        Assignment.Wait(60) -- works correctly; no need to think about context
        runHourlyMaintenance()
    end
end)
```

---

## Repeat

Calls a function at a fixed interval for a set number of iterations or indefinitely.

**Scenario A — Finite countdown with early exit**
```luau
local countdownHandle = Assignment.Repeat(10, 1, function(iteration: number, handle: Assignment.Handle)
    local remaining = 10 - iteration + 1
    updateCountdownUI(remaining)

    if roundEnded then
        handle:Break() -- stop the loop early if the round ends before 10 ticks
    end
end)
```

**Scenario B — Infinite loop with external cancellation**
```luau
local pollHandle = Assignment.Repeat(-1, 30, function(_iteration: number, _handle: Assignment.Handle)
    refreshLeaderboard()
end)

onGameModeEnd:Connect(function()
    pollHandle:Break()
end)
```

**Scenario C — Forwarding arguments to the callback**
```luau
Assignment.Repeat(6, 5, function(iteration: number, handle: Assignment.Handle, target: Player, amount: number)
    grantResource(target, amount)
    if target.Parent == nil then
        handle:Break() -- player left; stop awarding
    end
end, player, 10)
```

**Scenario D — Zero or nil interval — one-frame yield between iterations**
```luau
-- Passing nil or 0 as the interval yields one frame between iterations
-- rather than a timed wait. Useful for batch processing that should stay
-- responsive but run as fast as possible.
local batchHandle = Assignment.Repeat(#recordsToProcess, nil, function(iteration: number, _handle: Assignment.Handle)
    processRecord(recordsToProcess[iteration])
end)
```

---

## Cancel

Stops a scheduled task before it runs. Accepts a `Handle` from any pooled call or a `thread` from any isolated call.

**Scenario A — Cancel a handle when a player disconnects**
```luau
local respawnHandle = Assignment.Delay(5, respawnPlayer, player)

player.AncestryChanged:Connect(function()
    if player.Parent == nil then
        Assignment.Cancel(respawnHandle)
    end
end)
```

**Scenario B — Cancel an isolated thread by reference**
```luau
local thread = Assignment.DelayIsolated(3, doLongAsyncWork)

onConditionChanged:Connect(function()
    Assignment.Cancel(thread)
end)
```

**Scenario C — Passing nil is safe**
```luau
-- Assignment.Cancel silently does nothing when passed nil.
-- Safe to call even if the handle was never assigned.
local handle: Assignment.Handle? = nil

if someCondition then
    handle = Assignment.Delay(2, doWork)
end

Assignment.Cancel(handle :: any)
```

---

## Migrate

Hands all pending scheduled work to the engine's native scheduler, preserving remaining time for each task.

**Scenario A — Automatic shutdown (no action required)**
```luau
-- Assignment handles this automatically when the server closes.
-- All pending work is handed off with remaining time intact.
-- No code is needed — shown here for reference only.
```

**Scenario B — Manual migration for a controlled shutdown sequence**
```luau
if RunService:IsServer() then
    Assignment.Migrate() -- hand everything off now

    -- All Assignment calls from this point forward go through the
    -- native scheduler, but the API surface stays the same.
    Assignment.Delay(1, announceShutdown)
end
```
