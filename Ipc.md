# Saiph / Musashi / Rigel Roadmap

## Machine Takeover Architecture for Bellatrix

**Status:** Investigation and implementation roadmap  
**Primary target:** Raspberry Pi 3B  
**Host:** Emu68 + AROS m68k  
**Takeover CPU candidate:** Musashi  
**Amiga chipset:** Rigel  
**Compatibility reference/runtime:** JST / WHDLoad ecosystem  

---

# 1. Objective

Saiph exists to solve a specific problem:

> Allow classic Amiga software that expects full machine takeover to execute while AROS and the Raspberry Pi hardware drivers remain alive.

Classic takeover software may assume ownership of:

- CPU vectors;
- interrupts;
- custom chipset;
- CIA;
- Chip RAM;
- Copper;
- Blitter;
- Paula;
- floppy;
- timing;
- low 24-bit address space.

On Bellatrix this cannot be allowed to stop the host AROS environment, because AROS owns the Raspberry Pi services required for:

- USB;
- Bluetooth;
- storage;
- HDMI/video;
- audio;
- input devices;
- filesystem access;
- host UI.

The intended responsibility split is:

~~~text
Bellatrix
    owns the physical Raspberry Pi.

AROS
    owns host operating-system services.

Saiph
    owns the takeover machine boundary.

Musashi
    executes the takeover 68k CPU.

Rigel
    provides Amiga chipset semantics.

JST
    provides a compatibility path toward the
    WHDLoad/JST software ecosystem.
~~~

The fundamental principle is:

> Saiph virtualizes machine ownership, not the Amiga programming model.

Classic software should believe it owns an Amiga.

AROS must continue to own the physical Raspberry Pi.

---

# 2. Architectural evolution

The initial idea was something similar to:

~~~text
AROS
  |
musashi.library
  |
Musashi
  |
guest 68k
~~~

This would be useful as a prototype, but a m68k `musashi.library` would itself execute through Emu68:

~~~text
ARM
 |
Emu68
 |
m68k Musashi
 |
interpreted guest m68k
~~~

This introduces an unnecessary execution layer.

The PPC implementation already present inside Emu68 provides a much better precedent.

The preferred architectural direction is now:

~~~text
Raspberry Pi 3B

Core 0
  |
Emu68
  |
AROS
  |
Bellatrix


Secondary ARM core
  |
Saiph native AArch64
  |
Musashi
  |
takeover guest 68k
~~~

A future AROS library may still exist, but as a control-plane frontend:

~~~text
AROS
  |
saiph.library
  |
shared control block
  |
inter-core signalling
  |
Saiph native service
~~~

Possible control API:

~~~c
SaiphCreateMachine();
SaiphLoadADF();
SaiphLoadSlave();
SaiphStart();
SaiphPause();
SaiphResume();
SaiphStop();
SaiphGetStatus();
~~~

The library should not execute Musashi itself.

---

# 3. Emu68 PPC as the primary precedent

The PPC subsystem of Emu68 is important because it already demonstrates the class of architecture needed by Saiph.

It provides examples of:

- secondary Cortex-A53 bootstrap;
- independent execution context;
- per-core stacks;
- MMU/cache setup;
- startup synchronization;
- shared-memory communication;
- atomic signalling;
- doorbell-style inter-core communication;
- communication between the m68k environment and another CPU engine;
- execution of a translator independently on another ARM core.

Saiph should reuse or generalize these mechanisms instead of creating a second multicore runtime from scratch.

---

# 4. PPC investigation roadmap

Before implementing Saiph multicore integration, study the PPC subsystem systematically.

## 4.1 Secondary-core startup

Determine:

- how cores 1–3 are released;
- how the PPC core is selected;
- where secondary-core stacks are allocated;
- how MMU tables are installed;
- how cache configuration is inherited or established;
- how VBAR and exception handling are installed;
- whether idle secondary cores park using `WFE`;
- how startup uses `SEV`;
- how the PPC engine is held until the main environment is ready;
- how lifecycle and shutdown are handled.

Identify whether a generic interface can be extracted:

~~~text
secondary_service_init()
secondary_service_start()
secondary_service_signal()
secondary_service_wait()
secondary_service_stop()
~~~

Do not duplicate this infrastructure inside Saiph unless necessary.

---

