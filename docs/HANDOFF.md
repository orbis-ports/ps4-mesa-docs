# Handoff — RADV on the PS4, 2026-08-16 (afternoon)

## Where this is

**OpenGothic runs at 45-48 fps at FULL 1920x1080.** It was 7.0 this morning and 9.2 by lunchtime.
The native GNM backend this project wrote before RADV ran the same save at 30.

No artefacts, no faults. `Gothic.ini` has `vidResIndex=0` - full internal resolution.

⚠ **The env file now carries six lines, and three of them switch off diagnostics the TITLE compiles
in.** `main.cpp` does `setenv(..., 1)` - overwrite - for `ORBIS_DRM_TRACE=1`, `RADV_DEBUG=info,nohiz`
and `ORBIS_DUMP_SUBMIT=7`, and `/data/tempest-env.txt` is read afterwards, so it is the last word:

    MESA_LOG_FILE=/data/opengothic-mesa.log      ORBIS_NO_TESS=1
    MESA_LOG_LEVEL=info                          RADV_DEBUG=info
    ORBIS_DRM_TRACE=0                            ORBIS_DUMP_SUBMIT=18446744073709551615

`ORBIS_DUMP_SUBMIT` cannot be switched off with 0 - the driver reads it with `strtoull` and 0 means
"dump submission zero". UINT64_MAX is what an unset variable produces.

**Full resolution costs about 10%**, which is the shape of the whole result: 0.75 scale gives 51.7
fps and 1.0 gives 45-48, for 1.78x the pixels. The frame is CPU-bound almost everywhere - the
present waits 1-8% of the window - and the GPU only takes the floor back in heavy views, where two
windows out of ten fell to 29-31 fps with the wait at 34-38%. Those views are where the remaining
GPU work is, if anyone wants it: 20.15 screenfuls of 1080p a frame from 18 passes, plus 24
dispatches of 100252 workgroups.

⚠ Antialiasing still does not run. `aaEnabled` is `(aaPreset>0 && vidResIndex==0)` so full
resolution was expected to switch CMAA2 on for the first time, and the pass counts say it did not -
18 passes and 24 dispatches, unchanged from 0.75. `aaPreset` is 0. The run is a clean one-variable
change after all, and CMAA2 remains a compute pass this driver has never executed.

The frame, measured by the title itself through the driver's accounting:

| | | |
|---|---|---|
| `app:Renderer::draw` | 7.8 ms | 40% — Tempest building the frame; 54 us of it is RADV recording |
| `app:world tick` | 5.7 ms | 29% — NPCs, triggers, scripts, physics |
| `app:animation` | 5.1 ms | 26% |
| `app:submit+present` | 0.43 ms | 2% |
| `app:camera` | 0.16 ms | |
| sum | 19.3 ms | against a frame of 19.5 |

**The driver is no longer a meaningful part of the frame.** 91 draws record in 54 us, the whole
descriptor machinery costs 148 us, submit 0.3 ms, and the present is now 112 us. What is left is the
title's own work, and the largest single item is Tempest's, not RADV's.

## The three things that did it, in the order they were found

**1. The console had six cores and the title was using one.**
`std::thread::hardware_concurrency()` returns **1** here. The standard permits that - it is a hint
that may give up - so every `parallelFor` and `parallelTasks` in OpenGothic had run serially for the
life of the port, on a machine with six cores. `scePthreadGetaffinity` reports the real answer
(mask `0x3f`); the fix is in the title's `workers.cpp`, and `OG_WORKERS=n` overrides it.

Animation fell 27.2 ms → 9.5 ms. **And the frame got worse**, 61.6 → 77.2, because the GPU lost
about 15 ms at the same instant. That failure is what found the next one.

**2. Everything the GPU touched was on the cache-coherent bus.**
The arena has always been `WB_ONION`, so every GPU access to every surface crossed the bus the CPUs
are on and snooped their caches. GARLIC exists on this console so that GPU traffic does not; a
native GNM backend would have used it without thinking. It cost nothing visible while one core ran.

VRAM-domain BOs now get real physical backing from the GARLIC pool, mapped `MAP_FIXED` over the
address RADV already chose - the arena became an address-space reservation whose pages can be
replaced. GTT stays on ONION deliberately: those are what the CPU reads and writes, and CPU reads
from GARLIC are an order of magnitude slower. `ORBIS_VRAM_GARLIC=0` turns it off.

| | ONION only | VRAM on GARLIC |
|---|---|---|
| `app:animation` | 9.5 ms | 5.1 ms |
| `app:world tick` | 9.1 ms | 5.4 ms |
| `app:Renderer::draw` | 10.1 ms | 8.2 ms |
| `app:submit+present` | 48.1 ms | 2.8 ms |
| frame | 77.2 ms | 21.9 ms |
| GPU wait | 56% | 0% |

⚠ **The size of this is the lesson.** The GPU's wait vanishing was predicted. That the game's own CPU
code got 40% faster was not - world tick and animation both, code the driver never touches - and so
did the scan-out memcpy, 2130 → 3572 MB/s. Coherent GPU traffic was not merely competing for
bandwidth; it was invalidating the CPUs' caches continuously, under **every measurement this port
has ever taken**. Numbers from before this change describe a machine that was fighting itself.

**3. The frame was copied to the screen.**
8100 KiB a frame, 2.3 ms of 21.9. The swapchain's own images are now registered with video-out, so
the present is a flip. `QueuePresentKHR` went 2503 us → 112 us.

## What zero copy cost, and it took three attempts

The maintainer reported tearing and black bands **in the main menu only**, with the world clean.
That pattern is the answer: a menu frame is nearly free so the title laps the display, while 15 ms
of GPU a frame kept the world behind it.

While the WSI owned the buffers, the copy was a **snapshot** - a flipped image was free the instant
it returned. With the images registered there is no snapshot: a flip consumed from the queue is
still on screen until the next replaces it. Three attempts:

* tightening the flip throttle to `count-2` — **wrong**, would have locked the frame to vblank and
  spent the 54 fps to fix a menu. Reverted before it shipped
* holding the last flipped image — **right in kind, wrong in size**. Five images, four flips allowed
  outstanding, one held
* holding every image the display still owns, `numFlipPending + 1` from `sceVideoOutGetFlipStatus`

⚠ **And then the maintainer asked the question that mattered**: does guessing the queue depth make
this a port for one game? It did. `count-3` fitted five images because OpenGothic asks for five.
What replaced it is the invariant - `P + 2 + R <= count`, where R is what the WSI declares it holds
- and `acquire_next_image` now says so loudly if that is ever wrong for another title instead of
stalling with an empty log.

## What was fixed today, and what established each

| defect | how it was established |
|---|---|
| `SET_PREDICATION` with `PREDICATION_OP_BOOL64`, undefined on GFX6/7 | per-packet progress markers narrowed 157000 dwords to one packet |
| the tail pad emitted as `0xFFFF1000` whenever the stream came out 7 mod 8 | the arm's own validator had said so for the life of the port |
| the SH-pointer dump dereferenced the arena instead of the live mapping | a host probe took the process down before any flash did |
| tess offchip ring sized 72 buffers where the hardware uses 256 | poison-fill watermark measured 3.46x; `sceGnmGetOffChipTessellationBufferSize` says `0x800000` |
| tess factor ring sized 55296 where 262144 is needed | correcting offchip moved the factor ring and the write faults moved with it |
| **the tessellation stage itself** | disabling it removed the GPU fault, the random artefacts and a third of the frame time |
| the CPU and GPU serialised at the present | one-frame deferral plus narrowing the wait to this frame's own submission: 7.0 -> 9.2 fps |
| **`hardware_concurrency()` returns 1**, so every parallel section in the title was serial | printed it and read it back; `scePthreadGetaffinity` says `0x3f` |
| **every GPU surface on the cache-coherent bus** | VRAM-domain BOs moved to GARLIC: 77.2 ms -> 21.9 ms a frame, and the game's own CPU code 40% faster |
| the frame copied to the screen, 8100 KiB and 2.3 ms | the swapchain's images registered with video-out; present 2503 us -> 112 us |
| the display's buffers handed back while still on screen | hold every image `sceVideoOutGetFlipStatus` says it owns |

## The tessellation finding

The faulting reads landed on **Sony's tessellation factor ring at `0xff0000000`**, confirmed by calling
`sceGnmGetTheTessellationFactorRingBufferBaseAddress()` in our own process rather than quoting notes.

**Why this driver hit it and nothing else ever has**, three sources found independently:

* the old GNM backend never implemented tessellation — "milestone B advertises neither tessellation nor
  geometry shaders, so a pipeline is exactly HW VS + HW PS"
* a reimplementation of `libSceGnmDriver` refuses LS/HS by name for want of ground truth: "no title in
  the corpus programs `VGT_SHADER_STAGES_EN.HS_EN`"
* **3057 command streams captured from four retail PS4 titles** carry that register only as `0x0` or
  `0xb0`. `HS_EN` is never set. Not once.

OpenGothic tessellates. This driver is the first thing to drive that stage on this silicon.

⚠ **Switched off is not repaired.** `ORBIS_NO_TESS` defaults to OFF in the tree — the feature stays
advertised — and the console env is what disables it. Task #21 carries three routes back.

## The top-left patch: SOLVED, and the term is now found too

`ORBIS_3D_LINEAR=1` removes it. 3776 frames, zero faults. The artefact was OpenGothic's fog LUT — the
only layered pass in the title, 32 instances one per layer of a 3D image — with its slices landing in
the wrong place and then applied to the frame as fog. That is why it appeared only where geometry had
been drawn.

**The term is the 3D tile mode selector, and it was found on the laptop** (`861d33dc4f5`, and
`tools/thick3d.sh` is the measurement). `gfx6_select_3d_tile_idx` chooses among eight modes, each with a
`supported` flag whose comment says it "comes from the tile mode arrays in the kernel". It does not — it
is a compile-time constant. The kernel assigns `tile[0..14]`, `tile[16]`, `tile[17]`, `tile[27..30]` and
nothing else, in **all four** pipe-count branches, and six of the eight candidates name indices it never
assigns: 15, 19, 20, 21, 25, 26.

The chain, every link read out of the source rather than reasoned about:

* `CiLib::HwlComputeMacroModeIndex` reads `m_tileTable[idx].mode`, finds `LINEAR_GENERAL`, takes its
  `!IsMacroTiled` arm and returns `TileIndexNoMacroIndex` — **which is -3** — with `ADDR_OK`
* `ac_surface` then evaluates `cik_macrotile_mode_array[-3]`. `si_tile_mode_array[32]` is the field
  immediately in front of it, so that reads `si_tile_mode_array[29]` — a real `GB_TILE_MODE` word that
  decodes into perfectly plausible bank fields. **Wrong data that looks right, never a crash**
* with those alignments the loop accepts `3D_TILED_XTHICK`; addrlib asserts on it, and with asserts
  compiled out `HwlPostCheckTileIndex` matches nothing and the surface carries `tileIndex -1` into the
  image descriptor — **the "invalid tile mode index" the earlier effort saw on this very surface**

The fix gates on the table, which is what `supported` claims to do, and bounds the returned macro mode
index. Not gated on the platform: it asks the device about its own registers. On Liverpool it leaves
`2D_TILED_THIN1` and `1D_TILED_THIN1`, both populated.

**Confirmed on the console.** `ORBIS_3D_LINEAR` out of the env, 2496 frames, 6818 submissions, zero
faults, no patch. 3D images are tiled again and the workaround is off.

⚠ **Two things went wrong on the way and both are worth keeping.**

**The first attempt killed the title before the menu.** The function has TWO loops: the first computes
alignments, the second selects on them and gates only on `supported`. Skipping with a bare `continue`
left `align_width/height/depth` at zero, and `util_align_npot` divides by its alignment with the assert
compiled out. SIGFPE at the first 3D image. **Clearing the flag is both the fix and what the field
means.**

**And `thick3d.cpp` could not have caught it**, because it replayed the selector in ONE loop. It
reported the gated selector picking `2D_TILED_THIN1` — true, and blind to the driver crashing before
reaching that choice. An instrument shaped differently from the thing it models cannot see that class of
defect. It now has two loops.

⚠ **The criterion given for reading the run did not exist.** The prediction was "the fog LUT's
allocation should be 8388608 instead of 3686400" — the driver logs no image sizes, and 27501 lines had
no answer. The run was read by elimination instead, which is sound but is not a measurement.
`2d403d88f3f` adds one line naming the chosen 3D tile mode so the next one answers directly.

⚠ **The old tree was never an oracle here.** Its fog LUT draw never executed — "the fence still stops
one short", `ac_surface.c:1712` in mesa-old — so its lack of this artefact is the absence of the pass,
not a correct layout. The route that said "compare against the old tree" was worth nothing on this one,
and is not what found it.

The eliminations that preceded it are kept below, because each is a place the next defect will not be.

## The artefacts

**1. The rectangular patch — SOLVED above.** Kept for its eliminations. Visible ONLY where geometry was
drawn, reproducing the shape of the mesh under it, transparent where there is none. Colours follow the
scene as the camera moves — transparent, green, purple, white. **The old tree never had it; this one
has since its first rendered world.**

Eliminated, each by a run:

* not the presentation copy — the scan-out matches the image byte for byte over its rows, every frame
* not an aliased allocation — no live mapping overlaps the swapchain images, and none of the sixteen
  re-let ranges comes near them (nearest ends `0x21b274000`, images start `0x21b400000`)
* not uninitialised memory — poisoning the presented image magenta changed nothing, so something writes
  there every frame
* not a scissor — the frame's distinct rectangles are the full screen and a mip chain of powers of two,
  nothing wide-and-short
* not CMASK — the gate the old tree had was restored and the patch is unchanged

⚠ **It is a DRAW, not corrupted memory.** Corrupted memory does not know where the geometry is.

**2. A soft wedge in the sky — SOLVED, and it was self-inflicted.**

**The cause was an old diagnostic patch inside the title, not the driver.** `renderer.cpp` in the build
checkout carried an uncommitted task-78 patch whose `fogDiag` machinery recorded on one frame and read
back on the NEXT through `Device::readBytes`, which **waits the device idle**. That is a periodic,
seconds-scale disturbance wired into every frame — exactly the cadence the artefact had.

⚠ **And it was found by accident, in a way that nearly went into the record as a false result.** The
patch was destroyed by a `git checkout` of `renderer.cpp` — a mistake, made while reverting a
diagnostic edit of my own on a file I had not checked for pre-existing local changes. In the same step
`RADV_DEBUG=fullsync` was tried, the artefact was gone, and the cure was attributed to the flag.
`syncshaders` "confirmed" it. Both were confounded: the artefact was already gone for the other reason.

**What caught it** was a counter added to the normal cache-flush path, which reported
`24577 total, PS_PARTIAL 20804` over 1920 frames. A barrier emitted that abundantly cannot be the
missing one, and that number forced a look at the build timeline:

| package | `renderer.cpp` | `RADV_DEBUG` | artefact |
|---|---|---|---|
| 22:41 | task-78 patch present | none | **present** |
| 22:56 | pristine upstream | fullsync | absent |
| 22:56 | pristine upstream | syncshaders | absent |
| 23:07 | pristine upstream | none | absent |

The only variable separating the two states is `renderer.cpp`, not the flag. Confirmed by a long play
session on the 23:07 package with no debug flag at all.

**The lesson is the method, not the bug.** A flag that appears to cure something must be tested against
the same build without it before the cure is believed — and a diagnostic wired into every frame is a
change to the thing being measured.

