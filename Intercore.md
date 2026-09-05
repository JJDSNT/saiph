# Rigel Inter-Core Ownership, IPC and Time Authority

## Findings from the Emu68 PPC Investigation and Implications for Bellatrix / Saiph

**Status:** Architecture Investigation / Refinement  
**Related document:** `Saiph/Ipc.md`  
**Primary target:** Raspberry Pi 3B  
**Projects:** Bellatrix, Rigel, Saiph, Emu68  
**Priority:** Evolution of the existing Bellatrix Rigel architecture

---

# 1. Purpose

This document records the architectural conclusions reached after investigating:

1. the current Bellatrix/Rigel multicore model;
2. the IPC concerns described in `Saiph/Ipc.md`;
3. the Emu68 PPC secondary-core implementation;
4. the relationship between Rigel ownership and inter-core communication;
5. the future Saiph takeover model;
6. CPU-based Rigel timing under Saiph;
7. the possibility of DIRECT and IPC Rigel transports.

The investigation supports the general direction already proposed in `Ipc.md`, but several concepts should be separated more explicitly.

The most important distinction is:

~~~text
CPU EXECUTION OWNER
        ≠
RIGEL STATE OWNER
        ≠
TIME AUTHORITY
~~~

These three responsibilities may belong to different components.

This separation allows Bellatrix to evolve its current dedicated-Rigel-core implementation without preventing Saiph from later driving Rigel timing from Musashi CPU execution.

---

# 2. Main Architectural Conclusion

The central invariant proposed for the evolved architecture is:

> **RigelContext has exactly one execution owner at any given time.**

Normally this should be the Rigel execution core.

Other CPUs should communicate with that owner rather than independently entering and mutating `RigelContext`.

Conceptually:

~~~text
CURRENT SHARED-STATE MODEL

Core0 / Emu68                 Rigel Core
      │                           │
      ├─────── lock ──────────────┤
      │                           │
      └────── RigelContext ───────┘
~~~

should evolve toward:

~~~text
SINGLE-OWNER MODEL

Core0 / Emu68                 Rigel Core
      │                           │
      │      request / state      │
      ├──────── IPC ─────────────►│
      │                           │
      │                      RigelContext
      │                           │
      │      result / events      │
      ◄───────────────────────────┤
~~~

The lock in the current model provides mutual exclusion.

It does not provide stable ownership of the mutable Rigel working set.

With a single owner:

~~~text
RigelContext
     │
     ▼
one core
     │
     ▼
stable ownership
~~~

becomes an architectural property rather than an accidental scheduling outcome.

---

# 3. What the Emu68 PPC Investigation Actually Demonstrates

The Emu68 PPC implementation is directly relevant as an architectural precedent, but it must not be overstated.

The investigation demonstrates several useful properties.

## 3.1 Independent execution context

The PPC engine owns a separate CPU context:

~~~text
m68k Core                       PPC Core
   │                               │
M68KState                       PPCState
   │                               │
m68k loop                      PPC loop
~~~

The second engine does not require both cores to execute directly against one shared CPU context.

This supports the principle:

~~~text
large mutable execution state
            │
            ▼
       one owner core
~~~

---

# 4. The PPC Precedent Is About Ownership More Than RPC

The most important lesson from the PPC implementation is not that Rigel should copy a particular RPC protocol.

The stronger lesson is:

> **Keep large mutable state local to its owner and communicate through small shared synchronization state.**

The PPC implementation establishes relationships between the m68k and PPC engines using small shared state and interrupt-related flags rather than having both execution engines manipulate each other's complete CPU contexts.

Conceptually:

~~~text
         large state

M68KState                 PPCState
    │                         │
    ▼                         ▼
m68k owner                PPC owner


         communication

    small shared state
    atomic synchronization
    interrupt/event flags
~~~

The corresponding Rigel architecture should therefore be:

~~~text
             large state

             RigelContext
                  │
                  ▼
            Rigel owner core


            communication

          MMIO requests
         completion state
         published IPL
          event flags
~~~

---

# 5. What the PPC Investigation Does NOT Prove

The investigation does not prove that Rigel should literally copy the PPC implementation.

In particular, it does not prove that:

~~~text
SEV/WFE
~~~

is already the established Emu68 PPC IPC mechanism.

The investigated PPC doorbell implementation uses atomic shared state with release/acquire semantics.

