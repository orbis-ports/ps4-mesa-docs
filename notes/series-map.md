# The former patch series, one line each — what to run, what to delete, what is only a log

The series is now `~/src/mesa-ps4`, branch `orbis`, one commit per patch, tagged `pNNNN`. This table
exists because the question "which patch does what" had no cheap answer while the answer lived in 52 diffs.

Classes, and they decide how each commit gets tested:

- **A — structural.** The port does not build or does not reach the television without it. Nothing to test:
  removing it is not a hypothesis, it is a broken tree.
- **B — behaviour, always on.** A workaround or a deliberate difference from stock RADV, with no knob. These
  are the ones worth re-testing one at a time, because each is a standing claim about this hardware and at
  least one (`p0010`) has already been measured obsolete.
- **C — env-gated experiment, inert by default.** Costs nothing when its knob is unset. Cheap to keep, and
  each is a run rather than an edit.
- **D — diagnostic.** Logging, auditing, counting. Every one of these is CPU spent inside the frame, and
  `p0034` proved the class is dangerous: its audit was 487 ms of every second. Unmeasured cost until priced.

`git show p0034` reads any of them; `git checkout p0034 && ~/src/orbis-mesa/build.sh` builds the tree as it
stood at that patch, and ninja rebuilds only the files that differ.

