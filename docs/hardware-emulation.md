# Hardware Emulation: Requirements and Implementation Path

Research notes on what it takes to emulate a piece of Apple SoC hardware in this
tree, what is already covered, what is missing, and the order in which the gaps
are worth closing.

All references are `path:line` against the tree at the time of writing.

> **This document is AI-generated**, as is some of the code it describes. See the
> warning at the top of the README. Nothing here has been validated by booting
> firmware.

---

## 1. Scope

Inferno emulates whole Apple application processors, not a generic board with
Apple-flavoured peripherals. Two machines exist:

| Machine | SoC | Device | CPU model | Source |
| --- | --- | --- | --- | --- |
| `s8000` | S8000 (A9) | iPhone 6s Plus | `TYPE_APPLE_A9` | `hw/arm/s8000.c:1478` |
| `t8030` | T8030 (A13) | iPhone 11 | `TYPE_APPLE_A13` | `hw/arm/t8030.c:2742` |

Both boot a signed Apple kernelcache (or SecureROM/iBoot) against a real Apple
device tree blob supplied by the user. That single fact drives nearly every
requirement below: the guest is unmodified production firmware, so emulation is
judged by whether XNU's own drivers bind and stay happy, not by whether a
register file looks plausible.

---

## 2. What already exists

### 2.1 Core SoC infrastructure

| Block | Type | Source |
| --- | --- | --- |
| AIC (interrupt controller) | v2 layout, 576 IRQs, 6 CPUs, AIC IPIs + fast IPI | `hw/intc/apple_aic.c` |
| DART (IOMMU) | per-stream TLB cache, atomic, exposes `IOMMUMemoryRegion` per stream | `hw/arm/dart.c`, `include/hw/arm/dart.h:42` |
| SART (ANS scatter-gather) | address filter for ANS DMA | `hw/arm/sart.c` |
| PMGR | power/clock register file, voltage states, bridge settings | `hw/arm/t8030.c:963` |
| AMCC / MCC | memory controller shim (MCC still `TODO`) | `hw/arm/t8030.c:1456` |
| A7IOP + RTKit | coprocessor mailbox framework, v2 and v4 register layouts | `hw/misc/a7iop/`, `include/hw/misc/a7iop/rtkit.h` |
| APCIe | Apple PCIe host + ports, S8000/T8015/T8030 modes | `hw/pci-host/apcie.c` |
| GXF / APRR | Apple guarded execution, as decodetree instructions | `hw/arm/a13_gxf.c`, `target/arm/` |
| Fast IPI, cluster state, timers | per-cluster deferred/no-wake IPI machinery | `hw/arm/a13.c:174` onwards |

### 2.2 Buses and peripherals

I2C (`hw/i2c/apple_i2c.c`), SPI (`hw/ssi/apple_spi.c`), SPMI
(`hw/spmi/apple_spmi.c`), GPIO (`hw/gpio/apple_gpio.c`), UART
(`hw/char/apple_uart.c`), watchdog (`hw/watchdog/apple_wdt.c`), AES engine
(`hw/misc/aes.c`), SIO DMA (`hw/dma/apple_sio.c`), NVRAM
(`include/hw/nvram/apple_nvram.h`).

### 2.3 Coprocessors and complex devices

- **ANS** (NVMe storage controller as an RTKit IOP, plus `nvme_mmu`) —
  `hw/block/ans.c`, `hw/block/nvme_mmu.c`.
- **SEP** — two implementations: a real one that runs actual SEPROM/SEPFW on a
  second CPU (`hw/arm/sep/emu.c` plus AES/PKA/TRNG/SSC/keystore peripherals in
  `hw/arm/sep/`), and a protocol-level simulator (`hw/arm/sep/sim.c`) used when
  no SEP images are supplied. Selection at `hw/arm/t8030.c:2633`.
- **SMC** — system management controller with a key store — `hw/misc/smc.c`.
- **AOP** — always-on processor with a typed endpoint framework
  (`AOP_EP_TYPE_HID` / `MUX` / `APP`) — `hw/misc/aop.c`, endpoint registration
  at `hw/misc/aop.c:746`.