## 4.2 PPC execution-context isolation

Study:

~~~text
PPCState
InitPPC()
StartupPPC()
PPC execution loop
context load/save
PPCStart
~~~

Separate:

~~~text
generic second-CPU runtime
~~~

from:

~~~text
PPC translator implementation
~~~

The desired long-term structure would be:

~~~text
Emu68 secondary-core runtime
           |
           +-- PPC service
           |
           +-- Saiph service
~~~

---

# 5. PPC inter-core communication is directly relevant to Rigel

The PPC precedent is not useful only for Saiph.

It may also teach us how to improve the current:

~~~text
Emu68 / AROS
      <->
Rigel dedicated core
~~~

communication in Bellatrix.

This deserves independent investigation.

---

# 6. Current Rigel multicore model

The current Bellatrix integration conceptually resembles:

~~~text
Core 0 / Emu68               Rigel core
      |                          |
      |      RigelContext        |
      |           |              |
      +-- lock ------------------+
      |  rigel MMIO access       |
      +-- unlock                 |
                                 |
                          lock   |
                      rigel_step |
                        unlock   |
~~~

This serializes access correctly, but there is an architectural concern:

> More than one core enters and mutates the Rigel context.

The lock prevents simultaneous execution, but ownership of cache lines and Rigel state can move between cores.

Possible costs include:

- lock contention;
- cache-line ping-pong;
- cache coherency traffic;
- Rigel state moving between cores;
- unpredictable MMIO latency;
- interference with the dedicated Rigel execution loop.

The current high Rigel CPU usage therefore must not be attributed exclusively to the scheduler until this has been measured.

---

# 7. Preferred Rigel ownership model

The better model is:

> One core owns `RigelContext`.

Other cores communicate with that owner through a small explicit protocol.

Conceptually:

~~~text
              Core 0
             Emu68/AROS
                 |
                 | request
                 v
          shared IPC area
                 |
                 v
             Rigel core
                 |
                 v
           RigelContext
~~~

The CPU frontend should not directly enter arbitrary Rigel state from another core.

This matches the lesson from the Emu68 PPC architecture.

---

# 8. Rigel IPC model

A first prototype should investigate PPC-style communication based on:

- shared control structure;
- atomic sequence/state variables;
- release/acquire ordering;
- lightweight signalling;
- `SEV/WFE` or equivalent mechanism where appropriate.

A minimal transaction could conceptually contain:

~~~c
struct RigelIPCRequest
{
    uint32_t request_seq;
    uint32_t complete_seq;

    uint16_t reg;
    uint16_t value;

    uint8_t operation;
};
~~~

This is only a conceptual starting point.

The actual implementation should derive from the proven PPC communication mechanism where possible.

---

# 9. Synchronous MMIO transactions

Some MMIO operations necessarily require request/response semantics.

Example:

~~~text
Core 0                        Rigel core

MOVE.W $DFF006,D0
    |
    v
publish READ request
    |
    +------------------------>
                               rigel_read()
                                   |
                               result
    <------------------------+
    |
resume m68k
~~~

The CPU cannot continue until a register read has returned.

Writes may initially also be synchronous to preserve ordering.

Example:

~~~text
MOVE.W D0,$DFF096
MOVE.W $DFF002,D1
~~~

The second operation must not incorrectly observe state preceding the first write.

Therefore:

> Start with strict synchronous MMIO semantics.

Only introduce asynchronous writes after proving that ordering remains correct.

---

# 10. Published Rigel state

Some state does not require a synchronous transaction.

Bellatrix already demonstrates the right pattern with IPL publication.

Instead of:

~~~text
Emu68
   |
rigel_get_ipl()
   |
RigelContext
~~~

prefer:

~~~text
Rigel core
    |
calculate IPL
    |
publish atomically
    |
    v
published_ipl
    |
    v
Emu68
~~~

This principle can be extended to:

- IPL;
- frame sequence;
- video-ready state;
- audio-ready state;
- disk events;
- status flags;
- diagnostic counters.

Conceptually:

~~~text
               RigelContext
                    |
          +---------+---------+
          |         |         |
         IPL      frame     audio
          |         |         |
          +---------+---------+
                    |
              published state
                    |
             other ARM cores
~~~

---

# 11. Rigel transport abstraction

The architecture should avoid forcing every host to use IPC.

