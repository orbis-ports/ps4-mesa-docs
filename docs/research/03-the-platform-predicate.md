# Research 3 — the port has been fighting the wrong condition

**Status: this replaces Phase 0's approach and deletes two shims. Mesa already has a predicate for
"there is no DRM kernel interface"; this port was busy faking one instead of setting it.**

## What went wrong, concretely

Round 8, against Mesa `f475fb0c`: **97 objects failed with 244 errors** — worse than the round that
compiled everything, because that round configured a narrower build. The errors were not 244 problems:

| errors | root cause |
|---|---|
| **176** | `radv_physical_device.h:154` — `drmPciBusInfo bus_info;`. 88 × the unknown type, 88 × a cascaded `offsetof` static assertion, because an incomplete struct makes `offsetof` non-constant |
| 19 | `wsi_common_drm.c` |
| 14 | `util/os_drm.h` — `#include <xf86drm.h>` |
| 13 | `u_sync_provider.c` |
| 9 | `vk_instance.c` — `drmDevicePtr devices[256]` |
| 2 | `ac_surface.c`, and `amdgpu.h` not found in 8 places |

Every one of them is the same thing: **code that only exists when a DRM kernel driver does.** And each
one is already guarded upstream — on `#ifndef _WIN32`, on `if dep_libdrm.found()`, on
`if system_has_kms_drm`. The port was compiling the DRM path and then trying to satisfy it with shim
headers, one declaration at a time, forever.

> I read `radv_physical_device.h:156` out of the error message and concluded the field was unguarded. It
> is guarded, two lines up, and I had the answer on screen. The lesson is the ordinary one: read the
> guard, not the line the compiler points at.

## The predicate already exists, in C, as a 0/1 macro

```meson
# meson.build:161
system_has_kms_drm = ['openbsd', 'netbsd', 'freebsd', 'gnu/kfreebsd', 'dragonfly',
                      'linux', 'sunos', 'android', 'managarm'].contains(host_machine.system())
# meson.build:344
pre_args += ['-DMESA_SYSTEM_HAS_KMS_DRM=@0@'.format(system_has_kms_drm.to_int())]
```

`'windows'` is not in that list, so **`MESA_SYSTEM_HAS_KMS_DRM` is already 0 exactly where `_WIN32` is
defined.** Which means replacing `#ifdef _WIN32` with `#if !MESA_SYSTEM_HAS_KMS_DRM` in the amd tree
**changes nothing on any platform Mesa builds RADV for today** — and gives this port a switch it can set.

**And Mesa already sets it false on an OS that does have DRM** (`meson.build:337`), for Turnip without the
DRM KMD:

> *"If DRM support isn't needed, we can get rid of it since linking to libdrm can be a potential
> compatibility hazard."*

So a Vulkan driver built with `system_has_kms_drm = false` on a DRM-capable kernel is not a new idea here.
The PS4 is the same case with a stronger reason: it is not that libdrm is *a hazard*, it is that there is
no `/dev/dri` at all.

## The change

`-Dplatforms=orbis` — a new entry in the `platforms` option, which is where Mesa keeps window systems, and
`sceVideoOut` is one. It sets `with_platform_orbis`, and:

```
meson.build         system_has_kms_drm = false when the option is set (before the C define is emitted)
                    libdrm_amdgpu is not required   ('and not with_platform_orbis')
amd/common          ac_linux_drm.c out of the build
amd/vulkan          winsys/amdgpu/* out of the build
15 × #ifdef _WIN32  ->  #if !MESA_SYSTEM_HAS_KMS_DRM     (and the #ifndef form to #if)
```

**Five of the twenty `_WIN32` sites in the amd tree are NOT about DRM and were left alone**:
`RADV_SUPPORT_CALIBRATED_TIMESTAMPS`, the RMV trace entrypoints, and three `open_memstream` paths in
`ac_debug.c`. Converting them blindly would have disabled working features to fix a header problem.

**One was deliberately left on the DRM side**: `radv_physical_device.c:2930`, which calls
`ac_drm_device_deinitialize`. Windows skips it because it never created one; we do create one, so taking
the Windows arm there would leak the device on every teardown.

### What this deletes

