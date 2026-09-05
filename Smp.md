# AROS m68k SMP — Bellatrix, Emu68, Musashi and Saiph CPU Ownership Architecture

**Status:** Architecture / Investigation Baseline  
**Primary target:** Raspberry Pi 3B / Bellatrix  
**OS:** AROS `m68k-emu68`  
**Primary CPU topology:** Emu68 + Musashi  
**Secondary / future topology:** Emu68 + Emu68  
**Related projects:** Bellatrix, Emu68, AROS, Rigel, Saiph

---

# 1. Purpose

This document consolidates the current investigation into adding SMP support to the Bellatrix AROS `m68k-emu68` target.

The primary proposed topology is:

~~~text
ARM Core 0                         ARM Core 3
    │                                  │
    ▼                                  ▼
 Emu68                              Musashi
    │                                  │
 68040                              68040
    │                                  │
    └──────────── AROS SMP ────────────┘
~~~

The Musashi CPU has an additional role.

When removed from the AROS SMP pool, it may be transferred to Saiph and reconfigured as the CPU required by a classic Amiga workload:

~~~text
AROS ownership                    Saiph ownership

Musashi                           Musashi
  │                                 │
68040                        68000 / 68010 /
  │                          68020 / 68030 /
AROS SMP                     68040 as required
~~~

The secondary, less immediate possibility is eventually supporting:

~~~text
Emu68 + Emu68
~~~

for a homogeneous high-performance AROS SMP configuration.

The objectives are therefore:

1. introduce an SMP backend for AROS m68k;
2. preserve the existing Bellatrix `m68k-emu68` architecture where possible;
3. use Emu68 as CPU0;
4. initially use Musashi as CPU1;
5. permit CPU1 to leave and later rejoin the AROS SMP pool;
6. allow Saiph to take ownership of Musashi while it is outside the pool;
7. permit Saiph to change the Musashi CPU model;
8. keep open a future path toward two concurrent Emu68 m68k CPUs.

The first implementation should prioritize correctness, observability, and architectural clarity over performance.

---

# 2. Important Finding: AROS Already Contains SMP Infrastructure

AROS m68k currently does not provide an SMP implementation.

However, SMP infrastructure already exists elsewhere in AROS.

The correct architecture is therefore not:

~~~text
write a new SMP scheduler for m68k
~~~

but rather:

~~~text
AROS common SMP-aware Exec
            +
m68k architecture SMP backend
            +
Bellatrix / Emu68 physical CPU transport
            +
second m68k execution engine
~~~

The common Exec already contains concepts such as:

- SMP-aware scheduler structures;
- task-ready/running/waiting locks;
- CPU masks;
- CPU affinity;
- per-CPU execution state;
- cross-CPU scheduling;
- IPI mechanisms;
- SMP-safe shared list handling;
- scheduler arbitration.

The m68k work should reuse this architecture instead of creating a parallel SMP design.

---

# 3. Reference Architecture

The useful reference is not one single AROS target.

Three sources must be considered together:

~~~text
AROS common Exec SMP
        │
        ├── scheduler semantics
        ├── locks
        ├── CPU affinity
        └── cross-CPU operations

AROS ARM/AArch64 SMP
        │
        ├── per-CPU state
        ├── IPI
        ├── CPU bootstrap
        ├── idle/wakeup
        └── ARM atomics

Bellatrix m68k-emu68
        │
        ├── actual m68k task frame
        ├── context switch
        ├── exception semantics
        └── Emu68-specific integration
~~~

The m68k SMP implementation should converge these three sources rather than copy one of them blindly.

---

# 4. Proposed Initial CPU Topology

The recommended first topology is:

~~~text
                     AROS m68k SMP
                          │
              ┌───────────┴───────────┐
              │                       │
           CPU #0                  CPU #1
              │                       │
           Emu68                   Musashi
              │                       │
        m68k / 68040             m68k / 68040
              │                       │
         ARM Core 0               ARM Core 3
              │                       │
              └──── coherent RAM ─────┘
~~~

The AROS-visible contract should make the two engines appear as compatible m68k CPUs even though their implementations are radically different.

AROS should not need to know whether a scheduled task is executing through a JIT or interpreter.

---

# 5. Why Emu68 + Musashi Is the Recommended Bring-Up Path

Originally, a homogeneous:

~~~text
Emu68 + Emu68
~~~

configuration appeared attractive.

Code inspection changed that conclusion for the initial implementation.

Emu68 already encapsulates architectural CPU state well, but important JIT/runtime structures remain global and are not currently designed for two concurrent m68k `MainLoop()` instances.

Musashi avoids that work.

The first implementation can therefore concentrate on:

- AROS SMP;
- CPU bootstrap;
- per-CPU state;
- IPI;
- atomics;
- scheduler integration;

without simultaneously having to make the Emu68 JIT runtime SMP-safe.