Introduce conceptually two transports:

~~~text
Amiga CPU frontend
        |
        v
 Rigel transport
      /     \
     /       \
 DIRECT      IPC
   |          |
   v          v
Rigel     Rigel owner core
~~~

`DIRECT` is useful when CPU and Rigel share a core.

`IPC` is useful when Rigel has a dedicated core.

This becomes particularly valuable for Saiph.

---

# 12. Saiph and Rigel topologies

Two valid topologies remain.

## 12.1 Same-core topology

Preferred if profiling and optimization show sufficient CPU budget:

~~~text
Core 0
    Emu68 + AROS

Core 3
    Saiph
      |
      +-- Musashi
      |
      +-- Rigel
~~~

Advantages:

- direct MMIO;
- no cross-core transaction per custom-register access;
- no Rigel locks;
- simpler CPU/chipset timing;
- easier takeover synchronization;
- fewer cache coherency transitions.

This is particularly attractive for demos, games and timing-sensitive software.

---

## 12.2 Dedicated Rigel topology

Fallback if Rigel remains too expensive:

~~~text
Core 0
    Emu68 + AROS

Core X
    Rigel

Core 3
    Saiph + Musashi
~~~

In this case:

~~~text
normal mode

Emu68
  |
 Rigel IPC
  |
Rigel core
~~~

and:

~~~text
takeover mode

Musashi
  |
 Rigel IPC
  |
Rigel core
~~~

Only the active Amiga CPU frontend may communicate with the machine chipset.

---

# 13. Execution ownership

Introduce an explicit machine-level CPU owner:

~~~c
enum amiga_execution_owner
{
    AMIGA_OWNER_EMU68,
    AMIGA_OWNER_SAIPH
};
~~~

Normal mode:

~~~text
owner = EMU68

Emu68 -> Amiga machine
Saiph -> blocked/stopped
~~~

Takeover mode:

~~~text
owner = SAIPH

Musashi -> Amiga machine
Emu68 low-24 -> blocked/controlled
~~~

Do not allow simultaneous CPU ownership.

---

# 14. Machine-centric Bellatrix integration

Rigel should no longer be conceptually owned by the Emu68 MMIO frontend.

Prefer an abstraction such as:

~~~text
BellatrixAmigaMachine
    |
    +-- Chip RAM
    +-- RigelContext
    +-- execution owner
    +-- timing state
    +-- machine configuration
~~~

Then:

~~~text
Emu68 frontend
      |
      v
BellatrixAmigaMachine
~~~

and:

~~~text
Saiph frontend
      |
      v
BellatrixAmigaMachine
~~~

Do not duplicate chipset semantics.

Avoid:

~~~text
emu68_amiga_bus_semantics
musashi_amiga_bus_semantics
~~~

Instead maintain one canonical machine decoder.

Example:

~~~text
classify(address)

RAM
ROM
CUSTOM
CIA
FLOPPY
UNMAPPED
~~~

The CPU frontend is replaceable.

The Amiga machine semantics are not.

---

# 15. Chip RAM model

Chip RAM should remain shared between the active CPU frontend and Rigel where possible.

~~~text
                 Chip RAM
                    |
          +---------+---------+
          |                   |
      active CPU             Rigel
      frontend               DMA
~~~

Normal mode:

~~~text
Emu68 <-> Chip RAM <-> Rigel
~~~

Takeover mode:

~~~text
Musashi <-> Chip RAM <-> Rigel
~~~

Do not create a second Chip RAM unless a concrete correctness requirement appears.

---

# 16. Low-memory/vector-page exception

The first page needs special investigation.

AROS/Emu68 uses low-memory structures, particularly address `$000004` / `AbsExecBase`.

Therefore:

~~~text
"AROS has no important state in low-24"
~~~

is too broad.

Most operational memory may be outside low-24, but the vector page remains special.

Potential solution:

~~~text
$000000-$000FFF

normal mode
    -> AROS / Emu68 low page

takeover mode
    -> Saiph guest vector page
~~~

while the remainder of Chip RAM remains shared.

This should be solved before arbitrary ADF or takeover execution.

---

# 17. Rigel performance problem

Bellatrix currently contains a measurement indicating roughly:

~~~text
~250 ns per colour clock
~~~

versus approximately:

~~~text
~282 ns available per colour clock
~~~

leading to around:

~~~text
~88% of one Cortex-A53
~~~

for the current Rigel execution loop.

This measurement should be treated as:

> A property of the current implementation.

It should not yet be treated as:

> An intrinsic cost of emulating an Amiga chipset.

The Bellatrix `legacy` material referring to the small UAE implementation running on much weaker microcontroller hardware is a strong reason to investigate this cost.

---

# 18. Two independent Rigel performance hypotheses

At least two major hypotheses now exist.

~~~text
                high Rigel CPU usage
                        |
              +---------+---------+
              |                   |
              v                   v
       fine CCK stepping     multicore access
              |                   |
     scheduler overhead      locks/cache bounce
~~~

Do not optimize only one side without measuring the other.

---

# 19. Rigel scheduler suspicion

Rigel exposes deadline-oriented APIs externally, but internally important Agnus paths still effectively operate at very fine CCK granularity.

Conceptually:

~~~text
CCK
 |
slot arbitration
 |
Copper
 |
DMA
 |
Blitter checks
 |
beam
 |
Denise
 |
next CCK
~~~

This may happen millions of times per second.

Even inactive intervals can therefore pay repeated dispatch costs.

The key performance question is:

> Can Rigel preserve exact observable bus semantics without dispatching a generic scheduler for every CCK?

---

# 20. Event/span scheduler direction

The desired optimization is not reduced chipset correctness.

The desired optimization is:

> Preserve accurate semantics while skipping intervals where nothing observable changes.

Instead of:

~~~text
slot 100
slot 101
slot 102
slot 103
slot 104
...
~~~

calculate:

~~~text
current time
    |
    +-- next Copper wake
    +-- next bitplane DMA
    +-- next sprite DMA
    +-- next Blitter slot
    +-- next disk event
    +-- next audio event
    +-- next line boundary
    |
    v
nearest meaningful event
~~~

Then fast-forward through inactive spans.

Potential directions:

- run-length processing of unused slots;
- specialized bitplane DMA loops;
- multiple granted Blitter slots per call;
- block beam advancement;
- event-driven Copper WAIT;
- skip completely inactive subsystems;
- scanline-level Denise processing where safe.

Target:

~~~text
same observable machine behavior
            +
less scheduler overhead
~~~

---

# 21. Rigel profiling must separate scheduler from IPC cost

The profiler must test at least three execution arrangements.

## Test A — standalone/direct Rigel

~~~text
single ARM core
no inter-core locking
no Emu68 interaction
~~~

Measure pure Rigel cost.

---

## Test B — current Bellatrix model

~~~text
dedicated Rigel core
current shared RigelContext
current locking
Emu68 MMIO path
~~~

Measure scheduler plus present multicore overhead.

---

## Test C — single-owner PPC-style IPC

~~~text
dedicated Rigel core
private RigelContext
shared transaction block
atomic request/response
PPC-style signalling
~~~

Compare:

~~~text
DIRECT
vs
CURRENT LOCK
vs
IPC
~~~

This experiment is essential.

Example possible outcomes:

~~~text
Standalone      70%
Current lock    80%
IPC             72%
~~~

would indicate primarily a scheduler problem.

Whereas:

~~~text
Standalone      30%
Current lock    80%
IPC             35%
~~~

would indicate the multicore communication model is a major problem.

Or both may contribute substantially.

---

# 22. Internal subsystem profiling

Measure accumulated ARM cycles and call counts for:

~~~text
rigel_step
Agnus scheduler
slot dispatch
Copper
Blitter
bitplane DMA
sprite DMA
beam
Denise tick
scanline compositor
Paula
disk
CIA
Chip RAM callbacks
MMIO handling
locking
inter-core waiting
~~~

Test workloads:

~~~text
A. Empty chipset
   DMA disabled

B. Simple one-bitplane display

C. Wanderer / Workbench

D. Game/demo

E. Timing active, rendering disabled

F. Denise disabled, DMA active

G. Copper disabled

H. Simplified/immediate Blitter
~~~

The empty-chipset case is especially important.

If it remains expensive, scheduler overhead is almost certainly significant.

---

# 23. Compiler/build verification

Before interpreting performance measurements, confirm the exact Rigel build flags.

Test:

~~~text
-O0
-O2
-O3
-O3 -mcpu=cortex-a53
~~~