| tag | class | knobs | files | what |
|---|---|---|---|---|
| `p0001` | A | — | futex.c | util/futex: take the Linux futex path only on Linux |
| `p0002` | A | — | detect_os.h | util/detect_os: the PS4 is a FreeBSD kernel with a musl libc |
| `psrc` | A | `ORBIS_API_COUNT` `ORBIS_ARENA_MIB` `ORBIS_ARENA_PRIVATE` | ac_orbis_drm.c, orbis_compat.c, orbis_tile_tables.h +3 | orbis: the driver arm's own sources |
| `p0005` | A | — | meson.build, meson.options, ac_drm_fourcc.h +10 | orbis: platform |
| `p0006` | A | — | meson.build, vk_drm_syncobj.c | vulkan/runtime: decouple vk_drm_syncobj from libdrm |
| `p0007` | A | — | ac_linux_drm.h, meson.build, radv_device.c +2 | orbis: build the amdgpu winsys |
| `p0008` | A | — | radv_amdgpu_bo.c | orbis: winsys must not reach the kernel directly |
| `p0009` | A | — | radv_amdgpu_bo.c | orbis: the pad page is one device page |
| `p0010` | B | — | ac_surface.c | orbis: no tile swizzle |
| `p0011` | C | `ORBIS_TILE_MODE` | ac_surface.c | orbis: force 1d tiling experiment |
| `p0012` | B | `ORBIS_CMASK` `ORBIS_NO_CMASK` | radv_image.c | orbis: no single sample cmask |
| `p0013` | A | `ORBIS_FORCE_CPU_IMAGES` | radv_wsi.c, radv_wsi.h, meson.build +2 | orbis: wsi a real swapchain over sceVideoOut |
| `p0014` | A | `ORBIS_BINARY_SYNCOBJ_TRANSFER` | radv_amdgpu_cs.c | orbis: a zero submission without sync files |
| `p0015` | A | — | vk_drm_syncobj.c | orbis: copy syncobj payloads by transfer |
| `p0016` | A | — | vk_device.c | orbis: breadcrumbs in copy semaphore payloads |
| `p0017` | C | `ORBIS_NO_PREFETCH` | radv_cp_dma.c | orbis: no cp dma prefetch experiment |
| `p0018` | B | — | ac_surface.c | orbis: no tile mode liverpool cannot name |
| `p0019` | C | `ORBIS_3D_LINEAR` `ORBIS_MAX_INSTANCES` | ac_surface.c, radv_cmd_buffer.c | orbis: two knobs to bisect a layered 3d draw |
| `p0020` | A | — | ac_linux_drm.h | orbis: a prototype for the gpu idle wait |
| `p0021` | B | — | wsi_common_headless.c | orbis: offer a unorm swapchain format first |
| `p0022` | D | — | radv_cmd_buffer.c | orbis: log every indirect draw |
| `p0023` | D | — | radv_cmd_buffer.c | orbis: how big was the recording |
| `p0024` | D | — | ac_descriptors.c | orbis: refuse descriptors outside the arena |
| `p0025` | A | — | ac_linux_drm.h | orbis: a prototype for the arena bounds check |
| `p0026` | D | `ORBIS_TRACE_BOS` | radv_cmd_buffer.c | orbis: check the vertex descriptors too |
| `p0027` | D | `ORBIS_CHECK_POINTER` | ac_cmdbuf.h, ac_linux_drm.h | orbis: the assert that is compiled out |
| `p0028` | D | — | radv_cmd_buffer.c | orbis: read the indirect arguments |
| `p0029` | D | — | radv_cmd_buffer.c | orbis: read the draw indirect arguments |
| `p0030` | D | — | radv_cmd_buffer.c | orbis: audit the descriptor sets in memory |
| `p0031` | A | — | radv_cmd_buffer.c | orbis: declare the arena check at file scope |
| `p0032` | B | — | radv_descriptor_pool.c | orbis: zero the descriptor pool |
| `p0033` | D | — | radv_cmd_buffer.c | orbis: check how far a descriptor reaches |
| `p0034` | C | `ORBIS_AUDIT_STEP` | radv_cmd_buffer.c | orbis: audit every flush in a rotating window |
| `p0035` | D | `ORBIS_WATCH` | ac_linux_drm.h, radv_descriptor_pool.c | orbis: watch the descriptor pool |
| `p0036` | D | — | radv_cmd_buffer.c | orbis: does the binding fit its set |
| `p0037` | D | — | ac_linux_drm.h, radv_cmd_buffer.c | orbis: is the set still mapped |
| `p0038` | A | — | radv_cmd_buffer.c | orbis: declare the live check |
| `p0039` | D | — | radv_cmd_buffer.c | orbis: check image extents |
| `p0040` | D | `ORBIS_TRACE_BOS` | radv_cmd_buffer.c, radv_descriptor_pool.c | orbis: per binding cursor and pool addresses |
| `p0041` | D | — | ac_linux_drm.h, radv_cmd_buffer.c | orbis: name the previous owner inline |
| `p0042` | D | — | ac_linux_drm.h, radv_cmd_buffer.c | orbis: is the descriptor target still mapped |
| `p0043` | C | `ORBIS_RASTER_CONFIG` `ORBIS_RASTER_CONFIG_1` | ac_gpu_info.c | orbis: override the raster config |
| `p0044` | D | — | radv_queue.c | orbis: log the scratch ring |
| `p0045` | D | — | radv_cmd_buffer.c | orbis: how far does an image reach |
| `p0046` | B | — | radv_queue.c | orbis: program the tessellation rings we own |
| `p0047` | C | `ORBIS_TESS_RING_SCALE` | radv_queue.c | orbis: headroom past the tessellation ring |
| `p0048` | B | `ORBIS_NANINF` `ORBIS_READ_REGS_AT` `ORBIS_READ_REGS_SET` | ac_cmdbuf.c | orbis: write the register clear state did not |
| `p0049` | D | `ORBIS_CHIP` | amdgpu_devices.c | orbis: teach mesas drm shim this device |
| `p0050` | C | `ORBIS_MAX_FLUSH` `ORBIS_SERIALISE` | radv_cmd_buffer.c | orbis: the heaviest barrier this driver can emit |
| `p0051` | D | `ORBIS_AB_A` `ORBIS_AB_B` `ORBIS_API_ALLOC_DESCSETS` | radv_buffer_view.c, radv_cmd_buffer.c, radv_descriptor_pool.c +5 | orbis: count the api work before timing it |
| `p0052` | C | `ORBIS_AUDIT_STEP` | radv_cmd_buffer.c | orbis: the descriptor audit was half the frame |
| `p0053` | C | `ORBIS_RASTER_AB` | ac_gpu_info.c | orbis: flip the raster config between frames |
| `p0054` | C | `ORBIS_TILE_SWIZZLE` | ac_surface.c | orbis: the tile swizzle becomes a test instrument |

## `orbis-min` — the same driver with class D taken out

Branch `orbis-min` reverts all twenty-four class D commits (`p0022`…`p0045`, `p0051`, plus `p0034`/`p0052`
which are the audit and its gate). It is not a rewrite: twenty-four reverts, every one clean, **1307 lines
removed**, then one commit restoring the diagnostic prototypes in `ac_linux_drm.h` — their definitions live in
`ac_orbis_drm.c`, which is the arm rather than a diagnostic, and `-Wmissing-prototypes -Werror` fails on six
of them otherwise.

It builds with **0 errors**, links, and is 67 KB smaller in the eboot (`linkprobe` 26,346,176 against
26,413,784). What one flash of it buys: **the whole diagnostic class priced at once**, instead of twenty
runs. If the picture and the frame time are unchanged, class D was inert and can be deleted; if either moves,
one of those twenty-four was never as passive as its comment claimed — which is exactly what `p0034` turned
out to be.

`git checkout orbis` goes back to the instrumented driver; the ORBIS_ counters and the WSI present timers
live in `ac_orbis_drm.c` and `wsi_orbis.c`, so `orbis-min` still reports `wait_gpu_idle`.