Musashi also has another important advantage:

> The same CPU engine can later be transferred to Saiph.

This creates a useful lifecycle:

~~~text
AROS SMP CPU1
      │
      ▼
Musashi 68040
      │
      ▼
quiesce
      │
      ▼
Saiph ownership
      │
      ▼
Musashi configured for workload CPU
~~~

---

# 6. Shared Memory Is the Fundamental SMP Boundary

Both CPU engines must access the same physical AROS memory.

~~~text
                  shared AROS RAM
                        │
               ARM coherent memory
                        │
              ┌─────────┴─────────┐
              │                   │
          Emu68 CPU0          Musashi CPU1
~~~

The engines must not maintain private copies of AROS memory.

Shared kernel structures, task structures, stacks and scheduling lists must therefore refer to the same underlying memory.

This is also what permits task migration.

---

# 7. Atomic Operations

## 7.1 Current m68k AROS atomics are not sufficient for SMP

The existing m68k atomic implementation uses operations such as:

~~~text
addq
subq
and
or
~~~

directly against memory.

These are sufficient in a uniprocessor environment but do not provide the required inter-CPU atomicity.

The m68k SMP backend therefore needs SMP-safe implementations.

---

# 8. Emu68 Atomic Behaviour

Code inspection of current Emu68 found that normal aligned m68k `CAS` operations are translated into genuine AArch64 exclusive operations.

Conceptually:

~~~text
m68k CAS.L
     │
     ▼
Emu68 JIT
     │
     ▼
LDXR
compare
STLXR
retry
DMB
~~~

Memory `TAS` similarly uses exclusive ARM operations.

This is an important result.

It means that Emu68 already participates in the physical ARM cache-coherency/exclusive-monitor domain.

---

# 9. Musashi Atomic Behaviour

Unmodified Musashi does not currently provide the same guarantee.

Its `CAS` implementation conceptually performs:

~~~text
read memory
compare
write memory
~~~

through memory callbacks.

Those operations are not atomic relative to another CPU.

Therefore Musashi must be modified for the SMP configuration.

The recommended design is:

~~~text
m68k CAS
    │
    ▼
Musashi instruction handler
    │
    ▼
Bellatrix/Saiph host atomic helper
    │
    ▼
ARM exclusive operation
~~~

This produces:

~~~text
                         shared lock
                              │
                      coherent ARM RAM
                              │
                  ┌───────────┴───────────┐
                  │                       │
               Emu68                  Musashi
                  │                       │
              m68k CAS                m68k CAS
                  │                       │
             LDXR/STLXR            host LDXR/STLXR
                  │                       │
                  └── same atomic domain ┘
~~~

The Musashi modification should be kept small and explicit.

---

# 10. CAS2

Current investigation has not found `CAS2` to be a structural requirement of AROS SMP.

This is useful because neither current engine should be assumed to provide a correct cross-core atomic `CAS2`.

The AROS SMP architecture investigated so far primarily requires:

- 32-bit atomic RMW;
- spinlocks;
- atomic INC/DEC;
- atomic AND/OR;
- memory barriers.

The m68k backend should therefore preferably use ordinary `CAS.L` loops rather than introduce a dependency on `CAS2`.

Example:

~~~text
AROS_ATOMIC_OR()
       │
       ▼
m68k CAS.L loop
       │
       ├── Emu68 → ARM exclusives
       │
       └── Musashi → host ARM exclusives
~~~

`CAS2` support may be investigated separately but should not block the first SMP implementation unless later code inspection proves it is actually required.

---

# 11. Alignment Requirement

Emu68 has a safe atomic path for properly aligned CAS operations and a different unsafe path for some unaligned accesses.

Therefore SMP atomic storage must have explicit alignment requirements.

At minimum:

~~~text
atomic16 → 2-byte aligned
atomic32 → 4-byte aligned
~~~

Debug builds should assert these conditions.

Kernel locks should never rely on unaligned atomic storage.

---

# 12. Memory Barriers

Atomicity alone is insufficient.

The m68k SMP backend must also provide the memory ordering semantics expected by AROS.

The desired host-level result is equivalent to appropriate ARM barriers such as:

~~~text
DMB ISH
~~~

The architecture should distinguish:

~~~text
acquire:
    atomic operation
    acquire ordering

release:
    release ordering
    unlock
~~~

The implementation must be checked against the exact AROS spinlock contract rather than merely inserting barriers arbitrarily.

---

# 13. AROS Spinlocks

Current AROS SMP uses shared spinlock words rather than a requirement for double-word transactional primitives.

The fundamental lock state is a 32-bit word containing lock state and reader information.

Conceptually:

~~~text
                    32-bit lock word
                           │
                    atomic RMW
                           │
               ┌───────────┴───────────┐
               │                       │
             CPU0                    CPU1
~~~

This fits naturally with `CAS.L`.