Its wait path currently performs polling rather than demonstrating a complete event-driven `WFE/SEV` RPC mechanism.

Therefore:

~~~text
PROVEN BY PPC PRECEDENT

shared-memory signalling
release/acquire atomics
small communication state
separate CPU contexts
explicit cross-engine interrupt state
single-context ownership
~~~

while:

~~~text
NOT YET PROVEN AS THE BEST RIGEL IMPLEMENTATION

WFE/SEV for every transaction
specific request queue layout
specific RPC protocol
specific batching policy
specific spin-vs-sleep policy
~~~

These remain design and profiling questions.

---

# 6. Do Not Copy `doorbell.h` Literally

The PPC doorbell is valuable as a semantic reference.

Conceptually:

~~~text
producer
    │
write payload/state
    │
release publication
    ▼
shared word
    │
acquire observation
    ▼
consumer
~~~

This establishes a useful memory-ordering model.

However, the current polling behavior should not automatically become the Rigel implementation.

Rigel MMIO behavior has different performance characteristics.

A custom-register access may occur extremely frequently.

Therefore the final Rigel IPC mechanism should be designed around Rigel workloads rather than copied mechanically from PPC.

---

# 7. Three Separate Ownership Concepts

The evolved Bellatrix/Saiph architecture should explicitly distinguish three concepts.

## 7.1 CPU Execution Owner

Who currently executes the Amiga CPU instruction stream?

Possible values:

~~~text
Emu68
Musashi / Saiph
~~~

---

## 7.2 Rigel State Owner

Who is allowed to mutate `RigelContext`?

Preferred answer:

~~~text
Rigel execution core
~~~

or, under a DIRECT topology:

~~~text
the single execution context currently running Rigel
~~~

There must still be only one owner.

---

## 7.3 Time Authority

Who determines how far virtual Amiga time is allowed to advance?

Possible values:

~~~text
Bellatrix timing policy
Saiph CPU progress
test harness
standalone emulator scheduler
~~~

The time authority does not need to own `RigelContext`.

This is the critical distinction.

---

# 8. The Core Invariant

The architecture should therefore enforce:

~~~text
TIME AUTHORITY
      │
      │ target time
      ▼
RIGEL STATE OWNER
      │
      ▼
RigelContext
~~~

The time authority says:

~~~text
advance the machine to T
~~~

The Rigel owner performs:

~~~text
Rigel state transition
current_time → T
~~~

These are different operations.

---

# 9. Bellatrix Normal Mode

During normal Bellatrix execution:

~~~text
              Emu68
                │
                │ CPU execution
                ▼
               AROS
                │
                │ MMIO
                ▼
          Rigel transport
                │
                ▼
         Rigel owner core
                │
                ▼
          RigelContext
~~~

The timing policy remains associated with normal Bellatrix operation.

Conceptually:

~~~text
execution_owner = EMU68

time_authority  = BELLATRIX

rigel_owner     = RIGEL_CORE
~~~

The exact representation does not necessarily require these literal enums.

The separation is architectural.

---

# 10. Saiph Takeover Mode

When Saiph takes over the Amiga CPU environment:

~~~text
             Musashi
                │
                │ CPU execution
                ▼
              Saiph
                │
        ┌───────┴────────┐
        │                │
       MMIO          CPU progress
        │                │
        └───────┬────────┘
                │
                ▼
          Rigel transport
                │
                ▼
         Rigel owner core
                │
                ▼
          RigelContext
~~~

Conceptually:

~~~text
execution_owner = SAIPH

time_authority  = SAIPH_CPU

rigel_owner     = RIGEL_CORE
~~~

Therefore:

> **Saiph assuming the Rigel clock does not require Saiph to assume ownership of RigelContext.**

---

# 11. CPU-Based Timing Under Saiph Is Compatible With Single Ownership

This was an important question arising from the investigation.

The answer is yes.

Musashi can determine virtual machine progress from executed CPU cycles.

Conceptually:

~~~text
Musashi executes N cycles
          │
          ▼
Saiph converts CPU progress
into machine/Rigel time
          │
          ▼
target_time
          │
          ▼
Rigel owner advances
RigelContext to target_time
~~~

Therefore:

~~~text
CPU determines WHEN

Rigel owner determines HOW
the chipset reaches that time
~~~

There is no ownership conflict.

---

# 12. Avoid IPC Per CPU Instruction

CPU-based timing must not imply:

~~~text
execute instruction
      ↓