* `stub-libdrm/` — the empty archive and its `.pc` files. **The idea was wrong**: it made meson believe
  libdrm exists, which turned on `wsi_common_drm.c`, `u_sync_provider.c`, `os_drm.h` and
  `vk_instance.c`'s DRM enumeration — five files' worth of shimming caused by the workaround itself.
* `shims/xf86drm.h`, `shims/libdrm/amdgpu.h` — not needed once the no-DRM arm is taken.
* README's *"the link error list is the specification of the work"*. It is not, and it never needed to be:
  the header states the surface directly, and that statement has now been cross-checked (below).

`shims/sys/ioccom.h` stays for now — `drm-uapi/drm.h` reaches other parts of the Vulkan runtime.

## The surface, confirmed from a second direction

`ac_linux_drm.h` declares **59** functions (`grep -c '^MESAPROC'`). Minus the five GFX11+ user-queue
entries this generation cannot have — `create_userqueue`, `free_userqueue`, `query_uq_fw_area_info`,
`userq_signal`, `userq_wait` — that is **54**, and the set difference against the caller-derived list in
`notes/` is *exactly* those five and nothing else.

Two independent derivations, one number. After being wrong at 43 and at 39, that is worth more than a
third count would be.

## The return conventions, which the plan had wrong

The declarations carry their own error convention in a macro suffix, so that Windows can stub them:

```c
#define TAIL    { return -1; }     /* 54 of them */
#define TAILV   { }                /*  3 */
#define TAILPTR { return NULL; }   /*  2 */
```

**So "every unimplemented body returns `-ENOSYS`" is impossible for five of them:**

| function | returns | a stub that does nothing is… |
|---|---|---|
| `ac_drm_device_get_sync_provider` | `struct util_sync_provider *` | NULL, and `has_timeline_syncobj` is read off `provider->timeline_wait` — **NULL provider dereferenced** |
| `ac_drm_get_marketing_name` | `const char *` | NULL, harmless |
| `ac_drm_device_deinitialize` | `void` | a leak, silent |
| `ac_drm_cs_chunk_fence_info_to_data` | `void` | **fills an out-parameter** — a no-op leaves a garbage submit chunk and no error anywhere |
| `ac_drm_query_has_vm_always_valid` | `void` | **writes into `radeon_info`** — a no-op leaves whatever was there |

The last two are precisely the failure mode Phase 1 named as *"the trap: a stub that returns 0"*, except
they cannot even return. They must be implemented in Phase 1 rather than stubbed, and the first must at
minimum return a provider whose 14 pointers are all NULL rather than NULL itself.

## The port's real shape: two functions

The whole insertion surface on the RADV side is two `#ifdef _WIN32` sites, both of which refuse on Windows:

```
radv_physical_device.c:2529   radv_physical_device_try_create(instance, drmDevicePtr, ...)
                              on Windows: assert(drm_device == NULL); return VK_ERROR_INCOMPATIBLE_DRIVER
radv_device.c:1325            radv_create_winsys(device)
                              on Windows: return VK_ERROR_INCOMPATIBLE_DRIVER
```

That is a better description than "3 890 lines of amdgpu winsys get re-aimed". Enumeration enters at the
first, the winsys is constructed at the second, and everything below them is `ac_drm_*`. **Phase 1 leaves
both refusing** — the driver links and reports zero devices, which is the honest state — and Phase 2/6
give each its own arm.

## What could still surprise

* `MESA_SYSTEM_HAS_KMS_DRM` undefined in some translation unit evaluates `#if !MESA_...` to *true*, i.e.
  silently takes the no-DRM arm. `pre_args` is project-wide so it should not happen, but a subproject or a
  generated file would not say so out loud.
* The substitution **does** change behaviour on macOS and Haiku, where `_WIN32` is undefined and
  `MESA_SYSTEM_HAS_KMS_DRM` is 0. RADV is not built there, so the blast radius is nil — but the patch is
  not literally a no-op for every platform, and should not be described as one upstream.
* `-Dplatforms=orbis` currently selects no WSI at all. RADV builds without a swapchain, which is fine
  until Phase 6, and then `orbis` becomes a real WSI entry rather than a marker.