The first m68k SMP implementation should favor correctness and use the common SMP locking path.

Do not preserve UP-specific lockless scheduler shortcuts until SMP correctness has been established.

---

# 14. Per-CPU State

One of the most important changes required in the current Bellatrix backend is separating global Exec state from CPU-local execution state.

Examples of CPU-local state include:

~~~text
ThisTask
IDNestCnt
TDNestCnt
ScheduleFlags
Quantum / Elapsed where appropriate
CPU number
~~~

The desired structure is conceptually:

~~~text
CPU0 local state

ThisTask
IDNestCnt
TDNestCnt
ScheduleFlags
Quantum
CPUNumber = 0


CPU1 local state

ThisTask
IDNestCnt
TDNestCnt
ScheduleFlags
Quantum
CPUNumber = 1
~~~

These values generally do not require inter-CPU atomic operations because each CPU owns its own local state.

Shared structures require locks.

CPU-local structures do not.

---

# 15. Current Bellatrix Dispatch Requires SMP Adaptation

The current `m68k-emu68` dispatch path still manipulates traditional ExecBase fields such as:

~~~text
SysBase->ThisTask
SysBase->IDNestCnt
SysBase->TDNestCnt
SysBase->Elapsed
~~~

This is valid for the current UP implementation.

It cannot represent two simultaneously executing CPUs.

Therefore the current context machinery should be preserved where possible, but accesses to CPU execution state must become SMP/per-CPU aware.

---

# 16. Current Bellatrix Fast Dispatch Path

The current Bellatrix `dispatch.S` contains a UP-oriented fast path that can manipulate `TaskReady` directly.

That is not appropriate for the first SMP implementation.

In SMP:

~~~text
CPU0 ─────┐
          ├── TaskReady
CPU1 ─────┘
~~~

requires scheduler locking.

The initial implementation should therefore bypass/remove the UP fast path and use the common SMP-aware dispatch machinery.

Only after SMP correctness is demonstrated should specialized fast paths be reconsidered.

---

# 17. Bellatrix Task Frame

The current Bellatrix `m68k-emu68` backend uses a persistent task frame of approximately:

~~~text
+00   PC
+04   SR
+06   format/vector word
+08   D0
      ...
      D7
      A0
      ...
+64   A6

total: 68 bytes
~~~

The frame is deliberately self-contained.

This differs from historical assumptions elsewhere in the m68k backend where parts of the exception state may remain associated with the supervisor stack.

The Bellatrix design was introduced to match the way Emu68 constructs and consumes exception frames.

This existing format should be treated as the current Bellatrix ABI for task context until there is a strong reason to change it.

---

# 18. The Task Context Is Not an Emu68 M68KState

This distinction is crucial.

The persistent task state lives in AROS memory.

Conceptually:

~~~text
AROS Task
    │
    ├── tc_SPReg
    │      │
    │      ▼
    │   m68k task frame
    │
    └── ETask / additional CPU context
~~~

The task is therefore not intrinsically owned by Emu68.

This opens the possibility of migration:

~~~text
CPU0 / Emu68
      │
      ▼
save task frame
      │
      ▼
shared AROS RAM
      │
      ▼
restore task frame
      │
      ▼
CPU1 / Musashi
~~~

This is one of the most important architectural properties enabling the proposed heterogeneous implementation.

---

# 19. Musashi Compatibility With the Bellatrix Frame

Musashi should initially run as a 68040-compatible CPU while participating in AROS SMP.

The first validation must not be a full scheduler test.

Instead, construct an exact synthetic Bellatrix task frame and prove that Musashi can restore it correctly.

Test:

~~~text
known D0-D7
known A0-A6
known USP
known SR
known PC
format = expected format

        │
        ▼
Bellatrix-style restore
        │
        ▼
RTE
        │
        ▼
test_entry
~~~

Then verify all architectural state.

This establishes whether the existing Bellatrix context ABI can be shared directly between Emu68 and Musashi.

No scheduler work should depend on this assumption until the test passes.

---

# 20. FPU State

FPU migration should not be part of the first SMP milestone.

Initial bring-up should be integer-only.

~~~text
Phase 1:
CPU0 ↔ CPU1
integer context only

Phase 2:
FPU context save/restore

Phase 3:
FPU task migration between engines
~~~

The relevant question is whether the AROS FPU context is sufficiently architectural to be restored by both engines.

This should be validated independently.

---

# 21. CPU Identification

For the initial two-CPU system there is no reason to dynamically discover a m68k CPU identifier.

The mapping can be explicit:

~~~text
Emu68   → AROS CPU #0
Musashi → AROS CPU #1
~~~

Each per-CPU structure therefore contains its logical CPU number.

The physical ARM core number and AROS logical CPU number should remain conceptually separate.

---

# 22. IPI and Remote Scheduling

AROS SMP already has the concept of scheduling another CPU.

The m68k backend needs a transport for:

~~~text
CPU0
  │
  ▼
request CPU1 scheduling
  │
  ▼
IPI
  │
  ▼
CPU1
  │
  ▼
set scheduling flags
  │
  ▼
dispatch at safe scheduling point
~~~

Bellatrix can implement this using ARM inter-core communication.

For Musashi:

~~~text
CPU0 / Emu68
      │
      ▼
Bellatrix inter-core event
      │
      ▼
ARM Core3 host loop
      │
      ▼
mark Musashi IRQ pending
      │
      ▼
Musashi enters m68k IPI handler
      │
      ▼
AROS CPU1 scheduling logic
~~~

The physical transport does not need to resemble native m68k hardware.

It only needs to preserve the AROS IPI semantics.

---

# 23. IPI Interrupt Level

The ARM AROS implementation uses architecture-specific interrupt mechanisms, including FIQ in some cross-CPU paths.

The m68k implementation does not need to reproduce that detail.

A dedicated m68k interrupt level/vector can be reserved for SMP/IPI use.

The important invariant is:

> An IPI must not re-enter code on the same CPU in a way that recursively attempts to acquire a scheduler lock already held by that CPU.

The m68k backend therefore needs appropriate local IPI masking around relevant critical regions.

---

# 24. CPU1 Bootstrap

The secondary CPU should not immediately enter general scheduling.

Recommended sequence:

~~~text
ARM Core3 starts
      │
      ▼
initialize Musashi
      │
      ▼
CPU_TYPE_68040
      │
      ▼
m68k CPU1 bootstrap entry
      │
      ▼
initialize CPU1 local state
      │
      ▼
create/enter bootstrap task
      │
      ▼
enable scheduler participation
      │
      ▼
CPU1 Idle
~~~

This mirrors the general design already used by AROS SMP implementations.

---

# 25. CPU1 Idle

Musashi should not continuously interpret an idle loop and consume an ARM core unnecessarily.

A desirable path is:

~~~text
AROS CPU1 Idle
      │
      ▼
m68k STOP or defined idle boundary
      │
      ▼
Musashi returns/yields to host
      │
      ▼
ARM Core3 WFE
~~~

On work:

~~~text
CPU0 sends event
      │
      ▼
SEV / inter-core notification
      │
      ▼
Core3 wakes
      │
      ▼
Musashi IRQ pending
      │
      ▼
resume m68k CPU1
~~~

This should be designed early even if the first prototype initially spins.

---

# 26. CPU Affinity

CPU affinity is especially useful because CPU0 and CPU1 have very different performance characteristics.

~~~text
CPU0 = Emu68 JIT
CPU1 = Musashi interpreter
~~~

Initially, the system should not assume equal CPU performance.

Early bring-up may use affinity such as:

~~~text
critical kernel/system tasks
        ↓
CPU0 / Emu68

selected test/worker tasks
        ↓
CPU1 / Musashi
~~~

General migration can be enabled later.

This is a bring-up strategy, not necessarily a permanent policy.

---

# 27. CPU1 Ownership State Machine

CPU1 must not simply disappear from AROS when Saiph needs it.

A formal ownership state machine is required.

Recommended states:

~~~text
AROS_ONLINE
     │
     ▼
AROS_QUIESCING
     │
     ▼
AROS_QUIESCED
     │
     ▼
SAIPH_OWNED
~~~

Return path:

~~~text
SAIPH_OWNED
     │
     ▼
AROS_QUIESCED
     │
     ▼
AROS_REJOINING
     │
     ▼
AROS_ONLINE
~~~

The critical invariant is:

> CPU model and machine ownership may only change after the CPU has stopped executing AROS code.

---

# 28. Removing CPU1 From the AROS Pool

A safe transition should resemble:

~~~text
CPU0 requests CPU1 removal
          │
          ▼
scheduler stops assigning new work
          │
          ▼
running CPU1 task reaches safe point
          │
          ▼
task context saved in AROS RAM
          │
          ▼
task migrates / becomes schedulable elsewhere
          │
          ▼
CPU1 enters offline trampoline
          │
          ▼
CPU1 acknowledges quiescence
          │
          ▼
CPU1 removed from scheduling mask
          │
          ▼
AROS no longer owns Musashi
~~~

Only after this point may Saiph reconfigure the CPU.

---

# 29. Musashi CPU Model Can Change After Leaving the Pool

While participating in AROS SMP:

~~~text
Musashi
   │
CPU_TYPE_68040
   │
AROS CPU1
~~~

After complete quiescence:

~~~text
Musashi stopped
      │
      ▼
discard/reset CPU-local AROS execution state
      │
      ▼
select Saiph CPU model
      │
      ├── 68000
      ├── 68010
      ├── 68020
      ├── 68030
      └── 68040
      │
      ▼
reset
      │
      ▼
Saiph workload
~~~