**Earlier eliminations on this artefact, kept because each is still a place a defect is not:**
the fog LUT (painted at 40x, flat), the cloud layer (cut, streaks returned), the direction mapping in
`textureSkyLUT` (artefacts were in the raw 128x64 image), the draw's arithmetic and inputs (it wrote a
constant and they survived), rasterisation coverage (clear to magenta showed **no magenta** — every
texel is written), live buffer overlap (the arm's own detector silent and not blinded), and the
surface geometry (`tools/thick3d.sh`: pitch 128, height 64, no padding, identical in all three tilings).

## The diff is the tool for artefact 1, and it already paid

The maintainer had to say twice that the old tree lacks this artefact before I stopped building
instruments and diffed. That was the right instruction and it immediately produced CMASK.

Comparing every `ORBIS_` knob between the trees, **seven exist only in the old one**:

    ORBIS_3D_LINEAR  ORBIS_MAX_INSTANCES  ORBIS_MAX_FLUSH  ORBIS_NO_PREFETCH
    ORBIS_SERIALISE  ORBIS_BINARY_SYNCOBJ_TRANSFER  ORBIS_NO_CMASK (an orphan - the gate is gone)

Three of those were built for **one pass**, and the old tree names it:

> OpenGothic's fog LUT pass is `DRAW_INDEX_AUTO 3 indices, 32 instances` with
> `PA_CL_VS_OUT_CNTL.USE_VTX_RENDER_TARGET_INDX` set: one instance per LAYER, rendered by a layered
> shader. **Nothing else this port has run is layered.**

So the only layered pass in this title draws into a 3D image, and 3D images are the only place slice
addressing exists. `ORBIS_3D_LINEAR=1` is ported and is the run in flight. If the patch survives it,
the next candidates are `MAX_INSTANCES`, `NO_PREFETCH`, `SERIALISE`, `MAX_FLUSH` — and the diff has more
than knobs in it: `radv_queue.c` 315 changed lines, `radv_physical_device.c` 383, `radv_cmd_buffer.c`
191, `wsi_common.c` 62.

## CLEAR_STATE does not cover PA_CL_NANINF_CNTL here

RADV skips nine context registers on the strength of `PKT3_CLEAR_STATE`. All nine were read back after
the preamble, with eleven registers RADV writes itself as controls. **Every control matched to the bit,
and eight of the nine held exactly what the `!has_clear_state` branch would have written.** This one
held `0x8b47008d` where Mesa writes 0 — and `0x8b470000` of that is **twelve bits outside every defined
field** (they occupy `0x00107fff`). A driver writing a considered value does not set reserved bits.

Among the defined bits that were set: `VTE_XY_INF_DISCARD` and `VTE_W_INF_DISCARD`, which make the
vertex transform engine discard primitives with infinite XY or W, where ACO compiles against 0.

`0a2c03bd4d8` writes it. Mesa already does so unconditionally on gfx10 and gfx12; only the gfx6/7 path
deferred. **It did not change the wedge** — but a value nobody wrote, with reserved bits set, is wrong on
its own terms. `ORBIS_NANINF=0x8b47008d` puts the measured value back as a control.

## How the frame is measured now, and it is the durable part of today

Three instruments, and between them the frame closes to within a third of a percent. That closure is
what makes any of the numbers above believable, so it is worth keeping:

* **`ac_orbis_api_account(slot, ns)`**, in the driver, times seven Vulkan entry points at their API
  boundary - submit, present, acquire, fence wait and status, descriptor update and allocate - and
  prints microseconds and call counts per frame every five seconds. `src/util/orbis_api_probe.h`,
  one line per function via the cleanup attribute rather than an edit at every return.
* **The title fills four more slots itself** (`mainwindow.cpp`), because nothing inside a driver can
  tell Gothic's simulation from Tempest building the frame. It links the driver, so it calls the same
  function and lands in the same line, on the same clock, over the same window. The slot numbers are
  shared between the two files and both say so.
* **`ac_orbis_note(const char *)`** carries a line from the title into the driver's log, which is the
  only stream that reaches the laptop. That is how `hardware_concurrency() = 1` was established.

Plus the older ones that still run: render passes and screenfuls, dispatches and workgroups, the
submit path's own time, the scan-out copy's rate, and the recording window.

⚠ **Three counters were wrong before they were right, and each looked like a finding.**
`orbis_submit_cpu_ns` incremented only on the empty-submission path, which no frame takes, and
reported "0 ms inside this driver's submit path" over 22 windows. Its sibling was gated behind
`ORBIS_COUNT_API`, which was off. The recording window printed `0 ms` because it divided to
milliseconds. **A zero is not a measurement until you know the counter fires.**

## History: the frame as it was measured this morning

⚠ **These numbers describe the machine BEFORE the GARLIC split, when coherent GPU traffic was
invalidating the CPUs' caches continuously. Do not compare against them.** Kept for the lessons in
them, which survived.

At 1080p in steady gameplay, 143 ms a frame (7 fps):

| | | |
|---|---|---|
| GPU | 103 ms | 72% — the present's wait, 3620 ms of every 5000 ms window over 35 presents |
| CPU | 49 ms | 34% — `getrusage`, and **constant across resolutions** |
| sum | 152 ms | against 143 observed |

Confirmed at half resolution: GPU ~26 ms + CPU 43 ms = 69 against 71 observed. **CPU per frame does not
change with pixels while GPU does**, which is what makes this a serial model rather than a coincidence.
Stable over 37 consecutive windows.

**They were serialised at `wsi_orbis.c:464`, and the overlap is now DONE and measured:**

| | before | after |
|---|---|---|
| frame | 143 ms | **109 ms** |
| fps | 7.0 | **9.2** |
| present waiting for the GPU | 103 ms, 71% | 56 ms, 51% |
| CPU | 49 ms/frame | 60 ms/frame |

−34 ms a frame, **+31%**, against a prediction of +39%. GPU work is still ~103 ms; 47 ms of it now runs
under the CPU's own 60 ms. Full overlap would give `max(60, 103) = 103 ms` — 9.7 fps — so **about 6 ms
of headroom is left in this lever and it is spent.**

⚠ **Deferring the present alone changed NOTHING** — 71% before and after, to the percentage point. The
wait targeted `orbis_submit_seq_no` as it stood when called, so once the copy was deferred it waited for
the frame *after* the one being copied. Narrowing the wait to the frame's own submission was dismissed
when the deferral was proposed, on the grounds that one submission per frame makes the two identical.
That was true of the immediate present and stopped being true the moment it was deferred: **the
narrowing is a precondition of the deferral, not an alternative to it.**

**From here the GPU is the floor** — 103 ms of it against 60 ms of CPU hidden underneath.

**And the GPU side is fully characterised.** Two resolutions give `F + P = 103` at 1080p and
`F + P/4 = 26` at 540p, so `P = 103, F = 0`: **there is no resolution-independent cost at all** — not
shadow maps, not geometry, not state. (Antialiasing is not a confound; `aaPreset` defaults to 0 so
CMAA2 never ran.) The driver then counted what the title asks for:

    18 render passes covering 20.15 screenfuls of 1920x1080, plus 24 dispatches of 100000 workgroups

A full-screen 8x8 compute pass over 1080p is 32400 groups, so the dispatches add roughly three more
screenfuls. **~23 screenfuls a frame, 4.5 ms each.** Pure fill at 32 ROPs and 800 MHz would be 0.08 ms,
so this is shader work — around 4000 flops per pixel per pass, which is a lot but not absurd for
lighting, SSAO, volumetric fog and sky LUTs.

So both things are true: the renderer does a great deal of full-screen work for 2013 hardware, and each
pass is expensive. **The levers are mostly in the title.** `vidResIndex=1` (0.75 scale, 56% of the
pixels) is the untested middle point, and it is where the CPU's 60 ms should become the floor instead —
at which point the driver's own waste starts to matter: three unconditional `getenv()` calls per
submission (`ac_orbis_drm.c:7774, 7919, 7953`) and the flattener copying every chunk.

⚠ Narrowing the wait to this frame's fence would gain nothing: the windows show 35 submissions for 35
presents, so "the latest submission" already is "this frame's". Checked, not assumed.

**Getting the instrument right took three attempts and the first two were caught rather than believed.**
A hand-instrumented version summed counters around individual poll loops and a code review found nine
reasons its number would be wrong — the worst being that it leaned on `orbis_idle_wait_ns`, which this
project had **already measured as 55x wrong with a written do-not-use instruction**. Reverted. Then the
clock probe was widened to ids 0–4 with two phases, a sleep *and* a spin, because one phase cannot tell
a CPU clock from a broken one; the console answered that all five ids are wall clocks there while the
laptop reports 2 and 3 as CPU time. `getrusage` works because libkernel exports it itself, so the link
misses musl's Linux-numbered syscall stub.

⚠ **And a conclusion withdrawn.** After the first budget run I wrote that optimising this driver's CPU
paths could not raise the frame rate, because 0.35 cores meant the CPU was idle. **Not saturated is not
the same as off the critical path when the work is serialised.** The half-resolution run exposed it.

## Capabilities and closed doors

**`sceKernelMprotect` reaches the GPU's page tables.** Measured. So amdgpu's teardown is reproducible
here, not merely imitable. `ORBIS_PROTECT_FREED=1` revokes on unmap and restores on map, whole 16 KiB
pages only.

**The GNM debugger family is present and shut** — `0x80000000` at the gate, `0x8eee00ff` behind it. The
address watch and the non-GRBM register route are both behind that door. Do not spend an evening there.

**Sony's builders write PM4 into our buffer.** `DrawInitDefaultHardwareState` gives 256 dwords and 39
registers; prepending all of them changed nothing. `SetHsShader`/`SetLsShader` refuse every reservation
from 1 to 64 dwords with two different blobs, so their argument shape is unknown.
`DrawInitToDefaultContextState` returns 0 dwords for the shape that worked for the other.

**The difference from FreeBSD is not the kernel.** FreeBSD runs Mesa through `drm-kmod`, which is Linux
DRM including `amdgpu`. The PS4 shares the FreeBSD kernel and has no amdgpu. The axis is amdgpu vs
no-amdgpu.

## Method, and it earned its length

**Every instrument must be seen to fire before its silence is evidence.** Eight produced confident false
numbers today, and every one was caught by a control rather than by the console:

* the dead-pool check was silent because it was broken — a host probe caught it
* the mprotect ladder's control rung caught `SRC_SEL=5` (IMM) where memory needs 1; the value read back
  was the victim's own address, which named the mistake
* that ladder's verdict branched on whether the end-of-pipe arrived, and printed "does not reach the
  GPU" in the same second the klog recorded the fault proving it does
* a scan mask of `0x0ff00000:0xfff00000` matches one dword in 4096 — 30024 "hits" is chance
* adding `0xf0000000` filled all 32 printed lines with a value common in real data
* the tessellation registers' zeros were read as data for two runs before a round trip showed the block
  does not answer at all
* a byte-grep of the retail captures reported "3057 of 3057"; a packet decode found zero, because the
  byte sequence matches unaligned
* the strip's first pixel sample fired on the black loading screen, so every value including the control
  came back near zero

**And twice I nearly reported a finding from a comment** — `ORBIS_NO_CMASK` in an include line whose gate
no longer exists. Read the code, not the note about the code.

**Absence of a log line is not absence of the event.** A run once looked like it had removed the write
fault; the next brought it back at the same address. **A log pulled from a running title is a moment,
not an ending** — I called a process dead at 6062 submissions while it was still rendering.

## Build and run

    build-support/orbis/build.sh                          # driver, cross
    build-support/orbis/build.sh --host-orbis             # regression gate: probes + self-tests
    ~/src/Tempest/ps4/opengothic/build.sh --radv          # title
    -> ~/.cache/tempest-og/build-ps4/opengothic/IV0000-TMPS10021_00-TEMPESTOPENGOTHI.pkg

    lftp -p 2121 192.168.100.2                            # console; port 2121, not 21

⚠ **The title builds from `~/.cache/tempest-og/OpenGothic`, NOT from `~/src/OpenGothic`.** A shader edit
made in the second one compiles nothing, the package builds cleanly, and the run reports "no change" —
which is the correct outcome of testing nothing. One diagnostic run was lost to exactly that. **Check the
compiled SPIR-V's hash changed before believing a shader experiment**, e.g.
`md5sum ~/.cache/tempest-og/build-ps4/shader/sprv/sky.frag.sprv`. The sky has two variants, `sky` and
`sky_sep`; VolumetricHQ uses `sky_sep`.

**The fault address lives only in the klog.** The mesa log has never carried it and
`has_gpuvm_fault_query = 0` means the amdgpu query interface never will. Check the receiver is growing
before starting the title: one died silently at console suspend and eight hours of missing faults read
as "no faults".

**The env file carries only what the run needs.** No `OG_ARGS=-bl 0` — it is not required and the
maintainer has asked twice. Every extra line is a second variable in a one-variable run.

## Closed by the tessellation fix

**Artefacts worsening with playtime** (was #19) only happened with tessellation enabled and is gone.
Of the two mechanisms recorded for it, the first was the right family: the tess ring watermark climbing
towards its ceiling (0.98x and still rising when last measured) was the stage corrupting more as play
continued, not freed addresses being re-let.

The second mechanism is still true and still unexplained by anything — the arena re-lets freed
addresses immediately, 16 warnings a run. It was simply not this. If a time-dependent artefact ever
returns, that is where to look, and `ORBIS_PROTECT_FREED=1` now makes a stale access fault at the point
of use instead of reading whoever moved in.

## The Vulkan CTS runs on this console, and it found five defects on its first day

`build-support/orbis/cts/` builds `deqp-vk` for the PS4: a platform file, a target file, six small
patches and a 107 MB package. `install.sh` puts them into a checkout. The README there carries the
full account; this is what a reader needs to know.

**Why it exists.** Every measurement this driver had ever taken came from one game, and the
maintainer named the risk before the tooling did: a queue bound written as `count-3` worked because
OpenGothic asks for five swapchain images and would have deadlocked anything asking for three.

**Where it stands.** `dEQP-VK.memory.*` is **6363 tests, zero failures**. `api.*` is largely passing
with three things open. Two families of failure were found and both were this driver claiming a
capability it did not have - `EXT_zero_initialize_device_memory` and `EXT_external_memory_host` - and
both were fixed by dropping the claim rather than faking the capability.

⚠ **Two defects had been wrong since the first day of the port and were invisible until something
multithreaded ran:**

* **`_umtx_op` is ENOSYS on this console.** Mesa's `simple_mtx` is built on futexes, so every
  contended lock in the driver was a pure SPIN - it could not sleep and could not be woken. Nothing
  this port ran was contended, so nothing showed it. `shims/sys/umtx.h` is now an implementation
  rather than a syscall declaration, built on the pthreads this console does have.
* **`sizeof(pthread_mutexattr_t)` is 4 and the implementation writes more.** One mutex creation
  cleared 24 bytes of an unrelated heap block 4704 bytes away; the identical calls in a different
  stack frame did no damage. Same code, different frame, different victim.

**Still open from the CTS:** nonexistent instance layer names are accepted; allocation callbacks are
not called for descriptor set layouts (not yet attributed - RADV may do this upstream too); and
`object_management.multithreaded_*` still needs re-running now that the futex exists.

⚠ **Ceilings the CTS walked into that a game never would**, all now growable: the BO table (4096) and
the live-mapping overlap table (8192). The second is the one to remember - it had gone silently
blind, and a blind overlap checker reads as a clean run forever after.

## ⚠ A repair that lives only in a working tree is one reset away from gone

The title came back with artefacts and a ten-times slower load. The cause was
`0008-og-intermediate-buffer-views`, a diagnostic patch of ours that adds `dbgStash()` at four
points inside the frame's command encoding. **It had already been fixed once - by hand, in the build
checkout, never as a patch.**

`ps4/opengothic/build.sh` keeps a stamp of the patch queue it last applied. The stamp said 35 while
37 patches were applied: the tree contained work the queue did not describe. That mismatch was read
as "make the tree match the queue" and answered with `git checkout -- .`, which destroyed the hand
fix permanently - no stash, no reflog, no dangling blobs, because `git checkout` on unstaged changes
leaves nothing in the object store.

⚠ **The mismatch was the warning.** It means the tree holds something unrecorded, which is exactly
the state whose contents must be saved before anything touches it. Dump `git diff` first, then turn
what it holds into a numbered patch.

**Finding it again cost a day and eleven console runs**, of which four were the bisection that
actually worked and the rest were mechanisms asserted from memory rather than measured:
tessellation (the env log showed `ORBIS_NO_TESS=1` applied), the driver (bisected across twelve
commits - exonerated), the fog-LUT patch, a jammed libc `FILE` lock, and slow loading blamed on the
six-core patch when the maintainer pointed out the title reached the menu before that patch existed.

⚠ **And three "hangs" were crashes.** The klog receiver had been dead since the console was switched
off overnight - its process was alive, so it looked healthy, and it had received nothing for twelve
hours. The rule in this file is "check the receiver is GROWING"; checking that it exists is a
different test. With it back, one run named a stack overflow in
`radv_graphics_shaders_compile` in a single line.

**What the bisection established**, four runs, one variable each: 18 diagnostics out -> clean;
0005-0012 -> artefacts; 0005-0008 -> artefacts; 0005+0006 -> clean; +0007 -> clean. So 0008, and
with it `0009 0011 0031 0035 0036`, which patch `renderer.cpp` on its hunks and cannot apply
without it. All diagnostics, none load-bearing. The queue is 31 patches and they are deleted from
it, so a re-clone cannot bring them back.

⚠ **The mechanism is still unknown.** `dbgStash` opens with `if(dbgStashAt!=stage) return;` and
`dbgStashAt` starts at 0 against stages 1..4, so the copy should be inert until F1 cycles it.
Something in that reasoning is wrong. The culprit is measured; how it does damage is not.
## What was fixed on 2026-08-15 and 16

| defect | how it was established |
|---|---|
| `orbis_arena_setup` claimed its once-flag before doing the work | a 180 s hang, twice; it passes now with 16 tests behind it |
| the default thread stack is **65536 bytes** against a 72 KB compile frame | klog backtrace; `tcuMain` prints the number at startup |
| `_umtx_op` is ENOSYS, and its timed arm used the wrong clock | driver self-tests plus `tools/umtxcheck.c` |
| `radv_create_cmd_buffer` destroyed a half-built buffer on the OOM path - **upstream** | SIGSEGV at `0x8`, derived from `list_head`'s layout first |
| the syncobj table was a fixed 1024 - the third such ceiling | `binary_semaphore.chain` ended a 1899-test session at test 1 |
| **`SET_PREDICATION` hangs this CP**, for conditional rendering as for meta | halted on the first predicated test; five byte-identical samples over three minutes |
| conditional rendering moved to **COND_EXEC**, its inverted arm needing `PFP_SYNC_ME` | 210 failures -> 121 -> **21**; `expect_noop` 24/104 -> **128 / 0** |
| **`DRAW_*_INDIRECT_MULTI` hangs this CP** - a GFX7 *firmware* feature this console lacks | two runs: count field cleared still hung, packet dropped gave 482 verdicts |
| indirect multi-draw replaced by a loop of plain `DRAW_INDIRECT` | **160 / 160** on `multi_command`, zero regressions |
| the count buffer honoured with `COND_WRITE` + `COND_EXEC` | 401 pass / 3 fail where the same family had 144 failures |
| `ORBIS_NO_TESS` left `multiviewTessellationShader` true - **half a switch** | two runs halted on the same test with zero tessellation NotSupported |
| the GS rings are undersized **in two independent ways** | see below - it took eight configurations and is not finished |

**CTS standing**: `memory.*` 6363/6363 · `object_management.multithreaded_*` 134/134 ·
`conditional_rendering` 906 for 21 failures · `geometry.*` 180/181 · `binding_model` geometry
328/328 · multiview 703 passing alone, 9/72 with geometry.

## ⚠ The GS rings: what is shipped, and why it is not a repair

Two faults, each mistaken for the whole story at some point:

    size register too small   the ring wraps early and overwrites itself -> WRONG IMAGES
    allocation too small      the hardware writes past the size it was told -> GPU FAULT, device dies

Shipped: `scale 2, pad 16x, total capped at 512 MiB`. 606 tests, 523 pass, 64 fail, no device loss.

⚠ **`multiview + geometry` is left broken deliberately.** Scale 4 fixes it 78/78, but there the
largest ring reaches upstream's clamp of `63.999 MB * num_se`, the padding asks for half a gigabyte
on top, and the arena loses the device - measured uncapped, at 512 MiB and at 256 MiB. **63 wrong
images beat a device loss that takes 328 passing tests with it.**

⚠ **Neither fault is understood, and the multipliers are margins.** The formula is identical in RADV
and radeonsi, term for term, and both run on real GFX7 under amdgpu. Its topology input is confirmed
by the published spec - 18 CUs, 72 TMUs, 32 ROPs, HD 7850 based - and `ORBIS_NUM_SE=4` was falsified
on hardware. Formula right, inputs right, hardware still overruns.

`cts/README.md` lists all eight configurations tried so nobody repeats them, and the one hard number:
**the arena limit is between 536 MB survived and 646 MB lost.**

## Where to start next

### A. No console needed

**A1. Find why this part writes past a ring size it was given.** The answer is not another
multiplier. `VGT_GSVS_RING_ITEMSIZE` and `VGT_GS_VERT_ITEMSIZE` are the per-vertex strides the
hardware computes offsets from; on-chip versus off-chip GS mode is the other candidate. Compare what
this driver programs against radeonsi for the same generation - that comparison has already paid off
twice today, for `has_draw_indirect_multi` and for the ring formula itself.

**A2. `geometry.basic.output_vary_by_texture`** - the last geometry failure, and narrow. Its GS picks
an emit count from a **texture fetch**; the same test with instancing passes, and so do the uniform
and attribute variants. The difference is that the failing one takes its texture coordinate from a
VS output through the ESGS ring.

**A3. Point `tilecheck` at a layered MSAA image.** Multiview + multisample halts a run and was never
investigated; the first suspect is layered MSAA surface layout, because the one layout defect this
port has found was a layered surface. `tilecheck.cpp` and the drm-shim Liverpool entry exist for it.

### B. Needs the console

**B1. Finish the survey.** 24800 of 25613 still unrun, the giants entirely: api 267k, binding-model
150k, transform-feedback 134k, fragment-shading-rate 110k, robustness 99k, synchronization2 82k.
Three session-enders were removed from its path yesterday and today; this is where the next ones
turn up.

**B2. Re-run the COND_WRITE probe, fixed.** ⚠ It killed the run it shipped in. Its answers stand -
every rung completed and printed - but it must not run again as written. Suspect its arena slots and
the eight submissions it injects into the driver's own submit path.

**B3. The three small conditional-rendering groups**, in this order: `draw_clear.clear.depth.*discard*`
(8, best understood - the colour equivalents all pass), `vkCmdSetEvent2` (3), `dispatch.condition_size`
(6, needs a 32-bit compare built without COND_EXEC, which is itself narrow).

**B4. Tessellation (#21)** stays last: most expensive, no oracle, and its cheapest route starts by
buying a game.

## ⚠ Traps that cost real time, in the order they will cost it again

**Ten readings of one defect were wrong, and every one generalised a real measurement past its
conditions.** "The first complete primitive fails" (it was position in a 754-test run). "Tests
expecting an empty image pass" (a passing test had drawn a triangle). "Two of four views are white"
(0% extra pixels across all 63 failures said otherwise). "The GS copy shader needs the view index"
(`radv_nir_export_multiview.c` puts LAYER in the ring). "A sampler in the geometry stage kills the
console" (it was the ring again). "The register is right and only the allocation was short" (true for
one family). **The pattern never varies: the measurement is sound and the conclusion is stretched.**

**Extract the images from the .qpa.** `cts/qpa-status.py --images` does it. One four-minute run of
eight tests with images on ended a day of theories about the geometry rings.

**Read a killed run with `qpa-status.py`, not by eye.** The last-started test is always in the file;
reading it by hand off truncated output is what misdiagnosed the tessellation halt.

**`--deqp-crashhandler` has never fired here** and is not a mechanism to rely on. Nothing in the
process runs after the system kills it for a GPU fault.

**Run configuration lives in `cts/deqp-args.txt` and `deqp-env.txt`.** Copy them. Three runs died to
a line rebuilt from memory.

**A switch that half-applies is worse than none**, and the giveaway is in what the results do NOT
contain: `ORBIS_NO_TESS` left a second door open and zero cases came back NotSupported.

**A stamp/queue mismatch is a warning to SAVE, not to reset.**

**"The receiver is running" is not "the receiver is receiving."** Check the file is GROWING.

**Build the case list to answer the question.** One run measured nothing because all 476 of its tests
exercised the single thing the change deliberately did not do.

## Open

* **the GS ring mechanism** - both faults, neither understood
* **multiview + geometry**, 63 tests, knowingly left broken
* **multiview + multisample**, halts a run, never investigated
* **#21** tessellation
* **the arena still re-lets freed addresses immediately**, 16 warnings a run
* **the retire drain runs on every unmap**; the repair must not be "protect less"
* **`orbis-watchdog.txt` wrote one line** on a healthy multi-minute run
* **`COND_EXEC`'s comparison is narrower than 32 bits** - `0x1` executes, `0x100` does not. Four data
  points, no documentation

---

# Handoff — the forks, 2026-08-17

Today moved off patch queues and onto forks, and ended with a 175x load-time fix. The driver was
barely touched; the work was in the three repositories around it.

## Where everything now lives

    ~/src/forks/OpenGothic        ps4-support       the title's PS4 port
    ~/src/forks/OpenGothic-linux  master            Linux build, for A/B without the console
    ~/src/forks/Tempest           ps4-mesa          the engine, Vulkan route only
    ~/src/forks/ZenKit            ps4-support       lazy VFS mounting
    ~/src/forks/ZenKit-locals     fix-locals...     one upstream fix, deliberately separate

⚠ **NOTHING IS COMMITTED IN ANY OF THEM.** All four carry working-tree changes only. The maintainer
asked to commit himself after reviewing.

⚠ **`git status` fails in `~/src/forks/OpenGothic`**: `lib/Tempest` and `lib/ZenKit` are symlinks to
the sibling forks, and git refuses a submodule path that is a symlink. Read it with

    git -c core.symlinks=false -c submodule.lib/Tempest.ignore=all \
        -c submodule.lib/ZenKit.ignore=all status --short

Symlinks are there because the forks are uncommitted; a submodule cannot point at an uncommitted
state. Once they are committed and pushed, `git submodule set-url` to the fork URLs replaces them and
the workaround goes away. `ps4/build.sh` checks for a FILE in each rather than a revision, so it will
keep working either way.

## The one that mattered: 144 s -> 824 ms

`Resources::loadVdfs` carried a dead line:

    auto in = zenkit::Read::from(i.name);   // never used

`Read::from(path)` without `_ZK_WITH_MMAP` allocates a vector the size of the file and reads the
whole thing through an ifstream. 2.69 GB read and thrown away, at ~24 MB/s.

It cost nothing for as long as that call mapped instead of reading. The moment the port stopped using
mmap it became two minutes, and it looked exactly like the lazy mount being slow.

    mounting 15 archives     144 s  ->  0.824 s
    whole boot to exec       ~150 s ->  1.7 s

⚠ **This is an upstream bug**: on any platform without mmap, OpenGothic reads every archive twice. It
is a one-line report to Try/OpenGothic and owes nothing to the PS4.

## What is ready for review, and what it is for

**ZenKit `ps4-support`** — 733 insertions across 7 files. `VfsMountMode::{FULL,LAZY}`, LAZY reading
only the header and catalog and serving file data through pread on demand; `ReadVfsRegion` streams a
window rather than materialising the entry; the descriptor holds a `shared_ptr<int>` so the archive
closes when the last node stops reading it. `bs_check_posix_pread` mirrors the file's own
`bs_check_posix_mmap`, and nine tests cover the LAZY paths.

  * FULL is the default everywhere, so every existing call behaves exactly as before.
  * There is not one `#ifdef __PS4__` in it. The guards are `_ZK_WITH_PREAD`, decided by a configure
    check, because the question is whether the platform has pread() and not what console this is.
  * `src/DaedalusScript.cc` is three lines: `span(pointer, count)` instead of the iterator-pair
    constructor, which libc++ 11 does not have in a form `vector::iterator` satisfies. Verified
    against the SDK's own `<span>` (`_LIBCPP_VERSION 11000`, constructors take `pointer`).
  * `src/Stream.cc` is one line: `if (len == 0) return 0;` before a memcpy that would otherwise be
    handed a null pointer for a zero-length read.

⚠ **Five test cases fail for want of `samples/`**, and that is not this change: the pre-existing
`Vfs.mount_host` and the Daedalus tests fail identically. Four LAZY tests that build their own
archive pass, including the empty-file and streaming ones.

**Where it goes**: `GothicKit/ZenKit` is upstream and `Try/ZenKit` is its fork, one commit behind.
Our base `083cc5f` is an ancestor of `gothickit/main` and the change applies to it cleanly - checked.
`DaedalusScript.cc` could go today; `Vfs.cc` is the bigger conversation.

## Three claims made and reversed today

Each was a real measurement whose conditions were then ignored. The pattern has not changed since the
GS rings.

**"mmap was never the fault."** The boot probe says `mmap agrees with read()` and it is true - of ONE
64857-byte archive. Mapping 2.69 GB across fifteen files freezes the whole console: `SceShellCore`
reports its own main thread frozen for 9 seconds. A small mapping proving correct says nothing about
a large one proving survivable.

**"144 s is a regression in our rewrite."** The pre-rewrite implementation measures 137.5 s on the
same data. Both were paying for the dead `Read::from`. The comparison package the maintainer asked
for is what settled it; without it the search would still be inside ZenKit.

**"FULL mounts in 0.9 s."** That number came from runs that were ALSO lazy - patch 0004 has been in
the queue since 2026-08-03. It was never a measurement of mmap.

## Traps, in the order they will cost time again

⚠ **`~/.cache/tempest-og` (27 GB) STAYS.** The patches themselves live in
`~/src/Tempest/ps4/opengothic/patches/` - 32 of them, in a real git repo - but the cache holds the
scratch tree whose uncommitted state `build-support/orbis/title/opengothic-ps4.patch` is a snapshot
of. `emurun` is 25 GB of that and is the only obvious candidate if space is ever needed.

⚠ **Diagnostics were cut too widely from the Tempest fork.** `scripts/` was dropped as "not this
route" and five files had to come back one at a time as each build step tripped over them:
`make-pkg.sh`, `gen-icon0.py`, `logs.sh`, `log-receiver.py`, `peerfilter.py`. Check the whole
dependency chain at once rather than adding whatever just failed.

⚠ **A worktree of a branch with no commits gets the branch's COMMIT, not the working tree.** An
`OpenGothic` worktree came up empty of every PS4 change for this reason, and the fix was to carry the
diff across by hand.

⚠ **`cd` failing does not stop the rest of a chained command.** A worktree that failed to create left
a `cd` unexecuted and a python edit ran in the main tree instead, destroying the LAZY call site. Put
`&&` between them.

⚠ **ZenKit's `vendor/CMakeLists.txt` fetches doctest unconditionally**, not gated on
`ZK_BUILD_TESTS`, and doctest 2.4.9's `cmake_minimum_required` is too old for CMake 4. Cross-builds
need `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` until that is fixed upstream.

⚠ **Do not launch the game unasked.** Several background runs put windows on the maintainer's desktop
while he was working.

## The upstreamable list, as it stands tonight

    ZenKit    DaedalusScript.cc span(pointer,count)      ready, 3 lines, no PS4 in it
    ZenKit    Stream.cc zero-length memcpy guard          ready, 1 line
    ZenKit    Vfs.cc lazy mounting                        needs the review this handoff is for
    ZenKit    find_locals_for_function at end of table    on its own branch, NEEDS A TEST
    OpenGothic  the dead Read::from in loadVdfs           ready, 1 line, real upstream bug
    Mesa      five Tier A fixes                           unchanged from yesterday; fd.o still
                                                          refuses project creation for the account

⚠ **`OpenOrbis/musl` PR #35 is open, unreviewed, 15 days old, and touches three of our findings** -
it fixes `mode_t` (and with it `struct stat`), adds FreeBSD `CLOCK_*` values, and routes musl's
internal futex through `_umtx_op`, which we measured as ENOSYS on this kernel. Nothing was sent; the
maintainer will write anything that goes there himself.

---

# Handoff — the ZenKit review, 2026-08-18

One topic today: making `ps4-support` look like code GothicKit would take. Three questions from the
maintainer, each of which found something.

## The two constants are derived, not magic

    VFS_HEADER_SIZE        = 296   read_string(256) + read_string(16) + 6 x read_uint()
    VFS_CATALOG_ENTRY_SIZE = 80    read_string(64)  + 4 x read_uint()

Both read straight off the parser's own field list, and checked against a real archive:
`Meshes.vdf` has its signature at 256, `catalog_offset` = 296, and entries at 296 / 376 / 456 -
stride 80, names decoding cleanly.

They exist because LAZY has to know both sizes BEFORE parsing anything: it fetches exactly
`VFS_HEADER_SIZE` and then `entry_count * VFS_CATALOG_ENTRY_SIZE`. FULL never needed them - the whole
file was already in the buffer.

⚠ `catalog_offset` is 296 in all fifteen Gothic II archives but the format does not require it. The
code handles a catalog elsewhere in the file and rewrites the offset field in its copied buffer so
the parser sees it at 296.

## A hole on Windows, and the fix that was already right

`VfsFileDescriptor`'s fd-backed constructor was guarded with `#ifdef _ZK_WITH_PREAD` in the header.
That was removed on the 17th as "an ugly split of public API by build flag". It left a real hole:
without pread, `open_read()` fell through to `Read::from(fd.memory, fd.size)` with `memory == nullptr`
and a non-zero size, so the first read did `memcpy(buf, nullptr + pos, len)`.

Unreachable from inside ZenKit - only the LAZY mount produces such descriptors and it is itself under
`_ZK_WITH_PREAD` - but `VfsFileDescriptor` is public with a public constructor, so a library user can
build one and get a segfault.

⚠ **The guard is back, and the convention is why.** ZenKit's own answer to "this build does not have
that feature" is to remove the DECLARATION:

    #ifdef _ZK_WITH_ZIPPED_VDF
        static std::unique_ptr<Read> from_zipped(...);   // Stream.hh:102
        ZKAPI void save_compressed(...);                 // Vfs.hh:206
    #endif

A compile error, not a runtime throw. The `std::runtime_error` briefly added for the no-pread case was
this port's invention and is gone.

`src_handle` and `src_off` stay OUTSIDE the guard, so the struct's layout is identical everywhere and
no ABI splits; only the ability to construct one is conditional. That is exactly `save_compressed`'s
shape.

## Where the exception types now land, and why each matches

    no pread in this build   declaration absent, compile error   like from_zipped / save_compressed
    a syscall failed         std::runtime_error                  like MmapPosix.cc / MmapWin32.cc,
                                                                 same wording: "Failed to open ..."
    the archive is corrupt   VfsBrokenDiskError                  like the rest of Vfs.cc

Counted upstream before deciding: 25 `DaedalusVmException`, 22 `ParserError`, 5 `std::runtime_error`
- all five in the two Mmap files, all five for a failed OS call.

## State

`~/src/forks/ZenKit` `ps4-support`, still uncommitted, 7 files. Builds clean on Linux and with
`-D_WIN32`; the LAZY tests stand at 4 passing, 5 failing for want of `samples/` - the same five that
fail for the pre-existing `Vfs.mount_host` and Daedalus tests, so not this change.

## What the review has not covered yet

  * **`Vfs.cc` carries three validation checks** that are not PS4 needs - an entry extending past the
    catalog, a directory pointing past it, and the extent bound. They are needed by LAZY, because
    `extent_limit` stops coming from the buffer, and they have tests. Decide whether they ship as one
    commit with the lazy mount or as a separate hardening commit.
  * **`ReadVfsRegion`'s buffer is 4 KiB**, chosen as a page size and never measured. It is NOT on the
    mount path - mounting does two preads per archive - so it did not matter for the load-time work,
    but it is the window every resource read goes through.
  * **`find_locals_for_function`** is on `~/src/forks/ZenKit-locals` and still has no test.

## Upstream context found while reviewing — three voices on one root

Read before writing the PR description. None of it changes the code; all of it changes how the
change should be introduced.

**Issue #112 — "VfsNode or Read buffer access"** (Katharsas, open since 2025-10-20). Asks for
`std::span<std::byte> as_buffer()` or pointer+size out of a `Read`, because DirectXTex and friends
want a buffer, not a stream. lmichaelis' reply matters twice over:

  * *"specifically with the **new mmap-less I/O system** that's not super easy to provide"* — upstream
    has already chosen to move away from buffer-backed reads. `VfsMountMode::LAZY` is that direction
    applied to archives, not a request to change course.
  * *"I would not be opposed to add a `remaining` or `size` function to `Read`."* — `ReadVfsRegion`
    holds `_m_length` and `_m_position`, so it implements both in a line each if that ever lands.
    `Read` has no such method today; the interface is `read`/`seek`/`tell`/`eof`.

⚠ It cuts against us too, and saying so first is cheaper than being told: every new `Read`
implementation without a contiguous buffer takes `as_buffer()` further out of reach.

**PR #130 — "support VDF archives larger than 2 GiB"** (FadedAtlas, open since 2026-07-26, zero
comments). Two bugs chained: `bs_check_win32_mmap` uses `CreateFile` with a wide literal, which
resolves to `CreateFileA` on GCC 14+ and fails to compile, leaving `_ZK_WITH_MMAP` undefined; the
`ifstream` fallback then does ONE `read()` of the whole file, which fails above `INT32_MAX` on MinGW.
Fixed with `CreateFileW` and 256 MB chunks.

  * ⚠ **It touches both of our files** - `src/Vfs.cc`'s `mount_disk(path, ...)` and
    `support/BuildSupport.cmake`. Textual conflict certain, semantic conflict none: his hunks are in
    the `#else` branch and in `bs_check_win32_mmap`, ours are the `LAZY` block before it and
    `bs_check_posix_pread` beside it. Whoever lands second rebases in a minute.
  * His FAQ names this port's neighbour: *"ZenKit's Windows job builds with MSVC… OpenGothic compiles
    ZenKit with MinGW, so its releases hit both bugs."*

**The three together are the argument for the PR.** The mmap-less path is not a corner: the
maintainer is deliberately moving onto it (#112), it breaks above 2 GiB where it is actually used
(#130), and it read 2.69 GB per boot here. Three people, three symptoms, one root. That is a better
opening than "a console port needs an exception".

## Could #130's author use LAZY instead?

Not as-is, and the reason is worth carrying:

  * LAZY would dissolve his bug rather than bound it - it never reads a whole archive, so a 2.39 GB
    file is unremarkable. His 256 MB chunks bound the symptom; FULL stays the default and **still
    needs his fix**.
  * It needs `pread`, which MinGW-w64 does not provide (msvcrt has `_read`/`_lseek`, no positioned
    variant). ⚠ Reasoned from the platform, not compiled - there is no MinGW here.
  * Win32 has the primitive: `ReadFile` with `OVERLAPPED` carries the offset explicitly.
    `ReadVfsRegion::pull` is the only place that would change.

⚠ **And `off_t` is the real portability edge, not pread's absence.** Both preads cast to `off_t`; on
a build without 64-bit file offsets that truncates above 2 GB and returns EINVAL. PS4 and Linux
x86-64 are 64-bit `off_t`, so it is invisible here. A Win32 backend sidesteps it by passing the
offset as two 32-bit halves.

⚠ Beyond either: **VDF catalog offsets are 32-bit** (`read_uint()`), so the FORMAT cannot address an
archive past 4 GB no matter who implements the reader. 2.39 GB fits; 5 GB never will.

## Where the test samples come from — nowhere, and that is deliberate

`tests/` reference `./samples/G1/DEMON_DIE_BODY.MAT`, `HUMANS-S_FISTRUN.MAN`, fonts and savegames -
Piranha Bytes files that cannot sit in a public repository. There is no samples submodule and no CI
download step; `doctest_discover_tests` runs with `WORKING_DIRECTORY .../tests` and
`-tse=CutsceneLibrary,DaedalusScript,World`, which excludes the suites needing the largest assets but
NOT `Vfs`.

So the five failing LAZY cases will pass for the maintainer, who has his own `samples/`, and cannot
pass here. Say that in the PR rather than implying a green run.

---

# Handoff — the toolchain is the bug, 2026-08-19

Two threads today. The CTS fork was tidied for publication, and then a question about one of its
patches turned into the discovery that the port has been working around the SDK in four places at
once. That became a repository.

## The CTS fork, from 697 lines to 515

`~/src/forks/VK-GL-CTS`, branch `orbis`, based on **`vulkan-cts-1.4.6.1`** - the tag this tree's own
`.gitlab-ci/container/build-deqp.sh` pins as `DEQP_VK_VERSION`. There is no GitLab mirror of the CTS;
Mesa's CI clones it from GitHub Khronos, so we do too.

What came out, and why:

* **`qpTestLog.c` reverted entirely, 47 lines.** Instrumentation from a closed investigation - its own
  comments asked "WHICH CALL CLOBBERS IT" and "the fix being tested now is in deMutexUnix.c, and
  putting this back is what tests it". The test happened.
* **The heap probe in `tcuMain.cpp`, 25 lines.** 512 MiB of allocate-and-free on every startup to
  print a number that has an answer.
* **`freopen` of stdout/stderr, 20 lines.** Its own comment says both files came back EMPTY.
* `#define attr` → `ATTR`, because redefining a four-letter identifier across seventy lines of
  function body is a mine, and the union initialiser removed a `memset` that had no `<string.h>`.
* `memfd_create` now takes **upstream's own** not-supported path - one `#if` widened by
  `&& !defined(__PS4__)` instead of a nine-line block of ours. The file already does this six times.

⚠ **Comment convention, measured rather than guessed.** `framework/delibs` has 1031 comment blocks
outside licence headers: median **1 line**, eight of five or more, longest ten - and the three longest
all quote IEEE-754 in `deFloat16.c`. Ours were median 5, twelve of five or more, longest 25. The two
longest blocks in all of delibs were both ours. Now median 2, longest 7 (two file-format
descriptions, kept deliberately). Long comments belong where the algorithm implements a spec; the
narrative of an investigation belongs in the commit message.

## Which patches are ours to keep, and which the toolchain owes us

The question "does the CTS need this once the typedefs are fixed" split the seven patches cleanly,
and the split is the whole reason `orbis-compat` exists:

    deMutexUnix padding      -> the overlay grows the type; the patch retires
    qpCrashHandler execinfo  -> the overlay supplies execinfo.h; the patch retires
    deThreadUnix 1 MB stack  -> policy, not a defect. Could move; measured below
    deMemory malloc.h        -> one line; cheaper here than wrapping <stdlib.h>
    deTimer sigevent         -> STAYS. The header lacks only a macro, but adding it would compile a
                                path whose runtime support is unknown - "a symbol that LINKS is not
                                evidence it works", again
    memfd_create             -> STAYS. Not a libc gap; dEQP's own mechanism covers it
    tcuMain, platform, cmake -> STAYS. That is the CTS, not the console

`deMutexUnix.c` now carries `DE_STATIC_ASSERT(sizeof(pthread_mutexattr_t) < sizeof(void *))`, so the
day the toolchain is fixed the build stops and says which workaround to delete.

## `~/src/forks/orbis-compat` — a repository meant to shrink

The port had been patching the same class of defect in four trees: `og_ps4_{stat,mmap,mem,paths}` in
OpenGothic (2843 lines), `shims/` here, `orbis_compat.c` inside a GPU driver, and Tempest's own. And
they were not shared - `Tempest/cmake/ps4-openorbis.cmake:67` passes only the SDK's include path, so
**Mesa was the only component compiled against our corrected headers**.

Its README is the plan and the evidence; every item is labelled with where it finally belongs, since
the OpenOrbis maintainers have agreed to take these upstream. Present so far, all verified by
`./check.sh` on the laptop with no console needed:

    include/bits/alltypes.h   four pthread types corrected
    include/execinfo.h + src  backtrace(3) over _Unwind_Backtrace
    include/{machine,sys}/    six headers copied verbatim from shims/, with a drift check that
                              deletes itself when the originals do

⚠ **`#include_next` made the header override cheap.** musl guards every typedef with
`__DEFINED_<name>`, so defining one first suppresses its own - 45 lines instead of a 441-line copy of
a generated file, and nothing to re-sync when the toolchain moves.

⚠ **`backtrace` is real, not a stub.** `libc++.a` defines `_Unwind_Backtrace`, clang ships
`unwind.h`, and the build already passes `-funwind-tables`. The port has read a crash as a hang three
times; it can now print where it happened. The host test caught a missing `<stdint.h>` that the cross
build had accepted through the PS4 headers - *a file that cross-compiles is not evidence that it is
correct*.

## What the hardware said

**Five pthread types, one run.** `ORBIS_PTHREAD_LAYOUT_PROBE`, since deleted:

    pthread_mutexattr_init    declared=4  touched [0..7]  span=8  OVERRUNS
    pthread_condattr_init     declared=4  touched [0..7]  span=8  OVERRUNS
    pthread_barrierattr_init  declared=4  touched [0..7]  span=8  OVERRUNS
    pthread_spin_init         declared=4  touched [0..7]  span=8  OVERRUNS
    pthread_once              declared=4  touched [0..0]  span=1  fits

Four confirmed as eight-byte pointers, which is what `ORBIS_PTHREAD_MUTEX_INITIALIZER` being `NULL`
had implied. ⚠ The fifth was predicted to be the WORST - `ORBIS_PTHREAD_ONCE_INIT` is
`{ NEEDS_INIT, NULL }`, FreeBSD's sixteen-byte struct - and needs nothing. FreeBSD's implementation
locks the mutex at offset 8 even uncontended; offset 8 was untouched. The planned `pthread_once`
interposer, and the claim that libc++abi's prebuilt four-byte flag was being overrun, are both
withdrawn.

**`_umtx_op`: the syscall was never the question.** OpenOrbis/musl PR #35 routes musl's futex through
`_umtx_op` and reports it working; `shims/sys/umtx.h` opens with "AND THIS KERNEL DOES NOT HAVE IT".
Both are right, about different things. `libkernel.so` exports `_umtx_op` at 0xd366 and it is a real
compare-and-wait:

    rung 1  raw syscall 454 WAKE                   -> -1 errno=78     ENOSYS, as recorded
    rung 2  libkernel WAKE (no waiter)             ->  0 errno=0
    rung 3  libkernel WAIT, value does NOT match   ->  0 errno=0
    rung 4  libkernel WAIT, value matches, 150 ms  -> -1 errno=60 after 149906 us

Rung 3 waited with NO timeout and returned instantly - it read the word rather than sleeping blindly.
Rung 4 is the same call with a different value in memory and slept the deadline to 94 us.

⚠ **The mismatch case returns 0, not EWOULDBLOCK.** That is not Linux's convention, and the probe's
own verdict string said "not a comparison, so not an implementation" - wrong. Neither rung settles it
alone.

⚠ And the driver had already written the real explanation, unnoticed:
`raw syscall check: syscall(SYS_getpid)=-1 errno=78 ... raw syscalls are refused, so _umtx_op may
exist behind libkernel`. **Syscalls are refused wholesale, even `getpid`.** ENOSYS on 454 was never
evidence about `_umtx_op`.

**And then the A/B said keep our 374 lines.** Same console, same 49 cases, same arguments, driver
distinguished per binary by whether `_umtx_op` is undefined:

    A  libkernel        15 tests, 14 passed,  63 devices   ABORTED - VK_ERROR_OUT_OF_HOST_MEMORY
    B  shims/sys/umtx.h 49 tests, 49 passed, 179 devices   completed, 100%

Run A does not hang, which is what the probe predicted and is worth something - this is the workload
that once sat 180 seconds writing nothing. But it does not finish, and B finishes the same list on
the same hardware, so the host-memory exhaustion belongs to the libkernel variant rather than to the
workload. `-DORBIS_UMTX_LIBKERNEL=1` stays as a switch, off by default. One run each; the mechanism
behind the exhaustion is not established.

## Five traps, all of the same shape

Every one of these made an unchanged thing look changed, or a stale thing look fresh:

1. ⚠ **The package must go to `/data/pkg`, not `/data`.** Uploaded to the wrong place, the console
   started the build from three days earlier and logged no probe lines at all. **A missing answer is
   indistinguishable from an answer** - what caught it was the driver printing `arm built Aug 16` on
   its first line. Any probe should identify its build.
2. ⚠ **`build-support/orbis/shims` is COPIED into `~/.cache/orbis-mesa/cross/include` by build.sh.**
   Editing the source and running `ninja` directly compiles the old copy. The first `_umtx_op`
   variant was built against a shim without the `#ifdef` it was testing.
3. ⚠ **`meson setup` by hand loses the environment build.sh sets.** `PKG_CONFIG_PATH=` and
   `PKG_CONFIG_LIBDIR=<empty dir>` are what stop nix's devShell from supplying the HOST's libelf; the
   CTS link failed on 22 undefined `elf_*`. The comment in build.sh says exactly this.
4. ⚠ **`build-support/orbis/cts/deqp-args.txt` is version-bound.** It was written for `918221c` of
   `main`; `--deqp-watchdog-{interval,total}-time-limit` do not exist in 1.4.6.1, and dEQP rejects the
   whole command line for one unknown option. The file opens by warning that rebuilding it from
   memory costs runs - it now also needs to say which CTS version it is for.
5. ⚠ **Copying `external/` between CTS checkouts overwrites TRACKED files.** The diff went from 509
   lines to 1 331 494. `git checkout -- external/` plus `git clean -fd external/` unwound it, but our
   one patch under `external/` had to be saved and restored by hand first.

## Numbers worth keeping

* **The default thread stack on this console is 65536 bytes.** `radv_graphics_shaders_compile` wanted
  about 72 KB for one frame. The 1 MB `deThread` now asks for is not headroom, it is the difference
  between running and not.
* `src/amd/common/orbis_compat.c` is **deleted**: 64 lines, of which 62 were comment and two were
  `#include`s, defining no symbols, compiled into an empty object on every build. The text is now
  orbis-compat's README §2.1.1 - it remains the best writeup of the half-wired C11 threads layer.
* `-Dshader-cache=disabled` is in the orbis options, so `disk_cache` never runs here and the
  uncorrected `struct stat` on Mesa's side does not bite.

## Where to pick up

1. **Finish run A's list.** More host heap, or a smaller set. Its abort is a separate ceiling from
   the futex question and is not understood.
2. **Move the load-bearing files** into orbis-compat: `sys/umtx.h`, then OpenGothic's
   `og_ps4_{stat,mmap,mem,paths}`. Move without editing - a move that also changes code cannot be
   bisected - then one console run of each binary to prove nothing changed.
3. **Wire the four builds** to the overlay, and delete the CTS patches it takes over.
4. `__PS4__` → `__ORBIS__` across 27 files. Clang defines `__ORBIS__` and `__SCE__` for its own PS4
   triple and does **not** define `__PS4__`; ours is invented. Do it once, while the files are moving.
5. Only then upstream, headers first.

---

# Handoff — orbis-compat reaches the console, 2026-08-19 (evening)

All four builds now compile against `~/src/forks/orbis-compat`, and the title has run on hardware
with it: 172 presents, four pthread types eight bytes wide in every binary, `backtrace()` linked in.

## Three things this cost, all worth remembering

⚠ **The port's env file is configuration, not options.** The first console run crashed entering 3D.
The overlay was not the cause: the env file had been cut to "only what this run needs", which dropped
`ORBIS_3D_LINEAR=1` and `ORBIS_NO_TESS=1`. Both are OFF in the driver by default -
`ac_surface.c:1684`, `radv_physical_device.c:1079` - and this file is the only thing that turns them
on. Same binary, restored env, no crash. The minimal-env rule is about DIAGNOSTIC knobs.

⚠ **The build date is not proof of which package is installed.** `build.sh` runs meson through
`nix develop`, which sets `SOURCE_DATE_EPOCH=315532800`, so every driver it builds reports
`arm built Jan 1 1980 00:00:00`. A build made by calling `ninja` directly reports the real date -
which is why the discriminator seemed to work earlier the same day, and why it then silently stopped.
`-Dradv-build-id=<string>` already exists and takes anything.

⚠ **`git checkout -- <file>` on an uncommitted file destroys it.** The PS4 block in OpenGothic's
CMakeLists.txt was never committed; reverting a one-line edit of mine took all 209 lines with it. It
was reconstructed from `build-support/orbis/title/opengothic-ps4.patch` plus the build directory's own
`build.make`, adapted (the patch predates the move out of the patch queue) and trimmed of the GNM
shader-bake apparatus this branch does not carry. It builds, compiles the same eight sources, and
produces a package of the same size - but it is a reconstruction, and comments nobody wrote down are
gone. **The same lesson as the fog-LUT patch, six days later.**

## What moved, and what refused to

    sys/umtx.h                   moved. Mesa rebuilt, futex.c still inlines our implementation
    six SDK-gap headers          copied; check.sh fails if the two copies drift, and the check
                                 removes itself when mesa-ps4/build-support/orbis/shims goes
    deMutexUnix.c padding        DELETED from the CTS - the overlay corrects the type, and the
                                 self-retiring DE_STATIC_ASSERT fired exactly as designed
    qpCrashHandler.c guards      DELETED - the overlay supplies execinfo.h
    og_ps4_{stat,mmap,mem,paths} REFUSED. See below.

⚠ **The four OpenGothic interposers are one knot.** `og_ps4_stat` looks self-contained by its
`#include` list and calls `Ps4Og::anchorPath` from `og_ps4_paths`, which calls `ps4_log` from the
Tempest fork - the only thing the other three take from it either. Moving stat alone put an object
with an unresolved symbol into the archive, and `--whole-archive` then broke CMake's compiler test in
every project using the toolchain file. Unpicking it means making `ps4_log` a hook the overlay
declares and Tempest registers: an edit, not a move.

## Still open on the overlay

* One GPU stall survived the working run - submit #1327 of 465+, fence stuck for four submissions,
  every address mapped. **Not established as new**: no run of the same scene without the overlay
  exists to compare against.
* No LICENSE. `.gitignore` added (build/). Nothing committed anywhere in the forks.
* The six copied headers are still duplicated in this tree.

---

# Handoff — where every tree stands, 2026-08-19 (close)

## State by repository

    mesa-ps4      COMMITTED. 6 commits today; build.sh and the cross file require orbis-compat
    orbis-compat  COMMITTED. One commit, 667d654, 27 files - the whole thing amended into it
    Tempest       UNCOMMITTED - cmake/ps4-openorbis.cmake, ps4/common/ps4_app.cpp
    OpenGothic    UNCOMMITTED - CMakeLists.txt (209 reconstructed lines), ps4/, 4 files deleted
    VK-GL-CTS     UNCOMMITTED - 8 files / 481 lines, on branch `orbis` at tag vulkan-cts-1.4.6.1

⚠ **Three of the five carry their half of the wiring loose.** That is exactly the state the 209
lines were lost from this morning.

## What orbis-compat is, now that it is finished for this stage

`~/src/forks/orbis-compat`, MIT, one commit. Corrects four pthread attribute types (measured), ships
`sys/umtx.h`, `execinfo.h` over the unwinder libc++ already carries, six headers the SDK omits, and
the four interposers moved out of OpenGothic: `orbis_{stat,mmap,mem,paths}`.

`./build.sh` builds and checks; `--no-check` skips, `--define NAME=VALUE` reaches a documented knob.
No `check.sh` any more - the build is the compile check, and only what compiling cannot tell you runs
after it.

Nothing carries an engine's name: `namespace orbis`, `ORBIS_MMAP_DIRECT`, and `orbis_log.h` as the
hook the interposers call instead of Tempest's `ps4_log` - which is the one dependency that had made
them unmovable. Tempest registers `ps4_vlog` into it in `ps4_app_init`.

All four builds consume it and two of them REFUSE to build without it.

## Verified on hardware today

* the title builds against the overlay, boots and plays - 157 and 172 presents in two runs
* four pthread types write 8 bytes into 4-byte declarations; `pthread_once` writes 1 and needs nothing
* `libkernel.so` exports a working `_umtx_op`, and our pthread implementation is still the better one:
  49/49 with ours, aborted at 15/49 with Sony's

## NOT verified

* ⚠ the renames (`namespace orbis`, `ORBIS_MMAP_DIRECT`) have **not** been on the console. Mechanical,
  but three regex passes today turned out not to be.
* `__mmap`/`__munmap`/`__madvise` are local (`t`) in the linked title; no pre-move binary survives to
  compare against.
* one GPU stall per ~1200 submits - pre-existing, agreed to leave.

## Next, in order

1. **Commit the three forks.** Nothing else on this list matters if they are lost.
2. One console run of the renamed build.
3. ~~`src/orbis_mmap.cpp:296` - `mapOwned` is unused~~ **WRONG, and checked before acting on it.**
   `mapOwned` is called from `orbis_mmap.cpp:374`, and a clean rebuild (`rm -rf build && ./build.sh`)
   emits no warning at all. The diagnosis was wrong, not the code. Nothing to do here.
4. CMake for orbis-compat: an exported `orbis::compat` target would replace the hand-spliced paths in
   Tempest's toolchain file. ⚠ Keep the checks in build.sh - two of them need a different toolchain
   (they run host binaries) and one has to fail to compile. And drop the toolchain file's requirement
   that the ARCHIVE exist, or the overlay cannot build with it.
5. Fork Mesa somewhere. 245 commits still exist on one disk only.
6. `__PS4__` -> `__ORBIS__`, 27 files. Clang defines `__ORBIS__` for its own PS4 triple and does not
   define `__PS4__`.

---

# Handoff — the overlay closes four SDK gaps, and the renames finally ran, 2026-08-19 (late)

Two commits in orbis-compat (`2836985`, `28f6c56`) and two here (`c71e43f242a`, `cba487fbb9b`).
Everything below was built, and the title ran for seven minutes on the console with all of it.

## What the overlay gained

    include/{stdlib,signal,errno}.h    three names missing from headers the SDK already ships
    include/orbis_prefix.h            replaces -include stdlib.h, measured rather than argued
    cmake/orbis-compat.cmake          locate / orbis::compat / verify, for the CMake consumers
    test/declarations.c               compiles all three names the way their consumers write them
    LICENSE headers                   1 file of 23 carried SPDX; all of them do now

`malloc_usable_size` is declared in `<malloc.h>` only, while FreeBSD puts it in `<stdlib.h>` and
clang defines `__FreeBSD__` for this triple - that is the whole of dEQP's `deMemory.c` patch.
`sigev_notify_function` was never a missing FIELD: `signal.h:144` is FreeBSD's union, and only the
two macros every caller writes were absent. `ENODATA` is absent as on FreeBSD, and is now defined to
`ECONNREFUSED` - which is what Mesa itself picks for FreeBSD in `radv_amdgpu_cs.c`, and 61 in this
SDK's FreeBSD-numbered errno table.

⚠ **`sigev_notify_function` makes code compile and proves nothing about delivery.** `timer_create`
is real here (0x1a1 bytes in `libc.a`, calling `ktimer_create`), but nobody has measured whether
`SIGEV_THREAD` spawns anything on this kernel. **The CTS's `deTimer.c` patch stays until someone
does**: a timer that never fires is worse than one that does not build.

## The prefix header, and the seven that no prefix reaches

The `<orbis/>` headers name `size_t` and the stdint types without including them. Measured across all
189 of them:

    no prefix                    26 fail to compile alone
    -include stdlib.h            16      <- what the port passed everywhere
    -include orbis_prefix.h       7

Better coverage from a smaller injection - two type headers instead of a whole libc one, which
matters because `stdlib.h` at global scope in every TU is how this port once got an integer-only
`std::abs` truncating floats at 37 call sites.

**The remaining seven are SDK bugs**, and they are the shortest upstream list this port has:
`JpegEnc.h:17` writes `OrbisJpgEncOutputInfo` for `OrbisJpegEncOutputInfo`; `SysCore.h:27` names an
undefined `OrbisAppInfo`; `libc.h:352` and `LibcInternal.h:432` redeclare clang builtins
(`__sync_fetch_and_add_16` and neighbours); `Font.h`, `FontFt.h` and `Usbd.h` complete it.

## The six duplicated headers are gone from this tree

`build-support/orbis/shims/` is deleted. The copies in orbis-compat are the only ones, the cross file
already put its include directory first, and orbis-compat's drift check retired itself as designed.
Two things fell out:

* ⚠ `tools/umtxcheck.c` had been **uncompilable since `sys/umtx.h` moved out of shims**. Only
  `--host-orbis` builds it, so nothing ran it. Its `-I` names the overlay now, and it passes.
* the cross file listed `@ORBIS_CROSS@/include` as a system include path holding nothing. Dropped.

## Verified on the console, 15:44:46 → 15:51:51

`build-ps4-logs/ps4-udp-20260819-154435.log` (netlog) and `/data/mesa-knot.log` (driver).

    mem mmap-direct [boot]: ACTIVE - direct held 131072 KiB in 1 carve-out(s)   orbis_mmap.cpp:347
    TEMPEST_PS4_MMAP_DIRECT                                                     0 occurrences
    vdf probe - VERDICT: stat interposition ACTIVE
    vdf probe - VERDICT: mmap agrees with read()
    ctype probe - VERDICT: case folding works
    crash handlers installed (sigaction rc=0)
    0 anomaly(s)

**So the renames ran** - `namespace orbis` and `ORBIS_MMAP_DIRECT` - which was the one item on the
previous list that needed hardware. All four interposers say after the move exactly what they said
before it, and `getcwd` still fails with errno 78, which is why `orbis_paths` exists.

⚠ One GPU stall, submit #1286: the fence label stuck at 1285 for four submissions while five were
queued, `0/16 DMA_DATA addresses NOT in any live mapping`. **Pre-existing** - yesterday it was #1327,
about one in 1200 - and the title played on through it.

## Three things I got wrong today, all caught by looking rather than by thinking

1. **`mapOwned` is not unused.** The previous list said it was; `orbis_mmap.cpp:374` calls it, and a
   clean rebuild emits no warning at all. The diagnosis was wrong, not the code.
2. **"The SDK's headers are self-contained now" was measured on the wrong set** - `include/*.h`, 93
   files, when the claim in Tempest's toolchain file names `orbis/Net.h`. Sweeping `include/orbis/**`
   is what produced the table above. The comment being corrected was right.
3. **The netlog receiver WAS running**; `pgrep` was given `nc -u|netlog|socat` and the receiver is
   python. The evidence was in `build-ps4-logs/`, where it always is. *A search that does not find
   something is not a finding.*

## Next, in order

1. **Commit the three forks.** Unchanged from this morning, and still the only irreversible item.
2. Delete the CTS's `deMemory.c` patch - the overlay covers it. `deTimer.c` STAYS (see above).
3. Point Tempest's toolchain file at `cmake/orbis-compat.cmake` and at `-include orbis_prefix.h`.
   ⚠ Do it after (1), and remember `-lc` must stay in `CMAKE_C_STANDARD_LIBRARIES`: ahead of the
   overlay it is a `duplicate symbol: __mmap` link error, measured.
4. Fork Mesa. 245 commits still exist on one disk only.
5. `__PS4__` -> `__ORBIS__`, 27 files.
6. Upstream to OpenOrbis, headers first, with the seven broken `<orbis/>` headers as their own PR.

---

# Handoff — plan item 1: the thread stack, 2026-08-19 (evening)

`orbis-compat` `89b5df0`. The overlay now interposes `pthread_create`, and the plan it belongs to
lives in `orbis-compat/PLAN.md` - eight items, in order, each with the condition that ends it. That
file exists because this work kept growing sideways mid-task.

## What was wrong

`scePthreadCreate` gives every thread **64 KiB**. A dEQP worker died inside
`radv_graphics_shaders_compile` with ~72 KB of frame under it, and **every thread that compiles a
pipeline on this console has that cliff** - the title survives only because Tempest sizes its own.

`pthread_create` is UNDEFINED in every member of the SDK's `libc.a` and resolves from
`libkernel.so:0xda78`, so the number is Sony's rather than the libc's.

```
                 main thread    other threads
Linux / glibc        8 MiB          8 MiB      both from RLIMIT_STACK (measured on the laptop)
FreeBSD / libthr     8 MiB          2 MiB      THR_STACK_DEFAULT = sizeof(void*)/4 MB
musl                 8 MiB        128 KiB      small on purpose
PS4, as shipped      2 MiB         64 KiB      measured on the console
PS4, with this       2 MiB          2 MiB
```

## The probe came first, and both answers changed the design

```
a fresh attr claims       65536 B     "asked for the default" == "asked for nothing", indistinguishable
the main thread has     2097152 B     the platform DOES know a sane number, and applies it once
attr = NULL                65536 B
default-init attr          65536 B    the same, so ONE policy rather than two
```

⚠ dEQP passes an attr rather than NULL, so an interposer catching only the NULL case would not have
helped the thing that actually failed. That was not obvious before the measurement.

## The policy, and why it is not a number we invented

**A thread that did not choose gets what the main thread has**, read at runtime. That is what glibc
does with `RLIMIT_STACK`. `ORBIS_THREAD_STACK=<KiB>` overrides it, `0` disables the interposer, so an
A/B needs no rebuild. `detachstate` and `guardsize` are carried onto a copy - the caller's attr is
`const` and may be reused for other threads.

Any failure - `attr_init`, `setstacksize`, or a floor below `PTHREAD_STACK_MIN` - hands the original
attr through untouched. **A broken interposer must behave like an absent one.**

## Confirmed on the console

The probe asks `pthread_attr_get_np` about **live threads**, so this is what the kernel gave rather
than what the overlay intended:

```
attr = NULL             2097152 B      was 65536
default-init attr       2097152 B      was 65536
cost                    7936 KiB of address space, under 8 threads in the whole run
failures                none
```

`pthread_attr_get_np` had to be declared in the overlay's `pthread_np.h`: libkernel EXPORTS it
(0xd88e) and nothing declares it, so there was no prototype with which to ask a running thread about
its stack. Without it every answer here would have been a fresh attr's opinion.

## Left open, and it is not ours to close

⚠ **The CTS's `deThreadUnix.c` patch should now be deleted** - the overlay covers it. VK-GL-CTS is
still uncommitted, so it waits with the rest of the consumer wiring (PLAN.md §5).

Next: PLAN.md §2, whether `SIGEV_THREAD` actually delivers on this kernel.

---

# Handoff — plan items 1-5 done, 2026-08-19 (night)

`orbis-compat/PLAN.md` is the list; five of its nine items are closed, all confirmed on the console.
This section records only what a future reader needs that the plan does not repeat.

## Where the title is built from NOW

    ~/.cache/opengothic-ps4/build       STALE. Its CMake cache holds August's flags.
    ~/.cache/opengothic-ps4/build-tc    CURRENT. Configured from scratch today.

⚠ **A green build in the old directory proves nothing about a toolchain-file change.**
`CMAKE_<LANG>_FLAGS_INIT` applies at FIRST configure only, so an edited toolchain file leaves
`flags.make` exactly as it was. The change to `-include orbis_prefix.h` built clean in `build/` while
`build/` was still compiling with `-include stdlib.h`. **Check the generated flags, not the exit
code.**

⚠ **A from-scratch configure needs `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`** - doctest's
`cmake_minimum_required` predates CMake 4. Until today, the working build directory could not be
reproduced from scratch at all, and nobody knew.

## What the overlay gained today, beyond the plan items

    include/{stdlib,signal,errno}.h   three names missing from headers the SDK ships
    include/orbis_prefix.h            replaces -include stdlib.h; 16 broken orbis/ headers -> 7
    include/orbis_thread.h            the pthread_create stack floor, and the probe that set it
    include/orbis_timer.h             the SIGEV_THREAD probe, off by default because it HANGS
    include/orbis_boot.h              ctype probe + crash handlers, moved out of OpenGothic
    cmake/orbis-compat.cmake          locate / orbis::compat / verify
    signal.h's sa_sigaction           the SDK's macro is one underscore pair short

## The four SDK defects this port now corrects, in one place

    pthread attribute types    four of them declared smaller than the kernel writes
    struct stat                mode_t is 4 bytes where the kernel writes 2 - and CANNOT be fixed
                               locally, because prebuilt libc++ reads st_size at the wide offset
    sa_sigaction               a macro that names a member the union does not have
    the default thread stack   64 KiB, Sony's own number, against a 72 KB compile frame

## Two measurements that are worth more than the code they justify

* **`timer_create` with `SIGEV_THREAD` never returns.** Not an error, not a silent timer. A
  `SIGEV_NONE` control in the same probe proves the kernel's timers do run, and `CLOCK_MONOTONIC` is
  refused while `CLOCK_REALTIME` is accepted. The CTS's `deTimer.c` patch is permanent, and its
  comment now cites this instead of claiming the struct lacks a field it has.
* **The thread-stack default is Sony's**, not the platform family's: `pthread_create` is undefined in
  every member of `libc.a` and comes from `libkernel.so:0xda78`. FreeBSD would give 2 MB, musl
  128 KiB. The overlay gives what the main thread has, which is what glibc does with `RLIMIT_STACK`.

## Still not committed anywhere but here

⚠ Tempest, OpenGothic and VK-GL-CTS all carry today's work uncommitted. The CTS's is now readable -
six files, and `build-umtx/` (9164 files, 2.3 GB) has been taken out of its index with `build*/`
added to `.gitignore`. The other two have not been tidied.

Backups of every file edited in an uncommitted tree today:
`build-support/orbis/title/backup/` and `.../backup/cts/`.

---

# Handoff — the overlay becomes the platform, 2026-08-19/20

Two days in one entry, because they were one piece of work: `orbis-compat` stopped being a header
shim and became the place this port keeps everything that is about the PLATFORM rather than about a
game, a driver or a test suite. Nine plan items, four repositories rewired, one pushed.

## Where everything lives now

    ~/src-ps4/orbis-compat     f627246   committed, clean
    ~/src-ps4/Tempest          4ccbc71d  29 paths pending - his to commit
    ~/src-ps4/OpenGothic       14b05a0b  13 paths pending - his to commit
    ~/src-ps4/VK-GL-CTS        cac71bbda PUSHED to origin/ps4-support
    ~/src/mesa-ps4             429fc9a…  5 paths pending

⚠ **The workspace moved from `~/src/forks` to `~/src-ps4`**, so that orbis-compat can one day ship a
script that puts a fresh clone in the right place. `~/src/mesa-ps4` did NOT move: this session's shell
was rooted there and moving it would have cut the session off mid-work. One `mv` plus three path
edits, whenever convenient.

⚠ **Two absolute symlinks had to be repointed**: `OpenGothic/lib/Tempest` and `lib/ZenKit`. Without
that the title would have built a DIFFERENT Tempest and nothing would have said so.

⚠ **`git status` in OpenGothic errors** - "expected submodule path 'lib/Tempest' not to be a symbolic
link" - and returns nothing, which reads as a clean tree. It is not clean; it has 13 paths pending.
Use `git -c submodule.recurse=false status --ignore-submodules=all`.

## What orbis-compat contains, 56 files, 15732 lines

    include/           bits/alltypes.h  execinfo.h  errno.h  signal.h  stdlib.h  orbis_prefix.h
                       sys/{umtx,cpuset,ioccom,param,sysctl}.h  machine/cpu.h  pthread_np.h
                       orbis_{log,mem,mmap,paths,stat,thread,timer,boot,netlog}.h
    src/               the archive: interposers, backtrace, log hook, thread floor, SIGEV_THREAD
    optional/          NOT in the archive: bigheap, thread_atexit stub, netlog
    vkloader/          the Vulkan thunk mechanism + 771 weak thunks, generated once
    cmake/             the PS4 toolchain file, the packaging rules, orbis-compat.cmake
    scripts/ps4/       make-pkg.sh, gen-icon0.py, log-receiver.py, logs.sh, peerfilter.py
    test/              sizes.c, declarations.c, and two host tests

⚠ **`optional/` is not in `liborbis-compat.a` and must not be.** Every consumer links the archive
with `--whole-archive`; those three files are POLICY - an unlimited heap that competes with the
driver's arena, a thread_local stub that leaks by design, and a logger that obliges the consumer to
link `-lSceNet`. A consumer adds them by name, having read why.

## The measurements, which are the part worth keeping

    default thread stack     65536 B, and it is SONY'S: pthread_create is UNDEFINED in every member
                             of libc.a and resolves from libkernel.so:0xda78. FreeBSD would give
                             2 MB, musl 128 KiB. We now give what the main thread has - measured at
                             runtime, which is what glibc does with RLIMIT_STACK.
    SIGEV_THREAD             timer_create NEVER RETURNS. Not an error, not a silent timer. A
                             SIGEV_NONE control in the same probe proves the kernel's timers do run,
                             and CLOCK_MONOTONIC is refused while CLOCK_REALTIME is accepted.
    struct stat              ONE typedef explains the whole shear: FreeBSD's mode_t is uint16_t and
                             musl's is unsigned int. ⚠ And it CANNOT be fixed locally - prebuilt
                             libc++ reads st_size at the wide offset and is right today.
    __mmap LOCAL binding     correct, not suspicious: 11 members of libc.a reference it as
                             GLOBAL HIDDEN UND, and a hidden symbol is emitted local.
    the include prefix       measured over all 189 orbis/ headers: none 26 fail, -include stdlib.h
                             16, -include orbis_prefix.h 7. The remaining seven are SDK bugs.
    the thunk table          dEQP references FIVE vk* symbols and was carrying Tempest's 98.
                             Now 771 weak thunks generated from the header, shared by everyone.

## What each consumer stopped carrying

    Tempest       13 files deleted: vkloader.{c,h}, vkthunks.c, gen.py, both cmake files,
                  make-pkg.sh, gen-icon0.py, ps4_netlog.{cpp,h}, and three receiver scripts
    VK-GL-CTS     deMemory.c and deThreadUnix.c reverted to UPSTREAM; deTimer.c's patch kept but
                  its comment now cites the measurement instead of claiming a missing field
    mesa-ps4      the six duplicated SDK headers, and orbisCtsRuntime.c

## Five mistakes worth more than the fixes

1. ⚠ **A green build that tested nothing, twice.** `CMAKE_<LANG>_FLAGS_INIT` applies at FIRST
   configure only, so an edited toolchain file left an existing build directory compiling exactly as
   before. Checking `flags.make` rather than the exit code is what caught it.
2. ⚠ **A probe with no way out.** The SIGEV_THREAD probe was shipped on the title's boot path with
   no knob, and hung the console twice. The knob was added AFTER the black screen.
3. ⚠ **Symbols that vanished from a linking binary.** Splitting orbisCtsRuntime.c removed the one
   referenced symbol that was pulling the heap variables in; they disappeared and nothing failed.
   `-u` now names them in the build system. Found by diffing readelf against the previous build.
4. ⚠ **Moving a caller without its callee.** Twice: vkloader's generated file, and make-pkg.sh.
   Both surfaced as a failure in a step nobody was watching.
5. ⚠ **"I could not find it" reported as "it is not there."** A netlog capture went to the directory
   the receiver was run from, and the transport was briefly blamed. The same shape as the day
   before, when a receiver that WAS running was declared absent. `logs.sh` now writes beside itself.

## Next

1. Commit Tempest and OpenGothic. They are the last two carrying work that exists nowhere else.
2. Move `~/src/mesa-ps4` to `~/src-ps4`, and fix the three references to it.
3. `mesa-ps4/build-support/orbis/cts/` still holds drifted copies of `tcuOrbisPlatform.{cpp,hpp}`
   and `orbis.cmake`. The CTS fork's are newer; mesa's carry longer comments. Decide: archive or bin.
4. PLAN.md §6 (CI) and §8 (upstream) remain. §8 has no date by decision.

# Handoff — the day the comments were audited, 2026-08-20

One long session of tidying, and it stopped being tidying about an hour in. Every question the
maintainer asked — "why is this needed?", "can't we keep one branch?", "what does this do?" — turned
up something that was not merely verbose but WRONG: a guard naming a file that had moved, a log line
pointing at a counter that no longer exists, a version number that had drifted, a preprocessor arm
that could never have linked. Six such findings, none of which any build or any console run would
ever have reported.

## What moved into the overlay

    orbis-compat/include/ps4_app.h          the console log channel's API
    orbis-compat/optional/ps4_app.cpp       + optional/CMakeLists.txt: ps4-netlog, ps4-app
    orbis-compat/vkloader/CMakeLists.txt    the loader target, beside the sources it names
    orbis-compat/include/orbis_clock.h      the clock_gettime interposer's contract
    orbis-compat/src/orbis_clock.cpp        the interposer AND its falsifier, in the archive

⚠ **ps4_app had ZERO Tempest includes.** orbis_log.h, orbis_netlog.h, orbis/libkernel.h, orbis/Net.h,
nothing else. It sat in the Tempest fork purely by history - its own CMakeLists still spoke of "demo
apps", ps4doom and unemups4. It is policy, not correction, so it went to `optional/` with the netlog:
a consumer adds it by name and accepts `-lSceNet` and "a dying process idles rather than returns".

⚠ **The clock was the one that mattered, and it was in the wrong repository.** `ps4clock.cpp`
interposes `clock_gettime` because the SDK ships the LINUX clock-id table while `clock_gettime` is
libkernel's entry into a FreeBSD kernel: id 1 is CLOCK_MONOTONIC to libc++ and CLOCK_VIRTUAL to the
kernel, so the "monotonic" clock returned process CPU time and ran at 14.96% of real time.

MEASURED, and this is the finding: the CTS did not have it.

    title (links Tempest):   clock_gettime  FUNC GLOBAL  section 1   <- our definition
    deqp-vk (does not):      clock_gettime  FUNC GLOBAL  UND         <- libkernel's, broken

Three places in RADV stand on that clock - `os_time_get_nano` (via `timespec_get(TIME_MONOTONIC)` ->
`c23_timespec_get` -> `clock_gettime`), `util/futex.c`'s deadlines, and `vk_device.c`'s
`VK_KHR_calibrated_timestamps`, which is a Vulkan API surface dEQP tests - plus the overlay's own
`sys/umtx.h`. So every futex deadline in the CTS was computed against a clock running at ~15% speed.
Moving the file into the archive gives the CTS both the fix and its falsifier.

The probe now names its own verdict rather than printing ratios and leaving the reader to judge.
Confirmed on hardware, 11:33: `VERDICT the monotonic clock is WALL TIME`, busy 1.0003, asleep 1.0002.
The asleep window is the whole test - a CPU-time clock stops dead across it.

## Six things that were dead and compiled anyway

1. ⚠ **`OpenGothic/ps4/build.sh` was broken in THREE places.** Its guard checked
   `lib/Tempest/ps4/vkloader/vkloader.h`, deleted the day before; its toolchain path named
   `lib/Tempest/cmake/`, which is now an EMPTY DIRECTORY; and the comment explained both. Every run
   exited 1 before compiling anything. Not noticed because the session had been configuring CMake
   directly. **A guard that names a moved file is worse than no guard: it fails correct builds.**
2. ⚠ **`logs.sh` used `$HERE` two lines above its definition** - my own fix from the day before,
   written and never run. `set -u` refused, the script died before binding the socket, and the
   maintainer found it by running it.
3. ⚠ **`ps4_capture.{cpp,h}` had no callers anywhere** - 399 lines, and its receiver
   (`recv-framebuffer.py`) and harness (`run-tests.sh`) had already been deleted. The proof it was
   dead is better than a grep: **the symbol table was IDENTICAL before and after removal**, 55198 =
   55198. An archive member nobody references was never being pulled in.
4. ⚠ **A log line sent the reader to a diagnostic that does not exist.** `og_ps4_boot.cpp` printed
   "see patch 0004's mount summary, which reports entries dropped for lying outside the mapped
   buffer" - the ZenKit fork carries no such summary, checked across the whole tree. So a short
   archive mapping is silent at BOTH ends. The log now says so.
5. ⚠ **`RADV advertises 1.3.358` was wrong, in two repositories.** `VK_HEADER_VERSION` is 359;
   `RADV_API_VERSION_1_3` is `VK_MAKE_VERSION(1,3,VK_HEADER_VERSION)`; Liverpool sits before the GFX8
   block in `amd_family.h` so it takes the 1.3 arm. Not corrected to 359 - the number tracks a header
   and had already rotted once. Both comments now point at where it is decided.
6. ⚠ **The whole GNM tail, six items, none of which reached the binary.** `TEMPEST_BUILD_GNM` and
   `TEMPEST_PS4_VULKAN` forced into the cache with no reader anywhere; a `#else` arm calling
   `GnmApi::setLogSink` when `class GnmApi` does not exist in the fork; `gnm_register_shader_archive`
   declared and called and **defined nowhere - that arm would not have LINKED**; 28 lines of
   CMakeLists comment describing a shader bake, followed directly by `# zip`, with no code and no
   `cmake/gnm-shaders.cmake`; and finally `PS4_RADV` itself, a flag separating two backends when one
   remains. Removing all of it changed the section sizes by zero bytes: text 46994234, data 712205,
   bss 17595808, before and after.

## Dead cross-references, swept

    task-NN     47 references, 21 distinct numbers, 4 repositories - no tracker has them
    patch 00NN  10 references - both patch queues (Mesa's, OpenGothic's) are gone

⚠ **Five of the 47 were RUNTIME STRINGS, not comments** - `ps4api.cpp`'s pad-focus line,
`orbis_paths.cpp`'s relative-path error, `og_sound_orbis.cpp`'s resample notice, `og_ps4_boot.cpp`'s
save-path banner, `peerfilter.py`'s `--allow-from` help. They were reaching the maintainer's screen
on every run.

The rule applied was: keep the FACT, drop the pointer. `but task-51 froze this console with one wrong
memory call` became `but one wrong memory call has frozen this console before`; `(task-58, console
run 2026-08-04 15:50)` kept the date and lost the number, because the date is the evidence.

Still outstanding, needing a read of each site: 26 files reference eight things that no longer exist
- `unemups4` x10, `ps4/ime` x4, `ps4/radv` x3, `ps4doom` x3, `gnmprof` x2, `ps4/audio` x2,
`ps4/runner`, `AMDLLPC`. Some is honest history of a measurement; some is an instruction that now
misleads.

## Comment volume in the PS4 files

    ps4api.cpp   619 -> 536 lines    comment 197 -> 114   (31% -> 21%)
    ps4pad.h     149 -> 117 lines    comment  74 ->  42   (49% -> 35%)
    ps4pad.cpp   388 -> 383 lines    comment  35 ->  30
                                     total  306 -> 186   (-39%)

Cut: the narrative of how a fact was reached, and every `menuroot.cpp:129`-style line number into
another repository - those rot on the first commit upstream. Kept: LEDGERs that explain a decision
VISIBLE in the code (`walkZone` defaults to 0 because a continuous quantity cannot drive a toggle by
its edges) - without them someone "fixes" it back.

An 8-line comment about the GNM profiler was attached to `#include "ps4pad.h"`, which has nothing to
do with GNM; the render-size comment cited `GnmSwapchain` and a 57ms/96ms measurement from before the
GARLIC split, which this file's own handoff says not to compare against.

## Engine/gapi is now purely additive

    before today:  4 files, 110 insertions, 2 deletions
    after:         2 files,  36 insertions, 0 deletions

Not one upstream line is changed or removed, so every merge from Try/Tempest passes untouched. Two
whitespace-only hunks were reverted for the same reason.

⚠ **The `#elif defined(__PS4__)` in vswapchain.cpp is EMPTY ON PURPOSE and must stay.** Its only job
is to keep the `#else`'s `#error "WSI is not implemented on this platform"` from firing; every other
arm defines `VK_USE_PLATFORM_*` and includes a window-system header, and headless needs neither
because `vkCreateHeadlessSurfaceEXT` is core `vulkan_core.h`. Its comment claimed the opposite of what
the file does 180 lines below - "createSurface returns VK_NULL_HANDLE, presentation support is false"
against `vkCreateHeadlessSurfaceEXT(...)` and `presentSupport = true`.

The `#if/#else` around `rqExt` was deleted outright: it existed to skip `VK_EXT_debug_report`, and its
own comment admitted RADV advertises it. `.EXT_debug_report = true` in `radv_instance.c` sits outside
every `#ifdef`. Confirmed on hardware - `stage: mkApi` passed and the GPU came up.

## Where every tree stands

    ~/src-ps4/orbis-compat   0aab8cb    62 files, 16583 lines, clean
    ~/src-ps4/Tempest        4ccbc71d   21 paths pending - his to commit
    ~/src-ps4/OpenGothic     14b05a0b   14 paths pending - his to commit
    ~/src-ps4/VK-GL-CTS      cac71bbda  pushed, clean
    ~/src-ps4/ZenKit         39bf134    clean, and carries none of our changes
    ~/src/mesa-ps4           f1e1d2c26  clean

Tempest now carries ONLY `Engine/`: the Orbis `SystemApi` backend (window, pad, events) and two
Vulkan-backend files. `Tempest/ps4/` is gone entirely; `.gitignore` is upstream's again, with
`.direnv/`/`__pycache__/` moved to `.git/info/exclude` - which had to be CREATED, it did not exist.

## The shape all six findings share

Text beside code has no test. Code that stops being true stops compiling, or dies on the console.
A SENTENCE about code can be false for months and nothing reports it - and all six read perfectly
well. They were found by one question each, and the question was always some form of "why is this
here?"

The only instrument that catches them is reading the code beside the comment rather than the comment.

## Next

1. Commit Tempest and OpenGothic. Still the last two carrying work that exists nowhere else.
2. Run the CTS. It has the clock interposer for the first time, and the probe will say in the log
   whether it arrived. Every previous CTS result was measured against a 15% clock.
3. Decide the 26 files referencing eight things that no longer exist.
4. `mesa-ps4/src/amd/common/ac_orbis_drm.c` and `radv_orbis_winsys.c` still carry `task-NN`. They are
   written for upstream, where nobody knows what task-58 was.
5. Move `~/src/mesa-ps4` to `~/src-ps4`, and PLAN.md 6/8, both still open from yesterday.

# Handoff — OpenGothic's diff, split three ways, 2026-08-20 (evening)

The morning's tidying became a question: of the 3770 insertions this port adds to OpenGothic, how
many are actually about the PlayStation 4? The answer is fewer than it looked. Eight lines belong
UPSTREAM and are now four pull requests; ~40 could be deleted outright because the overlay already
did the job; and TWO changes turned out to fix nothing at all.

The plan is `docs/PLAN-opengothic-split.md`. This entry records what happened when it was worked.

## Four pull requests, all open, all one commit on current upstream/master

    892d8ec1  fix(sound): memset sizes the silence buffer with the wrong element type
    77fad74d  fix: removed unused Read::from
    b5517494  fix(compatibility): include <unordered_map> where it is used     PR #968
    6338fd76  fix(compatibility): construct string_view by length              PR #969

None mentions the PS4. Each is defended by evidence a reviewer can reproduce in one command:

    memset        AL_FORMAT_STEREO16 -> {FmtStereo, FmtShort} (al/buffer.cpp:570) ->
                  sizeof(int16_t) (storage_formats.cpp:53). The buffer is numbytes; the
                  correct fill is numbytes; the code writes 2*numbytes. Arithmetic closes exactly.
    Read::from    mount_disk on the NEXT LINE opens the same path by the same mechanism, so the
                  same failures reach the same catches. 15 archives, 2 691 545 775 B, read and
                  discarded. ⚠ It halves the I/O, it does not remove it: mount_disk reads the same
                  bytes and KEEPS them in _m_data. The PR text says so.
    include       compiled each of the six headers alone against both libraries: <functional> is
                  the only one that provides unordered_map, and only on libstdc++.
    string_view   compiled both forms against the SDK's libc++: the iterator pair gives
                  "no matching constructor"; the length form compiles. C++17, same view.

## C1 — the thread-pool rewrite came out, and the console agreed

`workers.{cpp,h}` rewrote std::thread into pthread_create solely to ask for a 1 MB stack.
⚠ **The overlay's interposer has been raising every thread to the MAIN thread's 2048 KiB since it
landed**, so the explicit 1 MB was itself being raised. Hardware, before and after, identical:

    thread census [live]: 16 created, 16 raised to 2048 KiB

`workers.h` dropped out of the diff entirely; `workers.cpp` went 98 -> 69 insertions. Symbols moved
by exactly one: `Workers::trampoline` out, std::thread's proxy in.

## Two changes deleted, because the reason did not survive being asked for

⚠ **`sceneglobals.cpp`'s `float()` cast fixed nothing.** The claim was that `std::max` cannot deduce
from `std::log2`. Compiled the real translation unit with the real build flags, without the cast:
exit 0. The justification was never true.

⚠ **`ssao.comp`'s LDS array was measured under a compiler this branch no longer runs.** The 19.9 ms,
the 112 B of scratch and the four `-amdgpu-*` flags all came from amdllpc on the GNM path.
`radv_physical_device.c:2830` makes LLVM opt-in via RADV_DEBUG; shaders go through ACO. The word
`amdllpc` survived in exactly one place in both forks: our own comment in that shader.

Both are reverted. The title builds and links without either.

## The collective branch, and why it is shaped this way

    1d030e98*  feat: PlayStation 4 (Orbis) support        <- staged, NOT committed
    a967aaab   Merge branches 'fix/...' x4 into ps4-support
    6d2cb645   upstream/master

The four fixes arrive by MERGE from branches based on upstream, never baked into the PS4 commit.
When upstream takes one, the next merge from upstream absorbs it and the change leaves this diff by
itself. ⚠ Tempest's `251ed657` is one commit carrying 1203 insertions - and it contains an upstream
bug fix (vswapchain's `#else` had `#warning` but no `presentSupport` declaration). Extracting it now
means surgery. That is the cost this shape avoids.

⚠ **The commit is reverted to staged at the maintainer's request** - he wants to review first. Its
message is in `$CLAUDE_JOB_DIR/tmp/ps4-commit-message.txt`, the commit itself in
`refs/backup/ps4-commit-1d030e98`.

## Traps this cost

1. ⚠ **`git reset --hard` DESTROYS `lib/Tempest` and `lib/ZenKit`.** Upstream has gitlinks there;
   this fork has symlinks to the sibling forks. Caught by ps4/build.sh's guard - the one repaired
   that same morning for naming a moved file. Re-create both after any hard reset.
2. ⚠ **`git reset --soft <newbase>` does not move a branch's base.** It moves HEAD and leaves the
   index, so the staged diff became "revert all ten upstream commits, plus our change" - 47 files
   instead of one. Caught by comparing diffstat before and after rather than assuming.
3. ⚠ **`deqp-args.txt` had gone stale against the CTS.** Upstream deleted
   `--deqp-watchdog-{interval,total}-time-limit` and made them compile-time constants; an unknown
   option is not ignored, so the run ended before its first test having written only
   `/data/deqp-trace.txt`. The rule was "copy, do not retype" - that guards a typo, not rot.
   Check the args against `deqp-vk --help` after every CTS bump. Fixed and committed.
4. ⚠ **The CTS ran with the clock interposer for the first time** and passed 49/49 - but the good
   baseline was also 49/49 and predates the fix. It settles nothing about the ResourceError; that
   needs repeats.
5. ⚠ **`clockProbe` is in the CTS binary but nothing calls it.** The interposition works (section 1
   vs UND, measured), but the run does not state its own basis. One line in tcuOrbisPlatform.cpp.

## The pattern, stated once

Five times the maintainer asked "how do we know?" Three answers got stronger - a name became a
lookup, an unused variable became a redundancy, a guess became a table. Two changes did not survive
at all. Every one of the five would otherwise have reached a stranger's repository with a
justification that could be refuted in a minute.

Reading the code beside a comment catches comments that stopped being true. Nothing catches a
justification that was never true except being asked.

## Where every tree stands

    ~/src-ps4/orbis-compat   0aab8cb    master        clean
    ~/src-ps4/Tempest        251ed657   ps4-support   clean, pushed
    ~/src-ps4/OpenGothic     a967aaab   ps4-support   18 paths STAGED, his to review
    ~/src-ps4/VK-GL-CTS      cac71bbda  ps4-support   clean, pushed
    ~/src-ps4/ZenKit         39bf134    ps4-support   clean
    ~/src/mesa-ps4                      clean

Submodule pointers now match the forks: `lib/Tempest` 251ed657, `lib/ZenKit` 39bf1344. ⚠ Neither is
visible in `git diff` - `diff.ignoreSubmodules=all` is set repo-locally so that difit and plain
`git status` work at all past the symlink. Read them with `git ls-files -s lib/`.

## Next

1. He reviews the 18 staged paths; restore the commit from the saved message.
2. Push ps4-support.
3. The pad-disconnect fix from this morning's review is still unverified on hardware.
4. Review leftovers not acted on: L2/R2 masking most binds while held, strafe hysteresis,
   unvalidated cfg tunables (a negative deadzone reaches `int(NaN)`), and eight false comments -
   including one this session made false, about SceKeyboard being in NEEDED.
5. 26 files still reference eight things that no longer exist (`unemups4` x10, `ps4/ime` x4, ...).

# Handoff — the night eight console runs bisected noise, 2026-08-20/21

Tier 1 of the code review went in and was confirmed. Then the title stopped reaching the menu, and
the next five hours went into finding out why. Six hypotheses, eight console runs, every one of them
a hard hang that cost a reboot. **None of the six was the cause.** The cause was a one-argument
defect that had been sitting in `resources.cpp` since 2026-08-18 and had nothing to do with anything
edited that day.

The maintainer found it by asking "to my nie robimy lazy?" after the eighth run.

## The defect, and it is ours

    game/resources.cpp   inst->gothicAssets.mount_disk(i.name, VfsOverwriteBehavior::OLDER);

Two arguments. ZenKit's third defaults to `VfsMountMode::FULL`, which mmaps each archive WHOLE and
keeps the mapping alive in `Vfs::_m_data_mapped` for the life of the process. **PS4 mmap POPULATES
EAGERLY** - 37 s to touch three bytes of a 722 MB archive, measured weeks ago and written down. So
FULL faults ~2.69 GiB of Gothic II into a process with 387 MiB of flexible memory.

⚠ **We introduced the default ourselves.** ZenKit fork commit `39bf1344` (2026-08-18,
"feat(vfs): Support for lazy loading VDF files") added `VfsMountMode` and chose `FULL` as the
default. `resources.cpp` was never updated to ask for `LAZY`. The branch has been compiled in the
whole time - `_ZK_WITH_PREAD=1` in every build - and nothing ever called it.

Before that commit, the two-argument form did the pread thing unconditionally (patch
`0004-zenkit-console-safe-pread-mount.patch` in the old queue). The feature commit turned a
guarantee into an opt-in and nobody opted in.

### The fix and the numbers

    mount_disk(i.name, OLDER, zenkit::VfsMountMode::LAZY);   // PS4 only

    FULL, cold          console dies at Speech2.vdf - no pad, no log, reboot
    FULL, warm          20.2 s   (og-c1 at 22:50)
    FULL, very warm      1.9 s   (16:51 - an outlier, not the norm)
    LAZY                 0.90 s  <- faster than FULL's best case, all 15 archives

Speech1.vdf, 722 MB: **0.119 s**. Speech2.vdf, the one that killed the console: **0.129 s**.

LAZY does not read 2.69 GiB at all. It preads fifteen catalogs and keeps one shared fd per archive.

## ⚠ THE METHOD FAILURE, WHICH IS THE PART WORTH KEEPING

The failure was **not deterministic**, and every run was read as a verdict. Proof, measured after the
fact: `og-c1` (works, 50 fps) and `og-nolog` (hangs) were compared object by object.

    all Gothic2Notr.dir objects     identical, except main.cpp.obj (build-date stamp)
                                    and resources.cpp.obj (same size, same symbols, embedded path)
    libTempest.a, libzenkit.a,
    libps4-app.a, libps4-vkloader.a byte-identical
    libGothicShaders.a              differs ONLY by the source path inside each of 316 blobs
    symbol tables                   identical, both directions
    data / bss                      identical

**Two functionally identical binaries, opposite outcomes.** And the same binary took 1.9 s at 16:51
and 20.2 s at 22:50 for the same work. The outcome was never a property of the binary.

So the bisect could not converge: each "this change broke it" was noise, and each "this change is
innocent" was luck. Six eliminations were nevertheless real and are recorded below so nobody repeats
them - they were eliminated by comparing artefacts, not by running.

Written to memory as [[one-console-run-is-not-a-verdict]]. Same shape as
[[counts-hide-nondeterminism]] three days earlier: an unstable measurement read as a verdict.

**The rule that would have saved the night:** when something worked this morning and does not
tonight, re-run this morning's package FIRST. That test cost nothing, needed no build, and was run
eighth.

## Eliminated, each by measurement rather than by a run

    Tier 1                a control build without it hangs identically
    the octopus merge     14b05a0b + the PS4 layer hangs identically, same archive, same timings
    `Read::from` removal  restoring it did not help - and it DOUBLES the eager mmaps, so the
                          "fix" made things worse. The maintainer said so before the run.
    the mmap interposer   __mmap/__munmap/__madvise present in both; it only intercepts ANONYMOUS
                          mappings (isOwnedRequest), never a file-backed one
    ZenKit itself         Vfs.cc.obj byte-identical (same md5) in the working and hanging builds
    ZK_ENABLE_MMAP,
    ZK_ENABLE_LAZY_VFS    =ON in both; _ZK_WITH_PREAD=1 in both
    PS4_RADV              zero readers across all three forks - the removal was right

## Traps this cost, all new

1. ⚠ **`git stash push -- ps4/` DELETED the whole directory.** Those files are ADDED, not in HEAD,
   so "revert to HEAD" means remove. The 18 staged paths awaiting review vanished. Recovered from
   the stash plus the pre-work backup, and the index/worktree split restored from
   `staged-before-tier1.patch`. Same family as `reset --hard` eating the submodule symlinks: **a git
   command that "undoes" a file which does not exist in HEAD deletes it.**
2. ⚠ **`git add -A` rewrites `lib/Tempest` and `lib/ZenKit` from gitlinks (160000) to symlinks
   (120000).** Mode shows as `T`. Caught before the commit; restored with
   `git update-index --add --cacheinfo 160000,<sha>,lib/<name>`. `diff.ignoreSubmodules=all` hides
   this, so check `git ls-files -s lib/` after every `add -A`.
3. ⚠ **The staged ZenKit gitlink was WRONG.** It read `083cc5f5`; the fork is at `39bf1344`, which
   is the commit that introduces `VfsMountMode`. With the old pointer `resources.cpp` does not
   compile. A previous handoff claimed this had already been fixed. It had not.
4. ⚠ **A build directory outlives the tree it was configured against.** `~/.cache/opengothic-ps4`
   still pointed at `/home/mikolaj/src/forks/OpenGothic`, deleted in the move to `~/src-ps4`. Two
   hours went into section-size comparisons against binaries from a tree that no longer existed.
   Check `flags.make` for the source path before treating any old build as a control. That directory
   is now deleted.
5. **`PS4_APP_BUILD_DATE_STICKY` is sticky per build directory**, so a binary stamped "built 14:48"
   may have been linked at 16:50. It identifies the CONFIGURE, not the link.
6. **git history is not a record of what ran.** The PS4 layer has never been committed, and the four
   fix branches sat uncommitted in the working tree for days. `git show 14b05a0b` describes a tree
   that was never built. Worse: a worktree at that commit does not compile, because the two fixes it
   needs were also uncommitted.

## Tier 1 of the review: done, confirmed on hardware

Eleven items, +90 -192 across 7 files. Verified in the running binary, not from memory:

    1.1  concepts shim collision      16 lines, was 27 with the concepts duplicated under a
                                      DIFFERENT guard - any TU doing #include <concepts> failed
                                      with 21 redefinitions. Repro was one clang command.
    1.2  census under the audio lock  ps4_log -> ps4_log_frame (klog is 8-15 ms a line, held
                                      across `sync`, block period 5.33 ms)
    1.3  OG_FOG_DIAG                  43 lines of comment and a -D on every target, zero readers,
                                      and it fired OUTSIDE if(PS4) - the only leak past the guard
    1.4  probeLargestArchiveMap       ~70 lines dead; 1 symbol before, 0 after
    1.5  OrbisMixer::announceOnce     never called
    1.6  two adjacent contradictory   the census's own retraction vs the paragraph explaining why
         comments                     the census exists
    1.7  the first census line lied   "the mixer thread is not running", printed from the mixer
                                      thread, on every run. censusAt 0 -> 1. Confirmed absent.
    1.8  six one-liners               redundant disjunct, "NOT under the lock", "the three above",
                                      CONTENT_LABEL 17->16 chars (was silently truncated),
                                      bounded toUtf8, ADPCM blockAlign underflow guard
    1.9  six includes that outlived   `#include "../../Engine/sound/*.h"` from OpenGothic/ps4/
         their file                   points ABOVE the repository; it resolved only because
                                      -I<root>/lib/Tempest/Engine/include sits exactly two levels
                                      below lib/Tempest. Now <Tempest/SoundDevice> and neighbours.

⚠ **1.9 was found by the editor's diagnostics, not by reading.** It was not a comment that stopped
being true - it was code that compiled by a two-level coincidence. A comment audit could not have
caught it.

## State

    ~/src-ps4/orbis-compat   0aab8cb    master        clean
    ~/src-ps4/Tempest        251ed657   ps4-support   clean, pushed
    ~/src-ps4/OpenGothic     df495d3a   ps4-support   clean, NOT pushed, UNSIGNED
    ~/src-ps4/VK-GL-CTS      cac71bbda  ps4-support   clean, pushed
    ~/src-ps4/ZenKit         39bf1344   ps4-support   clean
    ~/src/mesa-ps4           0332c0c18e9              PLAN-opengothic-review.md uncommitted

`df495d3a` is a **throwaway safety commit**, unsigned (gpg timed out waiting for a passphrase). Amend
confirmed changes onto it; drop it before the real commit. Its patch is backed up at
`$CLAUDE_JOB_DIR/tmp/og-tier1-backup/wip-commit-df495d3a.patch`. The maintainer reviews with
`difit .` and writes the real commit himself.

Cleaned up: 11.5 GB of build directories (23 of them, including the stale one that caused trap 4),
one worktree, and eleven experimental packages on the console (`/data/pkg` 1024 -> 587 MiB). Kept:
`og-lazy-20260820.pkg` (running), `og-c1-20260820.pkg` (the only known-good reference artefact -
its source tree no longer exists, so it cannot be rebuilt), and locally `og-build-lazy` and
`opengothic-ps4-c1`.

## Next

1. **Repeat the LAZY run two or three times.** One success is not proof - that is the whole lesson
   above. If it holds, LAZY has removed the non-determinism as well as the cost, because there is no
   longer 2.69 GiB being forced into 387 MiB.
2. Tier 2 of the review, untouched: the IME hang with no escape (2.1), the SoundEffect data race
   (2.2), the env example that ships neither of the two knobs it calls mandatory (2.3), the double
   copy of every sound (2.4), the boot probes' half-second (2.5), L2/R2 masking every bare pad bind
   (2.6). See `docs/PLAN-opengothic-review.md`.
3. ⚠ **`/data/tempest-env.txt` is missing `ORBIS_3D_LINEAR=1`.** It currently carries only
   `ORBIS_NO_TESS=1`, `ORBIS_PREDICATION=condexec` and two CTS log settings. That is review finding
   2.3 sitting live on the console; it will bite on entering 3D, not at boot.
4. The plan document needs LAZY added - it was not a review finding. It came out of the failure.

# Handoff — the review closes, and the env file was lying too, 2026-08-21

Tiers 2 and 3 of the OpenGothic review went in, each in one package, each confirmed on hardware. The
review is closed: 23 items across three tiers, plus `VfsMountMode::LAZY`, which it did not find.
OpenGothic is prepared for the maintainer's own `difit .` and his own signature.

## Tier 2, six items, confirmed

Run: 4.5 minutes in-game, 51073 audio blocks, 223 sounds decoded, 0 refused, 16 voices.

    2.1  the IME can no longer hang the save dialog. STATUS_NONE is bounded to ~6 s and reports
         Cancelled; Escape is let through keyUpEvent as a hatch the player can reach at once.
         ⚠ ARMOUR, not a repair - the hang has never been seen and cannot be forced on demand.
    2.2  SoundEffect's six setters, setGlobalVolume and currentTime take the mixer's lock. They
         wrote Voice::playing/finished/cursor/pos/volume with no lock at all while mixBlock read
         them: play() on a finished voice could be undone by an in-flight mixStatic, so a sound
         that was just started never played.
    2.3  the env example ships the two knobs it calls mandatory, with the diagnostics commented
         out. It used to ship NEITHER and enable a submission hexdump instead.
    2.4  a voice takes a share of Sound::Data and reads its buffer in place. Every sound existed
         TWICE - once in Data::ptr, once in a per-effect copy - and N times for N slots playing it.
         ⚠ shared_ptr<void>, because Sound::Data is private and only SoundEffect is a friend.
    2.5  the four heavy boot probes are behind `probes=1`. CONFIRMED: boot log 104 -> 55 lines,
         zero occurrences of readdir/inventory/save-paths/ctype, and the vdf probe's
         stat-interposition VERDICT still there as regression armour. The probe that wrote into
         the player's own Gothic II install every boot is gone outright.
    2.6  a pad modifier masks what it REBINDS, not everything. ⚠ STILL UNVERIFIED - nobody has
         held a trigger and pressed a masked input. It lives in Tempest, now pushed as d4d05d2f.

## Tier 3, six items, confirmed

Run: reached exec, five minutes, 56065 blocks, 21 voices, 221 decoded, 0 refused. The maintainer's
verdict was "wszystko działa jak działało", which is the right outcome for a tier of hygiene.

    3.1  OG_SOUND_NULL finally has a knob: option() plus `--sound-null`. PROVED, not assumed - a
         control build has no sceAudioOutOpen and the default has it. It had been documented,
         compiled and unreachable for months, the exact inverse of OG_FOG_DIAG in Tier 1.
    3.2  workers.cpp logs through ps4_log instead of hand-declaring ac_orbis_note, a symbol
         private to Mesa's PS4 arm, from game code with no header between them.
    3.3  decodeWav refuses a data chunk larger than any sound Gothic ships (64 MiB) instead of
         handing a uint32 from the file straight to resize().
    3.4  imeBegin falls through to the teardown ladder when the FIRST panel is refused. It used to
         return early without incrementing g_opened, so the ladder - the one apparatus built to
         catch 0x80bc0008 returning - was unreachable for the life of the process.
    3.5  26 dead cross-references swept, keeping the fact and dropping the pointer.
    3.6  currentTime() documented rather than rewritten: nothing in OpenGothic calls it.

## ⚠ THE ENV FILE NAMED THREE KNOBS THAT DO NOT EXIST

Asked to check whether every option in `ps4/tempest-env.example.txt` still exists. Three of twenty
had no reader anywhere - not in mesa-ps4, not in any fork:

    ORBIS_NO_PREFETCH     0 occurrences in the whole Mesa tree
    ORBIS_MAX_INSTANCES   0 occurrences
    TEMPEST_PS4_KBD       its ONLY occurrence in ~/src-ps4 was the file documenting it

Each carried a sentence that reads as knowledge: "MEASURED INNOCENT - the stall survives it", "=1
turns the fog LUT's 32-layer draw into one layer", "It TRAPS as an unpatched function on this
firmware and takes the process with it, so the keyboard is off by default". There is nothing to
turn off and nothing being measured.

⚠ **This is the third time this project has met the same shape**, after OG_FOG_DIAG (a knob with no
code) and `deqp-args.txt` (options deleted upstream, so the run died before its first test). The
mechanism is identical every time: **an experiment that silently does not fire comes back as a clean
result.** This instance was the worst of the three because it is not code but an INSTRUCTION TO AN
OPERATOR - it told you to set a line and told you what to expect from it.

Removed, with the check recorded in the file so it can be repeated:
`grep -rl 'getenv("NAME")' src/` in mesa-ps4. Also added `ORBIS_NO_TESS` to the list, which is
titled "THE KNOBS THIS PORT UNDERSTANDS" and had been omitting a mandatory one.

## What the review was worth, honestly

Three items changed behaviour that a player would notice: LAZY, 2.5, and 2.4. Everything else was
either invisible (a race, an unreachable ladder, a bounded hang nobody had hit) or hygiene.

But the two findings with the longest reach were not code at all - the env example that shipped
neither mandatory knob, and the three knobs that do not exist. Both are documents that would have
sent a person to a wrong conclusion, and neither could be caught by a build, a test or a console
run. Only by reading them against the thing they describe.

## State

    ~/src-ps4/orbis-compat   0aab8cb    master        clean
    ~/src-ps4/Tempest        d4d05d2f   ps4-support   clean, PUSHED (one commit, the whole port)
    ~/src-ps4/OpenGothic     a967aaab   ps4-support   19 paths STAGED - the review set
    ~/src-ps4/VK-GL-CTS      cac71bbda  ps4-support   clean, pushed
    ~/src-ps4/ZenKit         39bf1344   ps4-support   clean
    ~/src/mesa-ps4           aea6d53b2ba              clean

OpenGothic's WIP safety commit has been DROPPED (`reset --soft`), so all 19 paths are staged against
`a967aaab` for review. Its patch is kept at
`$CLAUDE_JOB_DIR/tmp/og-tier1-backup/wip-final-d9ef83d1.patch`.

Gitlinks: `lib/Tempest` bumped to **d4d05d2f** (verified present on the remote's ps4-support),
`lib/ZenKit` **39bf1344** - the commit that introduces VfsMountMode, without which resources.cpp
does not compile.

Console: `/data/pkg` holds og-lazy, og-c1, og-tier2, og-tier3 and the canonical package.
`/data/tempest-env.txt` was patched live with the missing `ORBIS_3D_LINEAR=1`.

## Next

1. `difit .`, then the maintainer's own commit. Everything else waits on that.
2. ⚠ **2.6 is still unverified.** Thirty seconds: hold R2, press d-pad up. It is the only one of the
   23 without a hardware confirmation, and it is already pushed in Tempest.
3. Repeat the LAZY run a few more times. One success was never proof - see
   [[one-console-run-is-not-a-verdict]] - and the whole point of LAZY is that it should have removed
   the non-determinism, not merely survived it once.
4. Still open from before: `ac_orbis_drm.c` and `radv_orbis_winsys.c` carry `task-NN`; move
   `~/src/mesa-ps4` to `~/src-ps4`; PLAN.md 6 (CI) and 8 (upstream).

# Handoff — the six repositories, and where each one stands, 2026-08-21 (close)

Written for a session that will be working across the forks rather than inside one of them. The
previous entries are chronology; this one is a map.

## The map

    ~/src/mesa-ps4        orbis        b0fda7f9  NO REMOTE   the driver. 156 files, +34396
    ~/src-ps4/orbis-compat master      0aab8cb   NO REMOTE   the platform overlay. 13 commits, 109 files
    ~/src-ps4/Tempest      ps4-support d4d05d2f  pushed      the engine. ONE commit carrying the port
    ~/src-ps4/OpenGothic   ps4-support de1b93fe  pushed      the title. ONE commit, submodules pinned
    ~/src-ps4/ZenKit       ps4-support 39bf1344  pushed      VfsMountMode - LAZY lives here
    ~/src-ps4/VK-GL-CTS    ps4-support cac71bbda pushed      the CTS port

All six are clean. Four are published on `mikolajmikolajczyk/<name>`; **two are not published at
all**, and those two are the ones that carry the most work.

## How the pieces depend on each other

    OpenGothic ──gitlink──> Tempest d4d05d2f      https://github.com/mikolajmikolajczyk/Tempest
               ──gitlink──> ZenKit  39bf1344      https://github.com/mikolajmikolajczyk/ZenKit
               ──path─────> orbis-compat          -DORBIS_COMPAT_DIR, default ~/src-ps4/orbis-compat
    orbis-compat ─────────> the toolchain file, the overlay archive, vkloader, ps4-app
    mesa-ps4 ─────────────> built separately; the title links RADV through orbis-compat's vkloader

⚠ **`lib/Tempest` and `lib/ZenKit` are SYMLINKS in the working tree and GITLINKS in the commit.**
That split is deliberate - it lets edits in the forks be picked up without a submodule dance - but it
means the two never agree by themselves, and `diff.ignoreSubmodules=all` is set repo-locally so that
`git status` and difit work at all. Consequences, all of which bit today:

  * `git add -A` rewrites both as symlinks (mode 120000). Check `git ls-files -s lib/` after.
  * `git stash push -- ps4/` DELETES files that are staged-but-not-in-HEAD.
  * `git reset --hard` destroys the symlinks; re-create them by hand afterwards.
  * a gitlink is only half a pointer: `.gitmodules` must name a repo that HAS that commit.
    `lib/Tempest` pointed at upstream Try/Tempest for one push, which does not carry d4d05d2f.

Verify a published pointer without cloning:

    GIT_TERMINAL_PROMPT=0 GIT_SSH_COMMAND=false git ls-remote <url> | grep <sha>

## What is left, per repository

**mesa-ps4** — the largest body of work and the least finished as a publication.
  * no remote at all. 34396 insertions across 156 files, on branch `orbis` off `main`.
  * ⚠ **49 `task-NN` references remain in the driver**: 48 in `src/amd/common/ac_orbis_drm.c`, one in
    `src/amd/vulkan/radv_orbis_winsys.c`, plus 19 files under `build-support/`. No tracker has those
    numbers. They were swept everywhere else and these were left.
  * still to move to `~/src-ps4` alongside the other five.
  * PLAN.md 6 (CI) and 8 (upstream) untouched.

**orbis-compat** — no remote. 109 files, 13 commits. It is the only repository the assistant has
standing permission to commit in. Everything else in this port depends on it by path.

**Tempest / OpenGothic / ZenKit / VK-GL-CTS** — published, one commit each, nothing outstanding
except:
  * ⚠ **2.6 (the pad modifier fallback) is in Tempest, pushed, and NEVER TESTED.** Hold R2, press
    d-pad up; the inventory should open. It is the only one of the review's 23 items without a
    hardware confirmation.

## Working rules this port has paid for

**Console.** The assistant builds, packages and uploads; the maintainer installs and runs. Every
proposed run ends with `INSTALL + RUN` or `RUN, no install`. Packages install only from `/data/pkg`,
by dated name (`og-lazy-20260821.pkg`) so an install can be pointed at one build. FTP is
`lftp -p 2121 192.168.100.2`; it stops responding while the console is wedged, and a hung `rm` may
have done nothing - re-list before believing it.

**Logs.** `~/src-ps4/orbis-compat/scripts/ps4/logs.sh`, capturing to that repo's `build-ps4-logs/`.
It must be running BEFORE the title starts; the interesting lines print once.

**⚠ One console run is not a verdict.** The archive-mount failure was non-deterministic: two
functionally identical binaries gave opposite outcomes, and the same binary took 1.9 s and 20.2 s for
the same work. Eight runs were spent bisecting noise. When something worked earlier and does not now,
**re-run the earlier package first** - it costs nothing and halves the search space.

**Do not commit in the forks.** The maintainer reviews with `difit .` and signs his own commits.
orbis-compat is the exception. When a throwaway safety commit is wanted, say in its message that it
is throwaway, and expect to drop it with `reset --soft`.

**Back up before deleting.** Every deletion this project has regretted was reversible only because a
tarball or a patch existed first.

## The shape worth carrying forward

Four separate defects this week had the same form: **something that looks like it is doing work, is
not, and reports nothing.** `OG_FOG_DIAG`, a knob with no reader. `deqp-args.txt`, options deleted
upstream so the run died before its first test. Three env knobs documented with measurements
attached and no `getenv` anywhere. And `mount_disk` called without the mode that was added for it.

None of the four is visible to a build, a test, or a console run. Each was found by reading one
thing against the thing it claims to describe - and three of the four were found because the
maintainer asked a plain question about something that looked settled.

---

# Handoff — the trees moved, build-support was halved, and there is a review plan to execute, 2026-08-22

Written for a session whose job is to **carry out `mesa-ps4/PLAN.md`**. Everything below is either
where things now live, or something that will otherwise be re-derived.

## ⚠ Everything moved. `/home/mikolaj/src/mesa-ps4` is now a BACKUP, not the working tree

    ~/src-ps4/mesa-ps4        orbis        ac2f18a9149   the driver - WORK HERE
    ~/src-ps4/orbis-compat    master       0aab8cb       platform overlay
    ~/src-ps4/ps4-mesa-docs   main         03b651a       this file lives here now
    ~/src-ps4/VK-GL-CTS       ps4-support  2c6c32438     pushed
    ~/src-ps4/Tempest         ps4-support  d4d05d2f      pushed
    ~/src-ps4/OpenGothic      ps4-support  de1b93fe      pushed
    ~/src-ps4/ZenKit          ps4-support  39bf134       pushed
    ~/src-ps4/RetroArch       ps4-support  4920537011    clean; this session never touched it

`/home/mikolaj/src/mesa-ps4` still holds the FULL HISTORY - 267 commits - and its own `build-orbis`.
It was left untouched on purpose. **A whole exchange was lost this session to editing one tree and
reading the other**, so check `pwd` before believing "nothing changed".

## The driver is now one staged change set, not 267 commits

`~/src-ps4/mesa-ps4` was squashed with `git add -A` then `git reset --soft main`. HEAD is upstream
Mesa (`ac2f18a9149`); the whole port sits in the index:

    73 files, +17795 / -145        41 src/amd · 15 build-support/tools · 9 src/vulkan · 3 src/util · rest

⚠ **The 267 commits are reachable ONLY through `refs/backup/orbis-267-commits`** in that repo, and
normally on the branch in the backup checkout. The upstreaming shortlist names six of them by SHA;
they are objects without a branch now.

⚠ Mesa upstreaming is **closed** - decided 2026-08-21, they will not take Orbis as a target. PLAN.md
§8 and the "upstreamable list" above are dead text. ZenKit and OpenGothic are unaffected.

## `mesa-ps4/PLAN.md` - the review plan, and what verifying three of it found

Written by a separate `/code-review max` agent, 15 findings in 5 phases. It is in
`.git/info/exclude` **line 1** - the agent excluded it before writing it, so `git status` will never
show it and `git add -A` cannot take it. Do not "fix" that.

Its line numbers were checked against the tree and they MATCH: the review ran after this session cut
454 lines out of `ac_orbis_drm.c`.

Three findings were verified by reading the code rather than trusting the report:

**2.3 (`vk_drm_syncobj.c:698`) - REAL, and it is the one to do first.** `u_sync_provider.c` sets
`.transfer = drm_syncobj_transfer` unconditionally; `drmGetCap(DRM_CAP_SYNCOBJ_TIMELINE)` gates only
`timeline_signal` and `timeline_wait`. So on a kernel without timeline syncobj the new arm
`(device->sync->transfer != NULL && wait_count == 1)` routes payload copies through an ioctl that is
not there. **This is in `src/vulkan/runtime/`, shared by every Mesa Vulkan driver** - the only
finding that damages somebody else. The fix is one term: the condition already has
`vk_device_has_timeline_syncobj(device)` next to it.
⚠ The code carries a long ⚠ comment defending `wait_count == 1`. It defends the transfer's SHAPE
(many-to-many needs a real timeline). It never asks whether the ioctl exists. A comment answering a
different question reads like a comment answering this one.

**2.4 (tess factor ring) - mechanism REAL, conclusion UNVERIFIED.** Measured in the tree:
`(192/3)*16 = 1024` B per workgroup, `num_workgroups = 256` hardcoded for LIVERPOOL/GLADIUS, so
`tess_factor_ring_size = 262144` B = 65536 dwords = `0x10000`. The generated header says

    #define S_030938_SIZE(x)  (((unsigned)(x) & 0x1FFFF) << 0)     /* 17 bits, widened for GFX11 */
    #define C_030938_SIZE     0xFFFE0000

so `assert((tf_ring_size & C_030938_SIZE) == 0)` in `ac_cmdbuf_cp.c:417` does NOT catch `0x10000`.
That part is certain. **What is not established is the review's premise that the GFX7 hardware field
is 16 bits** - it cites no source, and this port has already been burned by a number taken from a
family that is not this family (`ORBIS_NUM_SE=4`, falsified on hardware). Establish the width before
changing the count.
⚠ And note `gfx_level >= GFX7` takes the `R_030938` path - the `R_008988` arm below it is GFX6.
Reading that backwards costs a wrong fix.

**Phase 1 is not the blocker it says it is.** The three build breaks are Linux-without-orbis,
Windows and MSVC. Nothing here builds those, and this session built the tree four times today.
Worth fixing for hygiene; not worth blocking on.

Suggested order: **2.3 first** (it is one term and it is not ours to break), then phase 2, then the
rest as written.

The review's own stated gap: the `build-support/` finder returned nothing. That area was halved
today, so it is cheaper to look at now than it was this morning.

## What this session removed, so nothing gets restored by accident

    build-support/orbis/{docs,notes}    -> ~/src-ps4/ps4-mesa-docs   (this file, PLAN, TODO, research, dumps)
    build-support/orbis/cts/*           -> CTS fork targets/orbis/   (deqp-args, deqp-env, qpa-status.py)
                                           and ps4-mesa-docs/cts.md
    build-support/orbis/cross/          -> orbis-compat/cmake/orbis.ini.in
    build-support/orbis/tools/umtxcheck.c -> orbis-compat/test/
    build-support/orbis/title/          deleted - the OpenGothic patch-queue era, superseded by the fork
    build-support/orbis/tools/          32 files -> 15. Deleted: stalepool, thickprobe, thick3d,
                                        dargs.*, vidx.*, and three .spv intermediates
    ac_orbis_drm.c                      -454 lines: orbis_probe_gnm_debugger, orbis_test_mprotect_reaches_gpu,
                                        orbis_test_cond_write, and the three knobs that drove them
    task-NN references in src/          49 -> 0

⚠ **`gen-tile-tables.py` was nearly deleted as a dead probe and must not be.** It GENERATES
`src/amd/common/orbis_tile_tables.h`, a driver source. `tilecheck` stays too - HANDOFF's "A3" wants
it pointed at a layered MSAA image.

`gen.py` now `unlink()`s its intermediate `.spv`; three of them had sat checked in looking like
inputs. Regenerating the three headers reproduces them byte-identically, so the committed products
match their sources.

## ⚠ Four instruments that report success without checking - three found today

    build.sh --host-orbis     reads probe EXIT CODES, never their output. Three futex self-tests print
                              FAILED in the same run it calls "every probe exited cleanly". Not a
                              regression - the untouched backup tree does the same. See
                              [[host-orbis-gate-does-not-read-its-own-log]].
    -Wno-unused-function      is ON in this build, so orphaning a static function is silent. After
                              cutting the probes it had to be checked by hand.
    git status --porcelain    `grep -v "^[MADR]"` HIDES the `AD` state (staged-add, deleted on disk).
                              25 files sat in that state for hours because the check filtered them out.
    byte-comparing artefacts  see below.

## Comparing built artefacts: identity is useful, difference is not

Three times today a byte difference was nearly read as a finding:

    PS4 .pkg                 NON-DETERMINISTIC. Two packagings of the same eboot differ. Compare
                             eboot.bin instead - that IS deterministic.
    deqp-vk                  embeds the CTS commit SHA (qpReleaseInfo.inl). One amend moved 587490 bytes.
    libvulkan_radeon.a       embeds constants in radv_physical_device.c.o. The differing-byte count
                             went 83 -> 80 -> 67 across builds whose code was identical.

⚠ **The build IS deterministic** - two from-scratch builds of the same tree give a byte-identical
archive, measured. So *identical* means the same driver and can be relied on. *Different* means only
that the input differed, and the count of differing bytes means nothing at all.
⚠ A comment-only edit to `ac_surface.c` changed 67 bytes of those constants. The mechanism was NOT
identified. It is benign for correctness; do not build an argument on it.

## State, and what is waiting for the maintainer

    ~/src-ps4/mesa-ps4        73 files staged. Cross build 768/768 clean, linkprobe links,
                              --host-orbis identical to the backup-tree control. NOT COMMITTED.
    ~/src-ps4/orbis-compat    M include/signal.h + 2 untracked moved files
                              (cmake/orbis.ini.in, test/umtxcheck.c). NOT STAGED - the maintainer
                              had not said whether to stage them.
    ~/src-ps4/VK-GL-CTS       3 untracked in targets/orbis/ - the run configuration moved there.
                              NOT COMMITTED.
    ~/src-ps4/ps4-mesa-docs   03b651a. ⚠ UNSIGNED - gpg timed out.
                              Fix: `git commit --amend --no-edit -S`

Open questions the maintainer has not answered: a GitHub organisation (`orbis-ports` was the
candidate; the three unpublished repos are free to create there, the four forks carry committed
`.gitmodules` URLs and four open upstream PRs), and whether `mesa-ps4`/`ps4-mesa-docs` should be
renamed `orbis-*` to match the code's own vocabulary - the `__PS4__` -> `__ORBIS__` rename is still
an open item in this file, twice.

---

# Handoff — the review plan is executed, and two of its fixes needed fixes of their own, 2026-08-22 (close)

All fifteen findings of `mesa-ps4/PLAN.md` are done. This entry is what a reader needs in order to
judge the work rather than repeat it.

## What was verified, and how

    cross build (build-orbis)     0 failed targets, 0 compile errors, libvulkan_radeon.a 36138470 B
    link probe                    linked 26455800 B -> build-orbis/linkprobe.elf
    --host-orbis gate             rc=0, identical to the documented control: 3 futex self-tests print
                                  FAILED and the gate still says "every probe exited cleanly"
    build-host (NO orbis)         ⚠ CONFIGURED AND BUILT FOR THE FIRST TIME IN THIS TREE.
                                  -Dplatforms= , MESA_SYSTEM_HAS_KMS_DRM=1 -> libvulkan_radeon.so links

That last line is the point of phase 1. `build.sh --host` has existed all along and had never been
run here, which is exactly why 1.1 survived: the only two configurations anyone built were the two
that define HAVE_ORBIS_PLATFORM.

⚠ **1.1 was confirmed by control, not by reading.** With the guard removed again and only that one
file recompiled:

    ../src/amd/vulkan/radv_queue.c:592:4: error: implicit declaration of function
    'ac_orbis_note_gs_ring_sizes' [-Wimplicit-function-declaration]

and `-Werror=implicit-function-declaration` is in that build's command line. The call site carried a
comment saying "the note is a no-op elsewhere". It was not: nothing declares it and nothing defines
it outside the Orbis build.

## The two premises the review asserted without a source, and what they turned out to be

**2.4, the tess factor ring — the premise was RIGHT, and now it is cited.** Mesa ships the register
database this port had never opened:

    src/amd/registers/gfx6..gfx10.json    VGT_TF_RING_SIZE.SIZE  bits [0,15]   16 bits, max 0xFFFF
    src/amd/registers/gfx11.json                                 bits [0,16]   17 bits

256 workgroups x 1024 B = 65536 dwords = 0x10000, one past the field on Liverpool. The generated
`S_030938_SIZE` / `C_030938_SIZE` pair is built from the WIDEST definition, which is why the assert
in `ac_cmdbuf_cp.c` had a hole exactly where the value sat. Clamped in two places - the count in
`ac_gpu_info.c` and the register in `ac_cmdbuf_cp.c`, the latter per generation and loud.

**4.3, COND_EXEC - also right, and citable from this tree.** `src/amd/packets/cp_pm4_table_data_gfx11.json`
and its gfx12 twin give EXEC_COUNT as bits 13:0 for both the PFP and MEC forms: 16383 dwords. The
loop was predicating `per_draw * n` in ONE packet, 46 dwords a draw by default, so ~357 indirect
draws under conditional rendering overflowed it and the PFP would resume inside a packet. Now one
COND_EXEC per draw, which cannot overflow. AMD publishes no gfx7 table in this tree, so the width is
established for gfx11/gfx12 and assumed unchanged backwards - say so rather than claim more.

## 2.3 could not be fixed the way the plan wrote it

The plan offered "gate the new arm on `vk_device_has_timeline_syncobj`", which would simply delete
the arm this port added, and "or gate `.transfer` in u_sync_provider.c". The second is right - the
DRM provider filled the entry in unconditionally while the kernel refuses the ioctl with -EOPNOTSUPP
unless the driver has DRIVER_SYNCOBJ_TIMELINE, the same bit DRM_CAP_SYNCOBJ_TIMELINE reports - but
doing only that turns two error paths into NULL dereferences: `radv_amdgpu_cs.c` calls
`ac_drm_cs_syncobj_transfer` on its `!has_fence_to_handle` arm, which is reachable on old kernels.
So both wrappers in `ac_linux_drm.c` now return -EOPNOTSUPP for a NULL entry, which leaves every
existing caller's error handling doing what it already did.

## ⚠ TWO FIXES BROKE SOMETHING AND THE GATE SHOWED ONE OF THEM

**5.2 turned the residue detector into a liar.** Freeing the enumeration's probe device - a one-line
leak fix - made `orbis_report_residue()` run on a device holding nothing, which recorded ZERO as the
previous sample. The next teardown, the first real one, then read as growth:

    MESA: warning: orbis-drm: RESIDUE GREW across devices - 4 BO(s) was 0, 2 syncobj(s) was 0 ...
                   - something is leaking per device

in a run with no leak in it. Fixed by not letting an all-zero teardown become the baseline. It now
reports four device cycles at a steady 4 BOs / 2 syncobjs / 4 VA ranges, "and it did not grow".

⚠ This is the fourth instance of the shape this project keeps meeting, and the first where the
assistant caused it: **a change that makes an instrument report the wrong thing is worse than the
defect it fixed**, because the next real warning is now ignored. It was caught only by re-running
the gate and reading its output - the same discipline that
[[host-orbis-gate-does-not-read-its-own-log]] is about.

**The unsequenced marker.** Not in the review at all; the cross build has been printing it:

    ac_orbis_drm.c:1338: warning: unsequenced modification and access to 'dst' [-Wunsequenced]
    *dst++ = (uint32_t)(dst - out_base);

Undefined behaviour, and the value differs by one dword depending on what the compiler chose that
day. The progress marker it writes is the thing that says where the CP stopped, so it was a
diagnostic that could be off by a packet boundary. Spelled out to the documented intent.

## Everything else, one line each

    2.1  the GARLIC chunk now rides on the retire entry with the mprotect revoke, so the pages are
         not swapped and released under a GPU still executing the frame. A MAP that cancels a parked
         entry hands the chunk back synchronously - the address is being re-let that instant.
         ⚠ ORDER IS LOAD-BEARING: the chunk goes back BEFORE the protect, or the fresh MAP_FIXED
         mapping hands the rights back to a range the drain is about to revoke.
    2.2  present, then take the scan-out down, THEN free the images. The destroy loop used to free
         them first and the flush then read prev->cpu_map from a destroyed image. wsi_orbis's
         teardown also drains pending flips when the buffers are the swapchain's own.
    3.1  the fence-slot spin has a deadline. It held orbis_submit_lock, so a GPU hang wedged every
         later vkQueueSubmit - the console had to be pulled from the wall. It is the ONE wait that
         keeps a hard cap, and it says why.
    3.2  one orbis_retire_lock for push, drain and cancel. Three disjoint locks around one ring is
         the same as none. Order: submit -> retire -> va.
    4.1  one format table, read by get_formats AND get_formats2. The Orbis restriction was in the
         first only, and Formats2 is the entry point dxvk uses - red and blue swapped.
    4.4  one exit from queue_present. Three paths returned without releasing or handing on the
         image, so every failed flip consumed one for the life of the swapchain.
    4.5  the 5 s cap is a WATCHDOG now, not an answer. vkWaitForFences(UINT64_MAX) may not return
         VK_TIMEOUT. It logs every five seconds and honours the caller's own deadline.
    5.1  the four per-chunk mesa_logi are behind orbis_trace(). 84 submits a second x 4 lines,
         flushed to storage inside the submit lock.

## The change set

    +8   -1    src/util/u_sync_provider.c
    +16  -0    src/amd/common/ac_linux_drm.c
    +7   -0    src/vulkan/runtime/vk_drm_syncobj.c
    +316 -52   src/amd/common/ac_orbis_drm.c
    +103 -101  src/vulkan/wsi/wsi_common_headless.c
    +22  -0    src/vulkan/wsi/wsi_orbis.c
    +35  -1    src/amd/common/ac_gpu_info.c
    +33  -1    src/amd/common/ac_cmdbuf_cp.c
    +48  -5    src/amd/vulkan/radv_cmd_buffer.c
    +11  -0    src/amd/vulkan/radv_orbis_winsys.c
    +12  -2    src/amd/vulkan/radv_queue.c
    +7   -3    src/amd/vulkan/radv_instance.c
    +10  -3    src/amd/vulkan/radv_physical_device.c
    +4   -2    src/amd/vulkan/radv_physical_device.h

The pre-plan state is kept as two patches under the job's scratch directory
(`staged-before-plan.patch`, `worktree-before-plan.patch`); that is scratch and will not survive, so
anything wanted from it has to be taken before the job is deleted.

## ⚠ THE INDEX HOLDS A STALE ac_orbis_drm.c

`git status` shows it as `AM`: the version STAGED is the one from before the previous session cut
454 lines of probes out of it. The working tree is the truth and everything above was built from it.
A plain `git commit` would commit the old file. `git add -A` before committing - PLAN.md is on line 1
of `.git/info/exclude`, so it cannot be swept in.

## What has NOT been done

  * ⚠ **Nothing here ran on the console.** Every one of these fixes is on a path a title exercises -
    the retire queue, the present, the submit lock, indirect draws. The cross build and the host arm
    say they compile and that the host-side probes still pass; they say nothing about the hardware.
    One console run is not a verdict either ([[one-console-run-is-not-a-verdict]]), so the CTS is the
    instrument for this, not one boot of the title.
  * 1.2 is Windows-only and was not compiled. The change is mechanical - the predicate for "this is
    the console" is HAVE_ORBIS_PLATFORM, and `!MESA_SYSTEM_HAS_KMS_DRM` was catching Windows too -
    but it is reasoned, not built.
  * The review's own stated gap is still open: its `build-support/` finder returned nothing.
  * Still waiting on the maintainer: the commits themselves (`difit .`), the unsigned
    ps4-mesa-docs commit, whether to stage the two moved files in orbis-compat, the GitHub
    organisation, and the `__PS4__` -> `__ORBIS__` rename.

## The plan on hardware: three CTS runs, and what the control settled, 2026-08-22 (evening)

The fifteen fixes were taken to the console. Three runs of the SAME 735-case list, two of them on the
new driver and one on the old one as a control.

    package    cts-plan-20260822.pkg     the driver with PLAN.md executed
    control    cts-nopatch-20260821.pkg  same CTS fork commit 2c6c32438 (its binary linked four
                                         minutes after that commit), same vkloader, same target
                                         file - ONLY ORBIS_RADV_LIB differs, pointing at the backup
                                         checkout's 21 Aug archive
    list       api.smoke + api.object_management.multithreaded_* (the 140 with a baseline)
               + draw.renderpass.indirect_draw.* + indirect_instanced.*
               + memory.allocation.* + synchronization.basic.*

                         Pass   Fail   NotSupported
    plan run 1            718     2         15
    plan run 2            717     3         15
    control (21 Aug)      719     1         15

### The baseline did not move

All 140 cases that have a 21 Aug baseline passed in every run, and the test-by-test diff of verdict
IDENTITIES is empty in both plan runs. That is the regression answer: executing the plan changed
nothing on the only set that could tell.

### One real defect, and the control proved it is DEBT

    dEQP-VK.synchronization.basic.event.device_set_reset    Fail on ALL THREE runs

and the driver log carries, identically in all three, five `syncobj wait timed out` at fence labels

    889 889 889 890 892 893 893 894

Byte-for-byte the same sequence on both drivers. `event.host_set_reset` - the same event set by the
CPU - passes; only the GPU-side `vkCmdSetEvent` fails, and the submission carrying it never retires.
⚠ That is the fourth instance of this port's recurring shape: **RADV emits a packet this command
processor does not implement and the CP stalls rather than failing.** Task #34 carries the leads.

### The indirect_draw failures are noise, and the third run is what settled that

    run 1   1 failure    indexed_data_from_compute_alloc_offset_16 ... no_first_instance.triangle_strip
    run 2   2 failures   two OTHER cases; run 1's failure PASSED
    control 0 failures

Three runs, three disjoint answers, out of 364 cases. Exactly [[counts-hide-nondeterminism]]: the
count reads like a measurement and the identity is a coin toss. ⚠ **Nothing about the indirect-draw
path was established by any of this**, in either direction.

### Two things the runs said about the plan's own fixes

**5.2 is visible in the numbers.** Teardowns per run: control 330, plan runs 455 and 444. The ~120
extra are one per physical-device enumeration - the probe device that used to leak and is now freed.

**The residue detector's noise is NOT from the plan.** `RESIDUE GREW` fires 122/330 in the control
(37%) against 124/455 and 127/444 in the plan runs (27%, 29%). The zero-baseline false positive this
session introduced and fixed is gone - only three all-zero teardowns remain and none becomes the
baseline. What is left is the detector's own weakness, pre-existing: it compares against the
IMMEDIATELY PREVIOUS sample rather than against the floor, and residue legitimately oscillates
between 11 and 17 BOs while the floor stays at 11. A real per-device leak would ratchet; this does
not. Worth fixing, but it is not today's.

### What these runs did NOT measure

  * **4.3 was never executed.** The per-draw COND_EXEC only emits under conditional rendering, and
    none of the 364 indirect-draw cases enables it - `radv_orbis_predicate_next` returned false every
    time. `dEQP-VK.conditional_rendering.*` is the set for it and has not been run.
  * **4.5's watchdog never fired**, in any of the three runs. No wait reached five seconds, so every
    timeout above came from a caller deadline SHORTER than the cap - a regime where the change is a
    no-op. The unbounded-wait path is still untested on hardware.
  * 4.1 (Formats2), 2.2 (swapchain destroy) and 2.1's GARLIC deferral were not covered by this list
    either; WSI is NotSupported on this target and nothing here destroys a swapchain.

### ⚠ The trap this nearly walked into

Both CTS build directories in the fork were configured with
`ORBIS_RADV_LIB=/home/mikolaj/src/mesa-ps4/build-orbis/...` - the BACKUP checkout. Rebuilding either
would have packaged the 21 August driver and the run would have said nothing about the plan, with
nothing anywhere announcing it. `cts.md` named that path too, and `~/src/Tempest/scripts/ps4/make-pkg.sh`
which moved to `orbis-compat/scripts/ps4/` some time ago. Both corrected, and cts.md now carries the
check: `grep ORBIS_RADV_LIB build-orbis/CMakeCache.txt`.

Verification that the right driver shipped was NOT the package size - all three CTS packages are
exactly 109117440 bytes - but `nm` on deqp-vk finding `orbis_garlic_put_back` and `orbis_retire_lock`,
symbols that exist only in today's tree.

### Console state

    /data/pkg/cts-plan-20260822.pkg      the driver with the plan executed
    /data/pkg/cts-nopatch-20260821.pkg   the control
    /data/deqp-results-plan.qpa          run 1        /data/deqp-mesa-plan.log
    /data/deqp-results-plan2.qpa         run 2        /data/deqp-mesa-plan2.log
    /data/deqp-results-old735.qpa        control      /data/deqp-mesa-old735.log
    /data/deqp-results-nopatch.qpa       the 21 Aug 140-case baseline, untouched
    /data/deqp-args-nopatch.txt          its run configuration, preserved before overwriting

## conditional_rendering: 4.3 answered, and 23 failures that are all debt, 2026-08-22 (late)

The three earlier runs never executed finding 4.3's code once - `radv_orbis_predicate_next` returns
false unless conditional rendering is enabled, and none of the 364 indirect-draw cases enables it.
This is the set where it runs. 1030 cases plus the 140-case baseline anchor, on the plan driver and
then on the pre-plan driver as a control.

                     Pass   Fail   NotSupported
    plan driver       627     23        520
    control (21 Aug)  627     23        520

⚠ **Zero verdict changes. The 23 failure NAMES are identical, and so is the 520-case NotSupported
set.** Not "the same counts" - this project has been burned by that - the same names.

### 4.3 works, in the regime the CTS can reach

    conditional_rendering.draw        168 ran, 168 Pass, 0 Fail   (on BOTH drivers)
      draw_indirect                    28 Pass
      draw_indexed_indirect            28 Pass
      draw_indirect_count              28 Pass
      draw_indexed_indirect_count      28 Pass

112 indirect draws under conditional rendering, each now emitting a COND_EXEC inside the loop rather
than one packet covering `per_draw * n` dwords. No timeouts, no GPU faults, no wedge; the fence-slot
throttle from 3.1 was entered once (its cap message logged) and never expired.

⚠ **The overflow itself is still untested.** Reaching the 14-bit EXEC_COUNT limit needs more than
about 357 predicated indirect draws in one command buffer and nothing in the CTS does that. What is
established is that the new shape is correct, not that the old one broke. Those are different
claims.

### The 23, in three coherent clusters - all pre-existing

    9 of 9    transform_feedback.*        "Expected value at index 6 was 2, but actual value was 0",
                                          identical in all nine INCLUDING the plain non-indirect
                                          draw - so it looks like XFB rather than predication
    8 of 139  draw_clear.clear            image comparison, max difference (0.5,0,0,0) and
                                          (0.3,0,0,0) - RED CHANNEL ONLY, and only the
                                          *_full_no_offset variants; every *_partial_offset passes
    6 of 90   dispatch.condition_size     second_byte, third_byte, fourth_byte x2. The non-zero part
                                          of the 32-bit condition placed at byte 1, 2 or 3;
                                          first_byte passes. Something evaluates less than the whole
                                          dword.

Task #35 carries the leads for each.

### Two of the plan's own fixes are visible in the logs

**5.1, and it is the largest single effect measured today.** Driver log for the same 1170 cases:

    control     3620 lines
    plan run     927 lines

~2700 lines gone. Those were the four per-chunk `mesa_logi` calls that ran on every submission and
were flushed to storage INSIDE the global submit lock. That is what "84 submits a second times four
lines" costs, and it is now behind `orbis_trace()`.

**5.2 again.** Teardowns 133 (control) against 246 - one extra per physical-device enumeration,
which is the probe device that used to leak. And the residue detector's noise is proportionally
lower on the plan driver in this run too: 118/133 (89%) against 123/246 (50%).

### Where the plan now stands on hardware

Five runs, two drivers, three case sets. Everything the CTS can reach says the plan changed nothing
it should not have and fixed what it claimed to. What remains unmeasured on hardware:

  * 4.5's watchdog - never fired in any of the five runs, so the unbounded-wait path is untouched
  * 4.1 (Formats2), 2.2 (swapchain destroy), 2.1's deferred GARLIC restore - no case set here
    reaches them; WSI is NotSupported on this target
  * 1.2 - Windows only, not compiled anywhere
  * 4.3's overflow - as above

## The title on the plan's driver: 3328 frames, and the coverage gap is closed, 2026-08-23

The five CTS runs of 2026-08-22 answered nothing about four of the fifteen fixes, because no case
set reaches them. OpenGothic does. Built against `~/src-ps4/mesa-ps4/build-orbis` - confirmed from
`link.txt`, not assumed - packaged as `og-plan-20260823.pkg`, installed and played.

    3328 frames, 1293 submissions, ~50 fps average - the usual figure for this title

    syncobj timeouts            0
    4.5's watchdog              0        no wait reached five seconds
    RESIDUE GREW                0
    fence-slot exhaustion       0        3.1's cap logged once, never expired
    Protection faults           0

⚠ **Zero protection faults over 3328 frames is the direct answer on 2.1.** Deferring the GARLIC
restore behind the retire fence changes when physical pages are swapped out from under a running
GPU; if the synchronisation were wrong, a fault or visible corruption is the first thing it would
produce. Neither appeared, and the title looks and performs as it did before.

One warning fired: `THE GPU HAS NOT FINISHED submit #1293 - the fence label has been stuck at 1292
for 4 submissions while 5 were queued`, with a stream dump. The title recovered and carried on.
**The maintainer identifies this as a long-standing defect, not new.** Its address audit is worth
recording anyway because it exonerates 2.1 specifically: `0/0 SET_BASE and 0/16 DMA_DATA addresses
are NOT in any live mapping` - nothing in the hung submission pointed at freed memory, which is
exactly what a mis-deferred GARLIC restore would have produced.

### ⚠ A THIRD STALE DEFAULT, and this one was live

`orbis-compat/vkloader/CMakeLists.txt` defaults `ORBIS_MESA_BUILD` to
`~/.cache/orbis-mesa/mesa/build-orbis` - the patch-queue era's path. OpenGothic passed no override,
and that path still holds a driver dated 2026-08-21 22:04. The title would have linked the previous
day's RADV, run perfectly, and every measurement taken from it would have been about changes it did
not contain.

That is the third variant of one trap in two days:

    VK-GL-CTS's two build dirs   -> ~/src/mesa-ps4        (the backup checkout)
    cts.md's recipe              -> ~/src/mesa-ps4
    vkloader's default           -> ~/.cache/orbis-mesa/mesa

All three named a path that looked right. The date was the only thing that separated them, and
nothing printed it. `orbis-compat/scripts/ps4/orbis-env.sh` now resolves the checkout once and
exports `ORBIS_MESA_DIR` / `ORBIS_MESA_BUILD` / `ORBIS_RADV_ARCHIVE`; `orbis_announce_driver()`
prints the archive AND its build time, and every entry point calls it before configuring.

### Two smaller things this run exposed

⚠ **The console's clock is ~46 minutes ahead of the host.** Measured, not assumed: a file uploaded
at host 09:35:43 lists as `sie 23 10:22`. FTP timestamps and local mtimes are therefore not
comparable, and comparing them produced a confident wrong conclusion ("the run was not captured")
about a capture that was fine. See [[ps4-console-access-and-logs]].

⚠ **The title writes its driver log to `/data/deqp-mesa-default.log`** - a name that says deqp. This
run overwrote a 4.3 MB CTS baseline. `tempest-env.txt` warns about exactly this in its own comments
and then does it; the file needs a name of its own.

---

# Handoff — the plan is done and verified, everything is published, 2026-08-23 (close)

Written for a session picking up after `mesa-ps4/PLAN.md` was carried out and taken to hardware.
The three entries above this one are the detail; this is the map and what is left.

## Everything now lives in the orbis-ports organisation

    orbis-ports/mesa-ps4        orbis        own        the driver
    orbis-ports/orbis-compat    master       own        the platform overlay - EVERY other repo needs it
    orbis-ports/ps4-mesa-docs   main         own        this file
    orbis-ports/OpenGothic      ps4-support  fork       default branch is ps4-support, not master
    orbis-ports/Tempest         ps4-support  fork
    orbis-ports/ZenKit          ps4-support  fork
    orbis-ports/VK-GL-CTS       ps4-support  fork
    orbis-ports/RetroArch       ps4-support  fork       the ps4-support branch existed ONLY locally until today

All public. The four Try/Khronos forks were TRANSFERRED, so the upstream relationship and the
redirects from `mikolajmikolajczyk/*` survive; RetroArch was forked fresh because its origin had
always been libretro's own repository.

⚠ **mesa-ps4's history is 227793 commits and a single push of it FAILS.** It disconnected mid-pack
and left an empty repository that still exited 0. It went up in twelve staged pushes of ~20000
commits each; the script is worth reconstructing rather than retrying the single push.

## The build contract - one entry point per repository

Clone the repositories NEXT TO EACH OTHER and nothing needs setting:

    ps4/build.sh                in mesa-ps4, VK-GL-CTS, OpenGothic
    ps4/build-core.sh           in RetroArch (cores only; the app has no script yet)

All of them source `orbis-compat/scripts/ps4/orbis-env.sh`, which resolves and verifies
`ORBIS_COMPAT_DIR`, `OO_PS4_TOOLCHAIN`, `ORBIS_MESA_DIR`, `ORBIS_WORK` and `ORBIS_JOBS`. Finding
orbis-compat is the one thing that cannot be shared, so each entry point carries six lines of path
search; copy that block verbatim rather than inventing a variant.

Deploy with `orbis-compat/scripts/ps4/deploy.sh --pkg <file> --name <short>`: it uploads to
`/data/pkg/<name>-YYYYMMDD.pkg`, verifies by reading back, and ends by saying INSTALL + RUN or
RUN, no install.

## ⚠ THE ONE PATTERN THIS WEEK KEEPS PRODUCING

**A path that looks right, pointing at a driver from another day, with nothing printing the date.**
Three instances in two days, all live:

    VK-GL-CTS's two build dirs   -> ~/src/mesa-ps4              the backup checkout
    cts.md's configure recipe    -> ~/src/mesa-ps4
    vkloader/CMakeLists.txt      -> ~/.cache/orbis-mesa/mesa    the patch-queue era

Each would have produced a build that ran perfectly and proved nothing. `orbis_announce_driver()`
now prints the archive and its build time before every configure, and the CTS entry point REFUSES
when its cmake cache names a different one.

⚠ **And the same shape kept appearing in the tooling itself - five times, all self-inflicted:**

    git push               exited 0 after disconnecting; the repository was empty
    deploy.sh              ran lftp with output to /dev/null under set -e: a refused transfer
                           ended the script between two lines, silently
    deploy.sh              read field 5 - the MONTH - as the byte count, and reported
                           "is sie bytes, sent 51314688" on a transfer that had succeeded.
                           Visible only because the locale is Polish; under English it would have
                           compared 51314688 against "Aug" and looked ordinary
    a ref verification     printed ZGODNE for all eight repositories without reading a single SHA
    a log search           reported "in none of them" against files that contain no MESA lines at all

The rule that catches all five: **make the check print what it measured**, not a verdict.

## Where the fifteen findings stand

Executed, built, and taken to hardware. Every failure found was attributed by a CONTROL run rather
than guessed at:

    CTS 735 cases x2 + control    baseline 140/140, zero verdict changes
    CTS 1170 conditional_rendering x2   verdict SETS identical between drivers
    OpenGothic 3328 frames        0 protection faults, 0 timeouts, ~50 fps as before

Three defects surfaced and all three are DEBT: `event.device_set_reset` (task #34), the three
conditional_rendering clusters (task #35), and the hung-submit warning at #1293, which the
maintainer identifies as long-standing.

⚠ **Still unmeasured on hardware**: 4.5's watchdog (no wait ever reached five seconds), 4.3's
EXEC_COUNT overflow (no CTS case emits ~357 predicated indirect draws), 4.1's Formats2, and 1.2
which is Windows-only and has never been compiled anywhere.

## State, and what is waiting

    mesa-ps4        b327f8afbc1   clean, pushed
    orbis-compat    7ba8720       3 uncommitted: orbis-env.sh, deploy.sh, vkloader/CMakeLists.txt
    ps4-mesa-docs   e915911       1 uncommitted: this file
    OpenGothic      6669da67      2 uncommitted: ps4/build.sh, ps4/tempest-env.example.txt
    VK-GL-CTS       b7463f0bc     clean, pushed
    Tempest         d4d05d2f      clean       ZenKit  39bf134  clean
    RetroArch       4920537011    1 uncommitted: ps4/build-core.sh - LEFT ON PURPOSE

⚠ **The console's clock is ~46 minutes AHEAD of the host.** Measured. FTP listings and local mtimes
are not comparable, and comparing them produced a confident wrong conclusion today.

Open, not decided: RetroArch's app build has no script (`TARGET=retroarch_orbis` by hand); the
`192.168.100.x` addresses are hardcoded in orbis-compat's scripts except deploy.sh; the residue
detector compares against the previous sample rather than the floor and cries in ~30% of teardowns;
and the review's own gap - its `build-support/` finder returned nothing and nobody has looked.

## What RetroArch found about this driver, 2026-08-23

RetroArch draws through RADV on the console, and `orbis-ports/RetroArch`'s own `ps4/HANDOFF.md`
carries four things that are OURS rather than its. Pulled across so they are not lost in another
project's log.

### 1. ⚠ Integer vertex attribute fetch is the standing suspect

Beetle PSX HW runs Spyro 3 through the libretro hardware-render callback - foreign graphics code
building its own pipelines on this driver, which had never happened before. 40 fps. **The static
scene is correct** - sky, terrain, castle, lighting, colours - **and every animated model is
shredded**, with one quad showing stripes of garbage where a texture belongs.

Three eliminations were already paid for, each removing a class:

    the core's own SOFTWARE renderer draws the same content correctly
        -> the geometry reaching the GPU is right; the GPU's reading of it is not. The CPU side is
           out entirely, interpreter included
    ORBIS_3D_LINEAR=1 and ORBIS_NO_TESS=1 applied
        -> no change. So this is NOT the tiling those exist to avoid
    internal resolution doubled
        -> no change. Different resolutions mean different pipeline variants, and a shader
           miscompile would have moved. ACO is not the first suspect

What is left is the attribute declaration - `rhi/rhi_lib_vulkan.c:6032-6038` declares seven, and
**four are integer formats**:

    2  R8G8B8A8_UINT         window
    3  R16G16B16A16_SINT     pal_x      <- palette
    4  R16G16B16A16_SINT     u          <- texture coordinates
    5  R16G16B16A16_UINT     min_u

A quad showing stripes instead of a texture is what wrongly fetched UVs look like. Integer vertex
formats are a thinly travelled path in any driver and more so on GFX7.

⚠ **This is answerable with the CTS already on this console, without RetroArch and without Spyro:**
`dEQP-VK.pipeline.*vertex_input*` tests attribute fetch per format and in combination. If one of
those four formats fails, the cause is named to the format. That is a cheap run and nobody has made
it - `VK-GL-CTS/ps4/build.sh` builds it and the case selection is one wildcard.

### 2. ⚠ ORBIS_TILE_MODE has no reader - the fourth such knob

Verified in this tree today, not taken on trust. One occurrence in all of `src/`:

    src/amd/common/ac_orbis_drm.c:4512
       getenv("ORBIS_TILE_MODE") ? getenv("ORBIS_TILE_MODE") : "unset",

It prints its own value into a log line. Nothing acts on it. And
`OpenGothic/ps4/tempest-env.example.txt:28` documents it as working - "clamp 2D tiling to 1D (2D
surfaces only)".

That file already carries a ⚠ about three knobs found readerless on 2026-08-21
(`ORBIS_NO_PREFETCH`, `ORBIS_MAX_INSTANCES`, `TEMPEST_PS4_KBD`) and the check that finds them:
`grep -rl 'getenv("NAME")' src/`. **The check was never re-run after that sweep**, and this is what
it would have caught. Either implement the clamp or delete the line - a knob that documents a
measurement it cannot make is the exact failure that sweep was for.

### 3. Vulkan teardown WORKS, and 2.2 finally has a test vehicle

Finding 2.2 reordered `wsi_headless_swapchain_destroy` so the pending frame is presented and the
scan-out taken down BEFORE the images are freed. Nothing in five CTS runs reached that path - WSI is
NotSupported on the CTS target, and the title only tears a swapchain down at exit.

RetroArch reaches it from a menu entry. Close Content destroys the video driver and rebuilds it:

    15:19:26.718  wsi/orbis: scan-out down after 493 flip(s)
    15:19:26.719  wsi/orbis: scan-out up - 1920x1080 ...

**One millisecond, clean, straight back up.** ⚠ That capture is from BEFORE the plan, so it does not
test 2.2's reordering - but it establishes that the path runs at all, repeatably, in one session,
with the log flowing. **That is the instrument 2.2 needs**, and it costs a menu entry rather than a
console restart.

### 4. The close-hang is not in the driver

Every title built against this Mesa hangs when the console closes it, and nobody knew whether RADV's
teardown was the thing that wedged. It is not: the teardown above is clean, and Close Content then
Quit exits cleanly with RADV live. So "RADV alive at kill time" is NOT the condition. What is left is
the combination with a loaded core. Not ours to fix, and worth knowing before anyone spends a day on
the driver for it.

### Also worth having: the zero-copy scan-out, measured over a long run

    19 456 frames at frame 16 683 us - the same 60 Hz the software driver holds
    the scan-out copy took 0 us for 8100 KiB (worst 27 us over 19 456 frames)

Zero-copy is confirmed on a workload that is not OpenGothic.

## The integer-attribute suspect has a name, and it is alignment rather than integers, 2026-08-23

RetroArch's finding (previous section) pointed at the four integer vertex formats in Beetle PSX HW.
Chasing it in this tree moved the suspect one step and made it precise. **The formats are not the
variable. The element size is.**

### One line in generic AMD code carries the whole assumption

    src/amd/common/ac_shader_util.c   is_fetch_size_safe()

    return (gfx_level >= GFX7 && gfx_level <= GFX9) ||
           (offset % vertex_byte_size == 0 && MAX2(alignment, 1) % vertex_byte_size == 0);

On GFX6 and on GFX10+, ACO splits a typed vertex fetch until every piece is naturally aligned. On
GFX7, GFX8 and GFX9 it declares **any** fetch size safe at **any** address, and never splits. That
arm is a claim about silicon: a Sea Islands part can fetch an 8-byte element from an address that is
a multiple of 4 but not of 8.

On a discrete CIK part the claim holds because amdgpu programs `SH_MEM_CONFIG.ALIGNMENT_MODE` to
UNALIGNED during init. ⚠ **On this console that register belongs to a kernel we do not write**, and it
sits in the same privileged window that already refused to give up `GB_ADDR_CONFIG`. So the claim is
inherited, not verified - and if it does not hold here the fetch returns the wrong bytes with no
fault, no log and no other symptom.

### Why that fits the picture exactly, where "integer formats" only fits loosely

The failure can only touch formats whose element is larger than its channel alignment. Everything
built from 32-bit channels is immune, because dword alignment *is* element alignment there. So a
mixed layout comes back with its float attributes correct and its 8-byte ones shredded - which is
Beetle's seven attributes, where the three `R32G32B32A32_SFLOAT` are fine and the three
`R16G16B16A16_*` are the ones under suspicion. `R8G8B8A8_UINT` is a 4-byte element and should be
immune too, which is a prediction this hypothesis makes and "integer formats are less travelled"
does not.

### Measured, on this laptop, with no console involved

`build-support/orbis/tools/vsfetchprobe.c` is new: it builds one graphics pipeline for a chosen
vertex format, offset and stride under the gfx7 drm-shim, and `RADV_DEBUG=asm` prints what ACO made
of it. Attribute 0 is `R32G32B32A32_SFLOAT` at offset 0, so the binding's known alignment is 4 -
which is what an interleaved float+integer layout produces.

    r16g16b16a16_sint at offset 36, stride 76      (4-aligned, NOT 8-aligned)

    default        tbuffer_load_format_xyzw  dfmt:16_16_16_16 nfmt:sint offset:36
    STRICT_ALIGN   tbuffer_load_format_xy    dfmt:16_16       nfmt:sint offset:36
                   tbuffer_load_format_xy    dfmt:16_16       nfmt:sint offset:40

One instruction against two, and the float attribute's `buffer_load_dwordx4` is untouched in both.
That is the difference the hypothesis is about, in the form it actually reaches the hardware.

### ORBIS_VS_STRICT_ALIGN=1 - the knob, and what it costs

It withdraws the GFX7-GFX9 exemption, so fetches split to natural alignment exactly as on GFX6.
Default off. Four places had to move together:

    ac_shader_util.c        is_fetch_size_safe() - the fetch itself
    radv_cmd_buffer.c       misalignment_possible in lookup_vs_prolog() - the prolog must agree with
                            the shader it is paired with
    radv_cmd_buffer.c       CmdSetVertexInputEXT's own copy of the same test
    radv_physical_device.c  the pipeline cache UUID

⚠ **The cache UUID is not optional.** The knob changes the instructions ACO emits, so without it in
the key a run with the knob on can be answered by the previous run's shaders - an experiment that
reports whatever it was told first, while reading as a clean measurement.

⚠ **And a fourth place that would have made it unreliable.** `CmdBindVertexBuffers2` decides whether
to recompute the alignment masks by comparing the low **two** bits of the address and the stride,
because upstream's strictest possible requirement is dword. An 8-byte element asks for three bits, so
a rebind that moved a buffer from 8-aligned to 4-aligned would have differed only in bit 2, the masks
would never have been recomputed, and the knob would have quietly stopped applying to that binding.
The comparison width now follows the knob.

**The honest cost:** the split happens whenever RADV cannot *prove* natural alignment, not only when
the address is actually misaligned - and it can only prove what the attribute offsets tell it, so a
binding holding both float and 16-bit-integer attributes is 4-aligned as far as it knows. Measured:
`offset 32, stride 80` - naturally aligned by arithmetic - still splits. One extra VMEM instruction
per oversized typed attribute. That is the price of the experiment and, if the hypothesis holds, of
the fix.

### ⚠ THE CTS CANNOT SEE THIS, AND THE PREVIOUS SECTION SAID IT COULD

`dEQP-VK.pipeline.*vertex_input*` was named as the cheap answer. Reading the test rather than
trusting the name:

    vktPipelineVertexInputTests.cpp:624
       getNextMultipleOffset(inputSize, bindingOffsets[b] + attributeOffsets[b])

Every interleaved attribute is aligned to **its own full size** before it is placed. The CTS never
builds the case under test. `legacy_vertex_attributes` does use unaligned layouts - and only
`attribute_offset_1`, `memory_offset_1`, `stride_1_byte`, all of which are ODD, so they trip the
*component* alignment check that RADV applies on every generation, and the workaround path runs.
Nothing in the suite puts an 8-byte element at a 4-aligned offset.

So the 1853-case run uploaded today establishes the baseline - that aligned fetch and the odd-offset
workaround both work on this driver - **and cannot confirm or refute the hypothesis**. Saying so here
because the previous section promised otherwise, and a promise like that is how a run gets read as an
answer to a question it never asked.

### What would settle it, and what it costs to get there

RetroArch reads `/data/retroarch-env.txt` as well as `/data/tempest-env.txt` (its commit
`100d4b1b0f`), so setting `ORBIS_VS_STRICT_ALIGN=1` and starting Beetle PSX HW on Spyro 3 is the
two-sided measurement: models come back and the class is named with the fix already written, or they
do not and a whole class is eliminated.

⚠ **But not with the RetroArch that is installed.** Every title on this console links
`libvulkan_radeon.a` statically into its eboot, so the knob does not exist in a binary built before
today - the env line would be read, `getenv` would return it, and nothing would be there to act on
it. **RetroArch has to be rebuilt against this driver first**, and its app build is 239 objects driven
by hand rather than by a script (`ps4/build-core.sh` is for cores only). That is the real cost of the
experiment and it should be stated before anyone sets the variable and reports a result.

The driver's log is what tells the two apart: `orbis: ORBIS_VS_STRICT_ALIGN=1 - typed vertex fetches
split to natural alignment` is printed once, and its absence means the knob did not reach the driver -
whether because the file was not read or because the binary predates the knob.

**The alternative, and it is arguably the better one:** add the missing case to our own CTS fork.
The suite is already ours and already runs on this console; a group that places an 8-byte element at
a 4-aligned offset and checks the values would answer the question with numbers rather than with a
picture, would cost one CTS build instead of a hand-driven frontend build, and would stay as a
permanent instrument for a case upstream does not cover on any generation.

## The CTS DID see it: 97 failures, one defect, 2026-08-23 (afternoon)

    1853 cases   Passed 1657   Failed 98   NotSupported 98
    the run ended on its own terms - the last block in the .qpa is closed

⚠ **AND THE SECTION ABOVE WAS WRONG ABOUT THIS.** It said the suite could not reach the case, having
read `getNextMultipleOffset(inputSize, ...)` and concluded that every attribute is naturally aligned.
The attribute OFFSETS are. What that reading missed is that the offsets then determine the **stride**,
and a stride that is not a multiple of the element size misaligns every other vertex on its own. Two
whole groups do exactly that, and both failed.

### 33 in single_attribute, and every one is a three-channel 8- or 16-bit format

    r16g16b16_sint  r16g16b16_uint  r16g16b16_sfloat  r16g16b16_uscaled  r16g16b16_sscaled
    r8g8b8_uscaled  r8g8b8_sscaled  b8g8r8_uscaled    b8g8r8_sscaled

Nothing else in the group failed. `r32g32b32_*` passed all eight variants; so did every 1-, 2- and
4-channel format.

**Why three channels is the case.** `ac_shader_util.c`'s table gives 8- and 16-bit formats
`has_hw_format` 0xb - one, two and four channels, never three. So a three-channel fetch has to be
split, and the stride is the format's own size: **6 bytes for `r16g16b16`, 3 for `r8g8b8`.** Vertex i
sits at 6i, so the first piece - a `tbuffer_load_format_xy`, a **4-byte** element - lands on a
2-aligned address for every odd vertex. `r32g32b32` has a hardware three-channel format, a 12-byte
stride, and 4-byte channels, so it is aligned at every vertex. It passes.

**And the two exceptions prove it rather than break it.** `r16g16b16_unorm` passed 8/8 and
`r16g16b16_snorm` 7/8 - the same layout, the same instructions, a different verdict. The difference is
the comparison:

    getFormatThreshold()  ->  1.5f * getRepresentableDifferenceUnorm(format)   =  1.5 / 65535

and the test writes component n of vertex v as the raw integer `totalComponents * v + n`. Two adjacent
components therefore differ by exactly **1.0** representable difference, which is inside a threshold of
1.5. ⚠ **A fetch shifted by one component passes UNORM and SNORM and fails everything else**, because
every other numeric interpretation is compared exactly. Those are not passes; they are a tolerance
wide enough to swallow the error. The one SNORM failure -
`mat3.as_r16g16b16_snorm_rate_vertex` - is the case where the shift crosses a matrix column and the
error is larger than one step.

### 64 in legacy_vertex_attributes, and every one has an odd stride

    single_binding.*_stride_1   *_stride_3   *_stride_7
    multi_binding.*_stride_1_byte, and the stride_normal cases with attribute_offset_1

⚠ **The `_attribute_offset_1` cases at normal stride PASS.** That is the distinction that matters: an
odd OFFSET trips the component-alignment check RADV applies on every generation, so the workaround
runs and the answer is right. An odd STRIDE does not - the component alignment is still satisfied,
while the ELEMENT alignment changes from vertex to vertex, and the only test for that is the one
GFX7-GFX9 skips. `r8g8b8a8_*` at stride 7: three vertices in every four are misaligned.

So the earlier reading had it exactly backwards - `legacy_vertex_attributes` is not blind to this, it
is the group that isolates it, and it does so on the axis nobody looked at.

### One failure left over

`misc.stride_change_vert_geom_frag` - a geometry-shader path, and the vert/frag, vert/tess and
vert/tess/geom variants of the same test did not fail. Not this defect. Untriaged.

### The 98 NotSupported are all legitimate

    50 + 34   sRGB vertex formats, which RADV does not advertise
    12        max_attributes above this device's limit
     2        misc, tessellation - ORBIS_NO_TESS=1 is in the env file

### What this makes of the morning's hypothesis

It stops being a hypothesis about Beetle PSX HW and becomes a measured property of this console:
**97 of 98 failures are multi-byte typed vertex fetches from addresses that are not multiples of the
element size.** The morning's reasoning about `SH_MEM_CONFIG.ALIGNMENT_MODE` is still unverified as a
MECHANISM - the register cannot be read here - but the behaviour it predicts is now on the record with
names and counts.

`ORBIS_VS_STRICT_ALIGN=1` is aimed exactly at it. Measured on the host for the CTS's own layout
(`vsfetchprobe r16g16b16_sint 0 6 split`):

    default        tbuffer_load_format_xy  dfmt:16_16  offset:0     <- 4-byte element, 2-aligned on odd vertices
                   tbuffer_load_format_x   dfmt:16     offset:4
    STRICT_ALIGN   tbuffer_load_format_x   dfmt:16     offset:0
                   tbuffer_load_format_x   dfmt:16     offset:2
                   tbuffer_load_format_x   dfmt:16     offset:4     <- every piece 2-aligned

⚠ **The probe had to be corrected before it measured the right thing.** Its first version always put
the `R32G32B32A32_SFLOAT` attribute on the same binding, which tells RADV the binding is 4-aligned and
makes the two-channel fetch look safe even under the knob. That is Beetle's layout, not the CTS's. The
tool now takes `same|split` and prints which it used, because the two compile differently and a run
that does not say which one it was is not evidence for either.

**The experiment is now one console run**, on the same 1853 cases, with the knob on. If it is this,
the 97 become passes and nothing else moves.

## CONFIRMED ON HARDWARE, and it is now the default, 2026-08-23 (late afternoon)

Same 1853 cases, same case list, same binary, one environment line apart:

    believing the GFX7 exemption   Passed 1657   Failed 98   NotSupported 98
    ORBIS_VS_STRICT_ALIGN=1        Passed 1754   Failed  1   NotSupported 98

Compared case by case rather than by count, because a count is what hid non-determinism twice on
this project:

    Fail -> Pass          97
    Pass -> anything      0
    any other change      0

The driver printed its line once, so the knob reached it. The single remaining failure is
`misc.stride_change_vert_geom_frag`, the geometry-shader case that was already separated out - a
different defect, still untriaged.

**So the claim in `is_fetch_size_safe()` is false on this console.** A Sea Islands part here cannot
fetch an N-byte vertex element from an address that is not a multiple of N, and it fails at it
silently. That is no longer a hypothesis about `SH_MEM_CONFIG.ALIGNMENT_MODE`; the mechanism is still
unread, but the behaviour is measured, named and counted.

### The default is now ON, and only here

`ac_orbis_strict_vtx_align()` returns true unless `ORBIS_VS_STRICT_ALIGN=0`, and only under
`HAVE_ORBIS_PLATFORM`. Every other AMD part this file compiles for - including the drm-shim that
pretends to be one - has a kernel driver that programs the alignment mode, so upstream's claim holds
for them and splitting would be pure cost. Verified three ways on the host:

    build-host      (no HAVE_ORBIS_PLATFORM)   unset -> upstream    =1 -> split
    build-hostorbis (HAVE_ORBIS_PLATFORM)      unset -> split       =0 -> upstream

⚠ **The OFF state is now the dangerous one, so it logs a warning rather than nothing.** A run made
with wrong vertex data has to be identifiable from its own log, not from somebody's memory of which
environment file was in place.

### What it costs OpenGothic: nothing, and that is measured rather than assumed

`can_use_untyped_load()` sends every format whose channels are 32 bits wide to an untyped
`buffer_load_dword*`, which has no format alignment rule at all. Only sub-32-bit typed formats reach
`is_fetch_size_safe()`. Measured with the probe on the orbis ICD, both knob states:

    r32g32b32_sint     buffer_load_dwordx3    identical with the knob on and off
    r32g32b32a32_sint  buffer_load_dwordx4    identical with the knob on and off

And `Tempest/Engine/gapi/abstractgraphicsapi.h`'s `Decl::ComponentType` has **no sub-32-bit member at
all** - float1..4, int1..4, uint1..4, every one of them 4 bytes per channel. The engine cannot
declare a vertex attribute that takes this path, so the title pays zero instructions. A confirmation
run is still worth one console trip, because the change recompiles every pipeline in the driver, but
there is no frame cost to look for.

Where it does cost: sub-32-bit typed attributes, and the split happens whenever RADV cannot PROVE
natural alignment rather than only when the address is misaligned - all it knows at compile time is
the widest attribute alignment in the binding. A binding of 16-bit attributes is known 2-aligned, so a
four-channel 16-bit format becomes four one-channel fetches. Beetle PSX HW's layout is exactly that
shape, and it is the workload where the cost should be measured if anyone wants the number.

### What this closes and what it opens

Closes: the RetroArch integer-attribute artefact has a named cause and a shipped fix, without ever
loading Spyro. The prediction that `R8G8B8A8_UINT` was innocent and the three `R16G16B16A16_*` were
not is what the CTS data says too - 4-byte elements passed, larger ones failed.

⚠ Opens: **every other place this port inherits a "GFX7 can do X" claim from upstream.** This one was
found because a foreign renderer drew something wrong and the CTS was pointed at it. There is no list
of the others, and the tool that would build one is a grep for `gfx_level >= GFX7 && gfx_level <=
GFX9` and its relatives across `src/amd/`.

### RetroArch rebuilt on the fixed driver, and one Makefile defect found doing it

The frontend links `libvulkan_radeon.a` statically like every other title, so the installed build
carried the old fetch code - `strings` on the 2026-08-22 ELF finds 2730 RADV markers and zero
occurrences of the knob. Rebuilt with `make -f Makefile.orbis HAVE_VULKAN=1 pkg`.

⚠ **AND THE FIRST ATTEMPT PRODUCED A PACKAGE THAT WAS NOT THE REBUILD.** `create-fself` reads
`OO_PS4_TOOLCHAIN` from the environment and refuses without it, although it is invoked by absolute
path out of that very toolchain. `Makefile.orbis`'s `pkg` target passes the variable; its `eboot.bin`
target did not. So the build compiled 239 objects, linked a new `.elf`, failed at status 255, and left
**the previous day's `eboot.bin` and `.pkg` sitting beside it** - the same size as always, ready to be
uploaded as "the rebuild". It had only ever worked because the shell that ran make happened to export
the variable. One line, fixed, uncommitted in the RetroArch fork.

The check that caught it was the same one the CTS build script exists for: read the timestamp of the
artefact you are about to ship, not of the one the build started from.

    retroarch_orbis.elf   12:39   knob present
    eboot.bin             12:40   knob present
    .pkg                  12:40   41680896 bytes - byte-identical in size to yesterday's

### ⚠ AND THE PACKAGE THAT FIXED THAT ONE HAD NO CORES

The rebuild was invoked as `make -f Makefile.orbis HAVE_VULKAN=1 pkg`, because `HAVE_VULKAN=1` is the
only flag the file's own comments put in a command line. It is not the configuration the console was
running. Loading a `.prx` core needs all three:

    HAVE_VULKAN=1  HAVE_STATIC_DUMMY=0  HAVE_DYNAMIC=1

⚠ **Nothing about the result says so.** The ELF is the same size either way, the `.pkg` came out
41680896 bytes for the fourth day running, it installed, it booted, it drew - and the core list was
empty. That reads as a lost core directory, a permissions problem or a stale info cache, and sends
whoever is looking anywhere but at the build command. The evidence that settles it is one grep:

    sceKernelLoadStartModule    static build 0    dynamic build 3
    libretro_dummy              25 either way, so it is NOT the marker to look for

⚠ **And it left damage behind.** The static build then wrote
`/data/retroarch/info/core_info.cache` - 65 bytes, decompressing to `{"version": "1.2", "items": []}`.
That cache would have kept the menu empty across every later install until somebody deleted it by
hand, which is exactly what `RetroArch/ps4/HANDOFF.md` warns about and what nobody would have
connected to a driver experiment. Backed up and removed; the three `.info` files and the three `.prx`
cores beside it were untouched and are from 2026-08-22.

**The durable part.** `Makefile.orbis`'s link step now prints what it built, every time:

    == retroarch_orbis.elf
       cores:    .prx modules via sceKernelLoadStartModule
       dummy:    HAVE_STATIC_DUMMY=0
       vulkan:   RADV linked from .../libvulkan_radeon.a
       driver:   built 2026-08-23 11:46:19 - 36141588 bytes

and the wrong configuration announces itself as `STATIC DUMMY ONLY - the core list will be empty`.
Same rule as `VK-GL-CTS/ps4/build.sh` and `orbis_announce_driver`: the build says what it made, with
the date of the driver it linked, because none of it is visible in the artefact.

⚠ Adding that check cost two more builds, both to make's own expansion rules. `$(if ...)` splits its
arguments on the FIRST comma, and the line to print contained one - the recipe was truncated inside an
open quote and the link failed **after** writing the ELF. Then a shell comment inside the recipe that
merely NAMED `$(if ...)` was expanded as a call to it and stopped the parse. Both are recorded above
the rule in the Makefile.

**What to expect from Spyro, stated before the run.** The prediction the CTS data supports is that
attribute 2 (`R8G8B8A8_UINT`, a 4-byte element) was never affected and attributes 3, 4 and 5
(`R16G16B16A16_*`, 8-byte elements) were - if their offsets in the vertex are dword- but not
eight-aligned. **Those offsets have not been read**; they were inferred from tight packing of the
seven attributes, and the core's source is not in this workshop. So:

    models come back        the layout is what it was inferred to be, and the defect is closed
    models still shredded   the offsets are eight-aligned after all, and this was never Beetle's
                            problem - which still eliminates a class, at the cost of one run

Either way the driver log settles whether the knob was live: `orbis: vertex fetches are split to
natural alignment` in `/data/retroarch-mesa.log`.

## A ladder for mirrored direct memory, because the dynarec's wall may not be one, 2026-08-23 (evening)

The RetroArch profile settled where Spyro's frame goes, and it is not here:

    frame 43967 us (23 fps)     everything inside the Vulkan API   263 us   = 0.6%
    23 draws, 4 render passes, 1.68 screenfuls of 1920x1080, 2 dispatches
    97 five-second windows: menu 0.05 cores at 60 fps, scene 1.00 cores at 40 and at 23 fps
    time waited for the GPU, in every window without exception: 0 ms

One CPU thread saturated, the GPU never waited on, this driver at 0.1-0.4% of the window. That is the
MIPS interpreter, and the audio running slow follows from it rather than from anything in the audio
path - the core is at 38% of realtime, so the samples come out at 38% of the rate.

`RetroArch/ps4/HANDOFF.md` names why the interpreter: Lightrec wants the emulated machine's RAM
visible at several addresses at once, builds that with `memfd_create`/`MAP_SHM`, and "neither exists
here". ⚠ **That is a true statement about POSIX shared memory and not about this platform.** The
direct-memory API this driver uses on every single run has the shape the requirement needs:

    sceKernelAllocateDirectMemory(..., &phys)                       physical pages
    sceKernelMapDirectMemory(&va, len, prot, flags, phys, align)    a view of them

Nothing in it forbids a second call with the same `phys`. And `ac_orbis_drm.c:6464` already maps
direct memory **at an address of our choosing** with `ORBIS_MAP_FIXED`, on hardware, every time a BO
moves to GARLIC - so placement, the other half of what Lightrec needs, is not in question either.

### What is in question, and the five rungs that ask it

`ORBIS_TEST_MIRROR=1`, `orbis_test_mirror_mapping()`. It runs once at device init, takes 2 MiB,
releases it, and touches nothing else. Each rung is meaningless without the one before it, and each
prints what it measured rather than a verdict:

    rung 0   allocate, map once, write two words 2 MiB apart, read them back through that view.
             The harness. If this fails, nothing below the line says anything.
    rung 1   map the SAME phys again. Return code, both addresses, and whether they differ - a
             kernel that hands back the existing view has not made a mirror.
    rung 2   coherence BOTH WAYS. Write through view 0, read through view 1; then write a different
             word through view 1 and read it through view 0. ⚠ One direction is not enough: a
             copy-on-map passes forwards and fails backwards, and would otherwise be reported as a
             mirror.
    rung 3   how MANY simultaneous views before the kernel refuses. Lightrec wants about four, and
             "more than one" is a different answer from "as many as you like".
    rung 4   independence: unmap one view, check the survivors still hold the pattern. Mirrors that
             die together cannot be managed separately.

⚠ **And it refuses to answer on the laptop.** The real ladder is inside `#if defined(__PS4__)`; a
second definition outside it exists only to say `NOT RUN. Ask a console.` when the variable is set on
the host arm. Without that stub the function would simply not exist there, the run would print
nothing, and silence reads as "found nothing" rather than "was not there".

### What each outcome is worth

    rungs 0-4 pass    the wall is not a wall. Lightrec's remaining requirements are its own -
                      executable pages for the code buffer, which this ladder deliberately did not
                      ask about - and the interpreter stops being the only option.
    rung 1 refuses    one phys, one view, measured, with a return code. Nobody spends a day
                      rediscovering it, and the dynarec question is closed on evidence.
    rung 2 fails      the second mapping is a copy. Worse than a refusal, because it LOOKS like a
                      mirror to any test that only checks one direction.

Three builds verified: cross clean, `--host-orbis` clean and refusing out loud, plain Linux clean.

⚠ Building the ladder cost three failed cross builds, all placement rather than logic: it first
landed above the constants it uses, then inside another function's doc comment, then inside the
`#if defined(__PS4__)` region so the host arm lost it entirely and the call site got an implicit
declaration. Each was caught by the compiler. The one that would NOT have been caught is the third
in its other direction - a host stub that existed on both arms - and that showed up as a redefinition
only because the cross build was run again rather than assumed.

### ANSWERED ON HARDWARE: mirrored direct memory works, all five rungs

    MIRROR ladder - 2048 KiB of direct memory, prot 0x03, align 2048 KiB
    rung 0: phys 0x48200000
    rung 0 OK - view 0 at 0x248400000, holds what was written at offset 0 and at offset 2032 KiB
    rung 1 OK - view 1 at 0x248600000, 2048 KiB from view 0
    rung 2 OK - both directions, at offset 0 and at offset 2032 KiB. The two views are the same memory
    rung 3: 8 simultaneous view(s) of one phys, AND IT DID NOT REFUSE
    rung 4 OK - unmapped 0x249200000, the remaining 7 view(s) still hold the pattern

**One physical range, eight live virtual views, coherent in both directions, independently
unmappable.** `RetroArch/ps4/HANDOFF.md`'s "neither exists here" was true of `memfd_create`/`MAP_SHM`
and false of the requirement they were being used to meet.

⚠ **Eight is a floor, not a limit.** The ladder stops asking at eight; the kernel had not refused. The
number that matters is four, so the question is answered either way, but nobody should quote eight as
a measured maximum.

Worth noting: the two views landed at consecutive 2 MiB addresses, `0x248400000` and `0x248600000`.
Together with `ORBIS_MAP_FIXED` - which `ac_orbis_drm.c:6464` already uses on hardware every time a BO
moves to GARLIC - a contiguous mirrored region at addresses of our choosing is constructible.

### What is still unasked, and it is one question

The code buffer. A recompiler writes instructions and then executes them, and this ladder only ever
asked for `prot 0x03` - CPU read and write. Two candidate routes, and they are not equal:

    ORBIS_KERNEL_PROT_CPU_EXEC (VM_PROT_EXECUTE, 0x04) on a direct-memory mapping
        one more rung, the same shape as the five above, and it either returns 0 or it does not.

    sceKernelJitCreateSharedMemory / sceKernelJitCreateAliasOfSharedMemory /
    sceKernelJitMapSharedMemory / sceKernelJitGetSharedMemoryInfo
        ⚠ the SDK declares all four as `void f();` - symbols with no argument shapes at all. That is
        the same starting position as sceGnmSetHsShader in the tessellation work, and that one cost
        days and is still open. The name "create alias of shared memory" is exactly the mechanism, so
        the route is attractive, but the cost is reverse engineering rather than a rung.

Ask the cheap one first. If a direct-memory mapping can be made executable, Lightrec needs neither
the Jit API nor POSIX shm, and the interpreter stops being the only option on this console.

### Rung 5: the code buffer, split three ways because only one of them is dangerous

    5a  map the SAME phys with prot 0x05 - CPU read | CPU EXECUTE. A return code, nothing runs.
    5b  read the six instruction bytes back THROUGH the executable view, after writing them through
        the writable one. Free, non-fatal, and it separates "the mirror carries code" from "the page
        will execute" - two claims a single jump answers together and therefore answers badly.
    5c  CALL it. ⚠ A page that is not really executable does not return an error here; it takes the
        process down. So 5c is behind its own value: ORBIS_TEST_MIRROR=exec. With =1 the ladder
        stops after 5b and prints what to set.

The stub is six bytes and returns a value nothing else in this process produces:

    b8 ee ff c0 00    mov eax, 0x00c0ffee
    c3                ret

written through the WRITABLE mirror and called through the EXECUTABLE one - which is the W^X shape a
recompiler actually wants, rather than one mapping carrying both rights. x86-64 needs no cache
maintenance for self-modifying code, so a wrong answer here is about mapping and not about coherency.

⚠ **5a SUCCEEDING IS NOT AN ANSWER.** This console's kernel can accept a protection and not honour
it; that is the same lesson `GB_ADDR_CONFIG` and the tessellation registers already charged for,
where a call returned zero and the hardware disagreed. Only 5c is evidence about execution, which is
precisely why it is the rung that costs something.

⚠ **And 5c says so before it jumps.** `logger_file()` in `src/util/log.c` calls `fflush(fp)` after
every message, so the line naming the address reaches the file first. If it is the last MIRROR line
in the log, the mapping was granted READ|EXECUTE and did not execute - **that is the result, not a
crash to be investigated.**

Shipped with `ORBIS_TEST_MIRROR=1` on purpose: if 5a is refused, 5c never matters and nothing was
risked. Turning it on afterwards is one line in `/data/tempest-env.txt` - no rebuild, no reinstall.

If 5a is refused, the remaining route is `sceKernelJitCreateSharedMemory` and
`sceKernelJitCreateAliasOfSharedMemory`, which OpenOrbis declares as `void f();` - no argument shapes
at all. That is the same starting position as `sceGnmSetHsShader` in the tessellation work, which
cost days and is still open, so it is reverse engineering rather than another rung.

### Rung 5 answered: EACCES. Direct memory will not carry executable pages

    MIRROR rung 5a: a READ|EXECUTE view of phys 0x48200000 was REFUSED -> 0x8002000d

`0x8002000d` is `ORBIS_KERNEL_ERROR_EACCES` (`orbis/_types/errors.h:19`). ⚠ **EACCES and not EINVAL,
and the difference is the whole content of the result:** the call was well formed and the kernel
understood it. This is a policy gate on executable direct memory, not a malformed request, so no
amount of fiddling with the argument shape will move it.

5b and 5c never ran, which is what the split was for - nothing was risked to learn this.

### ⚠ AND THE QUESTION IS SMALLER THAN THIS SECTION HAS BEEN TREATING IT

Two requirements were tangled together above, and they are separable:

    the emulated machine's RAM   must be MIRRORED     -> works, 8+ coherent views, rung 1-4
    the recompiler's code buffer must be EXECUTABLE   -> refused on direct memory, rung 5a

**The code buffer does not need to be mirrored.** It needs to be executable, anywhere. And this
process is not categorically barred from executing memory - its own `.text` runs. Only direct memory
is refused.

So the remaining question is "can this process obtain executable pages by ANY route", and there are
three cheap ones left before the expensive one:

    sceKernelMprotect adding EXEC to an already-mapped RW direct range
        a different syscall and a different policy check. This driver already uses mprotect on direct
        memory and established it reaches the GPU's page tables (the 2026-08-?? mprotect ladder), so
        the call is known to work there for other bits.
    sceKernelMapFlexibleMemory with an exec protection
        a different pool entirely, with its own accounting and plausibly its own policy.
    sceKernelMmap with MAP_ANON and PROT_EXEC
        the plainest form, and the one musl's own allocator already uses without exec.

Only if all three are refused does the Jit API become the route, and that one has no argument shapes
in the SDK at all.

**Nothing about the mirror result is diminished by this.** Mirrored RAM was the claim recorded as
impossible, and it is possible. The code buffer is a second, separate lock, and three keys have not
been tried.

### Rung 6: three routes to an executable page, none of them direct memory

    6a  sceKernelMprotect adding EXEC to a range ALREADY mapped read-write
    6b  sceKernelMapFlexibleMemory        - a different pool, its own accounting and policy
    6c  sceKernelMmap MAP_PRIVATE|MAP_ANON - the plainest form there is

⚠ **Each is asked twice: RWX (0x07) first, then RX (0x05), and BOTH codes are logged.** A route that
refuses RWX and grants RX is a W^X platform and a perfectly good answer; one that refuses both is a
closed door. Collapsing the two attempts into one would turn the first case into the second and
report a working platform as a dead end.

6a is worth its place because it is a **different syscall from the one that refused**, and this
driver already establishes that `sceKernelMprotect` on direct memory reaches even the GPU's page
tables - so the call works there for other bits, and only this bit is in question.

⚠ **Nothing in rung 6 executes.** A granted protection is not an honoured one; that was rung 5's
whole structure and it holds here. Rung 6 collects return codes and addresses and names which route
the jump should be pointed at. Turning the jump on stays a deliberate second run behind
`ORBIS_TEST_MIRROR=exec`.

Refusals now print a name as well as a number - `orbis_kernel_error_name()`, from
`orbis/_types/errors.h`. **EACCES against EINVAL is the distinction that carries the meaning**:
understood-and-refused against malformed-and-never-reached. Rung 5a's `0x8002000d` was the entire
result, and reading it needed a header open beside the log.

If all three refuse, the door is `sceKernelJitCreateSharedMemory` and its alias call - `void f();` in
the SDK, no argument shapes - and a recompiler on this console goes through it or not at all.

### Rung 6 answered: two routes grant RWX, and the third was my own bug

    6a sceKernelMprotect on an already-mapped RW direct range   prot 0x07 -> GRANTED at 0x248400000
    6b sceKernelMapFlexibleMemory                                prot 0x07 -> GRANTED at 0x249200000
    6c sceKernelMmap MAP_ANON|MAP_PRIVATE      prot 0x07 and 0x05 -> 0x80020009 both times

⚠ **The striking one is 6a: the SAME direct-memory pages that could not be MAPPED read-execute can
be MPROTECTED to read-write-execute afterwards.** `sceKernelMapDirectMemory` refused with EACCES at
map time; `sceKernelMprotect` granted the identical rights on the identical pages a moment later. The
policy is applied where the mapping is created and not where its protection is changed. Whether that
is a deliberate gate or an oversight is not something this ladder can say - but a recompiler only
needs it to be true, and it is.

⚠ **AND 6c WAS NOT A REFUSAL, IT WAS MY MISTAKE.** `0x80020009` is EBADF. The rung passed `0x20` for
`MAP_ANON`, read straight out of `$OO_PS4_TOOLCHAIN/include/sys/mman.h` - which is **musl's header
carrying Linux's value**. This kernel is FreeBSD-derived and wants `0x1000`, so the flags word was
not anonymous, the kernel went on to validate `fd = -1`, and refused the descriptor. Both protections
returned the identical code, which is the signature of a call that failed before it ever looked at
prot - and it read as "executable pages are refused" while being nothing of the kind.

The value is `0x1002`, and `orbis-compat/src/orbis_mmap.cpp:53` carries a `static_assert` for exactly
this. ⚠ **The mistake was reading a header with `sed` instead of asking the compiler**; the overlay
sits ahead of the SDK in the include path and corrects that constant, which `clang -dM` says in one
line. The rung now passes the symbols and logs the numeric value it used.

⚠ Three protection words in this file are still bare numbers - `0x03`, `0x05`, `0x07` for
CPU read/write/execute. Those come from `VM_PROT_*` in the SDK's own `_types/kernel.h` and agree with
`ORBIS_GRAPHICS_PROT`'s low nibble, so they are cross-checked rather than assumed - but they are the
same shape of hazard and the next person to touch them should ask the compiler too.

### Rung 7: run something out of the page rung 6 was given

The platform has now said yes to a question about PERMISSIONS twice and never once to a question
about EXECUTION, and this console's kernel has form for accepting a value and not honouring it -
`GB_ADDR_CONFIG`, the tessellation registers, and rung 5a's own map-time refusal against rung 6a's
protect-time grant.

Rung 7 puts the same six bytes into whichever buffer rung 6 obtained and, under
`ORBIS_TEST_MIRROR=exec`, calls it. If the grant was read-execute rather than read-write-execute it
does the **W^X dance** - `sceKernelMprotect` back to RW, write the stub, protect forward again - which
is what a recompiler does anyway, so performing it here measures the thing rather than avoiding it.

Shipped at `=1`: the read-back runs, the call does not, and the log says which line to change.

## The dynarec wall is gone, and every piece of it is measured, 2026-08-23 (close)

    MIRROR rung 7: CALLING 0x248400000 NOW, from 6a ... at prot 0x07
    MIRROR rung 7 OK - code placed in a page from 6a and CALLED returned 0x00c0ffee

Six bytes written into direct memory through an ordinary read-write mapping, that mapping
`sceKernelMprotect`ed to read-write-execute, and then **called**. It returned the value, the ladder
finished, and the title carried on.

### The three requirements, and what each one now rests on

    mirrored RAM for the emulated machine
        8 simultaneous views of one phys, coherent in BOTH directions at two offsets 2 MiB apart,
        and unmapping one leaves the rest intact. Rungs 1-4.
    each mirror at an address of our choosing
        ORBIS_MAP_FIXED, and this is not new - ac_orbis_drm.c:6464 does it on hardware every time a
        BO moves to GARLIC.
    an executable code buffer
        map read-write, then mprotect to 0x07, then execute. Rung 7, and it RAN.

⚠ **THE RECIPE MATTERS AND THE OBVIOUS FORM OF IT DOES NOT WORK.** Asking
`sceKernelMapDirectMemory` for read-execute up front is refused with EACCES (rung 5a). The same
pages, mapped read-write and then `sceKernelMprotect`ed, are granted and honoured. The policy lives
at map time, not at protect time. Anyone who tries the direct form first will conclude this is
impossible, because that is exactly what this ladder concluded one rung before it was true.

⚠ **Only route 6a was executed.** `sceKernelMapFlexibleMemory` and `sceKernelMmap` both granted
0x07 as well, and neither was called - granted is not honoured, which is the whole reason rung 7
exists. If the recompiler wants its code buffer outside direct memory, that is one more jump to
measure, not an assumption to inherit.

### What this costs the RetroArch record

`RetroArch/ps4/HANDOFF.md` states that Beetle PSX HW runs the MIPS interpreter because Lightrec needs
mirrored RAM and "`memfd_create`/`MAP_SHM` neither exists here". Both halves of that are now
answered: the mechanism exists by another name, and the code buffer works too. **That file should be
corrected rather than left to send the next person down the same dead end** - the note is not wrong
about POSIX shared memory, it is wrong about the conclusion drawn from it.

None of this makes Lightrec build. It removes the reason recorded for not trying, and the reason was
the whole of the case. What remains is ordinary porting work against a platform whose three relevant
mechanisms are now measured instead of assumed - and the prize is the profile from this morning:
one CPU core saturated, 0.6% of the frame in Vulkan, 0 ms waiting for the GPU.

### ⚠ Every rung of this ladder was needed, including the ones that were wrong

    rung 5a  refused, EACCES - and it would have been read as "impossible" without rung 6
    rung 6c  refused, EBADF - MY error, the SDK's musl MAP_ANON where FreeBSD's was wanted.
             Two protections returning the IDENTICAL code was the tell: a call that failed before it
             looked at prot. Corrected, and then it granted like the other two.
    rung 7   the only one that asked about EXECUTION rather than about PERMISSION, and the only
             one whose answer could not have been guessed from the others.

The split between "granted" and "honoured" is what this console has charged for three times now -
GB_ADDR_CONFIG, the tessellation registers, and rung 5a against rung 6a. It is worth building into
the shape of any probe written here.


---

## 2026-08-28 — the close-hang, handed over from the RetroArch port with the frontend eliminated

Written into this file by the RetroArch workshop (`~/src-ps4/RetroArch`, `ps4/HANDOFF.md`, entry of
the same date) because the answer was measured there and the remaining work is here.

**Every title built against this Mesa ends with CE-34878-0. A RetroArch built without the driver
exits silently.** Same code, same `main()`, same `exit(0)`. Four runs on hardware narrowed where in
the shutdown that happens, and the answer eliminates the application entirely.

### What the process survives, measured with markers in the frontend

    frontend_driver_shutdown            reached
    main_exit returned, main() ending   reached
    __funcs_on_exit entered             reached
    EVERY atexit handler returned       reached - util_queue's thread join included
    .fini_array entered                 reached

All of it inside two milliseconds, then the dialog.

⚠ **AND STATIC DESTRUCTORS ARE IN THAT LIST.** Clang registers them with `__cxa_atexit` from a
constructor in `.init_array`, so Mesa's and ACO's global objects are destroyed inside
`__funcs_on_exit` - the list that completed.

⚠ **THE DECISIVE RUN: `_Exit(0)` INSTEAD OF RETURNING FROM main().** That skips `.fini_array`,
`__stdio_exit` and every remaining step of musl's `exit()`, and goes straight to the kernel. **It
died exactly the same way.** There is no step left between the last line the application can write
and the dialog.

Two hypotheses died on the way, both plausible and both wrong: `util_queue`'s atexit thread join
(`src/util/u_queue.c:83`), and `MESA_LOG_FILE` - the one stdio object linking Mesa adds, a `FILE*`
opened with `fopen` and never closed. Removing the log file changed nothing.

### Where that leaves it, and the qualifier worth reading

The driver's own teardown reports itself clean:

    wsi/orbis: scan-out down after 4703 flip(s)
    orbis-drm: at teardown 0 BO(s), 0 syncobj(s), 0 context(s), 0 VA range(s), 0 KiB held
               - winsys-lifetime, and it did not grow

⚠ **"winsys-lifetime" is the qualifier, and it says nothing about allocations of other lifetimes.**
The RetroArch workshop's own record already establishes that **direct memory is not reclaimed when
a mapping goes away** (2026-08-23, from the Lightrec port). Physical pages taken with
`sceKernelAllocateDirectMemory` and still held when the process ends are the shape that makes a
process's death the system's problem rather than its own - which is the other half of the symptom,
because Close Application from outside wedges the console rather than freeing it.

`wsi_orbis_release()` does call `sceVideoOutClose` and does release its own direct memory when it
owns the buffers, so the scan-out path is not the obvious offender. **The audit is of everything
else this driver takes from the kernel, and of what is still held at `_Exit`.**

### One platform fact found on the way, for the collection

`access()` is refused on this console. Measured beside its own control, same path, same instant:

    access() = -1, errno 1 (EPERM)        stat() = 0

It is a real `libkernel.so` export (`T access`), while the SDK's `libc.a` ships `access.lo` as an
empty object and `faccessat.lo` as a stub returning ENOSYS. Sixth member of the family after
`fcntl(F_SETFL)`, `ioctl(FIONBIO)`, `dup2` onto fd 2, `sceKernelGetCpuUsage` and
`sceKernelQueryMemoryProtection`. Presence is not permission.
