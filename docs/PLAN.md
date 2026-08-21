# The implementation plan

Walk in, do the port. Ordered so that every phase ends in a **milestone that either happened or did
not**, and so that anything testable on the laptop is tested there — a console flash costs ten minutes
plus a human, and this project has spent whole afternoons on flashes that a host run would have caught.

## The architecture decision, and it is the one that makes this maintainable

**Do not patch `ac_linux_drm.c`. Replace it.**

* It is **one entry** in `src/amd/common/meson.build:203`.
* `ac_linux_drm.h` declares every one of them **out-of-line** — zero `static inline` (checked). So a file
  of ours providing the same symbols is a drop-in.

So the Mesa side of this port is **one meson line**: drop `ac_linux_drm.c`, add `ac_orbis_drm.c`. Our
implementation lives in this repo, never fights an upstream rebase, and is not a diff against somebody
else's file. The alternative — a third `is_gnm` arm inside `ac_linux_drm.c` — means a large patch whose
hunks move every time upstream touches that file, which is the exact maintenance trap
[`task-92`](https://github.com/Try/Tempest) exists to get out of on the OpenGothic side.

    patches/0005-meson-swap-ac_linux_drm-for-ac_orbis_drm.patch     ← one line
    src/ac_orbis_drm.c                                              ← ours, 54 functions
    src/orbis_sync_provider.c                                       ← ours, the 14-pointer vtable
    src/orbis_wsi.c                                                 ← ours, sceVideoOut

---

## Phase 0 — unblock the link — **DONE, 2026-08-08**

**Why it was first:** until the link runs, the function list is a grep's inference rather than the
linker's statement, and every later phase is built on it.

**What actually unblocked it was not dependency plumbing.** The port had been compiling Mesa's DRM path
and shimming libdrm to satisfy it. Setting Mesa's own no-DRM predicate instead took **97 failed objects
and 244 errors to zero** — see [`research/03-the-platform-predicate.md`](research/03-the-platform-predicate.md).
Three things fell out of it that change later phases:

1. **`libvulkan_radeon.a`, 35.6 MB, 781 targets, zero failures.** RADV had to become `library()` rather
   than `shared_library()` for `-Ddefault_library=static` to apply — a platform with no ICD loader links
   the driver in, and our link line is an executable's, so a `.so` link fails on an undefined `main`.
2. **The undefined list is 13 `ac_drm_*`, not 54**, because the amdgpu winsys is excluded and it is the
   winsys that calls the other 41. **That is a staging tool**: 13 functions reach an enumerated device,
   and memory/submit/syncobj arrive with the winsys rather than before it.
3. **`thrd_current` is declared in OpenOrbis' `threads.h` and missing from `libc.a`** — the third
   toolchain lie, after `struct stat`'s layout and `std::abs`. One line, in our code, not a Mesa patch.

**Falsifier, and it held:** the 13 are a strict subset of `notes/ac_drm-surface-gfx7.txt`, and the header
declares 59 of which exactly five are the GFX11+ user-queue entries — so 54 by two independent
derivations after being wrong at 43 and at 39.

The empty stub `libdrm` is **deleted**, and so is the claim it was built on. It made meson believe libdrm
existed, which switched on five more files of DRM code that then needed shimming.

---

## Phase 1 — the skeleton arm — **DONE, 2026-08-08**

`src/ac_orbis_drm.c` + `src/orbis_compat.c`, wired in by `patches/0006` (five meson lines) and copied into
the scratch clone by `build.sh`.

**Milestone reached, and verified from a clean clone:** six patches apply, zero failures,
`libvulkan_radeon.a` at 35.6 MB, and **zero `ac_drm_*` symbols left undefined**.

**And past the milestone: the archive LINKS into a PS4 executable** — `ELF64 / DYN / x86-64`, 26.1 MB, zero
undefined symbols (`tools/linkprobe.sh`, now a build step). Which immediately caught 21 undefined symbols
from the HOST's zlib and libelf, reached through meson's cmake fallback and lld's default library search
paths, on a configure that had reported success. See README's *cross-build hygiene*.

Thirteen bodies, not 54, for the reason phase 0 uncovered — the winsys is excluded, so nothing references
the other 41 and writing them now would be writing them blind. Ten are `ORBIS_DRM_TODO()` returning
`-ENOSYS`; three are `ORBIS_DRM_REFUSED()` with the reason in the log, because a refusal is a decision and
must not read as an unfinished task.

Three could not be stubbed that way and were handled instead of skipped:

* `ac_drm_device_get_sync_provider` returns a real static `util_sync_provider` whose five **required**
  entries fail loudly and whose other nine are NULL. NULL for the struct itself would be a null
  dereference during device init, because `has_timeline_syncobj` is read straight off
  `provider->timeline_wait`. Leaving that member NULL is the load-bearing part: RADV then does not
  advertise `KHR_timeline_semaphore` or `KHR_present_wait`, an honest absence.
* `ac_drm_device_deinitialize` returns `void`, so it carries the free from the start rather than acquiring
  it later.
* `ac_drm_get_marketing_name` returns a real string, because NULL shows up as an empty device name in
  every tool that lists devices — precisely during the phase where enumeration is what is being debugged.

**And the C11 threads gap is four symbols, not one.** `thrd_current`, `thrd_detach`, `thrd_equal` and
`tss_get` are declared in OpenOrbis' `threads.h` and absent from its `libc.a` (42 defined, 25 declared,
measured). Only `thrd_current` is referenced by Mesa — `thrd_equal` never becomes a call because
`threads.h:55` gives it a macro — so only that one is defined, and the other three are named in the file
so the next link failure is recognised rather than re-measured.

---

## Phase 1b — the enumeration hook, which the plan did not have — **DONE, 2026-08-08**