The exact model may be selected according to the game/demo/environment being executed.

This is a strong advantage of using Musashi for CPU1.

---

# 30. Why 68000 Does Not Conflict With the Bellatrix 68-Byte Frame

The Bellatrix frame only matters while Musashi participates in AROS SMP.

When Saiph owns the engine:

~~~text
AROS task frame
      │
      X
      │
Musashi 68000
~~~

There is no requirement for the Saiph 68000 environment to restore an AROS task frame.

Instead:

~~~text
AROS ownership
Musashi 68040
Bellatrix task ABI
        │
        ▼
QUIESCE
        │
        ▼
RESET
        │
        ▼
Saiph ownership
Musashi 68000
classic 68000 exception semantics
~~~

The two contexts do not coexist.

---

# 31. Do Not Convert 68040 CPU State Into 68000 State

There is no useful reason to translate live CPU-local state between CPU models.

When CPU1 leaves AROS:

~~~text
AROS task state
      │
      ▼
saved in AROS RAM
~~~

The Musashi execution state may then be discarded.

Saiph starts from a fresh reset state.

Likewise, returning to AROS should not restore a frozen Musashi 68040 internal state.

Instead:

~~~text
stop Saiph
    │
    ▼
reset Musashi
    │
    ▼
set CPU_TYPE_68040
    │
    ▼
reinitialize AROS CPU-local state
    │
    ▼
CPU1 rejoin/bootstrap path
~~~

This keeps the ownership boundary clean.

---

# 32. Saiph CPU Model Selection

The ownership architecture allows Saiph to select a CPU according to workload requirements.

Conceptually:

~~~text
game/demo metadata
        │
        ▼
required CPU
        │
 ┌──────┼──────┬──────┐
 │      │      │      │
000    010    020    040
 │      │      │      │
 └──────┴──────┴──────┘
        │
        ▼
Musashi configuration
~~~

This should remain a Saiph concern.

AROS should only require:

~~~text
Musashi == compatible SMP CPU
~~~

while CPU1 is online.

---

# 33. Relationship With Rigel

The CPU ownership transition should also correspond to machine ownership.

During normal AROS SMP:

~~~text
CPU0 Emu68 ── AROS
CPU1 Musashi ── AROS
~~~

During takeover:

~~~text
CPU0 Emu68
    │
    ▼
AROS remains alive
    │
    ├── Pi USB
    ├── Bluetooth
    ├── storage
    ├── VideoCore
    └── host services


CPU1 Musashi
    │
    ▼
Saiph
    │
    ▼
classic 68k environment
    │
    ▼
Rigel
    │
    ▼
Amiga chipset semantics
~~~

This permits AROS to remain alive while the takeover environment controls the classic low-24 machine.

---

# 34. Recommended First Bring-Up Sequence

Do not attempt full SMP immediately.

## Stage 1 — Musashi Bellatrix frame test

~~~text
synthetic 68-byte frame
        │
        ▼
Musashi 68040
        │
        ▼
Bellatrix-compatible restore
        │
        ▼
RTE
        │
        ▼
validate all registers
~~~

Pass criteria:

- PC correct;
- SR correct;
- D0-D7 correct;
- A0-A6 correct;
- USP correct;
- expected exception-frame behavior.

---

## Stage 2 — Shared RAM test

Run both engines concurrently against the same RAM.

~~~text
Emu68 CPU0 ──┐
             ├── shared counters
Musashi CPU1 ┘
~~~

Validate visibility and cache coherency.

---

## Stage 3 — Cross-engine atomic test

~~~text
Emu68 CAS
     │
     ├── same shared lock
     │
Musashi CAS
~~~

Stress for a large number of iterations.

No lost updates are acceptable.

---

## Stage 4 — CPU1 bootstrap

Start Musashi on Core3 and enter a known m68k CPU1 bootstrap routine.

No scheduler yet.

---

## Stage 5 — Per-CPU state

Implement:

~~~text
CPU0 local state
CPU1 local state
~~~

Validate simultaneous:

~~~text
ThisTask0 != ThisTask1
TDNest0 independent of TDNest1
IDNest0 independent of IDNest1
~~~

---

## Stage 6 — IPI

Validate:

~~~text
CPU0 → CPU1
CPU1 → CPU0
~~~

with counters before involving scheduling.

---

## Stage 7 — CPU1 Idle

Bring CPU1 online but schedule only its idle/bootstrap task.

---

## Stage 8 — Selected CPU1 task

Use CPU affinity to run a controlled integer-only task exclusively on CPU1.

---

## Stage 9 — Task migration

This is the critical proof:

~~~text
Task X
  │
CPU0 / Emu68
  │
save
  │
TaskReady
  │
restore
  │
CPU1 / Musashi
  │
execute
  │
save
  │
CPU0 / Emu68
~~~

Validate the task state after every transition.

---

## Stage 10 — General SMP scheduling