- **Display** — DisplayPipe v2 (S8000) and v4 (T8030) plus scaler and Synopsys
  MIPI DSIM — `hw/display/`.
- **Input** — multitouch over SPI (`hw/input/mt-spi.c`) and buttons
  (`hw/input/buttons.c`).
- **USB** — Apple OTG/TypeC plus DWC2/DWC3/XHCI and a TCP-remote USB transport
  for host-side tooling — `hw/usb/`.
- **PMU / power** — D2255 PMU, SPMI PMU, Chestnut display PMU, FAN53740 buck,
  Roswell — `hw/misc/`.
- **Audio** — MCA, CS35L27, CS42L77, AOP audio — `hw/audio/` (see §4.3 for why
  the guest does not see it).
- **Baseband** — PCIe + SPMI baseband model, behind a compile-time switch —
  `hw/misc/baseband.c`, gate at `include/hw/arm/boot.h:22`.

### 2.4 Accelerator status

TCG is the baseline. HVF work is active and recent (`hvf: get gtimer_phys
working`, `sep: vcpu state sync`, `hwaccel_enabled` introduced upstream-style),
with `hwaccel_enabled()` branches in `hw/arm/sep/emu.c:362`, `hw/arm/t8030.c:226`
and `target/arm/cpu64.c`. Note that `hw/arm/Kconfig:12` still reads
`depends on TCG && ARM && AARCH64`, so the Apple SoC code is not selectable in a
TCG-less build — worth revisiting once HVF is a first-class target.

---

## 3. The emulation contract

"Emulating a device" in this tree means satisfying six separate contracts. Any
one of them missed produces a boot hang or a kernel panic rather than a
degraded-but-working device.

### 3.1 The device tree gate

The guest sees a *filtered* copy of the user-supplied DT. `hw/arm/boot.c` holds
three lists:

- `KEEP_COMP` (`hw/arm/boot.c:43`) — a whitelist of `compatible` strings. Any
  node whose `compatible` is **not** on it is deleted
  (`hw/arm/boot.c:269`).
- `REM_NAMES` (`hw/arm/boot.c:160`) — nodes deleted by `name` regardless of
  compatible.
- `REM_DEV_TYPES` / `REM_PROPS` — deletions by `device_type` and property name.

Filtering runs in `apple_boot_populate_dt()` (`hw/arm/boot.c:492`, invoked at
`hw/arm/t8030.c:595`), which is **after** machine init has already instantiated
devices from the unfiltered tree. Two consequences:

1. Adding a device model is not enough — its `compatible` string must be added
   to `KEEP_COMP` and its name removed from `REM_NAMES`, or XNU never probes it.
2. A model can exist in the emulator while being invisible to the guest. That is
   exactly the current state of the audio stack (§4.3).

Deleting a node is also the *cheap* way to handle unimplemented hardware, and
the project uses it deliberately. The lists are therefore the authoritative
inventory of "what we do not emulate".

### 3.2 Address and IRQ binding

Devices are constructed from their DT node, not from hardcoded tables. The
convention is a `*_from_node(AppleDTNode*)` constructor (18 of them today, e.g.
`include/hw/gpio/apple_gpio.h:47`, `include/hw/arm/dart.h:42`), then in the
machine:

```c
child = apple_dt_get_node(t8030->device_tree, "arm-io");
child = apple_dt_get_node(child, name);
dev   = apple_foo_from_node(child);
object_property_add_child(OBJECT(t8030), name, OBJECT(dev));

prop = apple_dt_get_prop(child, "reg");
reg  = (uint64_t *)prop->data;
sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, t8030->armio_base + reg[0]);

prop = apple_dt_get_prop(child, "interrupts");
ints = (uint32_t *)prop->data;
for (i = 0; i < prop->len / sizeof(uint32_t); ++i)
    sysbus_connect_irq(SYS_BUS_DEVICE(dev), i,
                       qdev_get_gpio_in(DEVICE(t8030->aic), ints[i]));

sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
```

