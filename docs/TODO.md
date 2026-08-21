# What has to be implemented

The PS4 arm of `ac_linux_drm` — a third backend beside `amdgpu` and `virtio`, which is the shape Mesa
already anticipated with `bool is_virtio` on every function in that file.

**54 functions.** Derived, not estimated: the union of what `src/amd/vulkan/` and `src/amd/common/*.c`
call, minus four type names the pattern also catches, minus the five user-queue entries which are GFX11+
and dead on gfx7. Raw lists in [`notes/`](notes/) with the command that regenerates them.

> ⚠ TWO EARLIER COUNTS WERE WRONG AND BOTH ARE CORRECTED HERE. **43** counted `ac_drm_bo`,
> `ac_drm_device`, `ac_drm_fourcc` and `ac_drm_bo_import_result` — type names, not functions. **39** then
> grepped `src/amd/vulkan/` alone and missed that `src/amd/common/` (`ac_gpu_info.c`, `ac_surface.c`)
> calls twenty more and is compiled into the driver too. The device-query group is the one that grew:
> 10 → 20, and it is phase 2 of the plan.

| group | functions | phase |
|---|---|---|
| device queries | **20** | 2 |
| memory, BO, VA | **16** | 3 |
| submission and contexts | **7** | 4 |
| syncobj | **11** | 5 |

## 13 first, then 35 — and both counts came from the linker

`-Dplatforms=orbis` originally dropped `winsys/amdgpu/*` from the build, and **the winsys is what calls the
rest**. So the first archive's undefined list — [`notes/ac_drm-missing-link.txt`](notes/ac_drm-missing-link.txt),
13 symbols — was what RADV needs with no winsys at all: the device-query group, `device_deinitialize`,
`get_sync_provider` and `get_marketing_name`. Two of those 13 were refusals already
(`query_pci_bus_info`, `query_video_caps_info`) and one is GFX11+ (`query_uq_fw_area_info`), so phase 1 had
**ten** real bodies to write before a device could enumerate. It enumerated, on hardware.

**`patches/0007` then builds the amdgpu winsys for this platform**, because measuring it showed exactly one
call in 3 890 lines reaching DRM directly and everything else going through `ac_drm_*`. The linker's second
list is **35 symbols** — regenerate it rather than trusting this number:

```sh
grep -oE "undefined symbol: ac_drm_[a-z0-9_]+" build.log | sed 's/.*: //' | sort -u
```

13 + 35 = **48 referenced**, of the 54 that exist on gfx7; the other six are declared and never called here.
All 35 are stubbed, so the archive links and `vkCreateDevice` runs — which is what makes the order below a
*measurement* instead of a guess. **RADV asked for `ac_drm_cs_ctx_create2` first, not memory**: queues are
created before anything is allocated. Implement what the log names, in the order it names it.

## Two levels, and only one of them is the work

| level | what | count |
|---|---|---|
| `ac_drm_*` | **where the PS4 arm is written.** Mesa's own abstraction, with the `is_virtio` switch already in place | **54** |
| `amdgpu_*` / `drm*` | what `ac_linux_drm.c`'s amdgpu arm wants from libdrm — i.e. what the PS4 arm must **not need** | 22 |

Implementing at the lower level would mean writing a fake libdrm and translating ioctls. The upper level
is a third arm in a file that already has two.

---

## 1. Memory, buffer objects and VA — 16 functions

**The cheapest group, because every right-hand side already exists in the Tempest fork.** And the PS4 is
*easier* than a PC here: the CPU and the GPU share one address space, so there is no separate GPU VA to
map into.

```
ac_drm_bo_alloc                 sceKernelAllocateDirectMemory + GnmAllocator's arenas
ac_drm_bo_cpu_map / _unmap      sceKernelMapDirectMemory   (GARLIC/ONION already distinguished)
ac_drm_bo_free                  GnmAllocator::free
ac_drm_va_range_alloc / _free   our own VA bookkeeping
ac_drm_bo_va_op_raw / _raw2     map and unmap within the same space
ac_drm_bo_wait_for_idle         GnmDevice::waitFence over the label page
ac_drm_bo_query_info            from the arena record
ac_drm_bo_set_metadata          tiling and format metadata; ours lives in GnmTexture
ac_drm_create_bo_from_user_mem  importing somebody else's pages
ac_drm_bo_export / _import      cross-process BO sharing
```

**Two of these should be REFUSED rather than implemented**, and saying so up front is the point:
`ac_drm_bo_export`/`_import` exist for `VK_KHR_external_memory`, which has no meaning for a single
homebrew process. A refusal that names itself beats a stub that pretends.

## 2. Submission and contexts — 7 functions

**→ [`research/01-submission.md`](research/01-submission.md) — the unknown is nearly closed by reading.**
An IB chunk is an address and a size, `sceGnmSubmitCommandBuffers` takes arrays, chaining works on gfx7 and
is switchable with `RADV_DEBUG_NO_IB_CHAINING`, `BO_HANDLES` can be ignored because everything mapped is
resident here, and the GFX-versus-COMPUTE question is settled by a number we synthesise: report
`ip[AMD_IP_COMPUTE].num_queues = 0` and there is only one ring.

One of these is the whole port:

```
ac_drm_cs_submit_raw2           ← parses struct drm_amdgpu_cs_chunk and turns it into
                                  sceGnmSubmitCommandBuffers
ac_drm_cs_ctx_create2 / _free   Sony exposes ONE submission path; a context is synthesised
ac_drm_cs_query_fence_status    our fences
ac_drm_cs_chunk_fence_info_to_data, ac_drm_cs_ctx_stable_pstate    small
```