Only after controlled migration is reliable should general scheduling across both CPUs be enabled.

---

## Stage 11 — CPU removal

Test:

~~~text
AROS_ONLINE
   ↓
AROS_QUIESCED
~~~

without Saiph.

Repeatedly remove and re-add CPU1.

---

## Stage 12 — Saiph ownership

Only after CPU lifecycle is stable:

~~~text
AROS CPU1
   ↓
quiesce
   ↓
Musashi reset
   ↓
CPU_TYPE_68000
   ↓
Saiph
~~~

Then test the complete return path.

---

# 35. Debug Instrumentation

The existing Bellatrix task-frame validation should be preserved and generalized.

Useful trace format:

~~~text
CPU0 SAVE
task=...
frame=...
PC=...
SR=...
SP=...

CPU1 RESTORE
task=...
frame=...
PC=...
SR=...
SP=...
~~~

Add:

~~~text
cpu number
engine type
scheduler state
ownership state
IPI state
lock state when relevant
~~~

A useful engine field would make traces immediately understandable:

~~~text
CPU0 engine=EMU68
CPU1 engine=MUSASHI
~~~

---

# 36. SMP Correctness Tests

At minimum, implement stress tests for:

- atomic increment;
- CAS contention;
- spinlock contention;
- ready-list contention;
- cross-CPU Signal();
- IPI storm;
- repeated scheduling;
- task migration;
- Disable()/Enable();
- Forbid()/Permit();
- semaphore contention;
- memory allocation from both CPUs;
- task creation/deletion;
- CPU1 remove/rejoin cycles.

Do not rely only on successful Wanderer boot as evidence of SMP correctness.

---

# 37. Special Attention: Disable/Enable and Forbid/Permit

Legacy m68k code may assume that:

~~~text
Disable()
~~~

or:

~~~text
Forbid()
~~~

provides stronger global exclusion than is valid in an SMP system.

The common AROS SMP implementation already addresses many internal Exec cases through locks, but m68k-specific code and drivers must be audited.

Important distinction:

~~~text
Forbid
    → prevents local scheduling semantics

Disable
    → affects local interrupt semantics

spinlock
    → protects shared SMP state
~~~

Neither `Forbid()` nor `Disable()` should silently be treated as a global inter-CPU lock.

---

# 38. Priority: Emu68 + Musashi

The recommended implementation priority is therefore:

~~~text
                    PRIORITY 1

              Emu68 + Musashi
                     │
                     ▼
              AROS m68k SMP
                     │
                     ▼
              CPU ownership
                     │
                     ▼
                  Saiph
~~~

This path simultaneously solves two architectural needs:

1. second AROS CPU;
2. reusable classic CPU engine for takeover.

---

# 39. Future Alternative: Emu68 + Emu68

A homogeneous Emu68 configuration remains desirable as a future optimization.

~~~text
CPU0                         CPU1
Emu68                        Emu68
68040                        68040
JIT                          JIT
 │                            │
ARM Core0                    ARM Core3
~~~

Advantages include:

- similar performance between CPUs;
- same m68k implementation;
- same exception semantics;
- same JIT behavior;
- easier arbitrary task migration from the AROS perspective;
- substantially faster CPU1 than Musashi.

However, current Emu68 implementation details make this a separate engineering task.

---

# 40. Emu68 Architectural CPU State Is Already Well Encapsulated

Current Emu68 has an `M68KState` containing architectural state such as:

~~~text
D/A registers
USP/MSP/ISP
PC
SR
VBR
CACR
FPU state
SFC/DFC
MMU state
interrupt state
JIT-related context fields
~~~

The execution loop obtains its current context through a context pointer.

This suggests that:

~~~text
M68KState0
M68KState1
~~~

is conceptually possible.

The CPU architectural state itself is therefore not the primary obstacle.

---

# 41. Current Emu68 JIT/Runtime Is the Main Obstacle

Important runtime/JIT structures are currently global.

Examples identified during investigation include concepts such as:

~~~text
global EPOCH
global ICache
global LRU cache state
global LRU allocation state
~~~

The current design assumes one active m68k execution environment for these structures.

Simply doing:

~~~text
M68KState cpu0;
M68KState cpu1;

Core0 → MainLoop(cpu0)
Core3 → MainLoop(cpu1)
~~~

must not be assumed safe.

Concurrent cache insertion, invalidation and translation management require explicit design.

---

# 42. Possible Future Emu68 SMP Runtime Architecture

A promising eventual design is:

~~~text
                  shared translation store
                           │
                  shared immutable blocks
                           │
             synchronized translation creation
                           │
             synchronized invalidation/epoch
                           │
             ┌─────────────┴─────────────┐
             │                           │
        per-CPU hot cache           per-CPU hot cache
        per-CPU LRU0                per-CPU LRU1
             │                           │
        M68KState0                  M68KState1
             │                           │
         ARM Core0                   ARM Core3