Reference: `t8030_create_gpio()` at `hw/arm/t8030.c:1232`. Note `armio_base` is
added for `arm-io` children; devices whose `reg` is already absolute (the APCIe
DARTs) pass `absolute_mmio = true` (`hw/arm/t8030.c:1068`).

### 3.3 Power domains

A device that XNU powers up must see its PMGR `ps` register acknowledge the
requested state, or the driver spins. That is handled generically by
`pmgr_reg_ops` in `hw/arm/t8030.c:963`; new devices normally need nothing, but a
device sitting in a domain whose register is served by the `pmgr-unk-reg-N`
fallback will silently read back garbage.

### 3.4 DMA

Any bus-mastering device must go through its DART stream rather than
`address_space_memory`:

```c
dart   = APPLE_DART(object_property_get_link(OBJECT(t8030), "dart-foo", &error_fatal));
mapper = apple_dt_get_node(t8030->device_tree, "arm-io/dart-foo/mapper-foo");
prop   = apple_dt_get_prop(mapper, "reg");
dma_mr = apple_dart_iommu_mr(dart, ldl_le_p(prop->data));
```

Reference: display at `hw/arm/t8030.c:1896`, AOP at `hw/arm/t8030.c:2173`. ANS is
the exception, using SART's region instead (`hw/arm/t8030.c:1165`). Getting this
wrong usually shows up as `pmap_iommu_map failed` style panics — the same class
of failure already noted for `nvme-coastguard` at `hw/arm/boot.c:216`.

### 3.5 GPIO function properties

Apple DT nodes wire board-level signals through `function-<name>` properties
(reset lines, clkreq, perst). The tree has helpers for this:
`apple_dt_connect_function_prop_out_in()` and the `_gpio` variants
(`include/hw/arm/dt.h:68`, implementation `hw/arm/dt.c:373`), used by APCIe at
`hw/pci-host/apcie.c:1456`. A new device with reset/enable GPIOs should use
these rather than hand-resolving pin numbers.

### 3.6 Build plumbing

Each new device needs: a `config APPLE_FOO` stanza in the subsystem `Kconfig`, a
`select APPLE_FOO` under `config APPLE_SOC` (`hw/arm/Kconfig:9`), a
`meson.build` entry gated on that symbol, a header under
`include/hw/<subsys>/`, and its own `trace-events` lines (every Apple subsystem
directory carries `trace-events` + a generated `trace.h`).

### 3.7 Coprocessor-specific contract

For anything behind a mailbox (`iop,*` / `iop-nub,*` compatibles), the model
must additionally speak RTKit: EP0 control protocol, rollcall, endpoint
registration (`apple_rtkit_register_user_ep()`,
`include/hw/misc/a7iop/rtkit.h:88`), and the `start` / `wakeup` / `boot_done`
ops. The IOP either runs real firmware on a second CPU (SEP model) or fakes the
protocol above the mailbox (ANS, AOP, SMC model). Choosing between those two is
the single biggest design decision for any new coprocessor — see §5.

---

## 4. Gap analysis

### 4.1 Hardware removed from the guest's device tree

Taken from `REM_NAMES` (`hw/arm/boot.c:160`) and `REM_DEV_TYPES`:

| Node(s) | Hardware | Notes |
| --- | --- | --- |
| `accel`, `gyro`, `compass`, `prox`, `spherecontrol` | motion / proximity / ambient sensors | all attach above AOP or SPU |
| `SPUApp` | sensor processing unit app endpoint | prerequisite for the sensor set |
| `gfx-asc` | GPU (AGX) coprocessor | no GPU emulation at all |
| `dart-ane` | Neural Engine | engine itself absent |
| `dart-avd`, `dart-ave` | video decode / encode | engines absent |
| `dart-isp` | camera ISP | engine absent |
| `dart-jpeg0`, `dart-jpeg1` | JPEG engines | absent |
| `dart-pmp`, `pmp` | power management processor | also `TODO: PMP` at `hw/arm/t8030.c:2585` |
| `wlan`, `bluetooth-pcie`, `amfm` | Wi-Fi / Bluetooth combo | no model |
| `stockholm`, `stockholm-spmi` | NFC | no model |
| `rose` | auxiliary controller | no model |
| `dockchannel-uart` | dock debug channel | no model |
| `smc-control`, `smc-aop`, `aop-smart-cover`, `aop-mca` | SMC/AOP sub-functions | parent devices exist |
| `Lynx` | — | removed to get SEPFW 17 booting |
| `biosensor,pearl` | Face ID / TrueDepth | commented out of `KEEP_COMP` with `// not implemented` (`hw/arm/boot.c:88`) |
| `baseband*` | cellular | modelled, but behind `ENABLE_BASEBAND`, currently off |