**Nothing was ever going to call the thirteen queries.** RADV registers only
`physical_devices.try_create_for_drm` (`radv_instance.c:324`), and the Vulkan runtime reaches that
exclusively by walking DRM nodes — so on a platform with no DRM the driver enumerates zero devices and
every `ac_drm_*` body is dead code. Phase 2 as written could not be tested at all, and that was found by
asking what would call it rather than by running into it.

`vk_instance.h:155` names the replacement, and `vk_instance.c:448` shows it short-circuits the DRM walk
when it returns anything but `VK_ERROR_INCOMPATIBLE_DRIVER`, so nothing is faked or bypassed. Three pieces:

* `enumerate_orbis_physical_devices()` — creates the one device and `list_addtail`s it, the shape lavapipe
  and v3dv use.
* `src/amd/vulkan/radv_orbis_winsys.c` — the sibling of `radv_amdgpu_winsys_query_info`, which lives in the
  excluded winsys and is the only thing that calls `ac_query_gpu_info`. **This is the file that connects
  RADV to our arm.**
* `ac_drm_device_initialize` — now real, because it is what the above calls first.

**And the DRM version we report is derived, not chosen.** `ac_gpu_info.c` constrains it from both sides:
`assert(drm_major == 3)`, `drm_minor < 54` is a hard failure, `>= 55` claims a gpuvm-fault query we refuse
and `>= 59` claims zerovram behaviour we lack. **3.54.0 is the only value that is accepted and switches on
nothing this port does not provide.**

**The syncobj layer came for free, and that was the surprise.** The amdgpu winsys builds the whole
`vk_sync` implementation with `vk_drm_syncobj_get_type(fd)`, which looked like a layer we would have to
write. But `vk_drm_syncobj_get_type_from_provider(struct util_sync_provider *)` exists and Intel's anv
already uses it that way — so our provider plugs straight in, and phase 5's five counter operations are the
only thing left underneath it. Getting there needed one upstream fix that is worth having anyway:
`vk_drm_syncobj.c` was confined to `if dep_libdrm.found()` by an `#include <xf86drm.h>` it does not use,
while making **not one `drm*()` call** — every DRM reference in it is a constant from Mesa's own vendored
`drm-uapi/drm.h`.

**Milestone:** `libvulkan_radeon.a` builds and the link probe still comes out clean at 26.2 MB.

---

## Phase 2 — device queries — **DONE, AND CONFIRMED ON HARDWARE, 2026-08-08**

**RADV enumerates the GPU on a real PS4 and its `radeon_info` is IDENTICAL to the host's: 164 fields, zero
differences** (`notes/radeon_info-console.txt` vs `notes/radeon_info-orbis.txt`). `timestampPeriod 10.000
ns` on the console — the fork's measured 100 MHz reaching the Vulkan API.

That equality is worth more than either dump alone. It says the cross-compilation is faithful across 164
derived values including arrays, that our `ac_drm_*` arm behaves the same under musl and glibc, and
therefore that **the host loop is a valid proxy for the console for everything in this phase** - which
turns "test on the laptop until it cannot answer" from a policy into a measured fact.

Getting there cost four defects, each found by a log rather than a guess, and **two of them deleted patches
instead of adding code**: `0003` (C11 threads) and `0004` (its consequence) are gone. See README's *the
console deleted two of the six patches*.

---

## Phase 2 — the laptop record

**`vkEnumeratePhysicalDevices` returns one device and it describes Liverpool:**

    infoprobe: 1 physical device(s)
    infoprobe: [0] PlayStation 4 Liverpool (RADV ORBIS) (RADV BONAIRE)  api 1.3.358
               vendor 0x1002 device 0x9920  type 1        <- INTEGRATED, because IDS_FLAGS_FUSION
    infoprobe: [0] timestampPeriod 10.000 ns              <- the measured 100 MHz, all the way to the API
    GB_ADDR_CONFIG: 0x10020003                            <- 8 pipes, 256 B interleave, 2 KB rows

**The diff against the reference is clean where it matters.** 164 of Bonaire's 165 fields are present -
only `dev_filename` is missing, and Bonaire's is `/dev/null` - and of the 46 that differ, **exactly four
are behaviour flags, all four intended**:

| flag | bonaire | ours | why |
|---|---|---|---|
| `has_dedicated_vram` | 1 | **0** | `IDS_FLAGS_FUSION`; one unified GDDR5 pool |
| `all_vram_visible` | 0 | **1** | follows from the same |
| `has_timeline_syncobj` | 1 | **0** | deliberate: `timeline_wait` left NULL |
| `kernel_has_modifiers` | 1 | **0** | there is no DRM |

**Every `has_*_bug` flag matches Bonaire.** That is what the reference was for: the errata list is inherited
exactly as the identity decision intended, and no hardware workaround changed silently. The remaining 42
differences are counts, clocks, sizes, names and firmware versions, each with its provenance in
`ac_orbis_drm.c`.

Both dumps are kept: [`notes/radeon_info-bonaire.txt`](notes/radeon_info-bonaire.txt) and
[`notes/radeon_info-orbis.txt`](notes/radeon_info-orbis.txt).

### What was learned doing it

* **`ac_drm_query_pci_bus_info` may be refused, but only if the CALLER says so.** `require_pci_bus_info`
  was copied `true` from the amdgpu winsys; radeonsi passes `false`. One argument, and the refusal went
  from fatal to correct.
* **`ac_drm_query_sw_info` has two entries and both are asked for.** I wrote that the PRT one was not, and
  the next run refuted it immediately - gfx7 has `has_smem_with_null_prt_bug`, and the query is fatal when
  that flag is set. It is a BIT INDEX that a NIR pass clears from every SMEM address; 47 is the canonical
  GCN low/high split and is inert here because this port's VA window is 34 bits. **That is an invariant the
  allocator must keep.**