~~~

Sharing immutable translated blocks is potentially valuable because both CPUs execute the same AROS kernel and libraries.

Duplicating every translation would waste memory and JIT work.

However, mutable hot-cache/LRU state should likely become per-CPU.

---

# 43. Emu68 Translation Invalidation Is Critical

Any shared JIT design must correctly handle:

- self-modifying m68k code;
- AROS `CacheClear*()` semantics;
- executable memory modifications;
- JIT block invalidation;
- global translation epochs;
- simultaneous translation creation;
- simultaneous execution of a block being invalidated.

This is considerably more complex than adding a second `M68KState`.

Therefore it should not be mixed into the first AROS SMP bring-up.

---

# 44. Recommended Emu68 + Emu68 Experimental Sequence

When the Musashi SMP path is already functional, use it as an oracle for the AROS side.

Then investigate Emu68 SMP independently.

First experiment:

~~~text
ARM Core0
   │
Emu68 MainLoop
   │
M68KState0


ARM Core3
   │
Emu68 MainLoop
   │
M68KState1
~~~

with:

- simple shared RAM;
- no AROS;
- separate stacks;
- separate CPU state;
- controlled translated loops;
- per-CPU counters.

Before attempting full AROS, prove:

1. both loops execute concurrently;
2. neither corrupts the other's CPU state;
3. JIT cache operations remain valid;
4. invalidation works;
5. atomic operations work across engines.

---

# 45. Relationship Between Both Paths

The architecture should deliberately permit CPU1 engine replacement.

~~~text
                    AROS m68k SMP
                         │
                  CPU backend contract
                         │
              ┌──────────┴──────────┐
              │                     │
          Musashi CPU1          Emu68 CPU1
              │                     │
       initial/reference       future/performance
~~~

The AROS SMP layer should not contain assumptions such as:

~~~text
CPU1 == Musashi
~~~

Instead:

~~~text
CPU1 implements m68k SMP CPU contract
~~~

Musashi is simply the first implementation.

---

# 46. Proposed m68k SMP CPU Contract

Conceptually, an AROS-compatible CPU engine must provide:

~~~text
m68k CPU execution
compatible task context
shared coherent RAM
atomic 32-bit RMW
memory barriers
interrupt delivery
IPI reception
logical CPU identity
idle/wakeup
context switch
CPU quiescence
~~~

An optional ownership interface may additionally provide:

~~~text
offline
reset
change CPU model
change memory/bus ownership
rejoin
~~~

The latter is primarily needed by Musashi/Saiph.

---

# 47. Important Architectural Separation

Do not mix these three layers:

~~~text
AROS SMP semantics
        │
        ▼
m68k CPU backend contract
        │
        ▼
physical execution engine
~~~

For example:

~~~text
KrnScheduleCPU()
        │
        ▼
m68k IPI abstraction
        │
        ├── Emu68 transport
        └── Musashi transport
~~~

Likewise:

~~~text
AROS atomic operation
        │
        ▼
m68k CAS semantics
        │
        ├── Emu68 JIT exclusive operation
        └── Musashi host exclusive helper
~~~

This separation is important if Emu68+Emu68 is introduced later.

---

# 48. Recommended Development Phases

## Phase A — CPU engine validation

~~~text
Musashi 68040
Bellatrix frame compatibility
shared RAM
atomic CAS
interrupt delivery
~~~

No AROS SMP yet.

---

## Phase B — Minimal AROS m68k SMP

~~~text
per-CPU state
CPU identity
spinlocks
atomics
IPI
CPU1 bootstrap
CPU1 idle
~~~

---

## Phase C — Scheduling

~~~text
selected CPU1 task
affinity
controlled migration
general SMP scheduling
~~~

---

## Phase D — Stability

~~~text
Exec stress
memory allocator stress
Signal stress
semaphores
drivers
legacy Disable/Forbid audit
~~~

---

## Phase E — Dynamic CPU ownership

~~~text
AROS_ONLINE
    ↓
AROS_QUIESCED
    ↓
AROS_ONLINE
~~~

Repeated stress.

---

## Phase F — Saiph

~~~text
AROS_ONLINE
    ↓
AROS_QUIESCED
    ↓
SAIPH_OWNED
    ↓
change CPU model
    ↓
classic workload
    ↓
reset
    ↓
AROS_REJOINING
    ↓
AROS_ONLINE
~~~

---

## Phase G — Optional Emu68 SMP Runtime

Only after AROS SMP semantics are proven:

~~~text
second M68KState
per-CPU JIT runtime state
shared translation architecture
SMP-safe invalidation
two concurrent MainLoops
~~~

Then replace Musashi CPU1 experimentally:

~~~text
Emu68 + Musashi
       ↓
Emu68 + Emu68
~~~

without changing the AROS SMP contract.

---

