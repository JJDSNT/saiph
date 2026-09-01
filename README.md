# Saiph

**Saiph** is an experimental runtime for executing classic Amiga software that expects to take exclusive control of the machine, while allowing the host operating system and platform services to remain alive.

The initial target environment is **Bellatrix**, running AROS m68k on Emu68.

Saiph addresses a specific problem:

> Classic Amiga software that performs a machine takeover assumes that it owns the CPU, interrupts, vectors, Chip RAM and custom chipset. On Bellatrix, allowing such software to take over the actual AROS/Emu68 execution environment would also stop the AROS services and Raspberry Pi drivers that provide graphics, audio, USB, Bluetooth, storage and other platform functionality.

Saiph introduces a separate 68k execution context for this software.

The host AROS system remains alive.

The takeover software receives the Amiga environment it expects.

---

# 1. Project Goal

The primary goal of Saiph is to support **machine takeover scenarios** without allowing the takeover software to take control of the physical Bellatrix host.

Examples include:

- WHDLoad slaves;
- JST-compatible software;
- bootable ADF images;
- custom bootblocks and trackloaders;
- games and demos that disable the operating system;
- software that directly owns the Amiga custom chipset;
- software that replaces interrupt vectors or disables normal OS scheduling.

The fundamental model is:

~~~text
Physical machine
      │
      ▼
  Bellatrix
      │
      ▼
 Emu68 + AROS
      │
      ├── Raspberry Pi drivers
      ├── USB / Bluetooth
      ├── storage
      ├── physical video output
      ├── physical audio output
      └── host services
             │
             │ remains alive
             │
             ▼
           Saiph
             │
      ┌──────┴──────┐
      │             │
   Musashi         Rigel
      │             │
 guest 68k      Amiga chipset
      │             │
      └──────┬──────┘
             │
       takeover software
~~~

Saiph therefore does **not** attempt to replace Bellatrix or AROS.

It provides an isolated execution environment for software that cannot safely coexist with the host operating system.

---

# 2. Core Principle

Saiph virtualizes **machine ownership**, not the Amiga programming model.

A takeover program should still see the environment it expects:

~~~text
68k CPU
   │
   ├── Chip RAM
   ├── Amiga address space
   ├── Custom registers
   ├── CIA
   ├── interrupts
   ├── Copper
   ├── Blitter
   ├── Paula
   └── floppy hardware
~~~

The program should not need to know that:

- the physical CPU is ARM;
- AROS is still running;
- USB input comes from an AROS driver;
- audio may ultimately leave through HDMI;
- video may ultimately be presented by VideoCore;
- an ADF may physically reside on an SD card.

From the takeover program's point of view, it owns an Amiga.

From Bellatrix's point of view, it owns the physical machine.

---

# 3. Why a Second 68k CPU Context?

Normal AROS applications run directly under Emu68.

This remains the preferred execution path:

~~~text
68k application
      │
      ▼
    Emu68
      │
      ▼
     AROS
~~~

There is no reason for Saiph to intercept or emulate software that can coexist normally with AROS.

Machine takeover software is different.

Such software may:

- disable interrupts;
- disable multitasking;
- replace exception vectors;
- change the VBR;
- manipulate the supervisor state;
- reset or reconfigure hardware;
- assume exclusive ownership of Chip RAM;
- directly program the Amiga chipset;
- expect the operating system to disappear.

Executing this directly in the host Emu68 context would conflict with the requirement that AROS remain operational.

Saiph therefore introduces another 68k execution context:

~~~text
                    ARM host

             ┌─────────┴─────────┐
             │                   │
           Emu68              Musashi
             │                   │
            AROS           takeover 68k
             │                   │
       host services        Amiga domain
~~~

The initial CPU engine for this context is **Musashi**.

---

# 4. Memory Model

Saiph should not begin by assuming that a completely separate virtual Amiga address space is required.

Bellatrix already has a deliberate distinction between the classic Amiga low address domain and the memory used by the running AROS system.

The intended takeover model is therefore based on **ownership of the Amiga domain**.

Conceptually:

~~~text
                     Bellatrix memory

       AROS / host domain              Amiga domain
              │                            │
              │                         low-24
              │                            │
           Emu68                    ┌──────┴──────┐
              │                     │             │
             AROS                Chip RAM        MMIO
              │                     │             │
         host drivers               │           Rigel
                                    │
                              takeover CPU
