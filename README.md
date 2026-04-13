# The Diary Framework: Architectural Overview
Diary is a Cache-First, Session-Locked Hybrid Persistence Framework designed specifically for high-capacity (200+ player) Roblox MMORPGs. Its primary architectural goal is to completely decouple real-time gameplay logic from the latency and rate limits of Roblox's native DataStoreService.

To achieve this, Diary relies on a sophisticated hierarchy of active server memory, transient MemoryStore caching, and resilient background batching.

## The Core Philosophy: Read Instantly, Save Asynchronously
In a traditional Roblox game, developers often yield the server to read or write data. In an MMO, yielding during combat or trading is unacceptable.

Diary solves this by enforcing a Read-Reference, Mutate, and Mark pattern:

### 1. When a player joins, their complete data profile is hydrated into local server RAM (PlayerDataCache).

### 2. Gameplay systems (combat, inventory) read and modify this RAM directly with zero latency.

### 3. Systems call Diary.MarkDirty() to flag changes. Background workers autonomously bundle these changes and push them to the database, ensuring the main gameplay thread never stalls.

## The Four Pillars of the Architecture
The framework is divided into four highly specialized, decoupled modules:

### 1. Diary.lua (The Orchestrator)
This is the central nervous system and the primary developer-facing API.

#### * Lifecycle Management: It safely handles player connections, disconnections, and server shutdowns, ensuring no data is dropped when the server closes.

#### * Template Reconciliation: It ensures player data always matches the latest game schemas, automatically injecting missing keys or scrubbing deprecated ones.

#### * Cross-Server Handshakes: It uses MessagingService to communicate with other servers during "crash-joins", demanding old servers release their locks before letting a player load in.

## 2. MemoryStore.lua (The Locksmith & Fast Cache)
This module handles distributed locking and high-frequency, volatile data.

#### * Session Leasing: To prevent data duplication or wiping, this module grants a strict "Lease" to a specific Server ID. A server cannot save player data unless it exclusively owns this lease.

#### * Transient Storage: It acts as a fast cache for global MMO features, such as tracking "Online/Offline" status and saving HumanoidDescription data for "Ghost" bodies left behind after logouts.

## 3. DataStore.lua (The Resilient Pipeline)
This is the permanent storage engine, built to survive Roblox's strict API rate limits.

#### * Smart Queueing: Instead of saving instantly, requests are routed into a SaveQueue. A background processor drains this queue in controlled batches of up to 10 operations, utilizing exponential backoff if a save fails.

#### * Hybrid Saving: It intelligently chooses between SetAsync (for direct overwrites if the server owns the lease) and UpdateAsync (for merging data if there is a conflict).

## 4. DependentServices.lua (The Optimizer)
This module acts as the mathematical and memory-optimization engine.

#### * Binary Serialization: It takes expensive 3D datatypes (Vector3, CFrame, ColorSequence) and compresses them into raw byte buffers, encoding them in Base64. This drastically reduces the size of the DataStore payload.

#### * Delta Generation: Before saving, it compares the current data against a cached snapshot to generate a "Delta" map, ensuring the server only writes the exact variables that changed.

#### * Memory Pooling: It heavily utilizes table pooling to recycle arrays and dictionaries, preventing the Luau Garbage Collector from freezing the server during massive serialization events.