### 4.2 Regions mapped as `unimplemented`

`t8030_create_pcie()` maps four register windows per PCIe port as
`create_unimplemented_device()` (`hw/arm/t8030.c:1779-1785`) with a `TODO: Hook
up all ports` at `hw/arm/t8030.c:1771` (mirrored in S8000 at
`hw/arm/s8000.c:738`). `mtrtempsensor14/15` are likewise unimplemented stubs
(`hw/arm/t8030.c:2340`). These are the cheapest wins: the address ranges and
consumers are already known.

### 4.3 Modelled but not exposed

The audio stack is the clearest case. `hw/audio/mca.c`, `cs35l27.c`,
`cs42l77.c` and `aop-audio.c` exist and `t8030_create_mca()` runs
(`hw/arm/t8030.c:2190`), but every audio `compatible` string in `KEEP_COMP` is
commented out (`hw/arm/boot.c:45-113`), `aop-mca` is in `REM_NAMES`, and the AOP
audio endpoint construction is commented out at `hw/arm/t8030.c:2182`. There is
a related `TODO` about reverting product/vendor IDs "once audio support is
[done]" at `hw/arm/t8030.c:2358`. Finishing audio is therefore mostly a matter
of correctness work on existing models plus flipping the DT gate, not new device
bring-up.

Similarly `scaler,t8030` and `pmu,d2255` are commented out of `KEEP_COMP`
(the latter marked `// buggy`) while both models exist.

### 4.4 Machine parity gaps

S8000 lacks a number of blocks T8030 has: SMC, SPMI, SIO, MCA/audio, buttons,
multitouch, AOP, temperature sensors, Roswell, scaler, MIPI DSIM
(compare `hw/arm/s8000.c:1390-1417` with `hw/arm/t8030.c:2589-2672`). Some of
these are genuine hardware differences; buttons, touch and SMC are not.

### 4.5 Cross-SoC generalisation

Several blocks are hardcoded to T8030 and would need parameterising before a
third machine (T8010/T8015/T8020 are all referenced in passing —
`hw/pci-host/apcie.c:1583`, `hw/arm/sep/debug-trace.c:193`,
`hw/block/nvme_mmu.c:235`):

- ~~`AIC_INT_COUNT` / `AIC_CPU_COUNT` / `AIC_VERSION` are `#define`s with an
  explicit `// TODO: this is hardcoded for T8030`.~~ **Done** — the switch case
  ranges are now bounded by the limits the register map imposes
  (`AIC_MAX_INT_COUNT` / `AIC_MAX_EIR_COUNT` / `AIC_MAX_CPU_COUNT`) rather than
  by T8030's counts, the counts themselves were already taken from the device
  tree, `apple_aic_create()` validates them, and the reported revision is a
  `version` property. Compile-checked only.
- SEP boot-monitor and debug-trace carry per-chip address tables keyed off
  `chip_id` (`hw/arm/sep/debug-trace.c:174`).
- APCIe already branches on compatible string, which is the pattern to copy.

### 4.6 Correctness debt in existing models

Worth tracking separately from missing hardware, because these are the things
that make long-running guests fail:

- SEP AES: `// TODO: This is 100% wrong, but it works anyhow/anyway`
  (`hw/arm/sep/aes.c:237`), plus CMAC and iteration-register hacks
  (`hw/arm/sep/aes.c:244,266,284`).