IPC ADVANCE
      ↓
execute instruction
      ↓
IPC ADVANCE
      ↓
execute instruction
      ↓
IPC ADVANCE
~~~

That would create excessive synchronization overhead.

The interface should operate on useful temporal intervals.

For example:

~~~text
execute N CPU cycles
       │
       ▼
calculate target_time
       │
       ▼
ADVANCE_TO(target_time)
~~~

The correct size of `N` is a scheduling/performance question.

It should not be hard-coded into the architecture prematurely.

---

# 13. Absolute Virtual Time Is Preferable to Thread Progress

The conceptual Rigel timing model should be based on virtual machine time rather than on how much host CPU time the Rigel thread happened to receive.

Prefer reasoning in terms of:

~~~text
rigel_now

rigel_advance_to(target_time)

rigel_next_deadline
~~~

rather than:

~~~text
Rigel thread runs
therefore time passed
~~~

The host scheduler must not define Amiga time.

This distinction becomes especially important when Saiph drives timing from CPU execution.

---

# 14. `next_deadline()` Becomes Strategically Important

Rigel already exposes the concept of a next deadline.

This can eventually allow CPU and chipset execution to synchronize around observable events rather than every individual chipset clock.

Conceptually:

~~~text
Rigel
  │
  ▼
next_deadline()
  │
  ▼
Tnext
  │
  ▼
convert Tnext to available CPU cycles
  │
  ▼
Musashi executes
~~~

If nothing interrupts CPU execution:

~~~text
Musashi reaches Tnext
        │
        ▼
advance Rigel to Tnext
        │
        ▼
process chipset event
~~~

This provides a natural basis for efficient CPU/chipset synchronization.

---

# 15. MMIO Creates Mandatory Synchronization Points

CPU execution cannot always simply run until the next predicted Rigel event.

An MMIO access creates an earlier synchronization point.

Example:

~~~asm
MOVE.W #$8200,$DFF096
~~~

If Musashi has executed CPU cycles since the previous Rigel synchronization:

~~~text
Musashi
   │
execute CPU cycles
   │
   ▼
MMIO write detected
   │
   ▼
synchronize Rigel to CPU current time
   │
   ▼
apply write
   │
   ▼
continue CPU
~~~

The write must occur at the correct virtual machine time.

---

# 16. MMIO Reads Require Even Stronger Synchronization

Example:

~~~asm
MOVE.W $DFF006,D0
~~~

The returned value may depend directly on beam position.

Therefore:

~~~text
CPU reaches read
      │
      ▼
CPU current virtual time
      │
      ▼
advance Rigel to that exact observable point
      │
      ▼
rigel_read()
      │
      ▼
return value
~~~

The CPU must not observe stale chipset state.

This supports keeping MMIO synchronous during the first IPC implementation.

---

# 17. MMIO Writes Should Also Initially Be Synchronous

Consider:

~~~asm
MOVE.W D0,$DFF096
MOVE.W $DFF002,D1
~~~

The second instruction may depend on the effects of the first.

Therefore the safe initial contract is:

~~~text
CPU write
   │
   ▼
publish request
   │
   ▼
Rigel owner executes write
   │
   ▼
publish completion
   │
   ▼
CPU continues
~~~

Posted/asynchronous writes may be considered later for carefully classified registers.

They should not be part of the first implementation.

---

# 18. Recommended Initial IPC Categories

Not every interaction should use the same mechanism.

The evolved architecture should distinguish at least:

~~~text
1. synchronous transactions
2. published state
3. asynchronous events
~~~

---

# 19. Synchronous Transactions

Examples:

~~~text
register read
register write
advance-to synchronization
reset/state-changing control operations
~~~

Conceptually:

~~~text
CPU
 │
 │ request
 ▼
Rigel owner
 │
 │ operation
 ▼
RigelContext
 │
 │ result/completion
 ▼
CPU
~~~

---

# 20. Published State

Some state should not require an RPC.

The most obvious example is:

~~~text
IPL
~~~

Rigel can publish the currently effective interrupt level atomically.

~~~text
Rigel owner
     │
     ▼
published_ipl
     │
     ▼
CPU frontend
~~~

This allows interrupt observation without entering `RigelContext`.

Other possible published state may eventually include:

~~~text
frame state
audio-ready state
disk event state
selected status flags
~~~

Only state with clearly defined consistency requirements should be published this way.

---