~~~

AROS host memory remains owned by Emu68.

The takeover CPU receives access to the classic Amiga-visible domain.

This allows AROS to continue executing while the takeover program behaves as if it owns the Amiga.

## 4.1 Chip RAM

The initial direction is to reuse the existing Bellatrix Chip RAM rather than automatically creating another copy.

Conceptually:

~~~text
                     Chip RAM
                        │
          ┌─────────────┼─────────────┐
          │             │             │
        Emu68         Musashi        Rigel
~~~

Only the appropriate CPU frontend should own the Amiga domain at a given time.

This has several advantages:

- no Chip RAM synchronization;
- no takeover-time Chip RAM copy;
- Rigel DMA continues to operate against the same memory;
- addresses remain stable;
- takeover software observes the expected machine state.

This policy must be validated during the initial implementation.

---

# 5. Amiga Address Space

The Amiga address map must have one canonical definition.

Saiph must not create a second implementation of Amiga hardware semantics merely because it uses another CPU engine.

The conceptual topology is:

~~~text
classify(address)
      │
      ├── RAM
      ├── ROM
      ├── CUSTOM
      ├── CIA
      ├── FLOPPY
      └── UNMAPPED
~~~

Different CPU engines may reach this topology differently.

For the normal Bellatrix path:

~~~text
Emu68
  │
  ├── RAM  ───────────────► direct mapping
  │
  └── MMIO ──► fault ─────► Bellatrix bus ──► Rigel
~~~

For Saiph:

~~~text
Musashi
   │
   └── memory callback ───► Saiph/Bellatrix bus ──► Rigel
~~~

The important invariant is:

> CPU frontend differences must not result in duplicated chipset semantics.

---

# 6. Machine Ownership

Saiph introduces the concept of an active owner of the Amiga machine domain.

Initially this can be modeled as:

~~~c
enum amiga_execution_owner
{
    AMIGA_OWNER_EMU68,
    AMIGA_OWNER_SAIPH
};
~~~

During normal operation:

~~~text
owner = EMU68

Emu68
  │
  ▼
Amiga low-24
  │
  ▼
Rigel
~~~

During takeover:

~~~text
owner = SAIPH

Musashi
   │
   ▼
Amiga low-24
   │
   ▼
Rigel
~~~

Meanwhile:

~~~text
Emu68
   │
   ▼
AROS host memory
   │
   ├── USB
   ├── Bluetooth
   ├── storage
   ├── audio
   └── video
~~~

continues operating.

The exact enforcement mechanism for low-24 ownership is an implementation question and should be validated experimentally before being fixed as a permanent API.

---

# 7. Rigel

Rigel is the natural chipset implementation for Saiph.

Saiph should not implement another Amiga chipset.

The intended relationship is:

~~~text
              Saiph
                │
       ┌────────┴────────┐
       │                 │
    Musashi            Rigel
       │                 │
       └──── Amiga bus ──┘
~~~

Rigel remains responsible for Amiga hardware semantics such as:

- custom registers;
- CIA behavior;
- interrupt generation;
- DMA;
- Copper;
- Blitter;
- Paula;
- video timing;
- floppy behavior where implemented.

Saiph is responsible for the takeover execution environment around it.

## 7.1 Rigel as a Dependency

The initial repository may include Rigel as a submodule.

This is useful because it allows Saiph to be tested independently from Bellatrix:

~~~text
Saiph test harness
      │
      ├── Musashi
      └── Rigel
~~~

However, integration with Bellatrix may eventually require Saiph to operate against an existing Bellatrix/Rigel machine instance rather than creating an independent chipset instance.

The architecture should therefore avoid assuming that Saiph permanently owns the lifecycle of Rigel.

Rigel should remain independently reusable and host-neutral.

---

# 8. Musashi

Musashi is the initial CPU engine for takeover execution.

It is well suited to this role because it provides an explicit software 68k CPU context and does not depend on the architectural state of the Emu68 instance running AROS.

Musashi should be treated as a CPU frontend.

It must not directly implement Bellatrix or Rigel policy.

For example:

~~~c
m68k_read_memory_16(address)
{
    return saiph_bus_read16(machine, address);
}
~~~

Conceptually:

~~~text
Musashi
   │
   ▼