- DART has an explicit anti-panic hack (`hw/arm/dart.c:713`).
- TZ0 sizing is a workaround for SEPOS ≥ 16 (`hw/arm/t8030.c:334`).
- Both machines force a DEV hardware model to avoid FDR errors
  (`hw/arm/t8030.c:554`, `hw/arm/s8000.c:395`).
- SMC battery/sensor keys incomplete (`hw/misc/smc.c:587-594`).
- `kernel_patches.c` has a host-endianness `TODO` (`hw/arm/kernel_patches.c:56`).

### 4.7 Verification gap

There is no `tests/` directory in this tree and CI (`.github/workflows/build.yaml`)
only compiles on four runners. Every behavioural claim is currently validated by
booting real firmware by hand. This is the largest structural risk to any of the
work below: there is no mechanism that would catch a regression in, say, DART or
AIC other than someone noticing a boot failure.

---

## 5. Choosing an implementation strategy per device

Five archetypes cover essentially everything left:

| Archetype | Effort | When to use | Reference to copy |
| --- | --- | --- | --- |
| **Plain MMIO block** | low | fixed register file, no DMA, no firmware | `hw/watchdog/apple_wdt.c` |
| **Bus peripheral** (I2C/SPI/SPMI) | low | discrete chip on a bus that already exists | `hw/misc/roswell.c` (I2C), `hw/misc/spmi-pmu.c` (SPMI) |
| **AOP endpoint** | low–medium | sensors and HID-shaped data sources | `hw/misc/aop.c:746`, `AppleAOPEndpointDescription` |
| **RTKit coprocessor, protocol-faked** | medium–high | IOP whose firmware we do not want to run | `hw/block/ans.c`, `hw/arm/sep/sim.c` |
| **RTKit coprocessor, firmware-executing** | very high | IOP whose firmware is the point (SEP) | `hw/arm/sep/emu.c` |
| **PCIe endpoint** | medium | anything behind APCIe | `hw/misc/baseband.c` |

The protocol-faked path is almost always the right default. Running real
firmware is only justified when the firmware's *behaviour* is the artefact under
study, as with SEP; it costs a second CPU model, a private address space, its
own peripheral set (`hw/arm/sep/` is ~7.9 kLOC for one coprocessor) and a
continuous chase after firmware-version drift.

For the GPU specifically, note that neither path yields graphics acceleration:
AGX is a firmware-driven coprocessor plus a command-stream ISA, and the display
path here is a framebuffer scanned out of DART-mapped memory. A `gfx-asc` model
that satisfies binding without rendering is feasible; actual GPU rendering is
out of proportion to everything else in this list.

---

## 6. Proposed implementation path

Ordered by (value to a booting, usable device) ÷ (effort × risk).

### Phase 0 — make the work verifiable

Prerequisite for everything else, given §4.7.

1. Add a boot smoke test harness: drive a machine to a known DT/kernel state and
   assert on serial output, runnable without user-supplied Apple firmware where
   possible (SecureROM-only path, or a synthetic DT).
2. Extend CI beyond compile-only to at least run that harness.
3. Fold the scattered `TODO`s in §4.6 into tracked issues so correctness debt is
   visible next to feature work.

### Phase 1 — finish what is already modelled

Highest value per line of new code; no new device models required.

1. **Audio end-to-end.** Uncomment the audio `compatible` entries in `KEEP_COMP`,
   drop `aop-mca` from `REM_NAMES`, re-enable `apple_aop_audio_create()`
   (`hw/arm/t8030.c:2182`), fix the MCA channel-count `FIXME`
   (`hw/audio/mca.c:256`), then revert the product/vendor ID workaround
   (`hw/arm/t8030.c:2358`).
2. **Scaler and D2255 PMU.** Both models exist; re-enable their DT entries and
   fix what the `// buggy` note refers to.
3. **PCIe port register windows.** Replace the four
   `create_unimplemented_device()` calls per port (`hw/arm/t8030.c:1779`) with
   real backing, and hook up the remaining ports (`hw/arm/t8030.c:1771`).