# 21. Asynchronous Events

Some events may be represented through event bits or counters.

Conceptually:

~~~text
Rigel
  │
  ▼
event_flags
  │
  ▼
CPU / Bellatrix / Saiph
~~~

These should be distinguished from synchronous register transactions.

This prevents the IPC interface from becoming a generic RPC mechanism for every piece of state.

---

# 22. Suggested Request/Completion Semantics

A simple conceptual request block may use sequence counters:

~~~c
struct RigelIPC
{
    uint32_t request_seq;
    uint32_t complete_seq;

    uint32_t op;
    uint32_t address;
    uint32_t value;
    uint32_t result;

    uint32_t published_ipl;
    uint32_t event_flags;
};
~~~

This is a design direction, not a claim that the PPC code already implements this exact structure.

Producer:

~~~text
write request payload
        │
        ▼
release-store request_seq
~~~

Consumer:

~~~text
acquire-load request_seq
        │
        ▼
read request payload
        │
        ▼
execute Rigel operation
        │
        ▼
write result
        │
        ▼
release-store complete_seq
~~~

Producer completion:

~~~text
acquire-load complete_seq
        │
        ▼
read result
~~~

The exact data structure should be chosen after measuring actual traffic.

---

# 23. Avoid One Global Lock as the Permanent Cross-Core Contract

The current lock-based model is useful and valid as an implementation stage.

It provides correctness while the architecture is being developed.

However, it should not become the permanent API contract between CPU frontend and dedicated Rigel core.

The preferred long-term boundary is:

~~~text
CPU frontend
      │
      ▼
Rigel transport
      │
      ▼
Rigel owner
      │
      ▼
RigelContext
~~~

rather than:

~~~text
CPU frontend
      │
      ▼
lock RigelContext
      │
      ▼
mutate shared state
~~~

This allows ownership to remain stable.

---

# 24. Why Stable Ownership Matters

Even when a lock prevents simultaneous mutation, alternating access can move cache lines between cores.

Conceptually:

~~~text
Core0
  │
  ▼
RigelContext cache lines
  │
  ▼
Core2
  │
  ▼
Core0
  │
  ▼
Core2
~~~

This may produce:

~~~text
cache-line migration
coherency traffic
lock contention
working-set disruption
~~~

A single owner changes this to:

~~~text
RigelContext
     │
     ▼
Rigel core cache
~~~

while communication occurs through a deliberately small shared IPC area.

This is the principal performance hypothesis to test.

It should be measured rather than assumed.

---

# 25. DIRECT and IPC Must Share the Same Semantic Contract

The architecture should not make IPC itself part of Rigel semantics.

Instead:

~~~text
              Rigel transport
                    │
             ┌──────┴──────┐
             │             │
          DIRECT          IPC
~~~

Both should provide equivalent operations.

Conceptually:

~~~text
read
write
advance_to
next_deadline
get/published IPL
reset
~~~

The difference is execution placement.

---

# 26. DIRECT Mode

When CPU and Rigel intentionally execute under one ownership context:

~~~text
CPU frontend
     │
     ▼
DIRECT transport
     │
     ▼
Rigel
~~~

No cross-core IPC is necessary.

This remains useful for:

- standalone tests;
- deterministic harnesses;
- profiling;
- possible same-core Saiph topology.

---

# 27. IPC Mode

When Rigel has a dedicated core:

~~~text
CPU frontend
     │
     ▼
IPC transport
     │
     ▼
Rigel owner core
     │
     ▼
RigelContext
~~~

This is appropriate for normal Bellatrix if profiling confirms the benefit of dedicated ownership.

It can also be used by Saiph.

---

# 28. Saiph Does Not Require IPC Rigel

This is important.

Saiph taking ownership of CPU execution does not force a particular Rigel placement.

Two valid topologies remain possible.

## Topology A — Dedicated Rigel core

~~~text
Core3                       Core2
  │                           │
Musashi                    Rigel
  │                           │
  └──────── IPC ──────────────┘
~~~

Here:

~~~text
Musashi/Saiph = time authority

Core2 = Rigel state owner
~~~

---

# 29. Saiph Same-Core Topology

Alternatively:

~~~text
             Core3
               │
       ┌───────┴───────┐
       │               │
    Musashi           Rigel
       │               │
       └──── DIRECT ────┘
~~~

Here Saiph can interleave CPU and chipset execution directly.

The semantic model remains:

~~~text
CPU progress
     │
     ▼
target virtual time
     │
     ▼
Rigel advance
~~~

Only the transport changes.

This may be particularly attractive for timing-sensitive games and demos if cross-core MMIO latency proves expensive.

It must be measured.

---

# 30. Transport Must Not Change Machine Semantics

This is an important invariant.

The following:

~~~text
Musashi
   │
DIRECT
   │
Rigel
~~~

and:

~~~text
Musashi
   │
 IPC
   │
Rigel owner core
~~~

must produce the same observable Amiga machine behavior.

Transport selection should affect:

~~~text
latency
host CPU utilization
cache behavior
parallelism
~~~

but not:

~~~text
register semantics
interrupt ordering
chipset timing
visible machine state
~~~

This property also makes DIRECT mode useful as an oracle for IPC validation.

---

# 31. CPU-Based Timing Must Also Be Transport-Neutral

Saiph should conceptually request:

~~~text
advance machine to T
~~~

Whether this becomes:

~~~text
DIRECT:
    rigel_advance_to(T)
~~~

or:

~~~text
IPC:
    send ADVANCE_TO(T)
    wait/coordinate completion
~~~

must not change virtual machine semantics.

Therefore:

> **Time authority belongs above the transport layer.**

---

# 32. Proposed Layering

The architecture should converge toward:

~~~text
             CPU FRONTEND
                  │
        Emu68 or Musashi
                  │
                  ▼
           MACHINE POLICY
                  │
       execution ownership
         time authority
                  │
                  ▼
          RIGEL TRANSPORT
             /         \
         DIRECT         IPC
            │            │
            └─────┬──────┘
                  ▼
             RIGEL OWNER
                  │
                  ▼
             RigelContext
~~~

This keeps machine policy independent from execution placement.

---

# 33. Bellatrix Evolution Path

The proposed architecture should be introduced incrementally.

It should not require a rewrite of the working Bellatrix/Rigel implementation.

Recommended evolution:

~~~text
CURRENT

shared RigelContext
       +
lock
       +
dedicated Rigel execution
~~~

then:

~~~text
STEP 1

introduce Rigel transport abstraction
       │
       ├── DIRECT
       └── existing shared/locked implementation
~~~

then:

~~~text
STEP 2

implement IPC transport
       │
       ▼
single RigelContext owner
~~~

then:

~~~text
STEP 3

move published IPL/events outside synchronous RPC
~~~

then:

~~~text
STEP 4

profile and optimize synchronization
~~~

then:

~~~text
STEP 5

reuse same interface from Saiph
~~~

This minimizes regression risk.

---

# 34. Do Not Optimize Before Establishing the Boundary

The first objective is not:

~~~text
minimum possible MMIO latency
~~~

The first objective is:

~~~text
correct ownership
correct ordering
correct timing
transport-independent semantics
~~~

Once these properties are established, optimization becomes much safer.

---

# 35. Spin vs WFE/SEV Is a Profiling Question

For synchronous requests there are several possible wait strategies.

## Pure spin

~~~text
publish request
      │
      ▼
poll complete_seq
      │
      ▼
complete
~~~

Potentially very low latency.

Potentially wastes an ARM core.

---

## Event wait

~~~text
publish request
      │
      ▼
signal owner
      │
      ▼
WFE
      │
      ▼
completion event
~~~

Potentially lower idle cost.

Potentially higher transaction latency.

---

## Hybrid

~~~text
publish request
      │
      ▼
short bounded spin
      │
      ├── complete → return
      │
      └── not complete
              │
              ▼
             WFE
~~~

This may eventually provide a useful compromise.

No option should be selected solely from intuition.

Measure first.

---

# 36. Rigel Internal CCK Stepping Remains a Separate Problem

The IPC investigation should not be confused with Rigel's internal scheduling efficiency.

These are separate questions:

~~~text
QUESTION A

How does another CPU communicate with Rigel?
        │
        ▼
IPC / DIRECT / ownership
~~~

versus:

~~~text
QUESTION B

How efficiently does Rigel advance internally?
        │
        ▼
CCK stepping / deadlines / event spanning
~~~

Fixing IPC does not automatically fix excessive CCK-by-CCK work.

Likewise, optimizing Rigel stepping does not remove cross-core ownership problems.

Both may be necessary.

---

# 37. Event-Span Fast Forward Complements CPU-Based Timing