`ac_drm_cs_submit_raw2` takes the **kernel's raw ABI structures**, so this layer abstracts the
*transport* and not the *model*: the arm implements amdgpu's submission semantics on top of Sony's
rather than swapping one call for another. The chunks carry IB descriptors, fences, syncobj in/out and
BO handles.

## 3. Device queries — 20 functions, and this is the group that grew

**Mechanical, because Liverpool is known.**

```
ac_drm_device_initialize / _deinitialize / _get_fd / _get_cookie
ac_drm_query_info                the big one: 36 fields of drm_amdgpu_info_device
ac_drm_query_gpu_info            the same struct by another door
ac_drm_query_hw_ip_info / _count num_queues - GFX 1, COMPUTE 0 (research/01)
ac_drm_query_heap_info           GARLIC and ONION sizes
ac_drm_read_mm_registers         start from the bonaire shim's dumped mmr_regs
ac_drm_query_firmware_version    the shim has these too (fw_gfx_me/pfp/mec)
ac_drm_query_sw_info             the driver's own view
ac_drm_query_sensor_info         clocks and temperature; constants or refuse
ac_drm_query_pci_bus_info        no PCI bus here - REFUSE, but the CALLER must pass
                                 require_pci_bus_info = false, or the refusal is fatal (ac_gpu_info.c:1465)
ac_drm_query_video_caps_info     no codecs in this build - REFUSE
ac_drm_query_gpuvm_fault_info    no GPUVM fault reporting - REFUSE
ac_drm_query_has_vm_always_valid one bool; true, since everything mapped is resident
ac_drm_get_marketing_name        a string
ac_drm_vm_reserve_vmid / _unreserve_vmid   no VMIDs here - REFUSE
```

**Seven of the twenty are refusals**, so the real work here is thirteen - and the drm-shim's `bonaire`
entry already carries `hw_ip_gfx`, `hw_ip_compute`, `fw_gfx_me/pfp/mec` and a full `mmr_regs` block dumped
from real gfx7 hardware, which is a starting value for five of them.

`ac_drm_read_mm_registers` has a shortcut worth knowing: the drm-shim's `bonaire` entry carries a
complete `mmr_regs` block dumped from real gfx7 hardware (`src/amd/common/amdgpu_devices.c`), so the
values can be taken from there and cross-checked against the console rather than derived.

## 4. Syncobj — a vtable to fill, not an emulation to design

**→ [`research/02-syncobj.md`](research/02-syncobj.md) — the unknown collapsed.** It is one struct,
`util_sync_provider`, with 14 function pointers; Mesa already has a second implementation and carves out
the place for a third under `#if HAVE_LIBDRM`. **Nine may be left NULL** — leaving `timeline_wait` NULL
makes RADV simply not advertise `KHR_timeline_semaphore` and `KHR_present_wait`, which is a clean
degradation. **Five must exist** - `create`, `destroy`, `signal`, `reset`, `wait` - because
`has_syncobj` is hardcoded true rather than probed, and all five are counter operations over the label
page the GNM backend already uses.

The original framing, kept because the function names are still the surface:

```
ac_drm_cs_create_syncobj2          ac_drm_cs_destroy_syncobj
ac_drm_cs_syncobj_timeline_wait    ← timeline semaphores
ac_drm_cs_syncobj_query2           ac_drm_cs_syncobj_transfer
ac_drm_cs_syncobj_export_sync_file / _file2 / _import_sync_file
ac_drm_device_get_sync_provider
```

The console has nothing of the kind. The Tempest fork has fence machinery over GPU label writes
(`GnmDevice::waitFence`, the label page) which covers the *wait* semantics, but timeline semaphores with
host waits, transfers between objects, and **export to a file descriptor** are a different shape.

**The three `sync_file` ones should be refused**, and the consequence has to be stated before anyone
writes a line: that removes the Vulkan extensions built on fd-based sharing. Better known now than
discovered inside `vkCreateDevice`.

---

## Order

1. **Device queries first** — DONE, and confirmed on hardware. Nothing runs until `ac_drm_device_initialize`
   answers.
2. **Whatever `vkCreateDevice` names next.** This replaces the ordering below rather than refining it: the
   groups are still the right way to *think* about the work, but the sequence is RADV's to dictate and it
   already disagreed once — contexts before memory. `./build.sh --host-orbis` prints the name.
3. **Memory and VA** — 14 to implement, 2 to refuse. All of it exists in the Tempest fork already. ⚠ The
   laptop has no `sceKernelAllocateDirectMemory`, so this needs a **backing-store seam from the first line**
   — malloc plus a VA counter on the host, direct memory on the console — the same shape the sync provider
   has. That separates "the arm's shape is right" from "sceKernel behaves as expected", which is the
   distinction this project has spent the most flashes on.
4. **Submission** — `ac_drm_cs_submit_raw2` alone, everything else around it is bookkeeping.
5. **Syncobj** — last, because a driver that submits and never signals can still be debugged, while one
   that cannot submit cannot.

Then WSI over `sceVideoOut`, and a loader-less entry: link RADV in and call
`vk_icdGetInstanceProcAddr` directly — no `.json`, no `dlopen`.

## Before any of it

The **link** has to complete, so the linker can confirm this list rather than `nm` inferring it. It is
blocked on cross-dependency plumbing, not on anything about the console — see README.md's *"where it
stops"*.