4. **SMC key coverage.** Battery heat map `BHT0` and the sensor-key split
   (`hw/misc/smc.c:587-594`) — this is what blocks the battery settings page.

### Phase 2 — sensors via AOP

The AOP endpoint framework already supports exactly this shape, so the marginal
cost per sensor is small once the first one lands.

1. Implement the `SPUApp` endpoint (`AOP_EP_TYPE_APP`).
2. Add HID endpoints for `accel`, `gyro`, `compass`, `prox`, ambient light.
3. Remove each from `REM_NAMES` and add its `compatible` to `KEEP_COMP` as it
   starts binding.
4. Wire host input where it makes sense (orientation from the UI layer).

Value: device orientation, auto-rotate, proximity blanking — user-visible
behaviour that is currently simply absent.

### Phase 3 — S8000 parity

Port buttons, multitouch SPI, SMC and the temperature sensors from T8030 to
S8000 (`hw/arm/s8000.c:1390`). These models are SoC-agnostic; the work is
machine wiring plus DT whitelist entries, not new emulation.

### Phase 4 — cross-SoC generalisation

Unblocks any third machine, and is cheaper to do before more devices harden
T8030 assumptions.

1. ~~Parameterise AIC — interrupt count, CPU count and version rather than
   `#define`s.~~ **Done**, see §4.5. Note the counts were already DT-derived
   into `numIRQ`/`numEIR`/`numCPU`; what was hardcoded were the compile-time
   `case` ranges around them, which silently sent any access past T8030's
   counts to the unimplemented-register path.
2. Extend the SEP chip-id tables (`hw/arm/sep/debug-trace.c:174`) to a table
   keyed by chip id rather than an `if` ladder.
3. Audit each `*_from_node()` for T8030-only assumptions.
4. Only then attempt a T8015-class machine, which has the most existing partial
   support already scattered through the tree.

### Phase 5 — large absent blocks

Each of these is its own project; listed in descending order of return.

1. **PMP** — referenced from the DT, removed wholesale today
   (`hw/arm/t8030.c:2585`), and its absence forces `pmp*` property stripping in
   `REM_PROPS`.
2. **Baseband promotion** — the model exists; the work is making it good enough
   to enable `ENABLE_BASEBAND` (`include/hw/arm/boot.h:22`) by default rather
   than as an opt-in build flag.
3. **`gfx-asc` binding stub** — a mailbox-level GPU coprocessor that lets the
   GPU driver attach without rendering.
4. **ISP / ANE / AVD / AVE / JPEG** — each needs a DART stream, an RTKit
   endpoint set and a plausible command-completion model. Value is limited
   unless there is a specific workload that needs them.
5. **Wi-Fi / Bluetooth / NFC** — PCIe and SPMI endpoints; large firmware
   protocol surface, and networking is already available through USB.

Deliberately not on this path: real GPU rendering, and any further
firmware-executing coprocessor beyond SEP.

---

## 7. Checklist for adding one device

1. Identify the DT node and its `compatible`, `reg`, `interrupts`, and any
   `function-*` properties in a real device tree for the target machine.
2. Pick an archetype from §5.
3. Header in `include/hw/<subsys>/`, model in `hw/<subsys>/`, with a
   `*_from_node()` constructor.
4. `config APPLE_FOO` in the subsystem `Kconfig`; `select APPLE_FOO` under
   `config APPLE_SOC` (`hw/arm/Kconfig:9`); `meson.build` entry gated on it.
5. `trace-events` entries; no bare `printf`.
6. `<machine>_create_foo()` following §3.2, called from the machine's init list.
7. DMA through the DART stream (§3.4) if the device masters the bus.
8. Add the `compatible` to `KEEP_COMP` and remove the name from `REM_NAMES`
   (`hw/arm/boot.c`) — without this the guest never sees the device.
9. Check the PMGR domain responds (§3.3).
10. Boot, and confirm from the kernel log that the real driver attached, not just
    that the registers were touched.