* **Firmware versions are inert on gfx7.** Every consumer in the tree is gated on GFX8 or later, so
  reporting zero is not merely conservative - it reaches nothing.
* **`mc_arb_ramcfg` is load-bearing and looks decorative**: `noOfBanks = ramcfg & 0x3`, and addrlib decodes
  2 as 16 banks, which is what the fork measured. A different value moves every 2D-tiled texel.
* **`vulkaninfo` is the wrong tool for the dump.** It calls `vkCreateDevice`, which fails without a winsys
  and dies in RADV's own cleanup path - truncating stdout and hiding 57 of the 165 fields. `tools/infoprobe.c`
  enumerates and exits, and only then was the comparison possible.
* **Phase 5 step 1 came early, because it had to.** With no sync provider entries RADV gets zero sync types
  and crashes later; the five required entries are now implemented over an array of counters, which is
  exactly what `research/02` prescribed as the host-first step.

### Still on hardware — this is the boundary

Five fields describe the shader-array topology and are reported as a **guess** that the log names:
`num_shader_engines`, `num_shader_arrays_per_engine`, `num_cu_per_sh`, `cu_bitmap`, `num_tcc_blocks`, plus
`GB_ADDR_CONFIG.NUM_SHADER_ENGINES`. They currently claim 2 SE x 1 SA x 9 CU. **task-94** measures them.
Until then every occupancy number RADV derives is provisional.

---

## Phase 2 — the original brief, kept for the field list

**Goal: `vkEnumeratePhysicalDevices` returns one device and `vkGetPhysicalDeviceProperties` describes
Liverpool.**

Implement in the order RADV trips over them — its own initialisation dictates the sequence, so let the
log drive it rather than guessing.

| function | what it answers |
|---|---|
| `ac_drm_device_initialize` | hand back an opaque `ac_drm_device*` of ours; the `fd` is meaningless here |
| `ac_drm_device_get_fd` / `_get_cookie` | a synthetic fd and a cookie; RADV only compares them |
| `ac_drm_query_info` | `AMDGPU_INFO_DEV_INFO` → the 36 fields of `drm_amdgpu_info_device` that `ac_gpu_info.c` reads |
| `ac_drm_query_hw_ip_info` / `_count` | **`num_queues = 1` for GFX, `0` for COMPUTE** — see `research/01-submission.md`; this is what removes the second ring |
| `ac_drm_query_gpu_info` | the same device struct by another door |
| `ac_drm_query_firmware_version` | the shim's `fw_gfx_me` / `_pfp` / `_mec` are dumped from real gfx7 |
| `ac_drm_query_sw_info`, `_has_vm_always_valid`, `get_marketing_name` | small; the second is `true` here, since everything mapped is resident |
| `ac_drm_query_heap_info` | GARLIC and ONION sizes from `sceKernelGetDirectMemorySize` |
| `ac_drm_read_mm_registers` | start from the drm-shim's `bonaire` `mmr_regs`, then verify against the console |
| `ac_drm_query_sensor_info` | clocks and temperature; refuse or return constants |
| `ac_drm_device_deinitialize` | free ours |

### THE LOOP THAT DRIVES THIS PHASE, AND IT RUNS ON THE LAPTOP

`./build.sh --host-orbis` builds this same driver - our arm included - for the host as an ordinary ICD and
runs it. RADV enumerates, reaches `ac_query_gpu_info`, and **the log names the first query that is not
implemented**. Implement it, rebuild, run, get the next name. No console, no harness.

The first iteration already corrected the plan: `ac_drm_query_pci_bus_info` is called first and, with
`require_pci_bus_info` true, its failure is fatal (`ac_gpu_info.c:1465`). That argument was copied from the
amdgpu winsys; radeonsi passes **false**, which leaves `info->pci.valid` false - the truth here. The
refusal was right, the caller was wrong.

### FIRST REAL IMPLEMENTATION: `ac_drm_query_gpu_info`, AND IT IS ALREADY SPECIFIED

The log stops here next, and this is the field group whose values the fork has **measured on hardware**.
RADV wants raw register words, `gb_tile_mode[32]` and `gb_macro_tile_mode[16]`, while `gnmtiler.h` carries
the same information as *fields* - so the work is bit-packing, and the layout is citable for this exact
generation from `oracles/mesa/.../registers/gfx7.json`:

    GB_TILE_MODE0       ARRAY_MODE [2,5]   PIPE_CONFIG [6,10]   TILE_SPLIT [11,13]
                        MICRO_TILE_MODE_NEW [22,24]   SAMPLE_SPLIT [25,26]
    GB_MACROTILE_MODE0  BANK_WIDTH [0,1]   BANK_HEIGHT [2,3]   MACRO_TILE_ASPECT [4,5]   NUM_BANKS [6,7]

Those are exactly the fields `gnmtiler.h` names. And once addrlib is fed them, its addresses must agree
with the GNM backend's own tiling code for the same surface - two independent implementations of CIK
addressing, one already passing on hardware, cross-checked on the laptop.

### ⚠ THE BLOCKER IN THIS PHASE IS NOT MECHANICAL, AND IT WAS ALREADY KNOWN

`ac_gpu_info.c` reads **39** fields off `drm_amdgpu_info_device` (regenerate:
`grep -oE 'device_info(\.|->)[a-z0-9_]+' src/amd/common/ac_gpu_info.c | sort -u`). Roughly ten are GFX11+
and are legitimately zero on gfx7. Of the rest, some are citable and some are **not**:

| citable, and measured on hardware | source |
|---|---|
| `num_rb_pipes` = 8, `enabled_rb_pipes_mask` = 0xff | `backlog/docs/gnm-tiling.md` H5 — "Liverpool has 8 RBs", MEASURED, and rung 2 of the tiling test passing is what confirms it |
| the tiling tables behind `ac_drm_query_gpu_info`'s `gb_tile_mode[32]` / `gb_macro_tile_mode[16]` | same document; `PIPE_CONFIG P8_32x32_16x16` and the 256 B pipe interleave are both MEASURED |
| `max_engine_clock` ≈ 800 MHz | `Engine/gapi/gnm/gnmtune.h:138` |