Saiph CPU adapter
   │
   ▼
Saiph machine/bus
   │
   ├── memory
   └── Rigel
~~~

This separation is important because Musashi may not remain the only possible takeover CPU engine.

If performance later requires a JIT engine or a second Emu68 context, the machine model should survive that change.

---

# 9. JST and WHDLoad Compatibility

JST is initially included primarily as a **compatibility reference and potential source of reusable implementation**.

Saiph should not initially depend on JST for its fundamental machine model.

JST contains knowledge useful for:

- WHDLoad slave handling;
- `resload_*` behavior;
- patch lists;
- game data loading;
- WHDLoad compatibility;
- existing takeover software expectations.

However, Saiph already exists specifically to provide a different machine takeover architecture.

Therefore the initial goal is not:

> Port JST wholesale into Saiph.

Instead:

> Study and reuse the parts of JST that are useful for building a Saiph-native WHD/JST runtime.

A future structure may therefore look like:

~~~text
Saiph
 │
 ├── machine
 │    ├── Musashi
 │    ├── Rigel
 │    └── memory/bus
 │
 ├── ADF boot
 │
 └── WHD runtime
      ├── slave loader
      ├── resload
      └── patch support
~~~

JST itself should initially remain a separate submodule rather than becoming tightly coupled to Saiph core code.

---

# 10. ADF Boot

Saiph is not intended to be limited to WHDLoad.

A second important execution model is booting a classic Amiga disk image.

Conceptually:

~~~text
Musashi reset
      │
      ▼
Kickstart / compatible ROM
      │
      ▼
Rigel DF0:
      │
      ▼
ADF
      │
      ▼
bootblock
      │
      ▼
game / demo / OS
~~~

The ADF itself can be supplied by the host.

The guest sees floppy hardware.

For example:

~~~text
AROS filesystem
      │
      ▼
   game.adf
      │
      ▼
Saiph host interface
      │
      ▼
Rigel floppy
      │
      ▼
guest DF0:
~~~

This allows trackloaders and bootblock software to execute in the environment they expect without requiring them to understand the Bellatrix filesystem.

ADF boot also provides an important validation path because it exercises the complete machine model without first requiring WHDLoad/JST compatibility.

---

# 11. Host Services

Saiph must not contain Raspberry Pi drivers.

In particular, Saiph should not implement or own:

- USB;
- Bluetooth;
- SD hardware;
- VideoCore;
- HDMI;
- physical audio devices;
- physical input devices.

These belong to Bellatrix/AROS.

Saiph should instead expose a narrow host boundary.

Conceptually:

~~~c
struct saiph_host_ops
{
    /* time / scheduling */
    uint64_t (*get_time)(void *host);

    /* input */
    void (*get_input)(void *host, struct saiph_input *input);

    /* guest output */
    void (*present_video)(void *host, const struct saiph_frame *frame);
    void (*submit_audio)(void *host, const struct saiph_audio *audio);

    /* storage / media */
    int (*read_media)(void *host, ...);

    /* host notification */
    void (*signal_event)(void *host, unsigned event);
};
~~~

The exact API is intentionally not fixed yet.

The important architectural rule is:

> Saiph consumes host services; it does not become the owner of physical platform hardware.

---

# 12. Input

Input illustrates the separation clearly.

A physical Bluetooth or USB controller may be handled entirely by AROS:

~~~text
USB / Bluetooth controller
          │
          ▼
      AROS driver
          │
          ▼
   Bellatrix / Saiph
       input bridge
          │
          ▼
        Rigel
          │
          ▼
JOYxDAT / POT / CIA
          │
          ▼
        guest
~~~

The guest therefore sees an Amiga joystick or mouse.

It never sees the physical USB or Bluetooth device.

---

# 13. Audio

The same principle applies to audio.

The guest programs Paula normally:

~~~text
guest
  │
  ▼
Paula registers
  │
  ▼
Rigel
  │
  ▼
PCM/audio stream
  │
  ▼
host bridge
  │
  ▼
AROS / Bellatrix audio driver
  │
  ▼
physical output
~~~

The guest does not need to understand HDMI or the Raspberry Pi audio hardware.

---

# 14. Video

Video follows the same ownership boundary:

~~~text
guest
  │
  ├── bitplanes
  ├── Copper
  ├── Blitter
  └── display registers
          │
          ▼
        Rigel
          │
          ▼
      guest frame
          │
          ▼
      host bridge
          │
          ▼
AROS / Bellatrix video path
          │
          ▼
      VideoCore
~~~

This allows the guest to believe it owns the Amiga display while AROS continues to own the physical display hardware.

---

# 15. Timing

CPU and chipset execution must remain synchronized.

The initial implementation should favor correctness and simplicity over premature parallelization.

A useful conceptual execution loop is:

~~~text
determine next chipset deadline
            │
            ▼
run Musashi for bounded cycles
            │
            ▼
advance Rigel
            │
            ▼
process resulting events
            │
            ▼
repeat
~~~

Musashi and the timing-critical Rigel core may initially execute on the same dedicated ARM core.

This avoids introducing cross-core synchronization into the CPU/chipset timing model before performance measurements show that it is necessary.

Expensive derived work such as physical frame presentation or audio delivery may later be moved away from the takeover execution core.

Performance must be measured rather than assumed.

---

# 16. Initial Repository Layout

A possible starting layout is:

~~~text
saiph/
│
├── external/
│   ├── musashi/
│   ├── jst/
│   └── rigel/
│
├── src/
│   ├── cpu/
│   │   └── musashi.c
│   │
│   ├── machine/
│   │   ├── machine.c
│   │   ├── memory.c
│   │   ├── bus.c
│   │   └── ownership.c
│   │
│   ├── host/
│   │   └── host.c
│   │
│   ├── adf/
│   │   └── ...
│   │
│   └── whd/
│       └── ...
│
├── tests/
│
└── README.md
~~~

This structure is only an initial direction.

The first implementation should remain small enough that the architecture can evolve as the takeover model is validated.

---

# 17. Initial Development Plan

The project should begin by proving the central architectural assumption before attempting complete WHDLoad or ADF compatibility.

## Phase 1 — Musashi Bring-Up

Integrate Musashi and execute a minimal 68k program.

Verify:

- reset state;
- registers;
- memory reads/writes;
- exceptions;
- interrupts;
- bounded execution.

No JST or ADF support is required.

---

## Phase 2 — Saiph Machine Bus

Introduce the Saiph machine/address-space layer.

~~~text
Musashi
   │
   ▼
Saiph bus
   │
   ├── RAM
   ├── ROM
   └── MMIO
~~~

The CPU adapter must not know chipset implementation details.

---

## Phase 3 — Rigel Integration

Connect Amiga MMIO to Rigel.

~~~text
Musashi
   │
   ▼
Saiph bus
   │
   ├── Chip RAM
   │
   └── Amiga MMIO
            │
            ▼
          Rigel
~~~

Start with a trivial 68k program that writes directly to custom registers.

For example, the first useful test does not need an operating system:

~~~asm
        move.w  #$0200,$dff100
        move.w  #$0f00,$dff180

loop:
        bra     loop
~~~

The purpose is to prove:

> Musashi can execute 68k takeover code against the Saiph/Rigel machine model.

---

## Phase 4 — Timing and Interrupts

Connect:

- CPU cycles;
- Rigel progression;
- IPL;
- CIA interrupts;
- custom interrupts;
- DMA deadlines.

The objective is a deterministic CPU/chipset execution loop.

---

## Phase 5 — Bellatrix Coexistence

Run the takeover environment while AROS remains operational.

Verify that:

~~~text
Emu68 / AROS
      │
      ├── continues scheduling
      ├── USB remains alive
      ├── Bluetooth remains alive
      ├── storage remains alive
      └── platform drivers remain alive

Musashi / Saiph
      │
      └── owns the Amiga takeover domain
~~~

This is the primary architectural milestone of the project.

---

## Phase 6 — Host Bridges

Add the minimum interfaces required for useful execution:

- input;
- video presentation;
- audio delivery;
- media access.

The host remains responsible for physical devices.

---

## Phase 7 — ADF Boot

Add enough machine support to boot a simple ADF.

Target progression:

~~~text
ROM executes
     ↓
DF0 recognized
     ↓
bootblock loaded
     ↓
bootblock executes
     ↓
simple game/demo runs
~~~

Start with standard ADF images before considering more complex protected disk formats.

---

## Phase 8 — JST/WHD Research

Only after the takeover machine works should JST integration become a major implementation task.