The two investigations actually reinforce each other.

If Rigel can efficiently advance:

~~~text
current_time
      │
      ▼
next meaningful event
~~~

then Saiph can efficiently execute:

~~~text
current CPU time
      │
      ▼
corresponding next Rigel deadline
~~~

This creates a potentially efficient synchronization loop:

~~~text
Rigel next deadline
        │
        ▼
calculate CPU budget
        │
        ▼
run Musashi
        │
        ├── MMIO early
        │      │
        │      ▼
        │  synchronize early
        │
        └── reaches deadline
               │
               ▼
          advance Rigel
~~~

This is a promising long-term model.

It should not block initial correctness work.

---

# 38. Interrupt Delivery Fits the Published-State Model

Rigel already conceptually computes an effective IPL.

That state should be published outside the full `RigelContext`.

~~~text
Rigel owner
     │
compute IPL
     │
     ▼
atomic published_ipl
     │
     ├────────► Emu68
     │
     └────────► Musashi
~~~

The active CPU frontend consumes it.

This avoids synchronous Rigel calls merely to determine interrupt level.

---

# 39. Only the Active CPU Frontend May Consume the Machine

During normal Bellatrix operation:

~~~text
ACTIVE FRONTEND
      =
    Emu68
~~~

During Saiph takeover:

~~~text
ACTIVE FRONTEND
      =
   Musashi
~~~

Both must not independently drive the same Amiga machine simultaneously.

Therefore:

~~~text
Emu68 ─┐
       ├── X simultaneous ownership
Musashi┘
~~~

must be prohibited.

The ownership transition must be explicit.

---

# 40. Ownership Transition

Conceptually:

~~~text
BELLATRIX / EMU68
        │
        ▼
quiesce Amiga frontend
        │
        ▼
synchronize Rigel machine time
        │
        ▼
transfer execution ownership
        │
        ▼
SAIPH / MUSASHI
~~~

Rigel itself need not be destroyed or recreated.

The same machine state can remain alive while the CPU frontend changes.

This is an important advantage of separating CPU ownership from Rigel ownership.

---

# 41. Timing Authority Transition

Execution ownership and timing authority should change together at a well-defined synchronization point.

Conceptually:

~~~text
Bellatrix time authority
        │
        ▼
advance Rigel to handoff time T
        │
        ▼
freeze old authority
        │
        ▼
handoff
        │
        ▼
Saiph CPU time starts from T
~~~

The new authority must continue from the same virtual machine timeline.

There must not be:

~~~text
time gap
time overlap
double advancement
clock reset
~~~

unless explicitly part of a machine reset.

---

# 42. Saiph CPU Clock Conversion

Saiph will need an explicit relationship between Musashi CPU progress and Rigel virtual time.

Conceptually:

~~~text
Musashi cycles
      │
      ▼
CPU/model timing conversion
      │
      ▼
machine master time
      │
      ▼
Rigel target time
~~~

The conversion must depend on the selected machine/CPU model where appropriate.

It should not be hidden inside generic IPC code.

This belongs to Saiph machine timing policy.

---

# 43. CPU Model Changes Do Not Break the Ownership Model

When Musashi leaves the AROS SMP pool and becomes Saiph-owned, it may be reset and reconfigured.

For example:

~~~text
AROS SMP

Musashi 68040
     │
     ▼
quiesce CPU1
     │
     ▼
SAIPH ownership
     │
     ▼
reset Musashi
     │
     ▼
Musashi 68000
~~~

The selected CPU model changes CPU timing characteristics.

It does not change:

~~~text
RigelContext ownership
Rigel transport contract
machine-state ownership rules
~~~

Only the Saiph time-conversion policy needs to account for the selected CPU model.

---

# 44. Important Distinction: Clock Source vs Rigel Clock Implementation

Avoid wording such as:

~~~text
Musashi becomes the Rigel clock
~~~

if that could imply that Rigel state is directly clocked from inside Musashi.

A more precise statement is:

> **During Saiph takeover, CPU execution progress becomes the authority from which Rigel target virtual time is derived.**

This preserves layering.

~~~text
Musashi progress
      │
      ▼
Saiph time authority
      │
      ▼
Rigel target time
      │
      ▼
Rigel owner
~~~

---

# 45. Snapshot and Determinism Implications

Explicit virtual timing also improves snapshot semantics.

A complete machine snapshot should conceptually contain:

~~~text
CPU architectural state
Chip RAM
RigelContext
Rigel virtual time
pending events
published interrupt state
machine ownership state
~~~

It should not depend on:

~~~text
host thread runtime
host scheduler timing
wall-clock timing
~~~

This is particularly important for deterministic debugging.

---

# 46. Testing DIRECT Against IPC

DIRECT mode provides a valuable semantic oracle.

For the same deterministic test workload:

~~~text
RUN A

CPU
 │
DIRECT
 │
Rigel
~~~

and:

~~~text
RUN B

CPU
 │
IPC
 │
Rigel
~~~

should eventually produce equivalent:

~~~text
register results
interrupt sequence
frame state
memory effects
Rigel state
virtual timestamps
~~~

Differences then identify transport-ordering or synchronization bugs.

---

# 47. Required Profiling Modes

Before deciding whether dedicated-core IPC is superior, compare at least three configurations.

## A. DIRECT

~~~text
CPU + Rigel
same execution context
no cross-core Rigel ownership
~~~

## B. Current Bellatrix multicore model

~~~text
shared RigelContext
lock
multiple cores touching context
~~~

## C. Single owner + IPC

~~~text
RigelContext touched only by Rigel core
request/completion IPC
published state
~~~

Measure:

~~~text
MMIO read latency
MMIO write latency
requests per second
IPC wait cycles
lock wait cycles
Rigel core utilization
CPU frontend utilization
Rigel steps per second
cache/coherency effects where PMU permits
frame timing stability
audio timing stability
~~~

Only these measurements can determine the best physical topology.

---

# 48. Expected Outcomes

Several outcomes are possible.

### Result A

~~~text
IPC clearly outperforms shared context
~~~

Then dedicated ownership becomes the preferred Bellatrix topology.

### Result B

~~~text
DIRECT is substantially better for Saiph
~~~

Then Bellatrix and Saiph may deliberately use different transports while preserving identical semantics.

### Result C

~~~text
IPC MMIO latency dominates
~~~

Then investigate:

~~~text
batching
posted writes where safe
published state
hybrid wait
same-core topology
~~~

### Result D

~~~text
Rigel internal stepping dominates everything
~~~

Then IPC is not the primary performance problem and event-span optimization becomes higher priority.

---

# 49. Architectural Invariants

The following should be treated as the current proposed invariants.

## Invariant 1

~~~text
RigelContext has exactly one active mutating owner.
~~~

## Invariant 2

~~~text
CPU execution ownership is separate from Rigel state ownership.
~~~

## Invariant 3

~~~text
Time authority is separate from Rigel state ownership.
~~~

## Invariant 4

~~~text
Saiph may determine Rigel target time without directly owning RigelContext.
~~~

## Invariant 5

~~~text
Only one Amiga CPU frontend may actively drive the machine.
~~~

## Invariant 6

~~~text
MMIO reads are synchronized with current virtual machine time.
~~~

## Invariant 7

~~~text
MMIO writes remain synchronous until proven safe to post.
~~~

## Invariant 8

~~~text
Published IPL does not require entering RigelContext.
~~~

## Invariant 9

~~~text
DIRECT and IPC transports must have equivalent machine semantics.
~~~

## Invariant 10

~~~text
Host scheduling must not define Amiga virtual time.
~~~

## Invariant 11

~~~text
CPU model selection affects Saiph timing conversion,
not Rigel ownership.
~~~

## Invariant 12

~~~text
IPC optimization and Rigel internal CCK optimization
are separate engineering problems.
~~~

---

# 50. Recommended Implementation Order

The current Bellatrix implementation should evolve rather than be replaced.

Recommended sequence:

~~~text
1. Instrument the current Rigel lock/context traffic.

2. Identify every path that mutates RigelContext.

3. Identify every path that only consumes published state.

4. Introduce a Rigel transport abstraction.

5. Preserve current behavior through the first transport implementation.

6. Introduce explicit Rigel single-owner mode.

7. Implement synchronous MMIO request/completion IPC.

8. Publish IPL independently.

9. Validate ordering and machine behavior.

10. Compare DIRECT vs current locked multicore vs owner+IPC.

11. Introduce explicit virtual-time synchronization.

12. Add ADVANCE_TO semantics at the transport boundary.

13. Integrate next_deadline into timing experiments.

14. Add Saiph as an alternative CPU frontend/time authority.

15. Test CPU-based timing first through DIRECT mode.