**NOT citable:** `num_shader_engines`, `num_shader_arrays_per_engine`, `num_cu_per_sh`, `cu_bitmap`,
`num_tcc_blocks`. The Tempest fork already says so in its own words —
`gnmtune.h:732`: *"the fork cannot cite Liverpool's CU/SIMD topology from any oracle in the stash"*.

So this phase must not silently invent them. Two rules for it:

1. Anything not citable gets a marked value and a **startup log line naming every uncited field**, so the
   `RADV_DEBUG=info` diff has a companion list of "known-unknown" rather than looking uniformly confident.
2. The drm-shim's `bonaire` entry is a **structural** reference, not a value one. Bonaire is gfx7 but it is
   not Liverpool: different CU count, different SE count. Copying its topology would produce a driver that
   works and renders subtly wrongly, which is the exact defect class this project has spent the most
   flashes on.

### AND THE UNCITED FIELDS CAN BE MEASURED RATHER THAN GUESSED

`SQ_WAVE_HW_ID` is in Mesa's own register database, which is in the oracle stash
(`oracles/mesa/mesa/src/amd/registers/gfx7.json`) — so this is a citable encoding for exactly this
generation, not an inference from a newer one:

| field | bits | width | what it bounds |
|---|---|---|---|
| `SE_ID` | [13,14] | 2 | up to 4 shader engines |
| `SH_ID` | [12,12] | 1 | up to 2 shader arrays per SE |
| `CU_ID` | [8,11] | 4 | up to 16 CUs per array |
| `SIMD_ID` | [4,5] | 2 | 4 SIMDs per CU |

**So a compute kernel that reads `HW_REG_HW_ID` and atomically ORs one bit per `(SE, SH, CU)` into a
storage buffer enumerates the topology from the hardware itself.** Launch enough workgroups to cover the
machine and the result is `num_shader_engines`, `num_shader_arrays_per_engine`, `num_cu_per_sh` and
`cu_bitmap[4][4]` — **measured on Liverpool**, which is stronger than any citation. `cu_bitmap` is already
indexed `[se][sh]` with a CU mask, which is precisely the shape the probe produces.

It runs on machinery that already works: the GNM backend dispatches compute and writes storage buffers
today. `s_getreg_b32` has no SPIR-V spelling, so the kernel has to be hand-assembled GCN and dispatched
through the existing bake path — a handful of instructions, and this fork already ships baked GCN blobs.

Tracked as **task-94** in the Tempest backlog. Until it has run, the fields above stay marked uncited and
the startup log names them.

**The 36 fields are the whole of this phase**, and they are listed in
`notes/` — regenerate with `grep -oE 'device_info->[a-z0-9_]+' src/amd/common/ac_gpu_info.c | sort -u`.

**Milestone, on the HOST:** `acoprobe` prints a device name and an `apiVersion`. It already does that
against the drm-shim, so **run it both ways and diff `ac_gpu_info`'s dump** — Mesa prints the whole
`radeon_info` struct with `RADV_DEBUG=info`. Two dumps, one from the shim's bonaire and one from our arm,
and the diff is the list of fields still wrong. That is the single most valuable check in the whole plan:
**a known-good reference for 165 fields.**
**The trap:** this phase is where a wrong field produces a driver that works and renders subtly wrongly —
the class of defect this project has spent the most flashes on. The diff above is the defence; do not skip
it because the device enumerated.

---

## Phase 2b — the winsys is BUILT, not written — **DONE, 2026-08-09**

**This phase did not exist in the plan, and it changes phases 3 to 5 rather than adding to them.**

Phase 3 as written could not have been tested, for the same reason phase 1b had to exist: with
`winsys/amdgpu/` out of the build, **nothing calls the memory functions**. `radv_create_winsys` returned
`VK_ERROR_INCOMPATIBLE_DRIVER` on this platform, so `vkCreateDevice` died before reaching a single
`ac_drm_bo_*` body. Fourteen functions written blind, with no way to run them.

So the winsys was measured instead of replaced. **In 3 890 lines there is exactly ONE call that reaches DRM
directly** - a `DRM_AMDGPU_GEM_MMAP` ioctl - and everything else goes through `ac_drm_*`, which is our arm.
`patches/0007` builds it here, and the price is the 35 functions the linker then names. The alternative was
44 `radeon_winsys` entry points of our own reaching the same calls.

**Milestone, reproduced from a clean clone:** 770 targets, zero compile errors, `libvulkan_radeon.a` at
35.8 MB, and the link probe still produces a 26.3 MB PS4 executable with zero undefined symbols.

**And the loop went one layer deeper.** `./build.sh --host-orbis` now also runs `infoprobe --create-device`:

    MESA: info: orbis-drm: device up, reporting amdgpu interface 3.54    <- the winsys' own device
    MESA: warning: orbis-drm: ac_drm_cs_ctx_create2 is not implemented yet
    radv/amdgpu: radv_amdgpu_cs_ctx_create2 failed. (-38)
    infoprobe: vkCreateDevice -> -1

**RADV's order is not this plan's order**, and that is the whole value of asking it: it wants a CONTEXT
first, because queues are created before anything is allocated. The phases below stay the right way to think
about the work; the sequence is RADV's to dictate.

Two things fell out of it that are worth carrying:

* **`ac_drm_cs_create_syncobj2` did not fail**, because it was wired to the same slot array the sync provider
  uses. Two separate syncobj pools would deadlock in a way that looks like a lost GPU - RADV creates them
  through both doors and waits through one.