Also confirm that debug, probe and hosted-environment features are not unintentionally active in the measured Bellatrix build.

Do not redesign the scheduler based on an unverified build configuration.

---

# 24. Rigel timing profiles

Bellatrix/AROS and Saiph have different goals.

A future Rigel integration may support two timing profiles.

## CLASSIC

Used by Saiph and compatibility-sensitive software.

~~~text
PAL/NTSC timing
authentic DMA slots
authentic bus arbitration
Copper races
Blitter timing
beam-visible timing
classic chipset behavior
~~~

---

## ACCELERATED

Used by normal Emu68/AROS operation.

Do not implement acceleration merely by:

~~~text
CCK frequency * 2
CCK frequency * 4
~~~

because this could multiply scheduler cost.

Instead accelerate physical chipset bottlenecks that do not need to constrain an accelerated AROS system.

Possible examples:

- faster Blitter completion;
- relaxed Chip RAM arbitration;
- batched DMA;
- event-driven inactive periods;
- accelerated internal chipset work.

Physical presentation can still remain:

~~~text
PAL  -> 50 Hz
NTSC -> approximately 60 Hz
~~~

The result is an accelerated Amiga-like machine rather than a simply overclocked PAL raster.

---

# 25. Musashi integration

Musashi remains a strong candidate for takeover execution because it provides:

- explicit CPU context;
- execution by cycle budget;
- IRQ injection;
- memory callbacks;
- multiple CPU-context support;
- portable C implementation;
- support for Amiga-relevant prefetch behavior.

The machine callbacks should remain thin:

~~~c
m68k_read_memory_8();
m68k_read_memory_16();
m68k_read_memory_32();

m68k_write_memory_8();
m68k_write_memory_16();
m68k_write_memory_32();
~~~

They should call the canonical machine decoder.

Do not embed Amiga chipset semantics directly inside Musashi callbacks.

---

# 26. Standalone Saiph first

Before AROS integration, build a standalone Saiph harness:

~~~text
native host
   |
Musashi
   |
Saiph machine decoder
   |
   +-- RAM
   +-- ROM
   +-- Rigel
~~~

Validate:

- CPU reset;
- memory accesses;
- ROM execution;
- Chip RAM;
- custom registers;
- CIA;
- IRQ;
- timing;
- simple diagnostic programs.

This isolates Saiph problems from Emu68/AROS problems.

---

# 27. ADF boot milestone

Before JST/WHD complexity, prove that the machine can boot a normal ADF.

Required:

- Musashi;
- ROM/Kickstart-compatible environment;
- Chip RAM;
- Rigel custom chipset;
- CIA;
- floppy backend;
- interrupt path;
- video;
- input;
- audio.

Expected flow:

~~~text
Musashi reset
      |
Kickstart
      |
Rigel DF0:
      |
ADF
      |
bootblock
      |
game/demo/OS
~~~

Start with normal ADF.

Protected formats can come later.

---

# 28. Host-service bridge

Saiph must never gain Raspberry Pi hardware drivers.

Physical hardware remains controlled by Bellatrix/AROS.

## Input

~~~text
USB / Bluetooth
       |
     AROS
       |
Bellatrix
       |
shared input snapshot
       |
     Saiph
       |
     Rigel
       |
JOY / POT / CIA
~~~

---

## Audio

~~~text
guest Paula
    |
  Rigel
    |
audio buffer/FIFO
    |
   Saiph
    |
   AROS
    |
Bellatrix HDMI/PWM
~~~

The host audio callback should not directly advance guest virtual time.

---

## Video

~~~text
guest bitplanes/Copper/Blitter
            |
          Rigel
            |
       completed frame
            |
      publish/swap buffer
            |
           AROS
            |
        VideoCore
~~~

Use explicit buffer ownership across cores.

---

## Storage

AROS owns physical storage.

Saiph receives logical media or file services:

~~~text
ADF
ROM
WHD slave
game files
disk blocks
~~~

The guest must not require Raspberry Pi storage drivers.

---

# 29. JST/WHD work comes later

Treat JST initially as:

- WHD/JST compatibility reference;
- `resload_*` reference;
- slave-loader reference;
- patching reference;
- filesystem bridge reference.

Do not make JST the core of Saiph architecture.

First prove:

~~~text
Musashi + Rigel + ROM + ADF
~~~

Then implement the WHD/JST layer.

---

# 30. Snapshot/save-state roadmap

Complete snapshot support is not required for the first milestone.

Eventually a Saiph snapshot should include:

~~~text
Musashi context
Chip RAM
guest Fast/Slow RAM if present
CIA state
custom-chip state
Copper
Blitter
Paula
disk state
beam position
Rigel timing state
~~~

This enables:

- suspend/resume;
- save states;
- deterministic debugging;
- reproducible tests;
- potentially session switching.

---

# 31. Recommended implementation order

## Phase 0 — Establish performance truth

### Phase 0A — Rigel standalone profiling

Measure pure Rigel cost without multicore interference.

### Phase 0B — Study Emu68 PPC IPC

Document:

- doorbells;
- atomics;
- memory ordering;
- signalling;
- shared control structures;
- request/ack model;
- `SEV/WFE` usage;
- interrupt interaction.

### Phase 0C — Prototype PPC-style Rigel IPC

Compare:

~~~text
DIRECT
CURRENT LOCK
PPC-STYLE IPC
~~~

### Phase 0D — Profile Rigel internals

Identify scheduler/render/device hotspots.

### Phase 0E — Optimize scheduler where justified

Introduce fast-forward/event/span execution without changing observable semantics.

Deliverable:

~~~text
Rigel Pi3 Performance and Inter-Core Report
~~~

---

# 32. Phase 1 — Generalize the Emu68 secondary-core model

Study the PPC implementation and extract the reusable architecture.

Deliverable:

~~~text
Emu68 Secondary CPU Service Architecture
~~~

The document should define:

- core bootstrap;
- lifecycle;
- IPC;
- context ownership;
- signalling;
- memory sharing;
- cache/MMU assumptions.

---

# 33. Phase 2 — Standalone Saiph

Build:

~~~text
Musashi
   |
Saiph machine
   |
Rigel
~~~

No AROS integration yet.

---

# 34. Phase 3 — Boot ADF

Add:

- ROM;
- floppy backend;
- input;
- video;
- audio.

Target:

> Boot a normal Amiga ADF using Musashi + Rigel.

---

# 35. Phase 4 — Decide Rigel core topology

Only after profiling and optimization decide between:

~~~text
Core 3
    Musashi + Rigel
~~~

and:

~~~text
Core X
    Rigel

Core 3
    Musashi
~~~

Do not freeze this earlier.

---

# 36. Phase 5 — Native Saiph on Emu68 secondary core

Reuse/generalize the PPC mechanism.

Target:

~~~text
Core 0
    Emu68 + AROS

Core 3
    Saiph native AArch64
      |
      +-- Musashi
~~~

If Rigel fits on the same core:

~~~text
Core 3
    Saiph
      +-- Musashi
      +-- Rigel
~~~

Otherwise use the IPC transport.

---

# 37. Phase 6 — Bellatrix machine ownership

Introduce explicit ownership:

~~~text
AMIGA_OWNER_EMU68
AMIGA_OWNER_SAIPH
~~~

Validate:

- Emu68 and Saiph never own low-24 simultaneously;
- Chip RAM remains correct;
- vector-page handling is safe;
- Rigel ownership is well defined;
- AROS continues running during takeover.

---

# 38. Phase 7 — Host bridges

Connect:

- USB;
- Bluetooth;
- storage;
- HDMI;
- audio;
- VideoCore;
- host notifications.

The success criterion is:

> Guest takeover code runs while AROS and all Raspberry Pi services remain alive.

---

# 39. Phase 8 — Accelerated AROS profile

After CLASSIC behavior and performance are understood, experiment with:

~~~text
RIGEL_PROFILE_CLASSIC
RIGEL_PROFILE_ACCELERATED
~~~

Investigate acceleration of:

- Blitter;
- Chip bus;
- DMA scheduling;
- inactive time spans.

Do not simply multiply CCK execution rate.

---

# 40. Phase 9 — JST/WHD integration

Implement enough of the WHD/JST environment to run takeover slaves without shutting down AROS.

Target:

~~~text
AROS stays alive
       |
     Saiph
       |
WHD/JST slave
       |
classic game/demo
~~~

---

# 41. Phase 10 — Advanced features

Later work:

- complete save states;
- debugger;
- rewind/reproducibility;
- protected disk formats;
- 68020+ performance tuning;
- alternate CPU engines;
- possible second Emu68 m68k-context investigation;
- richer host-service protocols.

---

# 42. Decisions that must remain open

Do not decide yet that:

~~~text
Rigel permanently needs a dedicated core.
~~~

Measure first.

Do not decide that:

~~~text
Rigel must run with Musashi.
~~~

That depends on the measured CPU budget.

Do not decide that:

~~~text
musashi.library is the execution architecture.
~~~

The PPC precedent strongly favors native secondary-core Saiph.

Do not decide that:

~~~text
Rigel IPC must use a brand-new protocol.
~~~

First investigate and reuse the PPC communication design.

Do not decide that:

~~~text
Rigel should simply run at 2x or 4x clock for AROS.
~~~

Accelerated mode should remove unnecessary physical bottlenecks instead.

Do not make JST fundamental to the machine model.

Do not duplicate:

- Rigel semantics;
- Chip RAM without necessity;
- Raspberry Pi drivers;
- machine address decoding.

---

# 43. Immediate priorities

The next work should be performed in this order:

~~~text
1. Study the PPC inter-core communication in Emu68.

2. Measure Rigel standalone on Pi 3B.

3. Measure current Bellatrix Rigel multicore overhead.

4. Prototype PPC-style single-owner Rigel IPC.

5. Compare DIRECT vs LOCK vs IPC.

6. Profile Agnus / Denise / scheduler internals.

7. Optimize Rigel only after separating IPC cost
   from scheduler cost.

8. Generalize the Emu68 secondary-core mechanism.

9. Build standalone Saiph with Musashi + Rigel.

10. Boot a normal ADF.

11. Decide final Rigel core topology.

12. Integrate Saiph into Bellatrix/Emu68.

13. Add host bridges.

14. Add JST/WHD compatibility.

15. Explore accelerated Rigel profile for AROS.
~~~

---

# 44. Critical questions to answer

The roadmap should not move to major architectural changes until these questions have concrete answers.

## Rigel

~~~text
How expensive is Rigel with no multicore communication?

How much of the cost is Agnus/CCK scheduling?

How much is Denise/rendering?

How much is lock/cache coherency overhead?

Can event/span scheduling retain existing semantics?

Can Rigel and Musashi fit together on one Cortex-A53?
~~~

## PPC / Emu68

~~~text
What exact mechanism does PPC use for inter-core IPC?

Which pieces are generic?

Can Rigel use the same mechanism?

Can Saiph use the same secondary-core bootstrap?

Can a generic Emu68 secondary service abstraction be extracted?
~~~

## Saiph

~~~text
How is low-24 ownership switched safely?

How is the first/vector page isolated?

Can Chip RAM remain fully shared?

How are host input/audio/video/storage services bridged?

Does classic takeover require tighter CPU/Rigel timing
than normal Bellatrix operation?
~~~

---

# 45. Final architectural direction

The current preferred conceptual model is:

~~~text
                        Raspberry Pi 3B

                           Bellatrix
                               |
              +----------------+----------------+
              |                                 |
           Core 0                         Secondary cores
              |                                 |
            Emu68                         Emu68 runtime
              |                                 |
            AROS                         +-------+-------+
              |                          |               |
        host services                  Saiph           Rigel
              |                          |               |
        USB / BT / SD                Musashi        chipset state
        HDMI / audio                   |
        VideoCore                    guest 68k
              |
              +------------ controlled bridges --------+
~~~

The exact placement of Rigel remains intentionally unresolved.

If optimization makes this practical:

~~~text
Core 3
    Saiph
      +-- Musashi
      +-- Rigel
~~~

is architecturally attractive.

If Rigel remains expensive:

~~~text
Core X
    single-owner Rigel

Core 3
    Saiph + Musashi

Core 0
    Emu68 + AROS
~~~

with PPC-style IPC becomes the preferred fallback.

The important architectural invariant is independent of this decision:

> Bellatrix owns the physical machine.  
> AROS owns the host services.  
> Saiph owns takeover execution.  
> Rigel owns Amiga chipset semantics.  
> The active CPU frontend owns the Amiga CPU view.  
> Inter-core communication must not leak host ownership into the guest machine.