16. Test the same Saiph timing semantics through IPC mode.

17. Only then optimize batching, waiting and event-span execution.
~~~

---

# 51. What Should Not Be Done Yet

Do not initially:

~~~text
copy PPC doorbell code verbatim
~~~

Do not assume:

~~~text
WFE/SEV is automatically faster than polling
~~~

Do not introduce:

~~~text
asynchronous posted MMIO writes everywhere
~~~

Do not make:

~~~text
Saiph directly mutate RigelContext
~~~

merely because Saiph becomes time authority.

Do not couple:

~~~text
Musashi CPU model
~~~

to:

~~~text
Rigel transport
~~~

Do not rewrite Rigel's internal scheduler merely to implement IPC.

Do not optimize multiple dimensions simultaneously before obtaining baseline measurements.

---

# 52. Resulting Architecture

The architecture emerging from these investigations is:

~~~text
                     Bellatrix / Saiph
                           MACHINE
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             │          ownership state        │
             │          virtual time           │
             │                                 │
             ▼                                 ▼
      ACTIVE CPU FRONTEND                RIGEL TRANSPORT
             │                              /       \
       ┌─────┴─────┐                    DIRECT     IPC
       │           │                       \       /
     Emu68      Musashi                     \     /
       │           │                         ▼   ▼
       │           │                    RIGEL OWNER
       │           │                         │
       │           │                         ▼
       │           │                    RigelContext
       │           │                         │
       │           │                         ▼
       │           └──── time authority ─► virtual time
       │
       └──── Bellatrix time authority
~~~

More precisely, only the active frontend contributes CPU execution.

Normal mode:

~~~text
CPU execution owner : Emu68
time authority      : Bellatrix
Rigel state owner   : Rigel owner
transport           : selected Bellatrix transport
~~~

Takeover mode:

~~~text
CPU execution owner : Musashi / Saiph
time authority      : Saiph CPU progress
Rigel state owner   : Rigel owner
transport           : DIRECT or IPC
~~~

---

# 53. Main Insight From the PPC Investigation

The strongest lesson from Emu68 PPC is not:

> "Use this exact doorbell implementation."

It is:

> **Large mutable execution state should have a clear owner. Cross-core communication should operate through a deliberately small, explicitly synchronized boundary.**

Applied to Rigel:

~~~text
DO NOT CENTER THE DESIGN ON

shared RigelContext + lock
~~~

Prefer:

~~~text
RigelContext
     │
single owner
     │
     ▼
small communication boundary
     │
     ├── synchronous MMIO
     ├── time advancement
     ├── published IPL
     └── asynchronous events
~~~

This is the most important architectural result of the investigation.

---

# 54. Main Insight From the Saiph Timing Investigation

The strongest conclusion regarding Saiph is:

> **CPU-based Rigel timing does not conflict with single-owner Rigel execution.**

Saiph can determine virtual machine progress from Musashi CPU execution while Rigel remains owned elsewhere.

~~~text
Musashi
   │
CPU cycles
   │
   ▼
Saiph
   │
target virtual time
   │
   ▼
Rigel transport
   │
   ▼
Rigel owner
   │
   ▼
RigelContext
~~~

This allows CPU/chipset synchronization without destroying the ownership model established for Bellatrix.

---

# 55. Final Direction

The current Bellatrix implementation should be treated as the working baseline, not as something to discard.

The proposed evolution is:

~~~text
shared Rigel state + lock
            │
            ▼
explicit transport boundary
            │
            ▼
single Rigel owner
            │
            ▼
small synchronized IPC surface
            │
            ▼
explicit virtual-time authority
            │
            ├── Bellatrix
            │
            └── Saiph CPU progress
~~~

This creates a common Rigel architecture capable of supporting both normal Bellatrix operation and Saiph takeover.

The physical topology remains a policy decision:

~~~text
Bellatrix:
    likely dedicated Rigel owner core

Saiph:
    evaluate both
        Musashi + Rigel same core / DIRECT
        Musashi + dedicated Rigel / IPC
~~~

Neither topology should require different Rigel machine semantics.

The immediate engineering objective should therefore be:

> **Establish ownership, transport and virtual-time boundaries first; profile physical placement second.**

This keeps the architecture compatible with the existing Bellatrix implementation while creating a clean path toward CPU-driven Saiph timing, dedicated-core Rigel execution, same-core deterministic execution, and future performance optimization.