* **The sync provider's `clone` and `finalize` were NULL**, with a comment saying clone was "only needed when
  a provider is duplicated". `radv_amdgpu_winsys.c:204` duplicates it through the pointer with no NULL check,
  and `vk_device.c:325` calls finalize the same way. The reasoning had been right; **the caller simply did
  not exist in the build yet.** The provider is now owned by the device and hands out heap copies.

---

## Phase 3 — memory, BO and VA — **DONE ON HARDWARE, 2026-08-09**

    radv: vkCreateDevice -> 0 (SUCCESS)
    radv: vkMapMemory -> 0, ptr 0x200420000        <- inside the arena the kernel gave us
    radv: readback OK (0/1024 dwords wrong)
    radv: vkQueueSubmit -> 0 / vkWaitForFences -> 0

**FIVE FLASHES, AND FOUR OF THEM MEASURED THE SYMPTOM.** Each found something real, and the order is worth
keeping because it is the order the console volunteered it:

1. `address32_hi` **is derived, not chosen.** The kernel placed the arena at 0x200400000, and reporting 0 - a
   "contract" this file had written down while the window was still ours to pick - left RADV's first shader
   arena unallocatable. The value is the arena's own high half.
2. **The page size is 16384.** Not a detail of ours: `radv_amdgpu_bo.c` rounds every mapping up with
   `align64(size, getpagesize())` BEFORE the arm sees it, so a 4 KiB BO legitimately asks to map one 16 KiB
   page. The VA allocator's alignment, the size a VA op accepts, and the reported
   `virtual_address_alignment` / `gart_page_size` all needed the real number - and two VA ranges closer
   together than one page would have been mapped on top of each other.
3. **`radv_amdgpu_winsys_bo_destroy` calls `munmap` directly** (patch 0008). This is the one that cost the
   flashes, and it is nasty for a specific reason: **the hole outlives the BO**, so the allocation that dies
   is the NEXT one handed that address. `0x200420000` looked healthy in every measurement until the write.
4. Its sibling: the `replace` path over-maps `PROT_NONE`. Same premise, different line.

**AND THE METHOD MATTERED MORE THAN THE REASONING.** Two of my own hypotheses died because I was arguing about
memory instead of asking about it. `sceKernelVirtualQuery` turned two contradictory measurements - the arena
self-test writing both ends successfully, a write 128 KiB in faulting "page not present" - into one fact:

    vq arena+128K 0x200420000: start 0x200400000 end 0x210400000 ... committed 1     <- at setup
    vq cpu_map page 0x200420000: NOTHING MAPPED (0x8002000d)                          <- later, same run

The page WAS mapped and stopped being. That is a statement no amount of touching could produce, because a
touch that faults takes the answer with it.

The probes stay in the code, bounded to the first four mappings. The defect class they catch is silent by
nature, and one of them is now a permanent argument against reasoning where a syscall will answer.

## Phase 3 — the host run that preceded it

    infoprobe: vkCreateDevice -> 0 (SUCCESS)
    infoprobe: vkAllocateMemory(type 2) -> 0
    infoprobe: vkMapMemory -> 0, ptr 0x220000        <- inside OUR VA window: one address, both processors
    infoprobe: readback OK (0/1024 dwords wrong)
    infoprobe: unmap + free returned

**The pointer is the milestone, not the return code.** 0x220000 is an address this arm's own VA allocator
handed out, so a CPU write and the address the GPU would be given are provably the same number - which is
what this hardware does and what a driver built for a separate GPU VA has to be shown to tolerate.

**THE LAYER SPLITS PHYSICAL MEMORY FROM ITS MAPPING, AND SO DOES THIS PLATFORM.** That is why `ac_linux_drm`
fits a console at all:

    ac_drm_bo_alloc      reserves PHYSICAL memory and nothing else   -> sceKernelAllocateDirectMemory
    ac_drm_bo_va_op_raw  maps a range of it AT AN ADDRESS RADV CHOSE -> sceKernelMapDirectMemory
    ac_drm_bo_cpu_map    hands back the address that mapping produced

The host arm is a **faithful analogue rather than a fake**: a memfd is a pool of pages addressed by offset and
mappable anywhere, which is what direct memory is. Same ordering, same aliasing (`RADEON_FLAG_VM_PAD_1PAGE`
maps one page at two addresses), same failure modes. Only "does sceKernel agree?" is left for hardware, and
the seam is one screen of code under `__PS4__` - not `MESA_SYSTEM_HAS_KMS_DRM`, which is 0 in both builds.

### Two defects, and the first is the interesting one

**`AMDGPU_VM_PAGE_*` DESCRIBE THE GPU'S PAGE TABLE, NOT THE CPU'S MAPPING.** Collapsing them - which one
shared address space invites - made RADV fault writing PM4 into its own command stream:

    0x200000-0x201000  rw-s   the fence BO
    0x201000-0x215000  r--s   the IB, and radv_create_flush_postamble died writing to it

RADV marks an IB `RADEON_FLAG_READ_ONLY` because the CP only ever *reads* it; on amdgpu the CPU keeps writing
the same pages through the mapping `GEM_MMAP` produced. Here there is one mapping, so the flag made the
command stream read-only for its own writer. The mapping is now always writable and the GPU-side intent is
**not enforced** - a real loss, stated rather than shrugged at: on amdgpu a GPU write to a read-only IB
faults, here it would corrupt silently. One page table, nowhere to record the distinction.

**And `ac_drm_bo_export` may not be refused wholesale.** `radv_amdgpu_bo.c:649` calls it with
`amdgpu_bo_handle_type_kms` under an `assert(!r)` to get the `uint32_t` handle it then passes to `va_op_raw`,
`query_info` and `set_metadata`. A KMS handle is this process's own name for its own buffer; only dma-buf and
flink leave the process. Same shape as `ac_drm_cs_syncobj_import_sync_file`: **a name with "export" in it can
serve an internal need.**