# 49. Non-Goals for the First Implementation

Do not initially attempt:

- CAS2 SMP semantics unless proven necessary;
- arbitrary CPU-model mixing while CPUs are online;
- FPU task migration;
- full scheduler performance optimization;
- retention of UP-specific dispatch fast paths;
- transparent live conversion between 68040 and 68000 state;
- two concurrent Emu68 JITs;
- sophisticated CPU load balancing;
- Saiph ownership before CPU offline/rejoin is reliable.

These can obscure the fundamental SMP bring-up.

---

# 50. Core Architectural Invariants

The implementation should preserve the following invariants.

### Invariant 1

~~~text
A task belongs to AROS, not to an execution engine.
~~~

### Invariant 2

~~~text
An online AROS CPU must obey the AROS m68k SMP CPU contract.
~~~

### Invariant 3

~~~text
Musashi is 68040-compatible while AROS owns it.
~~~

### Invariant 4

~~~text
Musashi CPU model may change only after AROS has completely quiesced CPU1.
~~~

### Invariant 5

~~~text
Saiph never takes ownership of a CPU still executing an AROS task.
~~~

### Invariant 6

~~~text
Shared kernel state is protected by SMP primitives;
Forbid()/Disable() are not substitutes for inter-CPU locking.
~~~

### Invariant 7

~~~text
Both engines must participate in the same physical atomicity domain.
~~~

### Invariant 8

~~~text
The first SMP implementation must not depend on CAS2 unless actual AROS code proves it necessary.
~~~

### Invariant 9

~~~text
The AROS SMP layer must not depend on CPU1 being Musashi.
~~~

### Invariant 10

~~~text
Emu68+Emu68 is an execution-engine optimization,
not a redesign of AROS SMP.
~~~

---

# 51. Target End State

The desired long-term architecture is:

~~~text
                         Raspberry Pi 3B
                               │
       ┌───────────────────────┴────────────────────────┐
       │                                                │
   ARM Core0                                        ARM Core3
       │                                                │
     Emu68                                      CPU engine slot
       │                                                │
 AROS m68k CPU0                         ┌───────────────┴──────────────┐
       │                                │                              │
       │                           Musashi                          Emu68
       │                                │                              │
       │                       AROS CPU1 / Saiph                 AROS CPU1
       │                                │                              │
       └────────────── AROS SMP ────────┴──────────────────────────────┘
                                        │
                                  when quiesced
                                        │
                                        ▼
                                      Saiph
                                        │
                              Musashi CPU model
                                        │
                         000 / 010 / 020 / 030 / 040
                                        │
                                        ▼
                                      Rigel
~~~

The initial implementation should establish the left side plus Musashi CPU1.

The future Emu68 CPU1 should plug into the same AROS SMP architecture without requiring a scheduler redesign.

---

# 52. Recommended Immediate Work

The next implementation work should be narrow and test-driven.

First:

~~~text
1. Validate the exact Bellatrix 68-byte frame under Musashi 68040.
~~~

Then:

~~~text
2. Patch Musashi CAS/TAS to use ARM host atomics.
~~~

Then:

~~~text
3. Build a standalone Emu68 ↔ Musashi atomic/shared-memory stress test.
~~~

Then begin the AROS backend:

~~~text
4. Introduce m68k SMP-safe atomics.

5. Introduce per-CPU execution state.

6. Implement logical CPU identity.

7. Implement IPI transport.

8. Bring CPU1 to bootstrap/idle.

9. Run one affinity-pinned task on CPU1.

10. Prove task migration between Emu68 and Musashi.
~~~

Only after these are stable should dynamic CPU removal and Saiph ownership be implemented.

---

# 53. Final Direction

The current investigation supports the following direction:

> **Use Emu68 + Musashi as the first Bellatrix AROS m68k SMP implementation.**

Musashi should present itself as a compatible 68040 while participating in the AROS CPU pool.

When CPU1 is safely quiesced and removed from that pool, its AROS-local execution state may be discarded, Musashi may be reset and configured as the CPU model required by Saiph.

This makes the second CPU a reusable execution resource rather than permanently assigning an ARM core either to AROS or to the takeover environment.

At the same time, the AROS SMP architecture must remain engine-neutral.

A future second Emu68 instance should therefore be treated as a replacement implementation of the CPU1 contract:

~~~text
AROS SMP
   │
   ├── CPU0: Emu68
   │
   └── CPU1:
          │
          ├── Musashi       ← first implementation
          │
          └── Emu68         ← future performance path
~~~

The major additional work required by the second path is not AROS SMP itself.

It is making the Emu68 m68k JIT/runtime safe for multiple concurrent m68k execution contexts, particularly translation-cache ownership, LRU state, epochs and invalidation.

Keeping that work separate allows Bellatrix to prove AROS m68k SMP first, while preserving a clear route toward a homogeneous high-performance Emu68 SMP system later.