Identify:

- required WHDLoad slave structures;
- minimum `resload_*` subset;
- patch handling;
- filesystem requirements;
- assumptions made by existing slaves.

Use JST as an implementation reference where appropriate.

---

## Phase 9 — First WHDLoad Slave

Implement the minimum runtime required to execute one deliberately selected, simple slave.

Do not attempt broad WHDLoad compatibility immediately.

Use each compatibility failure to identify missing runtime behavior.

---

# 18. Performance Strategy

The first goal is **correct execution**, not maximum performance.

The initial expected configuration is approximately:

~~~text
ARM core 0
    │
    └── Emu68 / AROS / host services

ARM core 1
    │
    └── Musashi + Rigel timing
~~~

Additional cores may later be used for work that does not need cycle-level synchronization.

The project should measure:

- Musashi host CPU usage;
- guest cycles per second;
- Rigel processing cost;
- audio generation cost;
- video generation cost;
- synchronization overhead;
- missed timing/deadlines.

If Musashi proves insufficient for demanding software, the CPU boundary should allow a future engine such as a dedicated Emu68/JIT context.

This is one reason the rest of Saiph must not depend directly on Musashi internals.

---

# 19. Non-Goals

At the initial stage Saiph is **not** intended to:

- replace Emu68 for normal AROS execution;
- emulate Raspberry Pi hardware;
- implement USB or Bluetooth stacks;
- replace Bellatrix;
- replace Rigel;
- become another general-purpose Amiga emulator;
- duplicate Amiga chipset semantics;
- provide immediate compatibility with every WHDLoad slave;
- solve every protected floppy format;
- implement Amiga SMP.

Its initial purpose is deliberately narrower:

> Provide a safe execution boundary for classic 68k software that expects to take over an Amiga machine.

---

# 20. Architectural Invariants

The following principles should guide early development.

### 1. AROS must survive takeover

The reason Saiph exists is to prevent guest machine ownership from becoming physical host ownership.

### 2. Normal software stays on Emu68

Saiph should not impose CPU emulation overhead on software that can run normally under AROS.

### 3. Musashi is an execution engine, not the machine

Machine semantics belong to Saiph and Rigel.

### 4. Rigel remains host-independent

Saiph must not push Bellatrix or Raspberry Pi dependencies into Rigel.

### 5. Physical hardware belongs to the host

USB, Bluetooth, storage, VideoCore and physical audio remain Bellatrix/AROS responsibilities.

### 6. The guest sees Amiga hardware

Host implementation details must not leak into the takeover programming model.

### 7. Avoid duplicated chipset implementations

Emu68 and Musashi may reach the Amiga bus differently, but ultimately they must use the same canonical hardware semantics.

### 8. Optimize only after measuring

Start with a deterministic, understandable CPU/chipset execution model.

Parallelize or replace the CPU engine only when measurements justify it.

---

# 21. Initial Dependencies

The initial project is expected to track three external projects:

~~~text
Musashi
    CPU execution engine

JST
    WHD/JST compatibility reference and potential reusable code

Rigel
    Amiga chipset implementation
~~~

They have deliberately different roles.

Musashi and Rigel participate directly in the initial execution architecture.

JST initially provides knowledge and compatibility reference material.

The relationship may evolve as implementation progresses.

---

# 22. Long-Term Direction

If the initial architecture proves successful, Saiph can become a common execution environment for several classes of classic Amiga software:

~~~text
                   Saiph
                     │
        ┌────────────┼────────────┐
        │            │            │
      ADF          WHD/JST     raw takeover
        │            │            │
        └────────────┼────────────┘
                     │
                 68k machine
                     │
            ┌────────┴────────┐
            │                 │
         CPU engine         Rigel
            │                 │
            └────────┬────────┘
                     │
               host services
                     │
                  Bellatrix
                     │
               Emu68 + AROS
                     │
          Raspberry Pi hardware
~~~

This creates a clear division of responsibility:

**Bellatrix owns the physical machine.**

**AROS owns host operating-system services.**

**Saiph owns the takeover execution boundary.**

**Musashi executes the takeover CPU context.**

**Rigel provides the Amiga chipset.**

**JST provides an important compatibility path toward the existing WHDLoad software ecosystem.**

The first objective is not broad compatibility.

The first objective is to prove that these boundaries work.