### Still to do in this group

`bo_wait_for_idle` and `create_bo_from_user_mem` are untouched, `AMDGPU_VA_OP_CLEAR`/`REPLACE` refuse (sparse
residency is not wired), and the host pool is a bump allocator that does not reclaim - which shows up as
address-space exhaustion with a loud message rather than as corruption. None of it is on the path to a submit.

---

## Phase 3 — the original brief, kept for the mapping

**Goal: `vkCreateDevice`, `vkAllocateMemory`, `vkCreateBuffer`, `vkCreateImage` all succeed.**

⚠ **THE HOST HAS NO `sceKernelAllocateDirectMemory`**, so this needs a backing-store seam from the first
line rather than a port afterwards: malloc plus a VA counter on the laptop, direct memory on the console,
chosen by `__PS4__` - and NOT by `MESA_SYSTEM_HAS_KMS_DRM`, which is 0 in both builds. That is the same
shape the sync provider already has, and it separates "the arm's shape is right" from "sceKernel behaves as
expected".

Every right-hand side exists in the Tempest fork; this is transcription, and the PS4 is *easier* than a PC
because the CPU and GPU share one address space.

```
ac_drm_bo_alloc            sceKernelAllocateDirectMemory, then GnmAllocator's arena logic
                           domain: AMDGPU_GEM_DOMAIN_VRAM → GARLIC, GTT → ONION
ac_drm_bo_cpu_map/_unmap   sceKernelMapDirectMemory / sceKernelMunmap
ac_drm_bo_free             the arena's free
ac_drm_va_range_alloc/free VA bookkeeping - one address space, so this is a bump/free-list over the
                           mapping we already made
ac_drm_bo_va_op_raw/_raw2  map and unmap; on this platform mostly a no-op with a recorded range
ac_drm_bo_wait_for_idle    the label-page fence wait
ac_drm_bo_query_info       from the arena record
ac_drm_bo_set_metadata     record it; GnmTexture is where tiling lives on our side
ac_drm_create_bo_from_user_mem   wrap an existing mapping
ac_drm_va_range_query            the VA space's own bounds; from high_va_offset/high_va_max
```

**Milestone:** `vkCreateDevice` succeeds and `vkAllocateMemory` returns memory a CPU write and a GPU read
both see. **Testable on the host** with a fake allocator (malloc + a VA counter) before any PS4 call
exists — which separates "the arm's shape is right" from "sceKernel behaves as expected".
**The trap:** GARLIC is write-combined. A CPU read-back of a GARLIC buffer to "verify" a write measures the
write-combine buffer, not memory. Use ONION for any host-visible test surface, as the backend already does.

---

## Phase 4 — SUBMISSION — **WORK REACHED THE GPU, 2026-08-09**

    radv: vkQueueSubmit -> 0
    radv: vkWaitForFences -> 0
    orbis-drm: submitting 4 DCB entries (3 IB chunk(s) + the arm's fence)

**AND IT IS NOT A FALSE PASS.** The label starts at zero and the sequence number is one, so the signed-delta
test refuses at first look; the poll had to wait, and on the PS4 arm the ONLY writer of that label is the
end-of-pipe packet. So the GPU executed it.

**TWO OPEN QUESTIONS CLOSED AT ONCE.** Sony's submit accepts **four DCB entries** - the question that hung
over three flashes, because this fork's own submit path had only ever passed one pair, and the fallback was
going to be chaining the IBs ourselves with `IT_INDIRECT_BUFFER`. And **RADV's whole fence layer works over
this arm**: `vkWaitForFences` went through `ac_drm_cs_query_fence_status` into a poll of the GPU's write.

### The fence is the arm's own end-of-pipe packet

amdgpu's kernel signals a submission's fence for its caller; nothing does that here. So the arm appends a
command buffer of its own as the LAST DCB entry, carrying **two** EOP events - the first drains the engines so
the second's write is trustworthy, which is mandatory on GFX7 and established by two oracles that agree on the
sequence (`radeon cik.c:3540-3570` field by field, `Mesa ac_cmdbuf_cp.c:518-536` neutering the dummy by its
data instead). This follows the radeon shape. It is the mechanism the Tempest fork's GNM backend already runs -
one monotonic ticket per submit, an EOP writing it into an ONION label the CPU polls - transcribed rather than
designed.

Its command buffer and its label live in a **private slice taken off the FRONT of the arena and never
reported**, so RADV's VA window starts after them. Front rather than back deliberately: an off-by-one in the
VA allocator then collides with the fence label immediately and loudly, instead of at the far end of 256 MiB
hours later. A clobbered label is a fence that never signals - a symptom with no visible cause.

The poll compares with a **signed delta**, not `>=`: the label is 32 bits and the sequence number 64, so
comparing truncated values makes the four-billionth submit look complete before the first.

### What is still a lie, and it is now on the critical path

**Out-syncobjs are signalled on the CPU when the packets are queued, not when the GPU reaches them.** Ordering
against the CPU is correct, because everything that waits goes through the fence poll. What is not correct is
**one submit waiting on another's syncobj** - which is what every real renderer does. That is phase 5, and it
is no longer optional.

## Phase 4 step 1 — decode and log — done on hardware, 2026-08-09

The console's submission has the SAME SHAPE as the laptop's - six chunks, three IBs, all `ip_type 0` on
`ring 0` - with the addresses inside the arena:

    SUBMIT #1 ctx 1, bo_list 0, 6 chunk(s) - DECODED, NOT SUBMITTED
      [0] IB va 0x200448000 bytes 192      [3] FENCE bo_handle 1 offset 0
      [1] IB va 0x20041c000 bytes  32      [4] SYNCOBJ_OUT x2
      [2] IB va 0x200404000 bytes  96      [5] BO_HANDLES 6 dw

So **three DCB entries for an empty command buffer** is measured on both machines rather than inferred from
one. And a probe read `0xc0012800` out of an IB page - a PM4 type-3 packet header - which incidentally proves
RADV's command stream physically lands in direct memory and reads back from the CPU.

## Phase 4 step 1 — the host run that preceded it

A trivial `vkQueueSubmit` of an EMPTY command buffer, decoded rather than submitted:

    SUBMIT #1 ctx 1, bo_list 0, 6 chunk(s) - DECODED, NOT SUBMITTED
      [0] IB va 0x23e000 bytes 608  ip_type 0 ip_instance 0 ring 0 flags 0x0
      [1] IB va 0x216000 bytes  32  ip_type 0 ip_instance 0 ring 0 flags 0x0
      [2] IB va 0x201000 bytes  96  ip_type 0 ip_instance 0 ring 0 flags 0x0
      [3] FENCE bo_handle 1 offset 0
      [4] SYNCOBJ_OUT x2
      [5] BO_HANDLES 6 dw - ignored, everything mapped is resident

**THREE IB CHUNKS FOR AN EMPTY COMMAND BUFFER, and that is the number that matters.** RADV's preamble, the
command buffer itself and the flush postamble each arrive as their own IB chunk - so step 2 is not "one IB
chunk becomes one DCB entry", it is **three DCB entries for the simplest submit there is**, and the fork's
own submit path has only ever passed one pair. Either `sceGnmSubmitCommandBuffers` takes them all, or the arm
chains the three itself before submitting. That question is now concrete and answerable on hardware instead
of being a guess about how many IBs to expect.

Everything else the reading predicted held: all three are `ip_type 0` (GFX) on `ring 0`, which is what
reporting one GFX ring and zero compute ones was for; `flags` is 0, so no IB flag needs interpreting yet; the
FENCE chunk points at the context's own fence BO with the offset `ac_drm_cs_chunk_fence_info_to_data` packed;
and `BO_HANDLES` really is ignorable.

⚠ **THE DECODER LIES TWICE AND SAYS SO EACH TIME.** It reports success while submitting nothing, and
`ac_drm_cs_query_fence_status` reports every fence expired - otherwise RADV either stops at the error or hangs
on a wait. Which means **any test that reads back GPU results will see stale memory and call it done.** Fine
while the goal is to observe chunks; wrong one line further, and the read-back test in `tools/infoprobe.c`
belongs after step 2 rather than before it.

---

## Phase 4 — submission (6 functions)

**Goal: a compute dispatch runs on the console and writes a value the CPU reads back.**

`research/01-submission.md` has the reading. The order inside this phase:

1. **`ac_drm_cs_submit_raw2` as a decoder first.** Walk the chunk array and log every chunk: kind,
   and for `IB` the `va_start`, `ib_bytes`, `ip_type`, `ring`. **Submit nothing.** Run it on the host
   against the drm-shim. The log tells you how many IBs arrive, whether they are chained, and in what
   order the chunks come — measured, not assumed.
2. Then turn `IB` chunks into `sceGnmSubmitCommandBuffers`' arrays. One DCB entry per IB chunk.
3. `CHUNK_ID_FENCE` → the end-of-pipe timestamp the backend already emits (`dcb.timestampEop`).
4. `BO_HANDLES` → **ignored, with a comment saying why**: no per-submit residency on this platform.
5. `ac_drm_cs_ctx_create2/_free` → synthesise a context; Sony exposes one submission path.
6. `ac_drm_cs_query_fence_status` → the existing fence poll.

**Milestone:** on the console, a `vkCmdDispatch` of a shader that writes a known pattern into an ONION
buffer, read back and compared. That is `ps4/texcompute`'s verdict pattern, which is already proven on this
hardware — so the *test* is not new, only the driver under it.
**The traps, all three already paid for once by the GNM backend:**
* `VGT_NUM_INSTANCES` and friends — a direct draw after an indirect one inherits state nothing chose.
  RADV emits its own PM4 and presumably handles this; do not assume, check the first hang against
  `task-33`'s notes.
* IB alignment is asserted by RADV (`% ip_alignment == 0`), and `ib_alignment` is a field **we** made up in
  phase 2. Get it wrong and the assert fires far from the cause.
* Sony's submit may cap the array count. Our backend has only ever passed one pair.

---

## Phase 5 — syncobj — **THE ORDERING WORKS, host-verified 2026-08-09**

    infoprobe: submit A (signals a semaphore) -> 0
    infoprobe: submit B (waits on it) -> 0
    infoprobe: ordered pair -> 0

**AND THE MEASUREMENT IS PARTLY AN ABSENCE.** The expected outcome is that NOTHING is emitted for the wait, so
a clean pass plus the absence of `waiting on the CPU for a syncobj no submission owns` from the log is the
result. Both held.

### One in-order ring is what makes this cheap, and it is a consequence rather than a shortcut

A syncobj signalled by an EARLIER submission is **already ordered**: the GPU cannot reach this submission's
packets without having finished that one's. So `CHUNK_ID_SYNCOBJ_IN` naming such a handle needs no packet at
all. That follows from `num_queues = 1`, which this device reports itself - **so that report has to stay true,
and the day a second ring appears this reasoning expires with it.**

An out-syncobj now carries the **sequence number of the submission that signals it**, so it becomes signalled
when the fence label reaches that number - i.e. when the appended end-of-pipe packet executes. Before this it
was CPU-signalled the moment the packets were queued, which told a waiting submit that the GPU had finished
work it had not started. Ordering against the CPU was accidentally correct, because every fence wait polls the
label; ordering between submits was not.

A CPU signal clears `gpu_seq`: the caller is asserting the object is signalled NOW, and leaving a pending GPU
sequence would let a later poll decide it is not.

### The one case that is still a CPU block, and it is a limitation rather than a stub

A wait on a syncobj that nothing has signalled and no submission owns cannot be expressed as a GPU-side wait
yet - the real answer is a `WAIT_REG_MEM` packet on the label - so the arm blocks before submitting. Correct,
and it serialises the calling thread against the GPU, which `WAIT_REG_MEM` would not. Bounded to a second and
loud, because a wait that never ends is indistinguishable from a hung driver. **A timeout proceeds with a
warning rather than refusing**: refusing would turn a lost signal into a permanently dead queue, while
proceeding turns it into a visible ordering bug.

Timeline entries stay NULL, so `KHR_timeline_semaphore` is still honestly unadvertised, and the three
`sync_file` entries stay refused.

## Phase 5 — the original brief

**Goal: `vkQueueSubmit` + `vkWaitForFences` returns, and a semaphore orders two submits.**

`research/02-syncobj.md` has the reading. Write `src/orbis_sync_provider.c`:

* **Five entries**: `create`, `destroy`, `signal`, `reset`, `wait` — slots in the label page, a CPU write
  for host-signal, and `GnmDevice::waitFence`'s poll for `wait`.
* **Nine NULL**, deliberately: leaving `timeline_wait` NULL makes RADV not advertise
  `KHR_timeline_semaphore` or `KHR_present_wait`, and the four fd entries are cross-process sharing.
* `ac_drm_device_get_sync_provider` hands it back.

**Milestone, on the host first:** the same provider over an array of `uint64_t` with `signal`
incrementing. Under the drm-shim every submit is a no-op so every fence signals trivially — which
exercises RADV's use of the interface without any hardware. Then the same file, backing store swapped for
the label page, on the console.
**The traps:**
* **Timeouts are absolute nanoseconds** in DRM's convention; `waitFence` takes milliseconds. This is how a
  1 ms wait becomes a 1 000 000 ms hang.
* `first_signaled` is an out-parameter for wait-any. Easy to leave unwritten, and then wait-any returns
  the wrong index and nothing looks broken until it does.
* `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT` means "block until a fence is even attached", which has no
  analogue here and needs a decision rather than a translation.

---

## Phase 6 — WSI over sceVideoOut

**Goal: a cleared screen, then a triangle.**

Independent of phases 2–5 and can be written in parallel. RADV's WSI goes through
`src/vulkan/wsi/wsi_common`; the PS4 needs a platform there or a private `VK_KHR_surface`
implementation. The Tempest fork's `GnmSwapchain` already owns the `sceVideoOut` side — flip queue,
buffer count, the deferred-flip decision — so this is a second consumer of knowledge that exists.

Also in this phase: **loader-less entry.** Link RADV in and call `vk_icdGetInstanceProcAddr` directly. No
`.json`, no `dlopen`, no Vulkan loader. Tempest's Vulkan backend resolves through `vkGetInstanceProcAddr`,
so a thin shim that answers from the static RADV is all that is needed.

**Milestone:** `vkAcquireNextImageKHR` / `vkQueuePresentKHR` put a solid colour on the television.

---

## Phase 7 — Tempest and OpenGothic

**Goal: the game runs on the Vulkan backend instead of the GNM one.**

This is the phase that pays for the other six, and the reason it is cheap: **Tempest already has a Vulkan
backend**. The work is build plumbing, not graphics — `TEMPEST_BUILD_VULKAN` for PS4, the static RADV in
the link line, and the shim from phase 6.

**Milestone:** OpenGothic boots and renders the world through RADV.
**The comparison that matters:** frame time against the GNM backend's `wall 34.19 ms | ph:record 22.06 |
ph:engine 11.79 | GPU 25.70`. RADV will add CPU overhead in the driver and probably win on the GPU, since
ACO's shaders beat amdllpc's and addrlib's tiling is correct by construction. **Which way it nets out is
not predictable and should not be predicted** — measure it with `gnmprof`'s own methodology, one report,
64 frames, knobs diffed.

---

## What will go wrong, from this project's own ledger

* **A phase that "works" is not a phase that is right.** Phase 2 will enumerate a device long before the
  36 fields are correct. The `RADV_DEBUG=info` diff against the bonaire shim is the only thing standing
  between that and weeks of chasing render artefacts.
* **Test on the host until the host cannot answer.** Phases 1, 2, 4-step-1 and 5-step-1 all run on the
  laptop under the drm-shim. Every one of them done on the console instead costs ten minutes a round.
* **Do not tune a threshold before checking what the thing does.** Three rounds went into a walk/run bug
  in this session before one `grep` showed the premise was wrong.
* **A shim can break more than it fixes and look like the port failing.** 109 build errors became 246
  because of a global `-include` that reached assembly files.
* **Count the surface with the right command.** This list has been wrong twice, both times because the
  grep was narrower than the build: 43 counted type names, 39 grepped `src/amd/vulkan/` and forgot that
  `src/amd/common/` is compiled in too. Phase 0's linker output is what settles it.
* **Refuse loudly rather than stub silently.** Every unimplemented function returns `-ENOSYS` and says so
  once. A stub returning success moves the failure somewhere it cannot be read.

## Sequencing at a glance

```
0  link                    host      the linker confirms the 54
1  54 stubs, 13 refusals   host      RADV links; acoprobe dies in our own log line
2  20 device queries       host+PS4  a device enumerates; RADV_DEBUG=info diffs clean
3  16 memory functions     host+PS4  vkAllocateMemory returns usable memory
4  7 submission            PS4       a dispatch writes a pattern the CPU reads back
5  11 sync (5 real)        host+PS4  vkWaitForFences returns
6  WSI + loader-less entry PS4       a colour on the television          (parallel with 2-5)
7  Tempest plumbing        PS4       OpenGothic renders through RADV
```
