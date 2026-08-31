# The running account of the PS4 port

Newest entry last. Everything here is an account of work; nothing in the build reads this file.
The plan it is executing is `ps4/PLAN.md`.

---

## 2026-08-22 — Phases 0 and 1, in one sitting

`ps4-base` = c59b1833f7, the tree before any of this. Work on `ps4-support`.

### What was expected, and what happened

The plan gave Phase 1 — "compile all 261 objects and read the link errors" — 2 to 6 sessions, and
called it the only real unknown in the whole estimate. **It cost nothing.** The build compiles 239
objects with **zero warnings** and links, and the only symbols missing at the end were four dangling
driver-table entries and one function deleted from the tree in 2021.

Not one libc or POSIX shim had to be written. RetroArch touches `stat`, `mmap`, `dirent`,
`pthread`, `clock_gettime`, POSIX timers, `open`/`rename`/`unlink` and sockets, and
`orbis-compat` already answered all of it — the overlay was built for a game engine, a Vulkan
driver and a conformance suite, and a frontend turns out to need nothing those three did not.

**This collapses the estimate.** PLAN.md §5 put the whole software leg at 9-16 sessions with the
spread almost entirely in Phase 1. That spread is gone.

### What actually had to change

Five things, none of them a shim:

1. **The Makefile.** Rewritten against OpenOrbis; see the commit. Its OBJ list had also been empty
   since 2021 (a `\` continuation running into a `#` line).
2. **`verbosity.h`** put every `RARCH_LOG` on `debugNetPrintf`. Now on the overlay's channel, and
   split: DBG/LOG/WARN over UDP only, ERR also on klog. klog costs 8-15 ms a line on this console.
   The `_V` variants were passing a `va_list` to a variadic function as an ordinary argument and
   printed garbage at every call site; they format now.
3. **`platform_orbis.c`** included ten orbisdev headers. Rewritten against `<orbis/*.h>`.
4. **`mem_stats.c`** called `get_user_mem_size()`, deleted in 2021 and replaced by an external
   `-luser_mem_sys` the link line still named. `ps4/ps4_mem.c` answers out of flexible memory.
5. **Four driver tables** named drivers that cannot build here.

### It boots

Under `unemups4`, from a plain `.elf`:

```
[retroarch] alive - main() entered, built Aug 22 2026 01:01:04
[SYSCALL] sceKernelOpen('/app0/ps4-run.cfg', ...)
[USER_SERVICE] sceUserServiceInitialize
[ERROR] [Config] Config not found at: "/data/retroarch/retroarch.cfg".
[INFO] RetroArch 1.22.2 (Git c81e53c9af)
[INFO] Capabilities: MMX MMXEXT SSE SSE2 SSE3 SSSE3 SSE4 SSE42 AES PCLMUL
```

The log channel works, the run config is read, UserService comes up, the config system runs and
says what it could not find, and RetroArch prints its own banner. Then it creates its first thread
and stops.

### ⚠ The emulator leg is blocked, and the port is not what is wrong

`unemups4` kills the process on the first call to `pthread_attr_get_np`, and behind it are
seventeen more its linker reports as `Stubbed missing`:

```
cpuset_getaffinity  getrlimit  mprotect  msync  _nanosleep
ktimer_{create,delete,getoverrun,gettime,settime}
pthread_attr_{getdetachstate,setdetachstate,getguardsize,setguardsize,setschedpolicy}
pthread_cancel  pthread_setcancelstate  pthread_set_name_np
```

**All eighteen are real PS4 exports** — every one of them is in `unemups4`'s own
`data/ps4_names.txt`, the NID name database. So this is not a port that calls something the console
does not have; it is an emulator that has not implemented what the console does. Nothing in
RetroArch or `orbis-compat` should be contorted to avoid these.

Several are one word of work in unemups4, because the Sce-spelled twin is already implemented and
the POSIX spelling merely is not in its `names = [...]` list: `scePthreadAttrGet`,
`scePthreadSetName`/`scePthreadRename`, `scePthreadCancel`, the detachstate and schedpolicy
setters, `SYS_NANOSLEEP`. The rest (`mprotect`, `msync`, the `ktimer_*` family,
`cpuset_getaffinity`, `getrlimit`, the guardsize pair, `pthread_setcancelstate`) have no
implementation there at all and want real ones — and getting their semantics wrong would be worse
than leaving them missing, in exactly the way that repository's own `sys_sigaltstack` comment
argues.

⚠ **One line of unemups4 was changed to get as far as the banner** and is left UNCOMMITTED in that
working tree: `crates/libs/src/libkernel/pthread.rs`, adding `"pthread_attr_get_np"` to
`scePthreadAttrGet`'s `names`. Its release binary was rebuilt from that. Keep it or drop it; it is
one word and it is correct. (That tree was already dirty with an unrelated, finished
`sys_sigaltstack` change from another session.)

### ⚠ orbis-compat creates `/data/OpenGothic` for every consumer

`src/orbis_paths.cpp:19` hard-codes the anchor root. RetroArch's run makes an empty
`/data/OpenGothic/` directory it will never use. Harmless — the anchor only rewrites RELATIVE
paths, and RetroArch's are all absolute — but it is a title's name compiled into a shared overlay,
and the fourth consumer is where that stops being invisible.

### Where the next session starts

Phase 2 is effectively done (the frontend driver). Phase 3 is the software video driver, and
nothing above blocks writing it. What IS blocked is watching it run, so the choice at the top of
the next session is: teach `unemups4` the eighteen exports first, or write Phases 3-5 blind and
verify them on hardware.

---

## 2026-08-22, later — the survey of what else is still orbisdev-shaped

A sweep of every `ORBIS`/`__PS4__`/`PS4` preprocessor condition left in the tree. Three of the
findings were live in code that compiles TODAY and are fixed (commits `63f7566167`, `9ba278d81b`);
the rest is a work list for the phases that revive each file.

### Fixed, because they were live

* `vfs_implementation.c` `dirent_check_err()` classed ORBIS with the Vita and compared a `DIR*`
  against 0 — always false, so a failed `opendir()` read as success.
* `vfs_implementation.c` `retro_vfs_truncate_impl()` excluded ORBIS from `ftruncate`, which this
  SDK has, so it returned -1 for everyone.
* `dylib.c`: a negative `sceKernelLoadStartModule` error cast to `dylib_t` made every failed load
  look successful; `SceKernelModule` is orbisdev's spelling; `dylib_error()` called `dlerror()` on
  a platform with no `<dlfcn.h>`. The whole branch had never been compiled.

### ⚠ `libretro-common/include/defines/ps4_defines.h` should be DELETED, not fixed

Nothing compiled still includes it — its only four includers (`psp_audio.c`, `orbis_ctx.c`,
`ps4_input.c`, `ps4_joypad.c`) are all out of the object list. Every macro in it is either supplied
correctly by an `<orbis/_types/*.h>` or is wrong, and the two that are wrong are exactly the ones
that would keep Phase 4 subtly broken:

* `SCE_USER_SERVICE_MAX_LOGIN_USERS 16` — **the SDK says 4**. `ps4_joypad.c:75` declares a
  16-element login-id list, `sceUserServiceGetLoginUserIdList` fills 4, and the loop then reads 12
  uninitialised `int32_t`s as user ids.
* `SCE_USER_SERVICE_USER_ID_INVALID 0xFFFFFFFF` — the SDK's is `-1` on a signed id, so
  `ps4_joypad.c:84`'s "reject invalid" guard never rejects anything.

Also invented with no SDK counterpart: `SCE_KERNEL_PROT_CPU_RW`, `SCE_KERNEL_MAP_FIXED`,
`SCE_PAD_PORT_TYPE_REMOTE_CONTROL`, the `SCE_MOUSE_*` set.

### The driver names are already chosen, and nothing answers to them yet

`configuration.c` sets the ORBIS defaults to audio `"orbis"` (via `msg_hash_lbl_str.h:48`), input
`"ps4"` and joypad `"ps4"`. None of those three drivers is registered any more, so each falls back
silently. Phases 4 and 5 must either adopt those exact `ident` strings or change these three
places with them — whichever, the pair has to move together.

`input_autodetect_builtin.c:800` binds "PS4 Controller" to the joypad driver `"ps4"` **using the
PS3 bind table**. A DualShock 4 is not a DualShock 3; Phase 4 owns this.

### Work list by phase

* **Phase 4** — rewrite `ps4_joypad.c` (half orbisdev: `orbisPad.h`, `orbisPadGetConf`) and
  `ps4_input.c` (`<orbis/libScePad.h>` is not a header; the SDK's is `<orbis/Pad.h>`); delete
  `ps4_defines.h`; fix the DS4 bind table.
* **Phase 5** — a real `sceAudioOut` driver. `psp_audio.c`'s ORBIS arm is the Vita's driver bent
  around orbisdev: `<libSceAudioOut.h>` does not exist, the user id is hard-coded `0xff`, and
  `psp_audio_stop` returns false unconditionally as a workaround.
* **Phase 7** — delete `orbis_ctx.c` rather than port it (it is Piglet end to end, and Phase 7 is
  Vulkan, not a GL context driver); strip the Piglet shader-binary cache from `shader_glsl.c`; drop
  ORBIS from `gl2.c`'s `glTexStorage` exclusion, which encodes "PS4 GL means Piglet" and is false
  under Mesa; fix the `stb.c` system-font paths, which are the Vita's PVF filenames with the
  directory swapped.
* **Housekeeping** — `Makefile.orbis.salamander` is orbisdev top to bottom and has been linking an
  empty `$(OBJ)` since 2019 (it defines `OBJS`). The CI jobs
  (`.github/workflows/PS4-ORBIS.yml`, `.gitlab-ci.yml:897`) run the orbisdev container, call
  `orbis-ld`, and expect a `.self` that the rewritten Makefile does not emit; the stub header
  `.github/workflows/scripts/stubs/orbis/orbis/libkernel.h` still declares orbisdev's
  `get_user_mem_size`. That stub build is what hid every defect above for six years.

---

## 2026-08-22, later still — Phase 3 written, not seen

`gfx/drivers/ps4_gfx.c` and `ps4/ps4_video_out.c` exist, the build links them and the driver
registers as `"ps4"`. **No pixel has been drawn.** The emulator stops before video init on symbols
it has not implemented, so everything below is reasoned, not observed.

What went in:

* `configuration.c` had **no ORBIS arm at all** in the default-video-driver chain, so this platform
  fell through to `VIDEO_NULL`. That is the real reason the port could boot to its banner without a
  display, and it is fixed.
* One scaler pass does the scale and the format conversion together, because video-out's only
  format is byte-for-byte RetroArch's `SCALER_FMT_ABGR8888`.
* RGUI's menu format comes from a table keyed on the driver ident; `"ps4"` is deliberately not in
  it, so it takes the RGBA4444 default, which the scaler reads directly.

Known gaps, written into the source at the point they matter rather than listed here:

* **No font backend** - on-screen messages go to the log. RGUI draws its own bitmap font, which is
  why the menu is unaffected.
* **The menu replaces the frame, it does not blend over it** - the scaler converts, it does not
  composite.
* ~~The aspect-ratio setting is accepted and ignored~~ - fixed once Vulkan ran beside it and
  honoured the same config, which is how the divergence became visible at all.
* **1080p is unmeasured.** Every frame is a scale into 2.07 million pixels on a Jaguar core. If
  60 Hz does not hold, 720p halves the fill; PLAN.md says settle this by measurement.
* **Alpha is left as the scaler wrote it.** ps4doom ORed `0xFF000000` into every pixel and never
  established whether scan-out reads the primary plane's alpha. If the screen comes up black or
  translucent, that is the first thing to try.

### The order of the next session

1. Decide the emulator question (teach `unemups4` the eighteen exports, or go straight to
   hardware). Nothing else can be verified until one of those happens.
2. Whichever way: the first thing to look for is a picture, and the first suspects if there is
   none are the alpha above and the direct-memory buffer registration.
3. Phases 4 and 5 are unblocked and independent of this - but `ps4_defines.h` has to go first, or
   the joypad driver inherits a 16-vs-4 user list and a guard that never fires.

---

## 2026-08-22 — on hardware: menu, pad, a game, and a core as a module

Everything below was run on a retail console under GoldHEN, in this order, each installed over
the last.

| build | result |
|---|---|
| dummy core, software video | **menu on screen, first try.** Phase 3 had never drawn a pixel anywhere. |
| + libScePad | **pad works.** D-pad, buttons, and analog all reach the frontend. |
| + sceAudioOut | port opens, volume set. Nothing to play yet. |
| 2048 linked statically | **a game runs.** Phase 6. |
| 2048 as a .prx, `HAVE_DYNAMIC=1` | **the module loads and runs**, and picked up the save the static build had written. |

### D3's three unknowns, answered

The plan said static first because it removed three questions at once, and that a core-sized PRX
was unproven. All three came back:

1. **A libretro core builds and loads as a PRX.** `create-fself --lib` with `crtlib.o` in place of
   `crt1.o`; `sceKernelLoadStartModule` finds it, `sceKernelDlsym` resolves `retro_*` by name.
   185 KB, so this says nothing yet about a core of tens of megabytes.
2. **The libc/heap question did not bite.** The module links its own `-lc -lc++` and the frontend
   has its own; nothing crashed, `retro_init` ran, and the core allocated and drew.
3. **`crtlib.o`'s init path is enough** for a core to be usable by the time `retro_init` is called.

⚠ **This was the first time `libretro-common/dynamic/dylib.c`'s ORBIS branch had ever executed.**
It could not compile until the fixes in `9ba278d81b`, so the code that loads every dynamic core on
this platform went from "never built" to "running a game" in one step. Treat its error paths as
unexercised.

### What a dynamic core costs that a static one does not

* It must carry its own `libretro-common`. `-DSTATIC_LINKING` omits those files for the static
  build because the frontend supplies them; a module cannot see the frontend's.
* It must NOT carry the copy it shipped with, if that copy predates this port - `libretro-2048`'s
  still has `#include <orbisFile.h>`. `ps4/build-core.sh --common` points it at the frontend's.

### Audio, confirmed — and the two things that had to be got right

A clean 300 Hz tone from `libretro-samples/audio/audio_no_callback`, built as a PRX and dropped
into `/data/retroarch/cores` with no reinstall. Two defects stood between the driver and that, and
both are worth carrying to any other consumer of this API.

**1. `sceAudioOutInit` returns ALREADY_INIT on the second call, and the constant was wrong.**
RetroArch initialises its audio driver twice — once at startup, again when content loads — so the
second call in a process always fails this way. The driver tolerated `0x8026000d` and bailed on
anything else, which killed audio at content load with "Failed to initialize audio driver".

    0x8026000D  ORBIS_AUDIO_OUT_ERROR_OUT_OF_MEMORY
    0x8026000E  ORBIS_AUDIO_OUT_ERROR_ALREADY_INIT

⚠ The wrong number came from `~/src/ps4doom/platform/doom_sound_ps4.c`, transcribed together with
a comment naming it ALREADY_INIT. **That comment is wrong there too** and has never shown, because
ps4doom initialises audio once. Worth fixing there before it travels again. Use the SDK's named
constant; a magic number carries its explanation with it, and a wrong explanation travels just as
well as a right one.

**2. `sceAudioOutOutput` blocks until the grain is CONSUMED, so the port needs a thread.**
When it returns, nothing is queued behind it. Fed from RetroArch's main loop — which also blocks on
vsync — the port runs dry between the last write of one frame and the first of the next, and a dry
port clicks once per frame. That is exactly what the first working build did.

It is also a throughput problem: at 48 kHz and 60 fps a frame is ~3.1 grains at 5.33 ms each, so
16.6 ms of vsync plus 16.6 ms of audio serialises into 33 ms — 30 fps.

The driver now has a dedicated thread permanently inside `sceAudioOutOutput`, pulling from a ring
that `write()` fills. An empty ring plays silence rather than skipping the call: the port takes
exactly one grain and will not take a partial one, so something must be handed to it either way.
Same shape as ps4doom's mixer thread and as `switch_thread_audio.c` in this tree.

### 1080p holds 60 Hz, measured

PLAN.md Phase 3 said to settle 1920x1080 against 1280x720 by measurement and not by choosing up
front. Measured, on hardware, over several minutes:

    300 frames: scale 6124 us/frame, frame 16683 us (59.9 fps)

Stable to within 30 microseconds across every report. So:

* **The frontend is paced by the display, not by itself.** 16683 us is the vsync interval; nothing
  is costing an extra refresh.
* **The scale costs 6.1 ms of a 16.67 ms budget — 37%.** That leaves ~10.5 ms per frame for a core.
* **1080p stays.** 720p would roughly halve the scale, and there is no reason to spend the
  resolution to buy time nothing needs yet.

⚠ The measurement is a 320x240 source. Point-scaling cost is dominated by DESTINATION pixels, so a
higher-resolution core moves this number much less than it moves its own; but a core that needs
more than 10.5 ms per frame will not hold 60 Hz, and 720p is then the lever - as a runtime option,
not a new default.

⚠ And it is measured with the default filter. `video_smooth` selects SCALER_TYPE_BILINEAR instead
of POINT, which is a different and larger number. Nobody has measured that one.

### Audio under load, measured

    audio: 1875 grains, 10 underruns
    audio: 3750 grains, 10 underruns
    audio: 5625 grains, 10 underruns

Ten underruns while the pipeline fills, then flat for as long as the run lasts. And a
free confirmation that the thread keeps the port's clock exactly: 1875 grains per 10 s times 256
frames is 48 000 frames a second, to the sample.

100% underruns while sitting in the menu is correct, not a fault: there is no core, the ring is
empty, and the thread plays silence to keep the port's continuity. A thread that skipped the call
instead would have to prime the port again on the way back into a game.

### Still unconfirmed

* **A large PRX.** 185 KB proves the mechanism, not the scale.
* **A demanding core.** Nothing has yet asked for more than a fraction of the 10.5 ms of headroom.
* **The bilinear scaler path.**

### A rough edge worth fixing

RetroArch creates `/data/retroarch/*` as `drwxr-x---`, and GoldHEN's FTP daemon runs as another
user, so `put` into `cores/` is refused until something chmods it. Copying a core over the network
is the only way to deliver one, so the directory mode is part of the delivery path, not a detail.

### ⚠ A correction to the record

An earlier entry in this session claimed `dir_check_defaults("/app0/custom.ini")` had stopped the
default directories being created on hardware. **It had not.** The directories were there all
along, dated from an earlier boot; the evidence for the claim was an FTP listing truncated by
`head -30` before it reached `retroarch/` alphabetically. The change that came out of it - passing
`NULL`, and checking the result - is kept on its own merits, and its comment now says what actually
happened.

---

## 2026-08-22 — Vulkan on RADV, and where the close-hang is not

RetroArch draws through Mesa's RADV on the console. 19 456 frames in one run at `frame 16683 us`
— the same 60 Hz the software driver holds — and the scan-out is the zero-copy path, not a copy:

    wsi/orbis: scan-out up - 1920x1080 pitch 1920, 4 swapchain buffer(s), A8B8G8R8_SRGB linear
               - ZERO COPY, the flip shows what the GPU rendered into
    wsi/orbis: the scan-out copy took 0 us for 8100 KiB (worst 27 us over 19 456 frames)

### ⚠ Vulkan teardown works, and this is the first evidence anywhere that it does

Every title built against this Mesa hangs when the console closes it. Nothing had established
whether the driver teardown was the thing that wedged, because **no capture from this console had
ever contained `wsi/orbis: scan-out down`** — the titles that came before end by idling or by
CE-34878-0, and both routes skip `vkDestroyInstance` entirely. The path had never run.

Forcing it from a live process — load a core, then Close Content, which tears the video driver
down and builds it again — gives:

    15:19:26.718  wsi/orbis: scan-out down after 493 flip(s)
    15:19:26.719  wsi/orbis: scan-out up - 1920x1080 ...
    15:19:26.719  [PS4] Vulkan up: 1920x1080 swapchain on RADV.

**One millisecond, clean, and it comes straight back up.** So the close-hang is not in destroying
the driver. It is in terminating the process while RADV is live.

That splits a problem this port did not create and cannot fix alone. And the next cut came back
narrower than expected: **Close Content, then Quit, exits cleanly even on the Vulkan driver** — and
Close Content re-initialises the driver, so RADV is live at that point. "RADV alive at kill time" is
therefore NOT the condition. What is left is the combination with a loaded core; the .prx alone was
fine for a whole session on the software driver, and Vulkan alone is fine here. Nobody has narrowed
it further than that yet.

Practically it also means the console no longer has to be restarted between tests, which is what
made every experiment above expensive.

⚠ RetroArch is a better instrument for this than the title that first hit it: the teardown is one
menu entry rather than an exit, repeatable in a single session, with the log flowing throughout.

### A side effect of the hang, worth knowing before it wastes an hour

**The config is never saved.** RetroArch writes `retroarch.cfg` on a clean exit, and there are no
clean exits yet — so a driver picked in the menu is forgotten on the next launch, every time. The
file on the console was four hours stale while the menu showed the right thing. The Vulkan default
now comes from `configuration.c` rather than from a file that does not get written.

### The software scaler is much more expensive for RGB565 cores

    XRGB8888 source:  scale  6 124 us/frame   (37% of a 16.67 ms budget)
    RGB565   source:  scale 14 450 us/frame   (87%)

Both at 320x240 into the same viewport, so the difference is the pixel format alone: an XRGB8888
source is already the scaler's internal format, and RGB565 costs a whole extra pass — convert in,
scale, convert out. 87% of the frame leaves almost nothing for a core, so on the software path a
16-bit core is the demanding case and a 32-bit one is not. Untouched for now because Vulkan is the
path that matters, but it is the number to remember if the software driver is ever the fallback for
a real core.

---

## 2026-08-22 — Phase 7 is done: XMB on RADV, icons and text

The GPU menus are back. XMB draws through the Vulkan driver's texture path with the monochrome
icon theme and readable text, which is the visible half of what Phase 7 was for — the software
build has RGUI and nothing else, because RGUI is the only menu driver that rasterises itself.

Two things had to be true and only one of them was:

* **The GPU menus need a driver that can carry them.** RADV provides it. They are enabled whenever
  `HAVE_VULKAN=1` and off otherwise, so the software build is unchanged.
* **They need assets, and the console had none.** `/data/retroarch/assets` was an empty directory
  that `dir_check_defaults` had created and nothing had ever filled. 16 MB of
  `libretro/retroarch-assets` — `xmb/monochrome`, `ozone`, and two fonts from `pkg` — copied over
  FTP is enough for both drivers.

⚠ **The system-font list was pointing at files that do not exist.** It was the Vita's seven PVF
names with the directory swapped to `/preinst/common/font`; a retail console's own directory has
`DFHEI5-SONY.ttf` and the SST family and none of those seven. Fixed, and ordered by what
`stb_truetype` can parse rather than alphabetically: SST is `.otf` with CFF outlines and
stb_truetype reads TrueType `glyf`, so the one real TTF goes first. In practice
`assets/pkg/fallback-font.ttf` is what XMB uses, so the system list is now a spare rather than a
requirement — but a spare that named seven absent files was worth nothing.

### Where the port stands

| | |
|---|---|
| build | 239 objects, no warnings, no libc shims written |
| video, software | 1080p @ 60 Hz measured, 6.1 ms/frame for a 32-bit core |
| video, Vulkan | RADV, 1080p @ 60 Hz, zero-copy scan-out |
| menus | RGUI on software; XMB, Ozone, MaterialUI, widgets on Vulkan |
| input | DualShock 4, digital and analog |
| audio | 48 kHz, threaded, no underruns under load |
| cores | static and `.prx`; saves survive across both |
| packaging | `.pkg`, `make send` over lftp |

### Still open

* **The close-hang**, which this port did not create: every title on this Mesa has it. Narrowed
  today — driver teardown is clean, and Close Content followed by Quit exits properly even on
  Vulkan, so the condition involves a loaded core rather than RADV being live.
* **Slang shaders** compile in and have never been run.
* **Assets are hand-copied.** Nothing ships them and nothing tells a user they are missing; XMB
  without them looks like a menu driver that failed to start.
* **The RGB565 software path** costs 14.4 ms of a 16.7 ms frame, against 6.1 ms for XRGB8888.

---

## 2026-08-22 — slang shaders run, and Beetle PSX HW builds as a 17 MB module

**Slang shaders work on hardware.** `crt/crt-geom.slangp` renders. That is the whole chain
executing for the first time: `.slang` -> glslang (compiled into the eboot) -> SPIR-V ->
SPIRV-Cross -> RADV/ACO -> GCN ISA on Liverpool, compiled at run time on the console. 12 MB of
`libretro/slang-shaders` (crt, interpolation, misc) copied to `/data/retroarch/shaders`.

**Beetle PSX HW is built and on the console** — 113 objects, a 17 MB `.prx` with 55 `retro_*`
exports. Three things had to be settled to get there, and each is general rather than specific to
this core:

* ~~⚠ **A dynarec needs a mirrored-mapping story this platform does not have.**~~ **SUPERSEDED
  2026-08-23 - see the section at the end of this file.** Lightrec maps the same PSX RAM pages at
  several addresses through `memfd_create`/`MAP_SHM`; neither exists here (`libretro.c:2345-2574`),
  and that part is still true. **The conclusion drawn from it was not.** The platform's own
  direct-memory API makes mirrored mappings, and it was measured doing so. `HAVE_LIGHTREC=0` is
  still what the shipped core is built with, and the reason recorded for it no longer holds.
* ⚠ **A core must use the Vulkan headers it was written against, not the driver's.** Forcing
  Mesa's current `vulkan_core.h` broke it on `VK_IMAGE_TYPE_RANGE_SIZE`, an enum removed from the
  spec years after this core started using it. The rule that the loader shim needs the exact
  headers RADV was built against does NOT generalise to consumers: a core is an ordinary Vulkan
  client and the ABI is backward compatible. The frontend is the special case, not the core.
* ⚠ **The core's Makefile has the stale-object problem too.** Turning `HAVE_LIGHTREC` off left
  `cpu.o` compiled against the old flags, and the link failed on `lightrec_destroy` for a source
  file that no longer referenced it. Same shape as the one fixed in `Makefile.orbis`; the fix
  there was a flags stamp, the fix here was `find -name '*.o' -delete`.

The core's own Makefile grew an `orbis` platform arm - the same shape as its `vita` one - which
compiles the objects; the archive and the `create-fself --lib` step are done outside it, because a
libretro module here is a PRX rather than a shared object.

### D3's last unknown, answered

The plan said 185 KB proves the mechanism and not the scale. 17 MB is the scale, and it links.
Whether it LOADS at that size is still for the console to say.

### What it needs before it can run

A PlayStation BIOS in `/data/retroarch/system` (`scph5500/5501/5502.bin`) and a disc image. Neither
is something this port can supply.

---

## 2026-08-22, close of day — hardware render works, and one core draws it wrong

Beetle PSX HW runs Spyro 3 through RADV with the libretro hardware-render callback: the core is
handed a Vulkan context and builds its own pipelines on this driver. Nothing about that path had
ever run — until today RADV only drew what RetroArch told it to, and now foreign graphics code
does. 40 fps on the MIPS interpreter, which is more than expected with no dynarec.

The picture is wrong in a specific way: the static scene is correct — sky, terrain, castle,
lighting, colours — and every animated model is shredded, with one quad showing stripes of garbage
where a texture belongs.

### What the elimination found, in order

Each of these cost one experiment and each removed a whole class of cause.

1. **The core's own software renderer draws the same content correctly.** So the geometry reaching
   the GPU is right, and the GPU's reading of it is not. That rules out the CPU side entirely,
   including the interpreter we are forced onto by `HAVE_LIGHTREC=0`.
2. **`ORBIS_3D_LINEAR=1` and `ORBIS_NO_TESS=1` were not applied, and now are.** RetroArch never
   read `/data/tempest-env.txt`, so it had been running the driver in a configuration no other
   title runs in — see the commit; the file itself says those two "are not options". Applying them
   did **not** change the picture, which is worth knowing on its own: this artefact is not the
   tiling that file exists to avoid.
3. **Doubling the internal resolution changes nothing.** Different resolutions mean different
   pipeline variants; a shader miscompile would have moved. It did not, so ACO is not the first
   suspect.

### ~~Where that leaves it: the vertex attribute path~~ SOLVED 2026-08-23

⚠ **Read the correction at the end of this file before spending time on anything below.** The
suspect named here was right in its neighbourhood and wrong in its name: the variable is not that
the formats are integer, it is that their ELEMENTS ARE LARGER THAN FOUR BYTES. The driver was
fetching them from addresses that are not multiples of their size, silently getting the wrong bytes,
and it is fixed. Spyro 3 draws correctly now.

### Where that leaves it: the vertex attribute path

`rhi/rhi_lib_vulkan.c:6032-6038` declares seven vertex attributes, and **four of the seven are
integer formats**:

    0  R32G32B32A32_SFLOAT   position
    1  R32G32B32A32_SFLOAT   color
    2  R8G8B8A8_UINT         window      <-- integer
    3  R16G16B16A16_SINT     pal_x       <-- integer
    4  R16G16B16A16_SINT     u  (UV)     <-- integer
    5  R16G16B16A16_UINT     min_u       <-- integer
    6  R32G32B32A32_SFLOAT   fog

Attributes 3 and 4 are palette and texture coordinates. A quad showing stripes of garbage instead
of a texture is what wrongly fetched UVs look like. Integer vertex formats are a far less travelled
path in any driver, and more so on GFX7.

⚠ **This is answerable with the CTS already ported to this console, without RetroArch, without
Spyro and without guessing:** `dEQP-VK.pipeline.*vertex_input*` tests attribute fetch per format
and in combination. If one of those four fails, the cause is named to the format.
`/data/deqp-cases.txt` currently holds 49 `api.object_management` cases from an earlier
investigation and has not been touched.

### ⚠ A fourth knob with no reader

`ORBIS_TILE_MODE` was going to be the next experiment. It has exactly one occurrence in mesa-ps4:

    src/amd/common/ac_orbis_drm.c:4412
       getenv("ORBIS_TILE_MODE") ? getenv("ORBIS_TILE_MODE") : "unset",

— printing its own value into a log line. Nothing acts on it. `tempest-env.example.txt` documents
it as working and lists three other names that were found readerless on 2026-08-21; this is a
fourth. Setting it would have produced exactly what that file warns about: "the run comes back
clean and reads as a measurement."

### What a console operator has to know

Delivery over FTP is a two-user problem in both directions, and the modes are not uniform:

    .prx  (modules)       777   must be executable; sceKernelLoadStartModule loads them as modules
    data files            666   BIOS, discs, .info, fonts, shaders
    directories           777

Setting `666` on a core makes it fail to load with no useful message. `mkdir` on this platform now
creates 0777 (`vfs_implementation.c`), but files written by the FTP daemon are its own and nothing
on the RetroArch side controls them.

Cores must be named `<name>_libretro.prx` with a matching `<name>_libretro.info`, and
`/data/retroarch/info/core_info.cache` must be deleted after adding one — a cache built while the
info directory was half-populated stays empty and the menu says "No cores available" for ever,
which then presents as an empty content browser because a frontend with no core info has no
extension filter.

### Still open

* The vertex-attribute question above.
* **The close-hang**, which every title on this Mesa has. Narrowed: driver teardown is clean, and
  Close Content followed by Quit exits properly, so the condition involves a loaded core.
* **`orbis_paths.cpp:19` hard-codes `/data/OpenGothic/`** as the anchor for relative paths. Every
  relative path RetroArch opens - and it does open some, `Main Menu.png` among them - lands in
  another title's directory. Flagged on 2026-08-21 as harmless; it is not.
* Assets, shaders and cores are hand-copied, and nothing tells a user when they are missing.

---

## 2026-08-23 — both open questions answered, from the driver side

Written into this file by the Mesa workshop (`~/src-ps4/mesa-ps4`, `~/src-ps4/ps4-mesa-docs`)
because both answers were measured there and both contradict something this file states as fact.
The full account, with every log line, is `ps4-mesa-docs/docs/HANDOFF.md` from
"The integer-attribute suspect has a name" onwards.

### 1. The shredded models: SOLVED, and it was alignment rather than integers

The suspect was one step off. Not "integer vertex formats are a thinly travelled path" - the
variable is **element size**. `src/amd/common/ac_shader_util.c`'s `is_fetch_size_safe()` exempts
GFX7-GFX9 from every alignment requirement: on those parts it declares any typed vertex fetch safe
at any address. That is a claim about silicon, inherited from upstream, and **it is false on
Liverpool.** A multi-byte element read from an address that is not a multiple of its size returns
the wrong bytes, with no fault and no log.

Measured with the Vulkan CTS, 1853 cases of `dEQP-VK.pipeline.monolithic.vertex_input`, same case
list and same binary, one environment line apart:

    believing the exemption   Passed 1657   Failed 98
    splitting the fetches     Passed 1754   Failed  1

97 Fail→Pass, 0 Pass→anything, 0 other verdict changes. The one survivor is a geometry-shader case
and a different defect.

**Why Beetle's picture looked the way it did:** formats built from 32-bit channels are immune,
because dword alignment IS element alignment there. So the three `R32G32B32A32_SFLOAT` attributes
were always correct and the three `R16G16B16A16_*` were not. ⚠ And the prediction this makes, which
"integer formats are less travelled" does not: **attribute 2, `R8G8B8A8_UINT`, was never affected** -
a 4-byte element is aligned wherever a dword is.

Fixed in the driver, `ORBIS_VS_STRICT_ALIGN`, **on by default on this platform** and only this one.
`ORBIS_VS_STRICT_ALIGN=0` restores the old behaviour and logs a warning, because the off state is now
the dangerous one. Nothing is needed on the RetroArch side except a rebuild against a driver from
2026-08-23 or later — the log line to check is

    orbis: vertex fetches are split to natural alignment

Its absence means the binary predates the fix. **Every title links `libvulkan_radeon.a` statically,
so an installed build carries whatever driver it was linked with.**

### 2. Where Spyro's frame actually goes, and it is not the GPU

From the driver's own BUDGET instrumentation during a Beetle PSX HW session, 97 five-second windows:

    menu           0.05 cores    60.0 fps
    3D scene       1.00 cores    40 fps, and 1.00 cores at 23 fps
    time waited for the GPU, in EVERY window without exception: 0 ms
    the whole Vulkan API, per frame:  263 us against a frame of 43967 us  = 0.6%
    23 draws, 4 render passes, 1.68 screenfuls of 1920x1080, 2 dispatches

One CPU thread saturated, the GPU never waited on, the graphics driver at 0.1-0.4% of the window.
⚠ **The audio running slow is the same fact, not a second one:** the core is at ~38% of realtime, so
the samples come out at ~38% of the rate. Precaching the disc does not help because the bottleneck is
not I/O. Internal resolution and renderer settings will not move it either - that is now measured
rather than assumed.

The interpreter is the entire cost, and the dynarec is the only lever.

### 3. Lightrec's wall does not exist. All three mechanisms measured on hardware

    mirrored RAM       sceKernelMapDirectMemory called repeatedly with the SAME phys gives EIGHT
                       simultaneous views - the probe's own cap, not the kernel's; it had not
                       refused. Coherent in BOTH directions at two offsets 2 MiB apart, and
                       unmapping one leaves the rest intact.
    fixed placement    ORBIS_MAP_FIXED puts a mapping at an address of our choosing. Not new -
                       ac_orbis_drm.c:6464 does it in production every time a buffer moves bus.
    executable code    map READ|WRITE, then sceKernelMprotect to 0x07, then EXECUTE. Six bytes of
                       x86-64 (b8 ee ff c0 00 c3) were written and CALLED; it returned 0x00c0ffee
                       and the process carried on.

⚠ **THE OBVIOUS FORM OF THE LAST ONE IS REFUSED.** `sceKernelMapDirectMemory` asked for
READ|EXECUTE up front returns `0x8002000d` = EACCES - understood and declined, not malformed. The
policy lives at map time, not at protect time. **Anyone who tries the direct form first will conclude
this is impossible**, which is exactly what the probe concluded one rung before it turned out true.

⚠ **Only the mprotect route was actually EXECUTED.** `sceKernelMapFlexibleMemory` and
`sceKernelMmap` both granted 0x07 as well and neither was called. On this console a granted
protection is not an honoured one - it has charged for that distinction three times now.

⚠ **`MAP_PRIVATE|MAP_ANON` is `0x1002` here, not `0x0022` - and the SDK already knows that.**
`$TOOLCHAIN/include/sys/mman.h` is musl's and does carry Linux's `MAP_ANON 0x20`, but it ends with
`#include <bits/mman.h>`, and `$TOOLCHAIN/include/bits/mman.h:59-62` `#undef`s `MAP_ANON` and
redefines it as `0x1000` - the same file also corrects `MAP_SHARED`, `MAP_PRIVATE`, `MAP_FIXED` and
the `PROT_*` set. **orbis-compat has no `sys/mman.h` and no `bits/mman.h`; it does not touch these
constants and never did.** Passing the wrong value would make the kernel treat the mapping as
file-backed, validate `fd = -1`, and return EBADF - which reads as a refusal of the protection and
is nothing of the kind, so the value matters; it is simply already right.
`orbis-compat/src/orbis_mmap.cpp:54` has a `static_assert` on `0x1002` that could not compile
otherwise. **Ask the compiler for constants (`clang -dM`), not a header you found with grep.**

The probe is `orbis_test_mirror_mapping()` in `ac_orbis_drm.c`, behind `ORBIS_TEST_MIRROR=1` for the
safe rungs and `=exec` for the jump. It runs once at device init in any title, takes 2 MiB and gives
it back. If the port hits a wall, that ladder re-establishes the ground truth in one run.

### What this does and does not promise

It removes the reason recorded for not building Lightrec, and that reason was the whole of the case.
**It does not say the port is short.** Lightrec brings its own code emitter, its own build system and
its own assumptions about the host; the platform blockers are gone and the size of the remaining work
is unmeasured.

⚠ And one trap already in this file, worth re-reading before starting: turning `HAVE_LIGHTREC` back
on needs `find -name '*.o' -delete` first. The core's Makefile has the stale-object problem, and
switching the flag the other way already cost a link failure on `lightrec_destroy`.

### ⚠ And the build line, because this file never wrote it down and that cost a package

The frontend needs **three** flags, not one. `Makefile.orbis`'s own comments put only
`HAVE_VULKAN=1` in a command line, and a rebuild made with just that shipped, installed, booted,
drew — and showed **no cores at all**:

    make -f Makefile.orbis HAVE_VULKAN=1 HAVE_STATIC_DUMMY=0 HAVE_DYNAMIC=1 -j$(nproc) pkg

⚠ **Changing any of them needs a `clean` first.** They are `-D` defines and this Makefile has no
header dependencies, so objects built under the other setting are silently reused.

⚠ **Nothing in the artefact says which configuration it is.** The ELF is the same size either way and
the `.pkg` has come out 41680896 bytes for four days running. The grep that answers it:

    sceKernelLoadStartModule    static build 0    dynamic build 3
    libretro_dummy              25 either way, so NOT the marker to look for

The link step now prints the configuration every time (`cores:`, `dummy:`, `vulkan:`, and the driver
archive's build date), so this should not be able to recur silently.

⚠ **A static build also POISONS `/data/retroarch/info/core_info.cache`** — it writes 65 bytes
decompressing to `{"version": "1.2", "items": []}`, and that empty cache keeps the menu empty across
every later install until somebody deletes it by hand. If the core list is empty after a good build,
delete that file first.

Two defects in `Makefile.orbis` were fixed while finding this, both uncommitted for review:

    the eboot.bin recipe did not pass OO_PS4_TOOLCHAIN, which create-fself reads from the
    environment and refuses without - although it is invoked by absolute path out of that very
    toolchain. The `pkg` target passes it; this one did not. The build linked a new .elf, failed at
    status 255, and left the PREVIOUS DAY'S eboot.bin and .pkg beside it, ready to be uploaded as
    "the rebuild". It had only ever worked because the shell that ran make happened to export it.

    the link step now prints what it built, as above.

### How everything here is compiled, so it is not rediscovered

Verified against the trees on 2026-08-23. Every path is a real entry point that was run that day.

**Order matters.** The overlay is what the driver and the frontend both compile against, and the
driver is what the frontend links, so a change low down means rebuilding upward:

    1  orbis-compat   ./build.sh                      -> build/liborbis-compat.a
    2  mesa-ps4       ./ps4/build.sh                  -> build-orbis/src/amd/vulkan/libvulkan_radeon.a
    3  RetroArch      make -f Makefile.orbis ... pkg  -> eboot.bin, IV0000-RTRA00001_*.pkg
    3b cores          ps4/build-core.sh               -> <name>_libretro.prx + .info

⚠ **Nothing rebuilds anything below it automatically.** The frontend links whatever
`libvulkan_radeon.a` is sitting there; it does not check whether the driver sources are newer. The
link step prints the archive's build date for exactly that reason - **read it, and ask whether that
is the driver you meant.**

    orbis-compat/build.sh
        Consumers need exactly two things, and both matter:
          -isystem <orbis-compat>/include   AHEAD of the SDK's include directory
          build/liborbis-compat.a           with --whole-archive
        ⚠ The include order is not a preference. The SDK ships musl's headers behind a FreeBSD
        triple, and the overlay corrects declarations that differ - four pthread types musl
        declares smaller than Sony writes, `sa_sigaction`'s macro - by defining musl's own
        `__DEFINED_<name>` guards before `bits/alltypes.h` is reached. Behind the SDK's directory
        it compiles, does nothing, and says nothing. (NOT the mmap constants: the SDK's own
        `bits/mman.h` already redefines those to FreeBSD's values - see "the MAP_ANON myth" below.)

    mesa-ps4/ps4/build.sh   [--host-too] [--host-orbis] [--sdk <dir>] [--work <dir>]
        no arguments   cross-build the driver for the console. This is the one that matters.
        --host-too     also build a plain Linux RADV, for the drm-shim probes
        --host-orbis   build THIS arm as an ordinary Linux ICD and run its self-tests. Catches
                       anything structural without a console trip.
        ⚠ It prints the driver path and modification time every run because a path that looks
        right pointing at another day's build has cost this workshop an evening.

    RetroArch  make -f Makefile.orbis HAVE_VULKAN=1 HAVE_STATIC_DUMMY=0 HAVE_DYNAMIC=1 -j$(nproc) pkg
        The three flags and the `clean` rule are in the section above. `make ... info` dumps the
        whole flag set if something looks wrong.

    RetroArch  ps4/build-core.sh --core <dir> --out <path> [--prx] [--name <label>] [--common <dir>]
        --prx     a loadable module rather than a static archive
        --name    also writes the minimal <out>.info RetroArch needs to show a readable name
        --common  use the FRONTEND's libretro-common instead of the core's own vendored copy,
                  which for anything predating this platform still has the orbisdev-era ORBIS
                  branch and will not compile

**All four resolve the toolchain and the overlay the same way**, through
`orbis-compat/scripts/ps4/orbis-env.sh`: `ORBIS_COMPAT_DIR` if set, else a sibling directory, else
`~/src-ps4/orbis-compat`; and `OO_PS4_TOOLCHAIN` or `~/.local/opt/openorbis`. A fresh clone of the
`orbis-ports` organisation with the repositories side by side needs no environment at all.

    deploy   orbis-compat/scripts/ps4/deploy.sh --pkg <file> --name <short>
                                                [--also <local>:<remote>]... [--host <ip>]
        Uploads to /data/pkg/<name>-YYYYMMDD.pkg - ⚠ this console installs from /data/pkg and
        nowhere else. Verifies by READING BACK: sizes for packages, byte-for-byte for anything
        under a megabyte. It ends by saying INSTALL + RUN or RUN, no install, and that line is
        the answer to "do I need to reinstall".

    configuration on the console
        /data/tempest-env.txt    read first, by every title
        /data/retroarch-env.txt  read second, ours, and only the log destination belongs in it
        Both are plain KEY=VALUE, applied with setenv() before anything touches Vulkan.

⚠ **The Beetle PSX HW source is NOT in this workshop.** The built `.prx` and its `.info` are on the
console under `/data/retroarch/cores/` and `/data/retroarch/info/`, and the checkout they came from
is not on the machine - a search on 2026-08-23 found nothing. Anyone picking up the Lightrec work
starts by fetching the core again, and the `orbis` platform arm its Makefile grew is not upstream.

### Where the Lightrec work goes: one fork, and probably only one

    orbis-ports/beetle-psx-libretro, branch ps4-support

Upstream is `beetle-psx-libretro`; the binary it produces here is `mednafen_psx_hw_libretro.prx`.
Same shape as the other eight repositories in the organisation.

**Why one fork and not two.** Lightrec is a separate project (pcercuei's) vendored into the core, so
the instinct is that a platform change belongs upstream in Lightrec. ⚠ **The evidence in this file
says otherwise:** the citation for the mirrored-mapping code is `libretro.c:2345-2574`, and that is
**Beetle's own file, not Lightrec's**. The recompiler is handed pointers; arranging the host mappings
is the core's job. If that holds, the whole change is one fork and Lightrec is untouched.

⚠ **UNVERIFIED - the core's source is not in this workshop and this was not checked.** After
cloning, two greps settle it:

    rg -n "memfd_create|MAP_SHM|mmap" --glob '!deps/lightrec/**' libretro.c
    rg -rn "memfd_create|MAP_SHM" deps/lightrec/

Hits only in the first: one fork. Hits in the second as well: Lightrec maps for itself, that is a
general "platform without POSIX shared memory" problem rather than ours, and it belongs upstream in
Lightrec rather than in a PS4 fork of the core.

⚠ **And the port work may not all be recoverable from a fresh clone.** The `orbis` platform arm this
file records the core's Makefile growing is **not upstream**, and the checkout it was written in is
gone. Look for a patch or a stashed tree before starting from zero; if there is none, that arm has to
be written again, and this file's own note that its `libretro-common` is too old to compile here
applies from the first build.

### ⚠ Do not switch cores to get a better recompiler

`pcsx_rearmed` has a mature x86-64 dynarec and is far lighter than Beetle, which makes it look like
the shorter road. **It has no Vulkan hardware renderer.** Beetle PSX HW is the only thing in this
port where foreign graphics code builds its own pipelines on our RADV, and it is what found the
vertex-fetch defect that had been silently corrupting every title. Moving to a software-rendered core
would trade the whole diagnostic value of this arrangement for a frame rate, and would throw away the
port work already spent on Beetle.

The core is right. The recompiler is what is missing from it.

---

## 2026-08-23 (evening) — Lightrec built, and the fork it lives in

The Mesa side's entry above removes the reason this file recorded for not building the dynamic
recompiler. This is the other half: the recompiler is built, it is on by default here, and the
core is on the console. **It has not been run on hardware yet** - what follows is what was
written and why, not what was measured.

### ⚠ First: the checkout this file said was gone is not gone

The entry above records the Beetle PSX HW source as unrecoverable - "a search on 2026-08-23
found nothing" - and tells whoever picks the work up to re-fetch it and rewrite the platform
arm. It was in the previous session's **scratchpad** (`/tmp/claude-1000/.../scratchpad/`), with
the `orbis` arm uncommitted in its working tree, exactly as it had been left.

The search was of `~` and `~/src-ps4`. A scratchpad is where a session is *told* to put
working files, so it is the first place to look for a missing one and it was not looked at.
The tree is now `~/src-ps4/beetle-psx-libretro`, branch `ps4-support`, origin
`git@github.com:orbis-ports/beetle-psx-libretro.git`, upstream `libretro/beetle-psx-libretro`.
(This said "not yet pushed - the repository may not exist yet"; checked 2026-08-28, the remote's
`refs/heads/ps4-support` is `b0b759e`, the same commit as the local branch. The work is not
stranded on one machine.) The recovered arm is its first commit, so
the Lightrec work reads as a diff against it rather than as one lump.

### The fork is one fork, and the grep the entry above asked for has been run

    rg "memfd_create|MAP_SHM|mmap" libretro.c        53 hits
    rg "memfd_create|shm_open" deps/lightrec/         0 hits

Confirmed: arranging host mappings is **Beetle's** job, not Lightrec's. The recompiler is
handed pointers. Nothing goes upstream to pcercuei; the whole change is in this fork.

### What the platform arm actually needed

`libretro.c` builds the maps behind five macros and a descriptor - `MAP`, `MAP_SHM`,
`MAP_CODE`, `UNMAP`, `MFAILED`, and a `MEMFDTYPE`. Every existing arm fills them with POSIX
shared memory, ashmem or Win32 file mappings. The PS4 arm fills them with Sony's allocator, and
the shapes line up one for one:

    memfd_create + ftruncate    ->  sceKernelAllocateDirectMemory   (physical pages, an off_t)
    mmap(MAP_SHARED, memfd)     ->  sceKernelMapDirectMemory        (a view of those pages)
    mmap(MAP_ANON|MAP_PRIVATE)  ->  the same call on its own allocation
    PROT_EXEC at mmap time      ->  REFUSED - map RW, then sceKernelMprotect

`MEMFDTYPE` is `off_t` here and the "memfd" is not a descriptor: it is the offset into the
console's physical memory. New files: `ps4/orbis_lightrec_mem.{c,h}` in the core.

### ⚠ Three things that differ from every other arm, and all three are load-bearing

**MAP_FIXED here has no `_NOREPLACE` form.** The Linux arm asks for an address and checks what
came back; on this kernel the check runs after the frontend's heap has already been replaced.
Every fixed mapping asks whether the range is empty first.

**The range check walks a granule at a time, deliberately.** The natural call is
`sceKernelVirtualQuery(addr, 1, ...)` - "the first mapping at or above" - one call for a whole
range. Nothing in this workshop has ever passed 1. The only established value is 0, from
`ac_orbis_drm.c`, where a nonzero return means nothing is mapped there. ⚠ **The two ways of
being wrong are not the same size:** a guessed flag that makes the call fail reports every
address as free, MAP_FIXED lands on the heap, and the result is a corrupted process rather than
a refusal. Being too conservative only loses the recompiler. ~18,000 queries per content load
buys that, which is nothing against the load itself. If flags=1 is ever established, it
collapses to one call.

**Direct memory is not reclaimed when a mapping goes away**, and `lightrec_init_mmap` is called
*twice* on the way in (hugetlb, then without). Released explicitly on both the failing and the
succeeding path, or it is 2 MiB of unswappable memory per content load.

### The code buffer is mandatory here, unlike everywhere else

Without one, Lightrec lets GNU lightning allocate its own with `mmap(PROT_EXEC)`
(`deps/lightning/lib/lightning.c:2516-2531`). This kernel refuses execute at map time. ⚠ **A
block emitted into non-executable pages does not fail, it ends the process** - so if the
promotion to RWX is refused, `psx_dynarec` is clamped to `DYNAREC_DISABLED` and the interpreter
runs, with a line saying why.

The guard is in two places because the core reads its options *before* it maps anything:
`check_variables` sees "refused", `InitCommon` sees "not available". "Not tried yet" must not
read as "refused", or the recompiler switches itself off on the way in every time.

### The option default is flipped on this platform only

Upstream defaults `beetle_psx_cpu_dynarec` to `disabled` and hides it under **Hacks**. Given
the measurement in the entry above - one CPU thread saturated, the GPU waited on for 0 ms in
every window, the whole Vulkan API at 0.6% of the frame - an interpreter default here is an
unplayable default. `#ifdef __ORBIS__` -> `"execute"`, everywhere else unchanged.

⚠ **A saved `.opt` file wins over a default.** `/data/retroarch/config/Beetle PSX/Beetle
PSX.opt` on the console has no `cpu_dynarec` line, because the core that wrote it had no such
option - so the new default does apply. Delete the line, not the file, if this ever needs
re-testing.

### ⚠ The build is reproducible now, which it was not

The `.prx` that ran for a day could not be rebuilt from the repository: the arm was
uncommitted and the link was a shell command nobody wrote down. `ps4/build.sh` in the fork is
that link. Two things it exists to prevent:

    make platform=orbis           tries to LINK, with $(LD) = $(CXX) = the host driver, and
                                  fails on host libstdc++. The previous route was to run make,
                                  let the link fail, and pick .o files out of the tree.
    make platform=orbis objects   a new target: compile and stop. This is what build.sh uses.

`STATIC_LINKING` is not the escape - it is a `-D` define about whether the core is built INTO a
frontend, not a link mode.

⚠ **`HAVE_LIGHTREC` needs a clean when it changes.** It is a `-D` define with no header
dependency, so objects built the other way are silently reused - which already cost a link
failure on `lightrec_destroy`. `build.sh` keeps a `.ps4-lightrec` stamp and cleans itself.

### On the console now

    /data/retroarch/cores/mednafen_psx_hw_libretro.prx   18903168 bytes, mode 777
    /data/retroarch/info/mednafen_psx_hw_libretro.info   mode 666
    /data/retroarch/info/core_info.cache                 DELETED, as it must be after any change

Nothing on the frontend side changed and no RetroArch rebuild is needed - the whole change is
inside the core.

### What the first run should say

The lines to look for, in order. `[PS4] lightrec:` is the prefix throughout.

    <addr> is taken (...)              one per rejected io_base, with what is there. This is
                                       the only account of where this process's address space
                                       is, and it is worth reading even on a successful run.
    N KiB code buffer at <addr>, writable and executable
                                       the promotion worked. Absent means it did not.
    no executable code buffer ...      the clamp fired; the interpreter is running.

Then the frame rate. The interpreter measured ~38% of realtime on Spyro 3 (40 fps in scene, 23
fps in the worst window). ⚠ **Audio pitch is the same fact as the frame rate here**, not a
second problem: the core runs slow, so samples come out slow.

### Still open, unchanged from the entry above

* The close-hang every title on this Mesa has.
* `orbis_paths.cpp:19` hard-codes `/data/OpenGothic/` as the relative-path anchor.
* `ORBIS_TILE_MODE` has no reader in mesa-ps4 and `tempest-env.example.txt` documents it.
* The temporary ORBIS instrumentation in `vfs_implementation.c` and `core_info.c` - both carry
  "Remove once ps4/HANDOFF.md has the answer", and both answers are in this file now.

### And one upstream defect found on the way, not fixed

`libretro.c`'s generic no-shared-memory arm does not compile, and has not for as long as the
mman deps have been there:

    #define MAP_SHM(addr,size,fd,offset)\
    #define MAP_CODE(addr,size,fd,offset)\
            MAP(addr,size,fd,offset)

The first `#define` ends in a backslash with nothing after it, so it swallows the `#define`
below it and `MAP_SHM` becomes an empty macro that is then called as a function. Any platform
falling into that arm gets four errors. Left alone here - the PS4 arm sits above it and fixing
it invites a conflict with upstream - but it is why nobody has exercised that path.

---

## 2026-08-23 (late) — the recompiler runs, and what was actually slowing it down

Measured on hardware, not predicted. Beetle PSX HW, Spyro 3, Vulkan renderer, 2x internal.

    [PS4] lightrec: calling 0x10800000 to see whether it executes...
    [PS4] lightrec: 0x10800000 returned 0x00c0ffee - the code buffer executes
    [PS4] lightrec: 8192 KiB code buffer at 0x10800000, writable and executable
    Lightrec map addresses: M=0x10000000, P=0x899a9c020, R=0x2fc00000, H=0x2f800000
    [Lightrec]: Using 32-bit LUT

The io_base search took the FIRST 32-bit candidate, 0x10000000, so the mirrors, the BIOS at
+0x1fc00000 and the scratch pad at +0x1f800000 all placed on the first try and the 32-bit LUT
is available. `mirrors_mapped` is true; the map is not "perfect" only because that requires the
guest's RAM at host address 0, which libretro.c deliberately refuses (psx_mem == NULL would be
indistinguishable from failure).

### ⚠ The first run failed, and the defect was in the check rather than the platform

    lightrec: sceKernelMprotect returned 0 for 0x10800000 but the range reads back as
              prot 0x03, without execute. Not using it.       (x16, one per base address)

**`sceKernelQueryMemoryProtection` reports the protection a range was MAPPED with, not the one
`sceKernelMprotect` has just set.** The pages were executable throughout; mesa-ps4's probe had
already established that by CALLING a stub in exactly this arrangement (`MIRROR rung 7 OK`,
same day). It never asked the query API, so nothing had caught it.

⚠ **This is the fifth call on this console that answers, and answers about something else** -
after GB_ADDR_CONFIG, the three tessellation registers, and the readerless env knobs. The rule
this workshop keeps re-learning has a sharper form now: **on this platform, a query API is not
a measurement of the thing it names.** The check was not deleted, it was replaced with the
probe's own: write `b8 ee ff c0 00 c3` into the buffer and call it, expect 0x00c0ffee.

⚠ **And the pair of log lines around that jump are BOTH at RARCH_ERR on purpose.** This
frontend puts everything below ERR on UDP alone (ps4/ps4_log.h). The warning before the jump
was at ERR and the confirmation after it was at INFO, so a klog reader saw "calling %p" and
then nothing - which is exactly what the death that line warns about looks like.

### Where the frame really went, and it was a core option

With the recompiler running the driver's BUDGET instrumentation still said, in twelve
consecutive five-second windows:

    1.00 cores busy    exactly one thread saturated
    GPU waited 0 ms    every window, no exception
    submit path        12-18 ms out of 5000 = 0.3%

The saved options carried **`beetle_psx_pgxp_mode = "memory only"`**. That is not cosmetic:
`mednafen/psx/cpu.c` in PGXP memory mode sets

    lightrec_map[PSX_MAP_KERNEL_USER_RAM].ops = &pgxp_nonhw_regs_ops;
    lightrec_map[PSX_MAP_BIOS].ops            = &pgxp_nonhw_regs_ops;
    lightrec_map[PSX_MAP_SCRATCH_PAD].ops     = &pgxp_nonhw_regs_ops;

Without PGXP those three are NULL and recompiled code reads memory directly. With it, **every
load and store to main RAM becomes a C function call**, which is the most expensive single
thing that can be switched on in a PSX dynarec. The whole benefit is geometry precision.

The set that made it smooth, all live-togglable (cpu.c:2890 watches pgxpMode, invalidate and
spgp and re-inits Lightrec without a reload):

    PGXP Operation Mode                 Memory Only -> Disabled     the big one
    Dynarec SP GP Hit RAM Optimization  Disabled    -> Enabled
    Dynarec Code Invalidation           Full        -> DMA Only
    Dynarec …Event Cycles               128         -> 512
    GTE Overclock                       Enabled     -> Disabled     an overclock costs host time
    Software Framebuffer                Enabled     -> Disabled     the only visual risk

⚠ **Internal resolution is NOT a lever here and raising it is nearly free** - the GPU waited
0 ms in every window measured, both before and after.

### ⚠ A stub that returns -1 is not a neutral stub

    [Lightrec]: Threaded recompiler started with 1 workers.

on six cores. `orbis-compat/include/sys/sysctl.h` answered every sysctl with -1, and **every
caller's fallback for "how many CPUs" is one** - the answer that switches parallelism off
rather than degrading it. Lightrec's `recompiler.c` takes the `__FreeBSD__` arm,
`sysctlbyname("hw.ncpu", ...) ? 1 : count`; Mesa's `u_cpu_detect.c` asks the same question with
a `{CTL_HW, HW_NCPU}` mib, in every title.

Fixed in the overlay, both spellings, six cores, `ORBIS_NCPU` to override. Six and not a guess
at seven: `sceKernelGetCpumode()` exists and reportedly separates the two modes, but this SDK
ships no constants for its return values and six against seven is a rounding error beside six
against one.

⚠ **Mesa does not get this until it is rebuilt**, and whether it should is an open question
rather than an oversight: more util threads in a driver the GPU never waits on could contend
with the emulation thread. That is a measurement somebody should make deliberately.

### Two console-facing traps found while doing this

⚠ **FTP hands back a `.prx` DECRYPTED.** A file uploaded as 18903088 bytes of self reads back
as 19807720 bytes starting `7F 45 4C 46`. md5 can never match, and an upload that worked looks
like an upload that silently failed. Verify by SIZE and MTIME. (`deploy.sh` dodges this only
because its byte-for-byte check is limited to files under a megabyte.)

⚠ **Two UDP log receivers were bound to 18194** - one from that day and one left over from
2026-08-21. `log-receiver.py` sets `SO_REUSEADDR` and not `SO_REUSEPORT`, so Linux delivers a
unicast datagram to exactly one of them and which one is not defined. "The log says nothing"
can mean "it went to a file from two days ago". Check with `ss -ulnp | grep 18194` before
reading silence as evidence.

### Confirmed on hardware, same evening

    [Lightrec]: Threaded recompiler started with 5 workers.

⚠ **The first attempt at this reported `1 workers` from a correct fix**, and that is its own
entry above: `-MMD` omits `-isystem` headers, so nothing rebuilt and the `.prx` came out
byte-identical in size. `beetle-psx-libretro/ps4/build.sh` now stamps the overlay's newest
mtime alongside HAVE_LIGHTREC and cleans when either moves.

What the driver's BUDGET saw, gameplay windows only:

    before (PGXP on, 1 worker)     1.00 cores flat    152-189 presents / 5 s   ~31-38 fps
    after  (options + 5 workers)   0.80-0.95 cores    197-260 presents / 5 s   ~39-52 fps

⚠ **That delta does NOT separate the core options from the worker count** - both changed
between the two measurements, and the frontend was never run with one without the other. If
the split matters, `ORBIS_NCPU=1` in `/data/retroarch-env.txt` isolates the workers without a
rebuild.

Audio, which is the other thing five compile threads on six cores could have wrecked:

    1 worker    warm-up burst to 1145 underruns, then FLAT - no new underruns for two minutes
    5 workers   warm-up burst to  879 underruns, then FLAT from 40 s after load

So the burst is recompilation warm-up in both cases and it got *shorter*, not longer. No
starvation. One window did show 70 submissions and 61 presents at 0.95 cores - about 12 fps -
which is a genuinely heavy moment rather than a regression.

The GPU still waited 0 ms. Internal resolution remains free, and the remaining cost is
Beetle's own C++ (GPU command translation, SPU) plus the recompiled code itself.

### ⚠ The isolation run first measured nothing, and found something larger

`ORBIS_NCPU=1` was written into `/data/retroarch-env.txt`, the console relaunched, and the core
reported **five** workers. Not a parse error, not a stale build.

**The SDK's `libc.a` is a real static musl archive** - 1481 objects, `getenv` and `setenv` as
defined text rather than stubs into a shared libc module. So the eboot and every `.prx` it
loads link their own copy, each with its own `environ`. `platform_orbis.c`'s reader setenv()s
into the *executable's*; a core calling `getenv()` reads its own, which nothing ever wrote.

⚠ **EVERY ENV KNOB THIS WORKSHOP HAS IS EXPOSED TO THIS, and none of them showed it.**
ORBIS_3D_LINEAR, ORBIS_NO_TESS, MESA_LOG_FILE, RADV_DEBUG, all of tempest-env.example.txt - all
read by Mesa, and Mesa is linked INTO the executable. The first knob that had to reach a
loadable module was the first to fail, and it failed by looking exactly like a knob with no
reader: a clean run that reads as a measurement. That is the shape tempest-env.example.txt
spends half its length warning about, and it had a second cause nobody had named.

Fixed in the overlay: `orbis_env_get()` (`orbis-compat/src/orbis_env.cpp`, `include/orbis_env.h`)
answers from the image's own environment first and falls back to parsing the env files itself.
`sys/sysctl.h` uses it. ⚠ **Anything in a `.prx` that reads a knob must use it rather than
`getenv()`.** The file list naming "retroarch" is a seam, not a design - the honest mechanism is
a loader handing its module what it applied, and libretro has no channel for that.

### What the workers are actually worth

With the fix in, `ORBIS_NCPU=1` took, and the comparison is:

    warm-up underruns before the count goes flat, two runs each
      1 worker    1145, 1124     mean 1134   spread  21  (1.9%)     flat at ~45 s
      5 workers    879,  831     mean  855   spread  48  (5.6%)     flat at ~38 s
    steady state  both FLAT afterwards - zero new underruns, held for two minutes
    frame rate    NOT distinguishable; presents per window overlap and the scenes differ

**24.6% shorter warm-up, and nothing at all afterwards** - which is what the workers should
buy: they compile blocks, and once the blocks are compiled there is no work left. The two
groups do not overlap - the worst 5-worker run (879) is comfortably below the best 1-worker run
(1124), and the 245-underrun gap is five times either group's internal spread.

⚠ **THE SMOOTHNESS WAS THE CORE OPTIONS, PRINCIPALLY PGXP** - not the worker count. Both changed
in the same interval earlier and the entry above said the delta could not separate them. It can
now, and the answer is that the part which felt like the win was the part that was not being
tested.

⚠ **The metric is warm-up underruns, and it is only comparable across runs that BOOT THE DISC.**
Loading a save state skips the code the first run had to compile, which is the whole quantity
being measured.

`ORBIS_NCPU` has been removed from the console's env file. The default of 6 stands, because a
shorter warm-up for free is still worth having.

### Where the frame goes inside the core - and the target it is already hitting

⚠ **The per-thread route is closed.** `sceKernelGetCpuUsage` links and returns `0x8002004e`,
ENOSYS - this kernel does not offer it to this process. `ps4/ps4_threads.c` keeps the attempt
and the return code so nobody spends an evening on it again.

The split therefore comes from inside the core (`ORBIS_CORE_PROFILE=1`, beetle's `retro_run`).
Twelve consecutive five-second windows, Spyro 3, hardware renderer, 2x internal:

    249 frames every window, without exception   = 49.78 fps
    CPU_Run              9.8-12.3 ms/f   49-61%   emulated machine + everything it schedules
    rest of retro_run    7.8-10.3 ms/f   38-51%   parallel-psx building the frame, audio batch
    outside the core                       0.1%   the frontend itself

⚠ **THE DISC IS PAL AND THE CORE IS AT 100% OF TARGET.**

    [Core] Geometry: 320x240, FPS: 50.0000, Sample rate: 44100 Hz
    SET_SYSTEM_AV_INFO: FPS: 49.7610

49.76 requested, 49.78 delivered. There is no deficit left to recover in steady state - but the
thread is 99.9% busy doing it, so there is also no slack. Anything gained from here buys margin
against the dips rather than frames.

⚠ **HALF THE FRAME IS NOT THE RECOMPILER**, which is the answer to "can the JIT be optimised
further": at most half of a frame that is already meeting its target. And `rest` is not the
software framebuffer - that option is already off - nor the driver, whose submit path BUDGET
puts at 0.3% of wall. It is parallel-psx's own command building, above the driver.

⚠ **AND THERE IS A PACING PROBLEM NO AMOUNT OF SPEED FIXES:**

    [Video] Timings deviate too much. Will not adjust. (Target = 59.94 Hz, Game = 50.00 Hz)

A 50 Hz game on a 59.94 Hz output judders by construction, and this console does not output 50
Hz. An NTSC copy of the same title would match the display exactly. Worth knowing before
anyone reads uneven pacing as a performance problem and optimises at it.

`beetle_psx_dynarec_spgp_opt` is still `disabled` - the one recommended knob not applied.

### Scan Directory: "Scanning unsuccessful, no database found"

Two causes, both present, and neither is a defect in this port:

`/data/retroarch/database/rdb/` was **empty**. `tasks/task_database.c:3447` reports exactly that
message when the database list comes back size 0 and the scan is in LOOSE or STRICT mode. The
file wanted is `Sony - PlayStation.rdb` from libretro-database (7709655 bytes).

The core's `.info` declared **no `database` field**, so even with the .rdb present nothing
associates a scanned disc with this core. `ps4/build.sh` in the core's fork now writes
`database = "Sony - PlayStation"`, and the name must match the .rdb's filename exactly.

⚠ **AND THE FILE-MODE RULE IS NOT ONLY ABOUT FILES.** The directories RetroArch creates come out
`drwxr-x---`, which the FTP daemon cannot write into - so the upload "succeeded" and the
directory stayed empty, twice, with no error anywhere. The writable ones (`cores`, `info`,
`assets`, `shaders`, `system`, `roms`) are 777 only because they were chmod'ed by hand at some
point. `chmod 777` the directory before putting anything in a fresh one.

Both were fixed on the console by hand. For a shippable package the `.rdb` belongs in `/app0`
alongside the core and its `.info`.

## 2026-08-23 (late) — how many cores exist, and how many build

`ps4/build-cores.sh` clones cores from a libretro-super recipe, builds them with this toolchain
and links each into a `.prx`. **It does not patch their Makefiles** - Beetle PSX HW needed a
hand-written `orbis` arm and there are 99 more candidates, which is 99 more of those.

The trick that makes it work: the toolchain flags go INSIDE `$(CC)` and `$(CXX)`, not into
`CFLAGS`. A libretro Makefile routinely does `CFLAGS := ...` and throws away what the caller
passed; almost none of them rewrite CC. `platform=unix` then gives the core a sane arm, its own
link fails (host driver, `-shared`), and the objects are collected and linked here.

### The field

    184   cores in libretro-super's cores-linux-x64-generic
    162     plain Makefile (GENERIC)          the tractable class
     17     CMAKE                             needs a toolchain file
      5     GENERIC_GL                        ⚠ IMPOSSIBLE HERE - no OpenGL, only Vulkan
     99   of the 184 also appear in the Vita recipe - already ported to a fixed console once

### First sweep: 20 of 29

    OK       fceumm gambatte snes9x2010 quicknes mednafen_ngp mednafen_wswan mednafen_vb
             mednafen_lynx gme handy prosystem stella2014 tyrquake prboom race pokemini
             snes9x2005 vba_next fmsx freeintv numero
    LINK     genesis_plus_gx mednafen_pce_fast nestopia gearboy gearsystem picodrive vecx
    COMPILE  mgba

### ⚠ Three things the sweep found that no single core would have

**LTO throws the whole libretro API away and the link reports success.** Several cores compile
with `-flto` under `platform=unix`, so their `.o` files are bitcode - `llvm-nm` prints dashes
where the address belongs. ld.lld links bitcode, runs LTO, and internalises everything
unreachable from an entry point; a module has no entry point. `snes9x2010` linked to a 915 KiB
ELF containing **zero** `retro_*` symbols, exit code 0. Fixed by naming all 25 libretro entry
points with `-u`. The harness's "linked without retro_run" guard is what caught it and it earned
its place on the first batch.

**A .prx cannot borrow the frontend's libretro-common.** Cores call `retro_vfs_*_impl`,
`cdrom_*` and friends without building them, because a shared object resolves them lazily
against the frontend at load time. A module has its own symbol table. The harness archives the
frontend's 116 libretro-common objects plus 138 it compiles itself (35 will not build here) and
puts the **archive** after the core's own objects - so a core that did build its own copy keeps
it, and nothing is duplicated.

**FIONREAD was a trap, not an absence.** The SDK defines no `FION*` at all. Taking musl's would
have been the obvious fix and would have been wrong: musl carries LINUX request numbers
(`0x541B`) and this kernel encodes direction, length and group into the number
(`FIONREAD = 0x4004667f`). `orbis-compat/include/sys/ioctl.h` now derives them rather than copying
them. ⚠ This is **not** the same shape as the mmap constants, which is a comparison this file used
to draw and which was wrong: the SDK corrects `MAP_ANON` itself in `bits/mman.h`, while it defines
no `FION*` at all. ⚠ **Nothing in this workshop has yet called ioctl() on this console**, so these are
reasoned, not measured.

### The remaining failure classes, each with a shape

    cdrom_lba_to_msf            libretro-common/cdrom/cdrom.c needs HAVE_CDROM, which CHANGES
    (genesis_plus_gx,           the layout of libretro_vfs_implementation_file. Forcing it into
     mednafen_pce_fast)         the shared archive would mix two struct layouts in one link -
                                silent and worse than the missing symbol. Per-core work.

    missing C++ class symbols   Their Makefiles compile straight from sources to the .so in ONE
    (gearboy, gearsystem)       command and never write a .o, so there is nothing to collect.
                                STATIC_LINKING=1 does not change it. Needs a real platform arm.

    nestopia                    12 objects only; its own vendored code fails earlier.
    picodrive                   dr_mp3.h: duplicate case value - a source/compiler disagreement.
    vecx                        glIsEnabled. Correctly impossible: this port has no OpenGL.
    mgba                        no objects and no errors - wrong makefile for the recipe entry.

⚠ **A core that builds is not a core that works.** This harness reports what compiled. Every one
of the twenty is unrun; the console is the only thing that can tell a working core from a
linking one.

### The whole recipe, built: 100 of 162

Every `GENERIC` core in `cores-linux-x64-generic`, one at a time. **453 MB of `.prx` in
`~/.cache/ps4-cores/out`**, with `cores.manifest` recording core, verdict, size and the upstream
commit each was built from.

⚠ **NOT IN /tmp, AND THAT IS NOT A DETAIL.** The first sweep's output lived in the session
scratchpad; a cold reboot cleared tmpfs and took 52 built cores, every clone and the manifest
with it. `build-cores.sh` already defaulted to `~/.cache/ps4-cores` and the default was being
overridden on every call.

    100  OK
     45  LINK       an undefined symbol, named in the manifest
     10  COMPILE    no objects at all
      6  NO-ABI     linked, but without retro_run
      1  CLONE      submodule fetch failed

### Two classes closed during the sweep, worth 7 cores

**`HAVE_CDROM=0`.** It gates passthrough to a *host CD device* - a real drive opened by path -
which this console does not have, so the feature could not work here whatever it linked. It also
adds a member to `libretro_vfs_implementation_file`, so the flag decides a **struct layout shared
between objects**. Satisfying the resulting `cdrom_lba_to_msf` from the shared archive would have
put two layouts of one struct into a single link: silent, and far worse than a missing symbol.

**`ZSTD_trace_*` are undefined WEAK.** ld.lld leaves them undefined, which is what weak means.
create-fself will not: it maps every remaining undefined symbol to a library NID, finds none, and
refuses the module - ⚠ **with exit status 0**, so a script trusting the exit code sees success and
no file. Fixed with real no-op definitions (`ps4/orbis_weak_stubs.c`).

⚠ **And the first attempt at that fix did nothing, instructively:** the stubs went into the
fallback *archive*, and **an archive member is never extracted to satisfy a weak undefined
symbol**. They have to be passed as a plain object.

### What is left, by cause rather than by core

     8  OpenGL / X11        ⚠ NOT FIXABLE HERE and should not be listed as work. boom3, boom3_xp,
                            craft, desmume, kronos, vecx, vitaquake2, vitaquake3. Note desmume2015
                            (same system, software renderer) builds fine - several of these have a
                            sibling that already works.
     8  no .o at all        blastem, higan_sfc, higan_sfc_balanced, mame2016, mgba, rustynes,
                            scummvm, squirreljme. Their Makefiles go from sources to the shared
                            object in ONE command and never write an object. STATIC_LINKING=1 does
                            not change it. Each needs a real platform arm - a patch.
     7  vice_*              562 objects each, then undefined `log_cb`, `pix_bytes`, `opt_vkbd_alpha`.
                            One family, one TU failing, one patch away from seven cores.
     6  NO-ABI              bsnes2014, freej2me, hbmame, mame, openlara, stella. Built something,
                            not a libretro core.
     2  FLAC                yabasanshiro, yabause - vendored libFLAC not in the source list.
     2  hiro::              bsnes, bsnes_hd_beta - byuu's GUI toolkit pulled into a libretro build.
    12  one-offs            SDL_GetTicks, unzOpen2, sk_num, JS_ToInt32, wasm_rt_trap, mp3dec_start,
                            osd_malloc, BurnDrvCps1944j, BurnSampleReset, inet_htons, ace::ace,
                            ARMJIT_Memory, AMeteor, cEmuSCV, GetKeyState, inflateInit2_.

⚠ **NONE OF THE HUNDRED HAS BEEN RUN.** The harness reports what compiled. A core that links and
draws nothing is a pass here and a failure on the console, and only the console knows.

---

## 2026-08-24/25 — running the hundred, and three loader defects underneath them

The sweep of the previous entry produced binaries. This one is about what happened when they
were run, and the answer is that **the first seventeen results were about the frontend, not the
cores**. `ps4/CORE-STATUS.md` carries the per-core table; this is what it cost to get there.

### ⚠ No .prx on this console had ever run a global constructor

Every C++-dominant core crashed on load with

    terminating with uncaught exception of type std::length_error:
    allocator<T>::allocate(size_t n) 'n' exceeds maximum supported size

and every C core was fine - seventeen results, the split exactly along the language. It was
never about C++. Three defects, each found only by running, and **two of them introduced by the
fix for the first**:

    1  crtlib.o marks module_start GLOBAL HIDDEN, so the linker correctly makes it LOCAL, and
       create-fself builds a module's export table from GLOBAL symbols in .symtab. The loader
       can never find it. Constructors never ran, so every C++ global stayed unconstructed and
       the first request for a size returned garbage.

    2  the frontend then ran them from dylib_load - on EVERY load. sceKernelLoadStartModule
       returns the SAME id for an already-loaded module and RetroArch loads a core several
       times, so mednafen_gba constructed its four globals EIGHT times. That presented as
       `failed_to_start_audio_driver` and an assertion deep inside Blip_Buffer.

    3  the once-per-module guard then held ids across unload, and the kernel REUSES them.
       nestopia unloaded, quicknes took its id, its constructors were skipped, and it died on a
       null read at 0x20 - a SIGSEGV that looks exactly like a broken core.

`libretro-common/dynamic/dylib.c` holds the whole chain in a comment; `ps4/orbis-module.ld`
brackets `.init_array` because OpenOrbis's `link.x` collects it and defines nothing around it
while `crtlib.o` carries the bounds as BSS variables eight bytes apart.

⚠ **THE METHOD LESSON IS THE ONE TO KEEP.** Four rounds were spent reading meaning into silence:
a probe that logged through klog printed nothing, which could equally mean "the constructor did
not run" or "klog from a .prx does not reach the host"; then a probe without a priority sat at
the END of the array, so it could say nothing about a module that dies earlier. What settled it
was a **control**: the same probe in `fceumm`, a core known to work. That should have been the
first experiment, not the fourth. And a verdict of `crash` against a core is worth very little
until the loader underneath it is known to be sound - seven cores sat condemned for this.

### The `.info` files under-declare, in three different ways

Every attempt to build a content list from core metadata came out short, and only the console
showed it:

    arcade cores   say their BIOS requirement in `notes` prose, with no firmware fields at all.
                   fbalpha2012_neogeo refused Metal Slug with `NeoGeo BIOS missing` and a list of
                   files nothing had asked for. neogeo.zip was in the repository all along.
    a5200          marks a mandatory BIOS `firmware0_opt = "true"`. It does not start without it.
    nxengine       needs a whole game data directory, described only in `notes`.

⚠ **A core's own metadata is a hint. The only reliable statement about what it needs is the core
refusing to start and saying so.** All 131 optional firmware files the repository carries are now
on the console alongside the required 34; arcade BIOS goes in the ROM directory, not `system/`.

### Two things that are now known and were not

**The archive path works.** `xrick` declares `zip` as its only extension and played straight out
of `data.zip`. The earlier warning that compressed formats were unexercised is answered for
`.zip`; `.chd` and `.7z` still are not. The corpus stays extracted anyway, and
`snes/Super Mario World.zip` still sits beside the `.smc` as a control.

**Arcade is blocked on romset vintage, not on the port.** Checked by CRC against FBA's own DAT
over all 231 sets to hand: 129 match FBA 2012, 98 are the wrong revision, 4 are unknown to it.
⚠ The first run of that check said "1 match" - a merged DAT lists the BIOS chips inside every Neo
Geo game while they live in `neogeo.zip`, so counting them as missing condemned a library that
had just been observed playing. The observation was right and the script was wrong.

### ⚠ The harness will overwrite a hand-ported core, and did

`build-cores.sh` clones UPSTREAM. Its `mednafen_psx_hw_libretro.prx` is plain Beetle - no orbis
arm, no `ps4/orbis_lightrec_mem.c`, no dynarec default - and it lands on the same filename as the
fork in `~/src-ps4/beetle-psx-libretro`. It did, and Spyro got slower for no visible reason. Only
the console copy was affected; the fork's source was never touched.

`PS4_CORE_FORKS` now names such cores. The harness builds them and reports `FORK - built, NOT
written`.

### Where testing stands

    31  plays        including every core that had been recorded as crash
     7  blocked      arcade, waiting on romsets of the right vintage
     1  broken       nestopia - loads, runs, exits cleanly, renders a green screen
    63  untested

Content staged on the console: NES, SNES, GBA, Mega Drive, Neo Geo (with `neogeo.zip`), PSX,
OutRun, Cave Story, Dinothawr, xrick, and Freedoom + `prboom.wad` for PrBoom. Of the untested,
about eight need no content at all; the rest want systems nothing on this machine has.

### ⚠ CORRECTION, 2026-08-25: the GL arm runs, and it is no longer an arm

This entry first recorded GL as *"a gate, not a product - it builds and it LINKS; it does not
run"*, quoting `build-support/orbis/build.sh`. **That comment was false when it was written**, and
mesa-ps4 has since deleted it (`62902e9a229`). GL runs on hardware: a frame, a triangle with
shaders, and Vulkan alive beside it in the same process.

The `--gl` flag is gone with it. `COMMON_OPTS` now carries zink, EGL and GLES2 beside RADV,
everything lands in one `build-orbis`, and the GL link probe runs on every build and fails it if
`libEGL.a` is missing. Two halves compiled separately are two halves that were never compiled
against each other, and this port needs both in one eboot with the core chosen at run time.

    build-orbis/src/amd/vulkan/libvulkan_radeon.a   36 MB
    build-orbis/src/egl/libEGL.a                     5.7 MB
    build-orbis/src/mesa/glapi/es2api/libGLESv2.a
    build-orbis/src/gallium/drivers/zink/libzink.a
    build-orbis/src/gallium/targets/dri/libgallium-*.a

⚠ **Vulkan-only consumers pay nothing** - they link `libvulkan_radeon.a` and the gallium archives
go unreferenced. So this frontend's current link line keeps working untouched.

Two things the GL link probe learned that any consumer will meet, both found on binaries an
earlier probe had passed:

    Mesa's dispatch tables reference their entry points WEAKLY, and a weak undefined reference
    does not extract an archive member - it resolves to zero. Without --whole-archive on the ICD
    the executable linked with vk_common_GetPhysicalDeviceProperties2 still undefined and jumped
    to address 0 on the first dispatch. (Same shape as the ZSTD_trace_* stubs in build-cores.sh:
    a weak undefined symbol never pulls an archive member.)

    .tdata.* is an orphan section under the SDK's own link.x, which cost the RW segment its page
    alignment and made the console refuse the file outright.

### What is still ours to do for GL

⚠ **This frontend has `orbis_vk_ctx.c` and nothing for `RETRO_HW_CONTEXT_OPENGL`.** A GL core asks
the frontend for a context through that mechanism; a working zink underneath changes nothing
until that driver exists. That half was always ours and still is.

What it would unblock, and what it would not: the eight cores the sweep listed as OpenGL-blocked
mostly have working software siblings (`desmume` against `desmume2015`). The real prize is
`parallel_n64`, `mupen64plus_next` and `ppsspp` - which are also the heaviest things this console
would be asked to run, on a port where Beetle PSX already spends a saturated core on the
interpreter. Worth starting now that the driver renders; worth expecting the frame rate to be the
next problem rather than the last one.

## Nintendo 64: what the frame is actually spent on

`mupen64plus_next` runs Shadows of the Empire at 22-25 fps. Everything below was measured on
hardware on 2026-08-25 with `ps4/orbis_profile.c`, which is linked into every core the harness
builds and turns itself on when `/data/retroarch-profile` exists. It reports every five seconds:

    129 frames in 5043 ms = 25.57 fps
      retro_run total  38.65 ms/f  98% of wall
      guest            37.44 ms/f  95%
      rsp-lle          33.94 ms/f  86%     <- rdp-submit is NESTED inside this
      rdp-submit       10.13 ms/f  25%
      rdp-enqueue       5.18 ms/f  13%
      rdram-h2g         0.48 ms/f   1%
      present           1.20 ms/f   3%

Unnested, per frame: **LLE RSP 24 ms, RDP command building 10 ms, R4300 dynarec 4.7 ms, scanout
1.2 ms.** The recompiler this port spent a day giving executable memory to is 12% of the frame.

⚠ **THE GPU IS NOT THE BOTTLENECK AND NEVER WAS.** `present` is one millisecond. A core sold as
"ParaLLEl-RDP on Vulkan" puts only the RASTERISER on the GPU; the RSP is a programmable vector DSP
running the game's own microcode, recompiled by GNU lightning onto the CPU, and it is the largest
single cost by a factor of two. Anyone reading "Vulkan" as "the graphics are free" will optimise
the wrong half.

### The thirteen milliseconds that were handshake

ParaLLEl-RDP processes its command stream on a worker thread, and `CommandRing::enqueue_command`
takes a mutex and signals a condition variable **once per RDP command** - thousands of round trips
a frame. Measured before the change:

    rdp-enqueue   17.08 ms/f    emulation thread blocked in the ring
    rdp-worker     7.09 ms/f    what the worker actually did

The consumer was busy for seven milliseconds and the producer waited seventeen. Setting
`single_threaded_processing` took the frame from 51.9 ms to 38.7 ms - **19 fps to 25.6** - and
`rdp-enqueue` from 17.08 to 5.18.

⚠ **THIS IS A FACT ABOUT THIS MACHINE, NOT ABOUT THE DESIGN.** The ring is right where a spare core
is cheap to reach; here the cores are 1.6 GHz Jaguars and **nothing in this port sets thread
affinity**, so a handoff can be a context switch on the same core. If affinity is ever set up,
re-enable the ring and measure again before believing this entry.

### A direction that was measured and abandoned

parallel-RDP warns at startup:

    VK_EXT_external_memory_host is not supported by this device. Application might run slower
    ... falling back to a slower path.

That message reads like the explanation for everything above, and it is not. The fallback path is
`Renderer::resolve_coherency_host_to_gpu`, page-granular memcpy of dirty RDRAM into a GPU-visible
buffer, and it costs **0.5 ms a frame**. Importing RDRAM as host memory - whether by re-enabling
`has_userptr` in mesa-ps4 or by allocating RDRAM from `vkAllocateMemory` - would buy half a
millisecond out of thirty-nine. Both were planned in detail before anyone measured them.

⚠ The driver's own comment (`ac_gpu_info.c`, "nothing on this console can use it either way") is
still correct in outcome, for a reason it does not give: something does want the extension, and
would gain almost nothing from it.

### What is left, and what it is worth

    LLE RSP           24 ms    only HLE removes this, and HLE needs a graphics plugin that
                               accepts display lists - GLideN64, which needs RETRO_HW_CONTEXT_OPENGL
    RDP commands      10 ms    CPU-side command building; the ring may return here with affinity
    R4300              4.7 ms  the recompiler, working as intended
    scanout            1.2 ms  the GPU

ParaLLEl-RDP implements `ProcessRDPList` and leaves `ProcessDList` empty, so the HLE RSP draws
nothing with it - tried on hardware, the log said `Plugins in use: RDP=ParaLLEl RSP=HLE` and the
screen stayed black. **N64 at full speed on this console is a GL context driver away, not a
tuning exercise away.**

## OpenGL, and Nintendo 64 at full speed

`gfx/drivers_context/orbis_gl_ctx.c` gives this frontend a `RETRO_HW_CONTEXT_OPENGL` context.
Built with `make -f Makefile.orbis HAVE_VULKAN=1 HAVE_OPENGLES=1 ...`; the Vulkan flag is not
optional, because there is no GL hardware path here at all:

    eglSwapBuffers -> kopper -> vkQueuePresentKHR -> VK_EXT_headless_surface -> wsi_orbis -> flip

On hardware, 2026-08-25: `OpenGL ES 3.1 Mesa 26.3.0-devel`, renderer `zink Vulkan 1.3 (RADV
ORBIS)`. mupen64plus-next with GLideN64 and the HLE RSP, Shadows of the Empire:

    303 frames in 5015 ms = 60.41 fps
      retro_run total  16.88 ms/f
      guest             5.18 ms/f  31%    <- all of the emulation
      present          11.68 ms/f  70%    <- idle, waiting for the flip

Five milliseconds of work in a 16.7 ms frame, against 43 ms on ParaLLEl-RDP. The arithmetic in
the section above said full speed was unreachable without this, and it was right about both
halves.

### Three things that each cost a round, and all three were the same mistake

⚠ **A statically linked Mesa does not make `__rglgen_` pointers into direct calls.** gl2.c guards
`rglgen_resolve_symbols()` with `#if !defined(RARCH_CONSOLE)`, on the reasoning that a console
links GL statically. This one does, and it changes nothing: the entry points glsym turns into
POINTERS stay pointers. Unresolved they are null, and the symptom was "Couldn't find any
supported shader backend" followed by SIGSEGV with `rip = 0`. Neither line says "unresolved".

⚠ **A context driver that reports no shader flags is not neutral.** `gl2_get_fallback_shader_type`
asks the CONTEXT driver which shader languages exist - "for gl2, shader support is completely
defined by the context driver shader flags" - and `gl2_shader_init` then logs an error and
returns TRUE. The driver comes up fully initialised with `gl->shader` NULL and presents empty
frames: black screen, no menu, and GoldHEN's counter reading a steady 60 fps.

⚠ **A core's GLES entry points must be THUNKS into the frontend's context, never a second copy of
Mesa's dispatch.** libGLESv2.a is the mapi dispatch table; the context is current in the
frontend's copy, so a copy inside the module would be empty. `ps4/orbis_gl_forward.c` forwards
133 entry points, resolved once from `context_reset` through the frontend's proc address. The
thunks are assembly because a C forwarder needs the exact prototype of all 133 and a wrong one
is not a compile error - it is arguments in the wrong registers, silently.

### ~~Open: framebuffer emulation and depth~~ SOLVED 2026-08-28

⚠ **Read the two entries at the end of this file before anything below.** The cause was GLideN64
throwing the depth attachment away over a one-pixel width difference, on GLES only; it is fixed in
`ps4/core-patches/mupen64plus_next/0008` and confirmed on hardware. The suspects named below were
in the wrong place, and the account of how that was narrowed is worth more than they are.

With `Framebuffer Emulation` ON the depth is wrong. Turning it off fixes it and is the current
recommendation. It is the first time GLideN64 has run over zink on this GPU, so the fault could be
in any of GLideN64, zink, RADV or these thunks.

⚠ **THE SYMPTOM WAS FIRST WRITTEN DOWN HERE AS "geometry behind the camera draws in front" AND
THAT WAS AN INTERPRETATION, NOT AN OBSERVATION.** Re-tested on hardware on 2026-08-28, Shadows of
the Empire: **the player's ship draws behind every other 3D object in the scene** - an enemy far
away appears in front of it, and bases on the map draw over it. Collisions still fire at the right
moment, so the game's own state is untouched and this is the renderer alone. That is a different
statement from the first one and it points somewhere else: "behind the camera draws in front" reads
as a clipping fault, and "the first thing drawn loses to everything drawn after it" reads as **no
depth test at all**.

**Read from the source, three candidates, ranked.** All three were found in the clone at `f275caf`
and none has been measured yet.

⚠ **1. GLideN64 can drop the depth attachment entirely, and only on GLES.**
`FrameBufferList::attachDepthBuffer` (`FrameBuffer.cpp`) will only attach a depth buffer whose
texture matches the colour buffer's width, and the test it uses is chosen by
`Context::WeakBlitFramebuffer`, which **is `isGLESX`**:

    GLES     goodDepthBufferTexture = depthTexture->width == colour->width     exact
    desktop  goodDepthBufferTexture = depth >= colour || |m_width difference| < 2

On a mismatch it sets `pCurrent->m_pDepthBuffer = nullptr` and the FBO is rendered into **with no
depth buffer**: every fragment passes, and the picture is ordered by draw order. And a mismatch is
reachable, because `DepthBuffer::initDepthBufferTexture` creates the texture once, sized to
whichever colour buffer happened to be current, and returns early ever after - so a game pairing
one depth buffer with colour buffers of two widths loses depth on the second. This is the only
candidate that is simultaneously framebuffer-emulation-only, GLES-only, and silent.

**2. The same thing from underneath: a `GL_DEPTH_COMPONENT24` texture attachment that zink or RADV
does not honour.** Indistinguishable from 1 in the picture, distinguishable in one log line -
either GLideN64 dropped the attachment or it did not. With framebuffer emulation off there is one
screen-sized buffer and the widths always agree, which is why that path is unaffected either way.

⚠ **3. A configuration this build splits that upstream treats as a pair.** `enableClipping` is
forced to 1 for every GLES context (`opengl_GLInfo.cpp`), which makes the vertex shader emit
`gl_Position.z /= 8.0` - deliberately, to stop the hardware clipping geometry the software clipper
is about to handle. The only code that scales that back is in `writeDepth()`, and `writeDepth()` is
compiled to `return 0.0;` when `enableFragmentDepthWrite == 0` **and** `N64DepthCompare` is
disabled. That is exactly this build: mupen64plus-next defaults `EnableFragmentDepthWrite` to
`False` under `HAVE_OPENGLES`, and it does not even expose `EnableN64DepthCompare` there. So the
`/8` is never undone.

⚠ **But candidate 3 predicts the wrong direction, which is why it is ranked last rather than first
despite being the one provable defect.** The software clipper runs *only* with framebuffer
emulation ON (`GraphicsDrawer::drawTriangles` calls `renderAndDrawTriangles` in that branch and a
plain `drawTriangles` in the other), so the `/8` leaves near-plane geometry unclipped in the
configuration the user says looks CORRECT. Depth ordering itself survives the `/8` because dividing
by 8 is monotonic. It is worth fixing on its own account; it is probably not this.

**The experiment that costs nothing runs first:** with framebuffer emulation ON, set
`GPU shader depth write` to True in the core options. No rebuild, no upload. It re-enables the
whole `writeDepth()` path and settles candidate 3 by itself.

**And `ps4/core-patches/mupen64plus_next/0006` answers 1 against 2 on the next build.** ⚠ It exists
because **GLideN64 is compiled with `LOG_LEVEL LOG_NONE`**, so every warning it writes - including
`GLInfo`'s own "your GPU does not support the extensions needed for..." lines - is discarded before
it is formatted. On this console that is silence by construction, and it is why none of the above
could be read off a log. The patch reports one line of capabilities at context creation and one
line per dropped depth attachment, through `orbis_report`, bounded at four because klog costs
8-15 ms a line.

## The toolchain's own bugs, and the instruments built to find them

Three defects this week were in the SDK rather than in any core, and all three presented as a core
crashing. They are recorded together because the shape repeats: **a prebuilt library compiled for a
different platform's constants, shipped for this one.**

### ⚠ libc++.a compares against LINUX'S ETIMEDOUT on a FreeBSD target

Observed loading a ROM in mupen64plus-next:

    libc++abi: terminating with uncaught exception of type std::__1::system_error:
               condition_variable timed_wait failed: Operation timed out

"Operation timed out" IS errno 60, ETIMEDOUT, and a timeout is the ordinary outcome of a timed wait -
`wait_for` returns `cv_status::timeout` and does not throw. It threw because the comparison that
filters that case out was compiled against a different number. Disassembling
`condition_variable.cpp.o` out of the SDK's `libc++.a`:

    322: 83 7d 94 6e    cmpl $0x6e, -0x6c(%rbp)     ; 0x6e = 110

110 is Linux's ETIMEDOUT. This console is FreeBSD underneath, its pthread returns 60, and the SDK's
own `bits/errno.h` agrees with 60. **Every timed wait that has ever expired on this platform threw.**

⚠ **AND IT IS NOT LIMITED TO condition_variable.** `std::future::wait_for`, `shared_timed_mutex` and
everything else with a deadline goes through `__do_timed_wait`. The call site that killed the N64
core is `CommandRing::thread_loop`, which does `cond.wait_for(holder, 500us, …)` whenever the RDP
worker has nothing to do - so the core died the first moment it went idle.

`ps4/orbis_cv_fix.cpp` rebuilds `condition_variable`'s four strong symbols from libc++ 11's own
source against this platform's headers. ⚠ All four, not just the broken one: an archive member is
all-or-nothing, so providing `__do_timed_wait` alone would leave `notify_one`, `notify_all` and
`wait` undefined. `notify_all_at_thread_exit` is deliberately absent - it needs libc++'s private
per-thread bookkeeping and no core here has referenced it.

⚠ **Fixing it at `pthread_cond_timedwait` instead would have been wrong**, and the reasoning is worth
keeping: everything compiled in this workshop sees ETIMEDOUT as 60 through the SDK's headers and
tests for 60. Only the prebuilt library expects 110. Translating the return value would fix libc++ by
breaking every honest caller. The mismatch belongs where it was introduced.

The real fix is a libc++ rebuilt against the target's headers, which is an OpenOrbis change.

### A core's dying words now reach a channel someone reads

`assert()`, libc++abi's `abort_message()` and `__cxa_pure_virtual` all say exactly what went wrong and
then call `abort()`. All of it goes to stderr, and on this kernel fd 2 goes nowhere.

⚠ **AND POINTING fd 2 SOMEWHERE IS NOT AVAILABLE:** `dup2` onto 1 or 2 returns **EPERM**. The
descriptor table is not ours to rearrange, so the message has to be caught before it is written
rather than after.

`ps4/orbis_abort_report.c` defines `abort_message`, `__assert_fail` and `abort` itself, overriding the
toolchain's by definition order - each lives in an archive member of its own defining that one symbol,
so a definition reaching the linker first means the member is never pulled. It reports through
`sceKernelDebugOutText` (synchronous; a UDP datagram needs the process to survive its own send) and
appends to `/data/retroarch-abort.log`.

⚠ **The one that matters most is `abort()` itself, and what it must produce is the RETURN ADDRESS.**
Plenty of code aborts without a message - libunwind's `_LIBUNWIND_ABORT`, GNU lightning, paraLLEl-RSP's
allocator. Those arrive as a bare SIGABRT with `abort` as the only frame, because a core is built
`-fomit-frame-pointer` and the console's backtracer cannot walk through it. The return address is on
the stack whether or not there is a frame pointer. It found `RSP::JIT::CPU::init_jit_thunks` in one
step, after two rounds of guessing had found nothing.

⚠ **`-DNDEBUG` IS PER TRANSLATION UNIT.** A core built with it still links libretro-common,
orbis-compat and vendored dependencies that were not. An assertion in one of those is
indistinguishable from a C++ runtime failure from the outside.

### Measuring instead of arguing

`ps4/orbis_profile.c` is linked into every core the harness builds and turns itself on when
`/data/retroarch-profile` exists - a file rather than an environment variable, because each image
carries its own static musl and `setenv` in the frontend is invisible to a `.prx`.

It exists because three performance stories in one day turned out to be wrong before it was written.
The section above on the N64 frame is entirely its output.

## Networking: the socket layer was never missing

⚠ **THIS SECTION REPLACES AN EARLIER ONE THAT WAS WRONG, AND THE MISTAKE IS WORTH KEEPING.** The
previous inventory checked `libc.a`, `libpthread.a` and "`libkernel.a`" at symbol level, found nine
of the twenty-two POSIX socket calls, and sized a rewrite of the whole API over `sceNet*` as an
archive-member override. **There is no `libkernel.a`.** The socket calls arrive through the DYNAMIC
stub `lib/libkernel.so`, which the OpenOrbis stub generator built from retail libkernel's export
list, and `-lkernel` has been on this port's link line since the first build.

    present, from libkernel.so (dynamic)   socket connect bind listen accept send recv sendto
                                           recvfrom shutdown setsockopt getsockopt getpeername
                                           getsockname select poll close fcntl sendmsg recvmsg
    present, from libc.a (static)          getaddrinfo freeaddrinfo getnameinfo gethostbyname
                                           gethostbyname2 inet_pton inet_ntop inet_addr inet_aton
                                           getifaddrs if_nametoindex in6addr_any accept4 signal
    missing                                nothing

Proven, not inferred, three ways:

1. **A link probe.** A translation unit taking the address of all twenty-two calls, compiled with
   this port's exact flags and linked with this port's exact `LIBS` line, links. The only symbol it
   could not resolve on the first attempt was `sceNetGetDnsInfo`, and `-lSceNet` was already there.
2. **The SDK's own libc is built on it.** `getaddrinfo.lo` in `libc.a` has undefined references to
   `socket`, `connect` and `close`; `fcntl.lo` calls libkernel's `_fcntl`; `send.lo` is a tail call
   to `sendto` and `recv.lo` to `recvfrom`; and musl's `socket()` is a wrapper over
   `__sys_socketex(name, domain, type, protocol)` - the same name-taking syscall `sceNetSocket`
   uses, passing `""`. There is ONE descriptor namespace here, and it is the kernel's.
3. **The SDK ships a sample that uses it.** `samples/networking/` opens a TCP listener with plain
   `socket`/`bind`/`listen`/`accept`/`close` over `<sys/socket.h>` and `<netinet/in.h>`.

⚠ **AND `errno` IS SHARED, WHICH IS THE PART THAT WOULD HAVE BEEN HARD TO GET RIGHT BY HAND.**
`libc.a`'s `__errno_location` is a single `jmp` to libkernel's `__error`, and `bits/errno.h` is
FreeBSD's table - the one those syscalls actually set. `EINPROGRESS` is 36, `EAGAIN` 35,
`ECONNREFUSED` 61. So `isagain()` and `isinprogress()` in `net_compat.h` read the right values with
no translation, and a `sceNet*` wrapper would have had to invent a mapping to replace something
that was already correct.

⚠ **SO DO NOT OVERRIDE `socket()`.** The two-namespace hazard the old section warned about is real,
but it is what the override would CREATE, not what it would fix: `sceNetSocket()` returning an
`OrbisNetId` into musl's `getaddrinfo`, which then calls libkernel's `connect()` and `close()` on
it. And `close()` cannot be overridden anyway - it is the file close as well as the socket close, so
an override would have to dispatch on descriptor type. There is nothing to gain and a working layer
to lose.

**What this platform DID need**, all of it landed in phase 5a:

* `network_init()` in `libretro-common/net/net_compat.c` had no ORBIS arm and fell through to the
  generic one, which only ignores `SIGPIPE`. `sceNetInit()` has to run first: musl's resolver reads
  the console's DNS servers through `sceNetGetDnsInfo` (`resolvconf.lo`), which answers with an
  error until the stack is up. Name resolution failing on a console that is plainly online is what
  that omission looks like. The return code is deliberately not fatal - it is negative when the
  stack is ALREADY up, which is the normal case here because `optional/orbis_netlog.cpp` calls
  `sceNetInit()` during early boot and then sends datagrams successfully on hardware.
* `HAVE_SOCKET_LEGACY` had to go from 1 to 0. Inherited from the console Makefiles this port was
  modelled on and inert while networking was off; with it set, the build stops on `redefinition of
  'addrinfo'`, because `net_compat.h` declares its own for platforms that have no `getaddrinfo` and
  this one has a real one in `netdb.h`.
* `DEFAULT_BUILDBOT_SERVER_URL` had no ORBIS arm. ORBIS is not `__linux__` and not any other case in
  that chain, so it fell to the final `""` - a Core Downloader that fetches nothing and explains
  nothing, and an invitation to "fix" it by borrowing the Linux/x86_64 URL, which would fill the
  list with x86-64 ELF objects that install and never load.

⚠ **`-lSceNet` IS NOW LOAD-BEARING FOR libc, NOT JUST FOR THE LOG CHANNEL.** `sceNetGetDnsInfo` is
the single symbol the whole socket API needs from libSceNet; `sceNetInit` is the second, from our
own arm. Removing `-lSceNet` because logging is off in a release build fails at link time.

⚠ **THERE IS NO `HAVE_NETPLAY` SWITCH IN THIS TREE.** `Makefile.common`'s `HAVE_NETWORKING` block
adds `network/natt.o`, `network/netplay/netplay_frontend.o`, `netplay_room_parse.o`, the three
`tasks/task_netplay_*.o` and `-DHAVE_NETWORK_CMD` unconditionally, and `runloop.c` defines
`core_set_netplay_callbacks` behind `HAVE_NETWORKING` alone. All seven compile clean for this
target and are dead code unless a session is started, so they are carried. Removing them means an
ORBIS `#ifdef` in `retroarch.c`.

**Timeouts need no work.** `socket_connect_with_timeout()` is non-blocking `connect` + `poll` +
`getsockopt(SO_ERROR)`, and `NETWORK_HAVE_POLL` is set for this platform because it takes the
generic POSIX branch. `poll` is a real libkernel export. There is no `sceNetSelect` and the
`sceNetEpoll*` family is declared in this SDK as `void sceNetEpollWait();` - names without
signatures - and neither fact matters, because nothing calls them.

**For TLS (5b):** `time()`, `getrandom` and `getentropy` are all present in libc. BearSSL and
mbedTLS are vendored under `deps/`; `HAVE_BUILTINBEARSSL := 1` in `Makefile.orbis` is the whole
switch, and it sets `HAVE_SSL` itself. Sony's `libSceSsl.so` and `libSceHttp.so` are present as
stubs and `orbis/Ssl.h` declares `void sceSslConnect();` - names without signatures - but
`samples/net_http/` shows the working call sequence for `sceSslInit`/`sceHttpInit` and could be
used to recover them if BearSSL's certificate story turns out worse than it looks.

## Distribution: mesa as a release, and where orbis-compat sits

The full plan for building and publishing all of this from GitHub Actions lives outside this file.
Two findings from it belong here because they are facts about the tree.

**A Mesa bundle is 87 MB and 15 MB compressed** - `libvulkan_radeon.a` 35 MB, `libgallium-*.a` 41 MB,
`libEGL.a` 5.5 MB, the GLES dispatch 228 KB, `include/` 5.5 MB. Building Mesa in every consumer's
pipeline pays for the same work repeatedly; it is the one input both expensive and slow-changing.

⚠ **AND MESA IS LINK-TIME COUPLED TO orbis-compat, not merely compile-time.** Its archives carry
undefined references only the overlay satisfies:

    libvulkan_radeon.a  ->  clock_gettime fstat open unlink pthread_create
                            orbis_sysconf  _Znam _Znwm _ZnwmRKSt9nothrow_t

`orbis_sysconf` is the tell: orbis-compat's `unistd.h` defines `sysconf` as a macro renaming it, so
every Mesa object that asks how many CPUs the machine has imports a symbol that exists nowhere else.

That settles two questions that will be asked again. **The direction cannot invert** - orbis-compat is
the base layer and Mesa its consumer, so publishing Mesa out of orbis-compat's repository would have
the base release the thing built on top of it, and every overlay change would drag a full Mesa build
behind it. **And the overlay does not belong inside the Mesa tarball** - those references resolve at
the FINAL link where `-lorbis-compat` is already on the line, and bundling a copy would freeze the
layer that changes most often inside the artifact meant to change least.

The proportionate check is the import list itself: assert that every symbol Mesa's archives import
from orbis-compat is still defined by the orbis-compat about to be linked. ⚠ It catches presence, not
meaning - a changed return value or a constant inlined at Mesa's compile time leaves nothing to check.

## The release pipeline, and the three things the plan got wrong about it

Phases 01-04 of the release plan are now in the tree: `.github/actions/orbis-toolchain/action.yml`,
`.github/workflows/frontend.yml`, `.github/workflows/cores.yml`, `ps4/shard-cores.sh`,
`ps4/make-index.sh`, disk hygiene in `ps4/build-cores.sh`, and a release workflow in mesa-ps4.
Nothing has run on a runner. What follows is only the part that was measured rather than written.

**The index CRC is of the `.prx`, not of the `.zip`.** The plan said to hash the archive that gets
uploaded. `tasks/task_core_updater.c:832-853` hashes `download_handle->local_core_path` and compares
it to `entry->crc`, and `local_core_path` is the *extracted* module -
`core_updater_list.c:466-471` strips the archive extension. Publish the archive's CRC and the
comparison never matches: nothing errors, and every core re-downloads on every visit to the
downloader, forever. The plan's *reasoning* survives intact and is why the index is cut inside the
shard that built the core - `create-fself` is not byte-reproducible, so a CRC computed from a later
rebuild of identical objects is a different number.

Two more parser facts worth not re-deriving. The CRC goes through `string_hex_to_unsigned`
(`libretro-common/string/stdstring.c:624-646`), which returns **0 on any parse failure**, and
`core_updater_list.c:367-379` treats 0 as a rejected line - a malformed CRC silently removes a core
from the list rather than reporting anything. And the date is `strtoul`-walked and must end at NUL
(`core_updater_list.c:331-362`), so a trailing space drops the line too.

**The `.info` files cannot ship inside the package the way phase 02 wanted.** Package contents mount
at `/app0`, which `frontend/drivers/platform_orbis.c:72-77` already documents as read-only, and the
same file sets `DEFAULT_DIR_CORE_INFO` to `/data/retroarch/info`. RetroArch will not look in
`/app0`. `make-pkg.sh` does have `--extra <src>:<targ>`, but `Makefile.orbis`'s `pkg` target passes
none. Making a fresh install legible therefore needs a first-boot copy out of `/app0/info`, or
`CORE_INFO_PATH` moved to `EBOOT_PATH` - not a packaging step. Left as a comment in `frontend.yml`
rather than a step that silently copies nothing.

**Two runner-environment dependencies nobody had written down.** `PkgTool.Core` dlopens
`libssl.so.1.1` (confirmed in its `strings`) and a GitHub runner has OpenSSL 3, so the frontend
workflow fetches the focal `libssl1.1` deb. And mesa-ps4's `build-support/orbis/build.sh` refuses to
run without `nix` and wraps every meson/ninja call in `nix develop nixpkgs#mesa`; it also runs
`ninja -k 0 ... || true`, so it exits 0 on a failed build and the release workflow has to assert the
five archives itself rather than trust the exit status.

**Still open, and both are cheap.** ⚠ Spike S1 - whether a Release asset keeps a leading dot - is
unanswered, and it decides Releases against Pages as the host. `cores.yml` answers it on its first
real run by re-fetching `.index-extended` from the release it just cut. ⚠ And the OpenOrbis SDK
release asset name could not be established: the copy at `~/.local/opt/openorbis` has a changelog
topping out at v0.5.2 (2021) but a `create-fself` dated 2026-01-04, so it is *not* that release
asset, and pinning `v0.5.2` would build against a 2021 SDK. Publishing a known-good copy under
`orbis-ports` is the honest fix.

Measured: 101 cores, 461 MB, 109 MiB zipped; 101/101 modules carry `retro_run`; all 101 index CRCs
cross-checked against `7z h` and Info-ZIP.

## Where the cores are hosted, and why not GitHub

Measured, not chosen on taste. **A GitHub Release asset cannot be named `.index-extended`.** Uploading
that exact filename to a throwaway release stored it as `default.index-extended`; the dotted name
returns 404 and the renamed one returns 200. The filename is a hardcoded string literal at
`tasks/task_core_updater.c:389`, joined onto the base URL - so hosting on Releases would have meant
patching the client with an `#ifdef` and diverging from upstream over a hosting quirk.

**Cloudflare R2 keeps the key verbatim.** Same test against the bucket: `PUT .index-extended`,
`GET /.index-extended`, 200, exact name. No client patch. The bucket is `orbis-cores` on account
`dde701a9fad0ed4ac032e5bfbbae1b56`.

**The domain is what makes it usable before TLS exists.** The bucket's own `pub-*.r2.dev` hostname
answers plain http with a 301 to https. `net_http.c:2329` follows redirects, so it would walk
straight into a handshake that `HAVE_SSL=0` cannot complete - a silently empty Core Downloader. A
custom domain does not redirect: `prx0.com` was registered, `cores.prx0.com` attached to the bucket,
and plain `http://cores.prx0.com/.index-extended` returns 200 with no redirect. ⚠ That is the only
reason the domain exists, and it is what takes phase 5b off the critical path - BearSSL is still
wanted, but nothing waits on it now. `config.def.h`'s ORBIS arm points there.

Ownership verification takes a few minutes and reports `error code: 1014` on http until it clears;
`wrangler r2 bucket domain list orbis-cores` shows `ownership_status` going pending → active.

**CI credentials, and why these names.** `wrangler` rather than rclone/aws-cli, so no S3 access keys
exist to leak; the R2 API token is scoped `Workers R2 Storage: Edit`. `CLOUDFLARE_API_TOKEN` and
`CLOUDFLARE_ACCOUNT_ID` are org secrets, the token restricted to this repository - the org is mostly
upstream forks, and a fork with an enabled workflow is the cheapest way for a secret to reach a log.
`R2_CORES_BUCKET` and `CORES_BASE_URL` are org *variables*, not secrets: they are not sensitive, and
as variables they appear in the log, which shortens "where did that URL come from" to one glance.

**Upload ordering is load-bearing.** `.index-extended` goes last, after every zip it names - a
console reading the index between the two states downloads a core that is not there yet. Pruning
runs after the new index is live, and the only inventory available is the *previous* index, because
wrangler has no `r2 object list` and no S3 keys exist. An object no index ever named is invisible to
the job and needs a manual sweep.

## The pipeline runs: what four red runs taught, and what is live

`http://cores.prx0.com/.index-extended` serves 101 lines over plain HTTP, no redirect. All 101
archives return 200. The first core's `.prx`, unzipped, hashes to exactly the CRC its index line
claims. That is every step the console performs except the console.

**Coverage is honest and it is not 163.** The recipe has 164 cores, 163 were attempted, **101 are in
the index**. Those 101 are the same set this machine has built by hand - CI introduces no difference
of its own, which is the most useful thing the first green run said. The other 63 now carry a
recorded reason: `bsnes` `fbneo` `desmume` `dosbox_svn` `bluemsx` fail to LINK on missing symbols;
`bsnes_hd_beta` wants `GOMP_parallel`, so OpenMP; `chailove` `geolith` `daphne` want PHYSFS, zlib and
SDL; `bsnes2014` and `freej2me` link *without* `retro_run` and are quarantined as NO-ABI; `citra` and
`blastem` produce no objects at all, so they build differently than the harness assumes.

**Four failures, and only one was environmental.**

⚠ *The runner is reclaimed sometimes.* Shard 0 ran 59 minutes and died with `The runner has received
a shutdown signal`. `kronos` had spent 57 of those minutes and its recorded `COMPILE` verdict is an
artifact - `trap ... EXIT INT TERM` does not stop bash, so the handler ran, deleted the clone, and
execution *resumed* to write a row about a core that had already been killed. Anyone chasing
`glsym/rglgen_private` should get a clean run first. The fix is a 25-minute per-core cap, chosen
from measurement: the slowest *successful* core anywhere is `mednafen_saturn` at 831 s. It must not
use `timeout --foreground` - tested, that leaves three orphaned `clang++` behind; without it, none.

⚠ *One core failing must never block 162.* `publish` has `needs: shard`, so a red shard sank the
whole release. Per-core outcomes are data now; a shard fails only if *zero* cores build.

⚠ *The leading dot bites a third time.* `actions/upload-artifact` has excluded hidden files by
default since v4.4, so `path: dist/.index-extended` matched NOTHING - on a file verified one step
earlier. `include-hidden-files: true`. Releases rename it, upload-artifact hides it, R2 keeps it.

⚠ *A non-empty secret is not a working one.* The first R2 attempt produced 101 consecutive
`"code":10000, "Authentication error"` with zero successes, after nine minutes of uploading, because
the credentials step only checked that the variables were set. ⚠ **AND THE CAUSE WAS THE BUCKET
SCOPING.** An R2-page token with *Apply to specific buckets only* authenticates the S3 endpoint, not
the REST path `wrangler r2 object put` uses. What works: a Custom Token with
`Account · Workers R2 Storage · Edit`, account-scoped, no bucket restriction. A one-object probe now
proves the token in three seconds before anything else runs.

And one of ours: the probe's own `echo` was committed with an unclosed quote, which YAML validation
cannot see because to YAML it is a perfectly good string. Every `run:` block now goes through
`bash -n` before commit.

Live: `orbis-mesa-c42aa135f234` (16.4 MB, past the link probe), the `cores` Release as the archival
copy (103 assets, where GitHub stores the index as `default.index-extended`), and the R2 bucket the
console actually reads. The frontend `.pkg` is still an artifact, not a Release - that needs a tag.

## RetroArchV, and why the title id had to move with the name

The package is now `RetroArchV`, title id `RTRV00001`, content label `RETROARCHV000000`.

⚠ The console decides collisions by **title id**, never by the name on screen. A second package
carrying `RTRA00001` would not have appeared beside an existing RetroArch install - the installer
would have treated it as an update and replaced it, silently, with something built by strangers.
Renaming the title alone would have left that exactly as dangerous while looking solved. Both moved
together, so the two are different applications as far as the system is concerned and either can be
removed without touching the other.

`ps4/icon0.png` is 512x512 and opaque on purpose: the system draws icon0 as a square, so the rounded
corners in `media/ico_src/icon.svg` would have appeared as transparent notches. It is the invader
from `media/retroarch-vector_invader-only.svg` - cropped to its alpha bounding box, because the
source has ~30% padding inside its viewBox and scaling the box rather than the glyph produced a
small mark floating in a large field - recoloured white over our own gradient, with a chevron mark.
Verified by content rather than by build success: the icon's bytes appear verbatim inside the
`.pkg`, alongside one `RetroArchV` and four `RTRV00001`.

Both workflow steps that used to spell `IV0000-RTRA00001_00-RETROARCH0000000.pkg` now glob
`IV0000-*.pkg` and assert exactly one. They were already stale when this landed - a hardcoded name
in CI for a value Makefile.orbis owns is a green run that copies a file which no longer exists.

## Networking works, and three defects of one family stood in the way

The Core Downloader lists 101 cores from `http://cores.prx0.com/`, downloads one, extracts it and
runs it. Databases update too - a 52 MB transfer. All of it measured on hardware.

⚠ **Every step of the diagnosis that reasoned from symbol tables was wrong, and every step that
measured was right.** Phase 5a concluded no wrapper was needed because `libkernel.so` exports all
22 POSIX socket calls. That is true and it is not the question. Presence is not permission, and the
two are indistinguishable at link time.

**1. The socket cannot be made non-blocking by any POSIX route.** Probed:

    fcntl(F_GETFL)  =  2   errno=0     reading flags is allowed
    fcntl(F_SETFL)  = -1   errno=13    EACCES
    ioctl(FIONBIO)  = -1   errno=13    EACCES
    connect()       =  0   errno=0     a blocking connect succeeds

`socket_connect_with_timeout()` calls `socket_nonblock()` FIRST and returns on failure without
reaching `connect()`, so `net_http` reported `socket_connect_failed` on a connect that never
happened. ⚠ That misattribution cost most of an afternoon: it reads as a host problem, then as a
process with no network authority, and it is neither. `sceNetSetsockopt(SO_NBIO)` works - **applied
to the descriptor musl's `socket()` returned**, which is also the experimental proof that the two
APIs share one descriptor namespace. The fix is an ORBIS arm in `socket_set_block()`.

**2. A downloaded core arrives without the execute bit.** The zip extracts to 0666, every `.prx`
that has ever loaded here is 0777, and `sceKernelLoadStartModule` will not open it. The download
reported success, the CRC matched, the file was byte-correct, and the core would not load. Nothing
server-side was wrong, so no assertion in the pipeline could have caught it. `chmod(0777)` in
`CORE_UPDATER_DOWNLOAD_END` - 0777 and not 0755 because that is the mode of the cores that already
work, under a different uid than the downloader writes as.

**3. musl's resolver cannot run here, and repairing its flags would not fix it.**
`sceNetGetDnsInfo()` reports a working nameserver and `getaddrinfo()` still answers -11 with EACCES:
`__res_msend` opens its query socket as `SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK`, and this SDK
defines those as 02000000 and 04000 - **Linux's values on a FreeBSD kernel**, which did not accept
them in `socket()` until FreeBSD 10. ⚠ And overriding `socket()` would not be enough: probed, the
flagged call returns a valid descriptor AND leaves errno at 13, because musl retries plain, tries
`fcntl(O_NONBLOCK)`, is refused again, and hands the descriptor back anyway - its resolver then runs
on a blocking socket it believes is non-blocking. `sceNetPoolCreate()` + `sceNetResolverStartNtoa()`
resolved in 73 ms first try. ⚠ The pool is the part the first attempt missed;
`sceNetResolverCreate(memid=0)` fails with 0x80410109.

That is three defects of one family in one day, with the libc++ `ETIMEDOUT` and the `LINUX_FIONBIO`
in `bits/ioctl.h`: **a musl and a libc++ built against Linux constants, shipped for a FreeBSD
target.** All of them compile, none of them warn.

**Instruments, kept.** `ps4/orbis_net_probe.c` runs from `frontend_orbis_get_env()` under
`-DORBIS_NET_TRACE` and answers "which API may this process actually use" on hardware. The same flag
un-gates `net_http_log_transport_state()`, whose stage names (`dns_lookup_failed`,
`socket_connect_failed`, `socket_send_failed`) are what turned each of these from a guess into a
measurement. ⚠ It logs through `ps4_log()`, not `RARCH_LOG` - the probe runs before RetroArch's
logger exists, and one build printed nothing at all for that reason.

⚠ **And `ps4_log()` writes klog ONLY while netlog is down** (`klogWanted()` in ps4_app.cpp is
`s_frameKlog || orbis_netlog_ready()==0`). A klog capture alone shows the first few lines of a boot
and then goes quiet. Capture UDP.

⚠ **This console's FTP does not return binaries faithfully.** The same 749,552-byte `.prx` came back
as 1,104,568 bytes through both lftp and curl, and it is not LF-to-CRLF expansion - the file holds
907 bytes of 0x0A. Uploads are fine; a 63 MB package installs and runs. Any forensic based on a file
pulled off this console is based on a corrupted copy.

## TLS, and the one line that made it impossible

HTTPS works, with certificate validation, verified on hardware against the same R2 bucket over
both schemes so TLS was the only variable.

⚠ **The blocker was never the crypto.** BearSSL is vendored at `deps/bearssl-0.6`, the backend is
written, and both things TLS needs from the platform - `time()` for validity windows,
`getrandom`/`getentropy` to seed - were present all along. What stopped it was
`net_socket_ssl_bear.c` reading its trust anchors from one hardcoded path,
`/etc/ssl/certs/ca-certificates.crt`, which is a Linux distribution's layout. On a console that
file does not exist, so BearSSL initialised with **zero** anchors and every handshake failed
validation.

⚠ And worse than "no TLS": `filestream_read_file` leaves its out-pointer NULL on failure and the
old code handed that straight to `append_certs_pem_x509()`, whose first act is `strstr(NULL, ...)`.
Turning on `HAVE_SSL` without fixing that would have made every https URL a crash rather than a
failed connection. A missing END marker in a truncated bundle had the same shape.

The path list is ordered `/data/retroarch/cacert.pem` then `/app0/cacert.pem`: the writable copy
wins so roots can be refreshed without a new package - a CA expiring is not a reason to reinstall -
and `/app0` is read-only but perfectly *readable*, so the shipped bundle needs no first-boot copy.
`ps4/cacert.pem` is 121 anchors, 185 KB, checked with `openssl s_client` against all three hosts
this port uses. It reaches the package through the first `--extra` this project has ever passed;
confirmed on hardware at `/mnt/sandbox/RTRV00001_000/app0/cacert.pem`, 185311 bytes.

`net_socket_ssl.h` also used `ssize_t` without declaring the dependency - transitive everywhere
else, absent here.

⚠ **Do not enable "Always Use HTTPS" on cores.prx0.com.** v0.1.1 and earlier have no TLS, and a
redirect they cannot follow turns their Core Downloader into a silent empty list. Plain http must
keep answering until those packages are gone. This is also why the CRC in `.index-extended` never
authenticated anything by itself: it arrives over the same channel as the file it describes.

## The Online Updater, and a 71 MB download that killed the process

Update Assets crashed the console at the end of the transfer - black screen, no abort report,
Mesa's log ending mid-frame-statistics with no error. Update Databases, 52 MB, had worked minutes
earlier.

⚠ **The first explanation was wrong and worth recording as such.** `net_http` grows its response
buffer by doubling, so a 71 MB body looked like it must pass through a 64 MB → 128 MB realloc
with both alive at once. It does not: `net_http.c:1556` sizes the buffer from `Content-Length`
in one allocation as soon as the headers land. The doubling only applies to a chunked response.
The theory was tidy, matched "at 99%", and was false.

What is true is simpler. The whole body is held in RAM until the transfer completes, then
`cb_generic_download()` writes it out and starts a decompress task - so the peak is 71 MB of
buffer, plus the write, plus inflating **7040 files and 85 MB** while the buffer is still alive.
52 MB survived that and 71 MB did not.

**The fix had been sitting in the tree with no caller.** `task_push_http_download_file()` streams
the body to a path as it arrives, so the peak is the receive window rather than the payload -
`task_push_http_transfer_file()`, which every updater entry used, passes `sink_path = NULL`.
Six downloads now take the streaming path: assets, core info, databases, overlays, cheats and
core system files. Confirmed on hardware: 6995 files extracted, all nine XMB themes, no crash.

⚠ **Only the enums whose destination is a plain settings directory.** The sink path must equal the
`output_path` the callback would have computed, and thumbnails, Content Downloader items,
autoconfig profiles and shader packs all derive a subdirectory that does not exist at push time -
from a playlist, a category, the joypad driver name. Those keep the in-memory path.
`download_stream_dir()` returns NULL for them and the old push runs, and a directory that cannot
be created also falls back rather than failing.

**Two other updater findings from the same session.** ⚠ `HAVE_UPDATE_CORE_INFO` and
`HAVE_UPDATE_ASSETS` were absent from Makefile.orbis on a reason that had expired - the comment
said the package ships the .info files, which it never did and cannot, because package contents
mount read-only at /app0 while RetroArch reads /data/retroarch/info. Measured: `/app0/assets` in
the installed package holds ZERO files. So the release notes and cores.prx0.com were telling a new
user to click a menu entry the build did not contain.

⚠ And the video driver defaulted to `gl`. `configuration.c` tests `HAVE_OPENGL || HAVE_OPENGLES`
before Vulkan, and this port builds with GLES for the sake of cores that need a GL context - so a
fresh install came up on zink, translating to the Vulkan that RADV was going to be handed anyway.
An ORBIS arm now selects Vulkan. It costs GL cores nothing: `video_driver_find_driver()` forces
the driver to match a core's hardware context and remembers the previous one.

## Why mupen64plus_next was never in a release, and what it says about every other core

It builds here and it has never once built in CI. Two causes, both of them a dependency this
machine happened to have and nobody had written down.

⚠ **clang falls back to /usr/include, and -isysroot does not stop it.** GLES3/gl3.h and its EGL
neighbours live in the Mesa tree - `$(ORBIS_MESA_SRC)/include`, the path Makefile.orbis:392 gives
the frontend - and `ps4/build-cores.sh` never passed it to cores. Measured directly:

    clang --target=x86_64-pc-freebsd12-elf -isysroot $TOOLCHAIN ... -E
      -> # 1 "/usr/include/GLES3/gl3.h"

So every GL core built on this machine has been compiled against **this Linux desktop's** GLES
headers. It works, because they are Khronos's and near-identical - an accident that holds. A
runner has no libgles-dev, nothing to fall back to, and the core fails with
`'GLES3/gl3.h' file not found`. ⚠ CI was right and the development machine was wrong, which is the
opposite of how this reads at first: seven objects short, then `undefined symbol: glsm_ctl` at the
link, and glsm.o was simply one of the files that never compiled.

⚠ **And `nasm` is not on a GitHub runner.** mupen's x86_64 dynarec assembles
`new_dynarec/x64/linkage_x64.o` with it - `make: nasm: No such file or directory`, Error 127.
`make` runs with `-k`, so both failures surfaced only as a missing symbol much later.

**The instruments this took, and why they were worth more than the fix.** Four attempts learned
nothing because the evidence kept being thrown away: a diagnostic step that reddened healthy
shards (GitHub runs `run:` under `bash -e`, and testing for a file that is absent *because nothing
failed* returns 1); an artifact with `if-no-files-found: error`, which discards the logs in the one
case where the logs are all there is; a selection keyed on a manifest that `build-cores.sh` never
writes when zero cores build - the single-core case exactly; and `WORK`, which is a step-level
variable, not a job-level one. Each was correct for the path its author had in mind and silent on
the path being walked, which is the same shape as every defect in this port.

⚠ **`cores.yml` now takes a `cores` input** - space-separated names, one shard, publishes nothing.
Chasing this cost two full 163-core runs before that existed. Publishing is blocked for a subset on
purpose: the prune step deletes bucket objects the new index does not name, so a two-core
diagnostic run would have taken the other ninety-nine down with it.

**Left open.** The host-header fallback is not specific to mupen: any core reaching for a header
the SDK and overlay lack will silently take this machine's. `-nostdsysteminc` would close it and
has not been tried.

## Open on hardware after v0.1.4: a save state, an audio port, and two PSX cores

**PrBoom crashes seconds after Load State, and the fault is inside the core.** Symbolized from a
console klog dump against a local build of the same pinned commit:

    signal 10 (SIGBUS), general protection fault
    rip 0x8009acf98 -> /data/retroarch/cores/prboom_libretro.prx +0x154f98
                    -> Z_Malloc +0x1d8

The zone allocator walking a corrupted free list. Not the frontend, not Mesa, not the platform.
⚠ The state loaded at 09:55:44 and the fault came at 09:55:47 - the load "succeeded" and the heap
was already wrong. The state file was **351640 bytes while the core's current
retro_serialize_size() reported 198200**: prboom sizes its state from live thinker, sector and
line counts, so it changes with the situation. ⚠ And the core declares no
RETRO_SERIALIZATION_QUIRK_CORE_VARIABLE_SIZE anywhere, so the frontend is never told - which also
means rewind, run-ahead and netplay are being driven on an assumption that does not hold.
retro_unserialize does bound its read by `size`, so it is not a naive overrun. **Not yet
established** whether this reproduces off this platform; the A/B test that settles it is save and
load at the same moment of play, where the two sizes agree.

⚠ **Audio died on core switches, and the driver was not leaking - the system keeps the port.**
FIXED. `sceAudioOutOpen failed: 0x80260005` is ORBIS_AUDIO_OUT_ERROR_PORT_FULL. The obvious
reading is a missing close, and it was wrong. Instrumented and measured across eight switches:

    eight opens, eight closes, EVERY close returning rc 0x00000000
    handles counted down 0x20000007, 0x20000006 ... 0x20000000, never reissued
    the ninth open failed

So `sceAudioOutClose` reports success and the port stays spent. Eight per process, then silence.
⚠ The fix is therefore not to close better but to stop reopening: `ps4_audio_init()` opens the
MAIN port ONCE per process and hands the same handle to every later init, and `ps4_audio_free()`
does not close it. Nothing is lost by holding it - the parameters are compile-time constants
(PS4_AUDIO_RATE, PS4_AUDIO_GRAIN, S16 stereo) and RetroArch is told the rate through *new_rate
and resamples - and a port that cannot be reused is worth nothing returned. Confirmed on hardware
over a dozen-plus core switches with sound throughout.

⚠ **The measurement overturned the hypothesis rather than confirming it.** Had the close been
"fixed" without instrumenting first, the change would have added a close that was already there
and the search would have gone on.

⚠ **Two Beetle PSX cores shipped and only one was ours. The stock one is now withheld.**
`mednafen_psx_hw` is the fork at `b0b759e`, ten commits ahead of upstream, with the ORBIS platform
arm, `orbis_lightrec_mem.c` and the Vulkan renderer - PS4_CORE_FORKS protects it from being
overwritten. `mednafen_psx` was built by an ordinary shard straight from upstream at `ef51860`:
`HAVE_LIGHTREC=1` in its Makefile and none of the port's work behind it, so it has no executable
code buffer, falls back to the MIPS interpreter, and runs at the ~38% of realtime this file
measured on Spyro 3 - beside a fork that holds full speed, under a name one letter apart from it.

The fix is a withdrawal, not a port. **Folding it into the fork was the other option and it buys
nothing**: the fork already IS upstream plus the platform work, and `_hw` is the same core with
the hardware renderer available. A second entry can only be the same emulator configured worse.

`PS4_CORE_DROP` in `ps4/build-cores.sh` withholds it. ⚠ **It is a separate mechanism from
`PS4_CORE_FORKS` because it answers a different question.** FORKS is about not overwriting a file;
DROP is about what the Core Downloader offers. A core reaches the menu by being in the index and a
user picks it by name, so every name in that list reads as a recommendation - and this is the only
list a console owner sees. Everything else missing from the index failed to build; this one builds
and is held back, which is why the verdict is `DROP` rather than a failure class.

⚠ **Withholding does not remove what is already published.** The publish job prunes bucket objects
the new index does not name, so `mednafen_psx_libretro.zip` goes on the next FULL run - a subset
run publishes nothing and prunes nothing. Until then the object is still fetchable by URL; it is
simply no longer named by anything a console reads.

**The other PlayStation options, measured rather than assumed.** `pcsx_rearmed` fails at
LINK on `lightrec_init_mmap` - the same executable-memory problem this port already solved for
Beetle, so it is the cheapest of these to try. `duckstation` and `swanstation` are CMAKE in the
recipe and the harness skips that build type for want of a toolchain file; they have never been
attempted. PS2 is `play` and `pcsx2`, also CMAKE - and beyond the missing infrastructure, PCSX2
wants an order of magnitude more CPU than this machine has, so treat it as arithmetic rather than
porting.

## 2026-08-28 — one core option, and the console had to be recovered from outside

`GPU shader depth write` (`EnableFragmentDepthWrite`) was suggested in the entry above as the
free experiment that would settle candidate 3 without a rebuild. **It is not free. Turning it on
hangs the core before its first frame and then takes the whole console down**, and it is now
clamped off on this platform and removed from the menu
(`ps4/core-patches/mupen64plus_next/0007`).

⚠ **THE SUGGESTION WAS MINE AND THE COST LANDED ON THE USER'S CONSOLE.** The reasoning behind it
was sound - the option is exactly the other half of a pair this build splits - and the reasoning
said nothing about what the option costs to *try*. A core option is not automatically a cheap
experiment on a machine with no way to interrupt a wedged process.

### What the log says, and it is not "lag"

Same ROM, same build, same session, thirty minutes apart. `ps4-klog-20260828-092942.log`:

    11:35:58.626  mupen64plus: Init new dynarec          option False
    11:36:00.426  [Audio] sinc resampler active path: float
    11:36:07.626  profile: 650 frames in 8972 ms = 72.44 fps

    12:06:45.669  mupen64plus: Init new dynarec          option True
    12:06:55.469  [WARN] [PS4] audio: 1772 underruns in 1875 grains
    12:07:05..35  1875 underruns per 1875 grains, four windows running
    12:07:41.070  [ScePthread/System] Internal Memory is running out.   x10851

**Nothing at all between `Init new dynarec` and the pthread pool giving out** - no first frame, no
profile window (and the profiler was on, `/data/retroarch-profile` exists and it reports every five
seconds *of frames*), 100% audio underruns throughout. Every line before that point is identical
between the two runs, down to EGL, the zink renderer string and `133 entry points resolved, 0
missing`. So this is not a slow frame: **the core never completes one**, and something between
dynarec init and the first `retro_run` allocates pthread objects without bound.

⚠ **THE EXHAUSTION IS SYSTEM-WIDE, WHICH IS WHY THE CONSOLE DID NOT COME BACK.**
`[ScePthread/System]` is the system's pool, not this process's heap - technote 235. Once it is
gone the shell cannot get itself back either: the PS button did not return to Orbis, and Close
Application from outside wedged the machine rather than freeing it. **A core option that can do
that must not be reachable from a menu**, whatever it is worth when it works.

### Where the clamp is, and why not at the option

`custom/GLideN64/mupenplus/Config_mupenplus.cpp`, immediately before `config.validate()`, which is
after every writer. ⚠ **Removing the core option alone would not have been enough:**
`LoadCustomSettings()` parses `generalEmulation\enableFragmentDepthWrite` out of GLideN64's
per-game ini, and with `GLideN64IniBehaviour == 0` that runs *last*. The core option is removed as
well, so nothing offers the switch, but the clamp is what makes it safe.

⚠ The removal produces one new line per content load, and it is not a fault:

    [ERROR] [Environ] GET_VARIABLE: mupen64plus-EnableFragmentDepthWrite - Invalid value.

`EnableN64DepthCompare` and `EnableShadersStorage` have printed exactly that on every GLES build
since the port began - upstream compiles those two out under `HAVE_OPENGLES` and asks for them
anyway. This is the third.

### What it does NOT settle

**Candidate 3 from the entry above is still open**, and now it cannot be tested the cheap way. The
`/8` that `enableClipping` puts in the vertex shader is still never scaled back on this platform;
the option that would scale it back is the one that hangs. If that pairing has to be tested, the
route is a build with `enableClipping` forced to 0 instead - which costs a core build and leaves
the software clipper doing the work it was always doing with framebuffer emulation on.

**And the cause of the hang is unknown.** Writing `gl_FragDepth` defeats early-Z, which is a frame
rate story, not a hang. The shape to chase is what creates pthread objects per shader variant when
every fragment program suddenly writes depth - zink compiling a program set it has never built, on
a driver where `orbis-compat` reports six CPUs to Mesa's `util_queue`. Nobody has looked.

⚠ **AND THE PUBLISHED CORE STILL HAS THE OPTION.** `mupen64plus_next` in the index was built before
this patch, so every installed copy can still be walked into this. It is fixed by the next cores
run, not by anything already shipped.

## 2026-08-28 — the depth bug reproduced on a laptop, and it is not this port's driver

The framebuffer-emulation depth fault is **confirmed, off the console, in one run**, and the cause
is candidate 1 from two entries above. `ps4/core-patches/mupen64plus_next/0008` fixes it.

### What the host says

mupen64plus-next built for Linux at the same commit `f275caf`, `FORCE_GLES3=1` so the flags are the
console's exactly (`-DEGL -DHAVE_OPENGLES -DHAVE_OPENGLES3 -DGLES3`), RetroArch from this branch
built `--enable-egl --enable-opengles --enable-opengles3`, and the same ROM - fetched over FTP and
checked against the md5 the console's own log printed. GLideN64 announces the same state it
announces on the console:

    GL 3.1 ES | depthTexture 1 weakBlit(GLES) 1 noPerspective 0 imageTextures 1
              | fbEmulation 1 fragDepthWrite 0 clipping 1 n64DepthCompare 0 copyDepthToRDRAM 2

and then, five thousand six hundred and seventy-two times in twenty-five seconds - **every frame**:

    depth attachment DROPPED for colour buffer 0000027f:
    depth texture 640 wide, colour 639 (m_width 320 vs 320)

⚠ **ONE PIXEL.** The colour texture is `m_width * m_scale` truncated, 639. The depth texture was
created before its colour buffer existed, took the *window* width instead, and is 640. GLideN64's
GLES branch tests `==`, so the depth buffer is thrown away and the FBO is rendered into with no
depth attachment at all. Every triangle passes, and the picture is ordered by draw order - which is
exactly "my ship draws behind everything, and the collisions are still right".

### ⚠ IT IS NOT ZINK, NOT RADV, NOT LIVERPOOL AND NOT OUR THUNKS

The same build on the host's **native radeonsi** - no zink in the process at all - drops just as
often:

    zink over RADV      26 000-32 000 drops / 25 s
    native radeonsi     26 000 drops / 25 s
    desktop-GL test     0 drops

So this is upstream GLideN64 on any GLES3 machine with framebuffer emulation on. Desktop GL never
sees it because the branch one line below tolerates a larger depth texture *or* an `m_width`
difference under two pixels - and here **both** clauses pass. That branch is the fix; `0008` takes
it on this platform and `depthWidthLoose=0` in the knob file puts upstream's back.

⚠ **AND THE FIX INTRODUCES NO NEW GL ERROR.** The worry was that a lenient size test would make
GLideN64's depth *blits* illegal, since ES3 will not scale a depth blit. Checked with `MESA_DEBUG=1`
over both variants: the same two error classes appear either way and no new one -
`GL_INVALID_FRAMEBUFFER_OPERATION` in `glReadPixels` and `glBlitFramebuffer`, plus three
`GL_INVALID_ENUM in glFramebufferTexture2D(unknown textarget 0x8d65)`. **0x8d65 is
`GL_TEXTURE_EXTERNAL_OES`** and those are a separate, pre-existing defect this port has never
noticed, present with and without the fix. Worth its own look; not this.

### The hang did NOT reproduce, and that was the right thing to doubt

`EnableFragmentDepthWrite=True` on the host runs. Sixty seconds, frames throughout, no thread
storm - where on the console the same option gives no first frame and then exhausts the system
pthread pool. So that fault is genuinely console-side: Liverpool, our Mesa build, or the console's
pthread pool, and the laptop cannot say which. **The user said so before the run and was right;
the run was still worth it, because it is what turned the depth fault into a fixed bug.**

### ⚠ Building this branch for the host found a defect of ours first

`libretro-common/dynamic/dylib.c`'s ORBIS constructor walker - `dylib_orbis_run_init_array` and
`dylib_orbis_forget` - was **outside every `#ifdef ORBIS`**, so it compiled on every platform and
the Linux build stopped on `sceKernelDlsym`. It has been that way since 2026-08-24 and only a
non-ORBIS build could see it. Fixed in `8cbea55517`. ⚠ Anything else this branch has added without a
guard is in the same position, and the host build is now the thing that would say so.

### ⚠ AND FTP DOES RETURN A DATA FILE FAITHFULLY

This file says "any forensic based on a file pulled off this console is based on a corrupted copy",
from a `.prx` that came back 900 KB larger. That reading was too broad: **a `.prx` is a signed
module and FTP hands it back decrypted**, which is a fact about self files, not about the transfer.
A 12 MB `.z64` pulled the same way came back at `c7b40352aad8d863d88d51672f9a0087`, the md5
mupen64plus itself printed on the console. Verify by content and the question does not arise.

### Where the workshop is

    ~/.cache/ps4-hostrepro/RetroArch      git worktree of this branch, configured for GLES3 + EGL
    ~/.cache/ps4-hostrepro/mupen-host     f275caf, FORCE_GLES3=1, with the instrumentation below
    ~/.cache/ps4-hostrepro/roms/sote.z64  md5-checked against the console
    ~/.cache/ps4-hostrepro/cfg/retroarch.cfg   video_driver gl, context x-egl, rgui, no audio

    MESA_GLES_VERSION_OVERRIDE=3.1   the console reports ES 3.1; without this the host offers 3.2
    MESA_LOADER_DRIVER_OVERRIDE=zink the console's stack; leave it out for the radeonsi control
    MESA_DEBUG=1                     GL errors on stderr, with no code change

⚠ **NOT in the session scratchpad and not in /tmp.** A cold reboot cleared tmpfs once and took
fifty-two built cores with it; `/tmp` here is tmpfs with the whole build in RAM besides.

The host clone carries three edits that are deliberately **not** patches in this repository, because
they are an instrument rather than a port: `LOG_LEVEL LOG_WARNING` in `GLideN64/src/Log.h`,
an `fprintf` in `attachDepthBuffer`'s drop branch, and one in `GLInfo::init` - the same two lines
`0006` reports through `orbis_report` on the console. `HOSTREPRO_DEPTH_LOOSE=1` selects the fix at
run time there.

### Confirmed on hardware, same day

The console now reports what the laptop reported, and reports it about a working picture:

    [orbis] gliden64: GL 3.1 ES | depthTexture 1 weakBlit(GLES) 1 noPerspective 0 imageTextures 1
                    | fbEmulation 0 fragDepthWrite 0 clipping 1 n64DepthCompare 0 copyDepthToRDRAM 2
    [orbis] gliden64: GL 3.1 ES | ... | fbEmulation 1 ...

    depth attachment DROPPED     ZERO, with framebuffer emulation on

Two lines because the first load still had the old workaround in `Mupen64Plus-Next.opt`
(`EnableFBEmulation = "False"`) and the second is after turning it back on. Shadows of the Empire
draws correctly with framebuffer emulation enabled - the ship in front of the bases, which is what
started this.

⚠ **The capability line is IDENTICAL on the two machines**, field for field, which is the retrospective
justification for the host reproduction: `depthTexture 1 weakBlit(GLES) 1 noPerspective 0
imageTextures 1 | fragDepthWrite 0 clipping 1 n64DepthCompare 0 copyDepthToRDRAM 2` on a GFX7
Liverpool under our Mesa and on a GFX11 Phoenix under Arch's. The fault was never in the half that
differs.

⚠ **AND THE PROOF CAME OFF /data/retroarch-abort.log, NOT OFF THE UDP OR klog CAPTURE.** The klog
receiver's connection died at 12:10 when the console wedged and nothing reconnected it, so the log
files on the workshop machine end there and read as if the console had stopped talking. `orbis_report`
writes klog *and* appends to that file, which is why the measurement survived a dead receiver. Check
the file before concluding a run produced nothing.

**Still to ship.** The core in the index was built before any of this. Nothing a user has installed
carries the fix until the next full cores run.

## 2026-08-28 — /data/OpenGothic is no longer every title's junk drawer

`orbis_paths.cpp:19` has been on the open list since 2026-08-22, first as "harmless" and then as
"it is not". Closed.

This process has no working directory - `getcwd` is ENOSYS and a relative `open` returns EINVAL,
not ENOENT - so orbis-compat interposes `open`, `stat`, `unlink` and `rename` and rewrites every
relative path under one root. That root was the string `/data/OpenGothic/` compiled into an overlay
that four titles now link. RetroArch has been creating that directory on every boot and opening
files inside it, because it does open relative paths - `Main Menu.png` among them.

**The root is now decided in three steps, and the choice is logged once:**

    1  orbis_set_anchor_root()          the application knows where its own data lives
    2  /data/<TITLEID>/                 from sceKernelGetAppInfo
    3  /data/orbis-compat/              neither of the above answered

⚠ **STEP 2 IS UNPROVEN ON THIS FIRMWARE AND IS WRITTEN TO SAY SO.** `sceKernelGetAppInfo` is
declared by the SDK, and this workshop has now been caught five times by a call that exists, links
and is refused at run time - `sceKernelGetCpuUsage` (ENOSYS), `fcntl(F_SETFL)` (EACCES),
`ioctl(FIONBIO)` (EACCES), `dup2` onto fd 2 (EPERM), `sceKernelQueryMemoryProtection` (answers about
a different thing). So the log line reports what it returned **whichever route wins**, and one boot
settles it without anyone arranging an experiment.

⚠ **AND THE TITLE ID IS VALIDATED BEFORE IT BECOMES A DIRECTORY NAME.** `OrbisAppInfo::TitleId` is a
fixed ten-byte field with no promise of a terminator. It is copied bounded, terminated here, and
rejected unless it is one to nine characters of A-Z0-9 - a field of garbage would otherwise become a
directory of garbage, and one containing '/' would become a directory somewhere else entirely.

**Both existing consumers name their own anchor**, which is why nothing moves underneath them:
RetroArch calls `orbis_set_anchor_root("/data/retroarch/")` in `frontend_orbis_init`, immediately
after the stderr capture and before the first file is opened; OpenGothic calls it with
`/data/OpenGothic/` as the first thing after `ps4_app_init`, so `save_slot_N.sav` stays where the
saves already are. ⚠ A call that arrives after the anchor has been decided **cannot** take effect -
the decision is made once because it sits inside `stat()` - and it logs that it was ignored rather
than pretending.

Confirmed by content, not by the build succeeding: `/data/OpenGothic` no longer appears anywhere in
`retroarch_orbis.elf`, and `/data/orbis-compat/`, `paths: title id is` and `orbis_set_anchor_root`
all do.

⚠ **The overlay's header had to grow a C guard.** `orbis_paths.h` included `<string>` at the top,
so a C consumer could not include it at all - and a frontend written in C is exactly the consumer
that needs to name its own anchor. The C++ half is behind `#ifdef __cplusplus` now; the setter is
`extern "C"` and was compiled from both C and C++ with the console toolchain before being believed.

orbis-compat `5f1e4e6`, OpenGothic `c0c202e2`.

## 2026-08-28 — the close-hang: two things believed about it are wrong

Not fixed. But it is narrowed further than it has ever been, and the narrowing came from the
maintainer's own account plus one log fetch - no experiment was needed to overturn either belief.

### ⚠ IT DOES NOT NEED A CORE LOADED

This file has said since 2026-08-22 that "Close Content followed by Quit exits properly, so the
condition involves a loaded core". **It does not.** Quit from the menu, dummy core, nothing loaded -
CE-34878-0 all the same, twice in the capture below. The "Close Content then Quit exits cleanly"
observation that produced that conclusion was made once and has not held up.

### ⚠ AND THE DRIVER TEARS ITSELF DOWN CLEANLY BEFORE IT HAPPENS

Mesa's log at exit - `/data/retroarch-mesa.log`, which is where `MESA_LOG_FILE` in
`/data/retroarch-env.txt` sends it - ends:

    wsi/orbis: scan-out down after 4703 flip(s)
    orbis-drm: at teardown 0 BO(s), 0 syncobj(s), 0 context(s), 0 VA range(s), 0 KiB held
               - winsys-lifetime, and it did not grow

Nothing leaked and the scan-out came down. The process dies **after** that.

⚠ **AND THIS FILE'S CLAIM THAT "NO CAPTURE FROM THIS CONSOLE HAD EVER CONTAINED `scan-out down`"
WAS ABOUT THE WRONG FILE.** Mesa writes to `MESA_LOG_FILE`, not to the UDP channel, so grepping a
UDP capture for `wsi/orbis` finds nothing however healthy the driver is - there are 1575 `wsi/orbis`
lines in the file and 0 in the datagram log of the same session. Two days of "the teardown path has
never run" rested on that.

### What is actually left, and it is a short list

The maintainer's account, which no log here contradicts:

* every title built against this workshop's Mesa ends this way, OpenGothic included;
* **a RetroArch built WITHOUT that driver exits with no dialog at all.**

⚠ **THAT SECOND POINT CONTRADICTS `ps4_app.h`.** Its termination note says returning from `main()`
on a retail console is outside the system's expected path and pops CE-34878-0 by itself - and if
that were the whole story, linking a graphics driver could not change it. The same code returning
from the same `main()` is silent without Mesa. So the dialog is not the price of returning; it is
the price of something Mesa leaves behind.

Which points at what linking Mesa adds AFTER the driver has already destroyed itself: **atexit
handlers and static destructors**. `src/util/u_queue.c:83` registers one that walks every
`util_queue` still on its list and joins its worker threads, and this console has already spent a
day proving how little it forgives around threads (`ORBIS_NCPU`, the pthread pool exhaustion of the
same date, `sceKernelGetCpuUsage` refusing outright).

### The instrument, in the build now on the console

Three markers, and the ordering is what makes them an answer rather than three log lines - `atexit`
runs handlers in REVERSE registration order:

    registered in frontend_orbis_init, before anything creates a util_queue   ->  runs LAST
    registered in frontend_orbis_shutdown, at the end of main_exit            ->  runs FIRST

With `ps4_log("exit: main_exit returned...")` at the end of `rarch_main`, four outcomes are
distinct and the next Quit picks one:

    no "main_exit returned"      died in the rest of main_exit, after frontend_orbis_shutdown
    no "atexit has begun"        died between main() returning and the first handler
    only "atexit has begun"      died inside a handler registered after startup - Mesa's
    both markers                 died after every handler, in the runtime's final teardown

⚠ **They go through `ps4_log`, which is klog AND UDP.** A process three instructions from death
cannot rely on a datagram leaving the machine; klog has already written by the time the call
returns. 8-15 ms a line, twice, once per process.

### And the anchor bug was confirmed in the field on the way past

The same capture, from the build that predates this morning's fix:

    paths: anchor '/data/OpenGothic/'
    paths: relative paths are anchored - 'Main Menu.png' -> '/data/OpenGothic/Main Menu.png'

Exactly the file this file guessed at on 2026-08-23, named by the console itself.

## 2026-08-28 — the exit markers answered, and the answer was not the guess

All three fired:

    14:23:27.968  shutdown requested - ending the process rather than idling; expect CE-34878-0
    14:23:27.968  exit: main_exit returned, main() is about to return
    14:23:27.968  exit: atexit has begun
    14:23:27.968  exit: every atexit handler returned, Mesa's included

⚠ **SO IT IS NOT THE atexit HANDLERS, AND util_queue's THREAD JOIN WAS THE WRONG SUSPECT.** The
entry above named `src/util/u_queue.c:83` as the thing to look at, on the reasoning that joining
worker threads at exit is where this console is least forgiving. That handler ran and returned. The
process survived the rest of `main_exit`, the return from `main()`, and every handler in the list -
and died after all of it, in under a millisecond.

**And the list that just completed is bigger than it looks**, which narrows this further than the
four outcomes did. Clang registers a C++ static object's destructor with `__cxa_atexit` from a
constructor in `.init_array`, so **static destructors are IN the atexit list** - Mesa's, ACO's, all
of them. They ran. What musl's `exit()` has left after `__funcs_on_exit` is exactly:

    __libc_exit_fini()   .fini_array - and by the above, largely empty here
    __stdio_exit()       flush and close every open FILE
    _Exit(code)          the kernel

⚠ **WHICH MAKES `MESA_LOG_FILE` THE SHARP SUSPECT, AND THE TEST FREE.** It is the one stdio object
that linking Mesa adds: a `FILE*` opened with `fopen` and never closed. A RetroArch built without
the driver has no such file, and exits silently - which fits. `/data/retroarch-env.txt` now has both
its lines commented out, with the reasoning in the file; one launch and one Quit settles it. **Put
them back afterwards** - without them Mesa's log goes to a stderr that goes nowhere on this console.

### Two stale things found on the way, both in this port's own record

⚠ **`/data/tempest-env.txt` DOES NOT EXIST ON THE CONSOLE.** This file, and a comment in
`platform_orbis.c`, both say `ORBIS_3D_LINEAR=1` and `ORBIS_NO_TESS=1` are "not options" and are off
in the driver unless that file turns them on. Checked in mesa-ps4 today: the driver flipped both
defaults, and the knobs now read `=0` to turn the behaviour OFF (`ac_surface.c:1691`,
`radv_physical_device.c:1081`). The frontend applies two lines from its own file, none from the
shared one, and renders correctly. The comment is corrected; the reader stays, because a file that
is not there costs one failed open and it is still how any knob is turned on.

## 2026-08-28 — the close-hang is not this frontend's, and now that is proven rather than assumed

Four runs, one afternoon, each one eliminating a step. The conclusion is negative and it is worth
as much as a fix: **nothing RetroArch can do changes this exit**, and the question moves to
mesa-ps4 with a boundary drawn around it.

### What the process survives

    shutdown requested                            frontend_driver_shutdown, late in main_exit
    exit: main_exit returned                      the rest of main_exit, and main() about to return
    exit: atexit has begun                        __funcs_on_exit entered
    exit: every atexit handler returned           ALL of them - Mesa's util_queue join included
    exit: .fini_array is running                  __libc_exit_fini entered

All five, every run, inside two milliseconds. And then CE-34878-0.

⚠ **THE atexit LIST IS BIGGER THAN IT LOOKS, WHICH IS WHY THE FOURTH LINE MATTERS SO MUCH.** Clang
registers a C++ static object's destructor with `__cxa_atexit` from a constructor in `.init_array`,
so **static destructors run in that list** - Mesa's, ACO's, every one. They all returned.

### The three eliminations, in order

⚠ **util_queue's thread join was the first guess and it was wrong.** The entry above named
`src/util/u_queue.c:83` on the reasoning that joining worker threads at exit is where this console
is least forgiving. That handler ran and returned.

⚠ **`MESA_LOG_FILE` was the second and it was wrong.** It is the one stdio object linking Mesa
adds - a `FILE*` opened with `fopen` and never closed - and a build without the driver has no such
file and exits silently. Both lines commented out of `/data/retroarch-env.txt`, confirmed by
`env: 0 line(s) applied`, same crash.

⚠ **AND THE THIRD ELIMINATION TOOK THE WHOLE REMAINING SPACE AT ONCE.** A build whose `main()`
called `_Exit(0)` instead of returning - skipping `.fini_array`, `__stdio_exit` and every step
musl's `exit()` has left, going straight to the kernel - **died exactly the same way.**

So the fault is in process termination itself, with this driver's state in the process. There is no
step left between the last line RetroArch can write and the dialog. **It belongs to mesa-ps4.**

### What mesa-ps4 has to go and look at

The driver's own teardown reports itself clean:

    wsi/orbis: scan-out down after 4703 flip(s)
    orbis-drm: at teardown 0 BO(s), 0 syncobj(s), 0 context(s), 0 VA range(s), 0 KiB held
               - winsys-lifetime, and it did not grow

⚠ **"winsys-lifetime" IS THE QUALIFIER TO READ, NOT TO SKIP.** It says nothing about allocations of
other lifetimes, and this workshop already knows that **direct memory is not reclaimed when a
mapping goes away** - recorded here on 2026-08-23, from the Lightrec work. Physical pages taken with
`sceKernelAllocateDirectMemory` and never released are the shape that would make a process's death
the system's problem rather than the process's, and that also fits the other half of the symptom:
Close Application from outside wedges the console rather than freeing it.

`wsi_orbis_release` does call `sceVideoOutClose` and does release its own direct memory when it owns
the buffers, so the scan-out path is not the obvious offender. The audit is of everything else the
driver takes from the kernel and of what is still held at `_Exit`.

### ⚠ access() IS REFUSED ON THIS CONSOLE, WITH EPERM

Found while building the third experiment, and measured in one line beside its own control:

    exit: fastexit switch - access()=-1 errno=1, stat()=0 errno=0

Same path, same instant. `stat` finds the file; `access` says EPERM. It is a real `libkernel.so`
export (`T access`), while the SDK's own `libc.a` ships `access.lo` as an **empty object** - no
`.text` at all - and `faccessat.lo` as a stub that sets `0x4e` (ENOSYS) and returns -1.

⚠ **THIS IS THE SIXTH CALL IN THAT FAMILY** - after `fcntl(F_SETFL)`, `ioctl(FIONBIO)`, `dup2` onto
fd 2, `sceKernelGetCpuUsage` and `sceKernelQueryMemoryProtection`. **Presence is not permission**,
and the first version of that experiment used `access()` alone, got a silent no, and was
indistinguishable from "the file was not there yet". A check that cannot report what it saw is not
a check.

Nothing in this tree relies on it: the only `access()` callers are `linuxraw_joypad.c`,
`parport_joypad.c` - neither built here - and a Wii U shim. A mine defused rather than a live bug,
but nobody should reach for it on this platform.

### What is left in the tree

The five markers stay; they are the record of how far the process gets, and the next thing to move
that boundary will be a driver change. The `_Exit` switch and its `access()` call are gone - a
switch that changes nothing except losing buffered writes is a hazard with no benefit.

## 2026-08-28 — linking the driver is enough; the frontend never has to call it

The zero-cost split, run before touching mesa-ps4: `/data/retroarch/retroarch.cfg` set to
`video_driver = "ps4"` and `menu_driver = "rgui"`, same eboot. The whole driver is still inside the
binary - its static constructors run at start and its destructors at exit - but nothing calls
`vkCreateInstance`. Confirmed from the log rather than from the intent:

    [PS4] video-out up: 1920x1080, 2 buffers, 16 MiB direct     <- the software driver
    (no "Vulkan up", no "vulkan: destroying the context")

**Same crash.**

⚠ **SO IT IS NOT GPU STATE, AND IT IS NOT ANYTHING THE DRIVER TOOK FROM THE KERNEL WHILE RUNNING.**
The software path opens video-out and takes 16 MiB of direct memory itself, and a build without the
driver has always done that and exited silently - so video-out and direct memory are not the
variable either. What is left is what the ARCHIVE contributes to the process: its static
constructors and destructors, its TLS, its data, its size.

### ⚠ AND THE CONTROL FOR THAT IS FROM MEMORY, WHICH IS NOT GOOD ENOUGH

"RetroArch built without our Mesa exits cleanly" is the maintainer's recollection of a build from
weeks ago, and everything under it has changed since - the frontend, orbis-compat, the exit path
itself. It is the whole premise of the hunt and it has never been re-measured on this tree.

So: `make -f Makefile.orbis clean` and a build with no `HAVE_VULKAN` and no `HAVE_OPENGLES`. The
link step prints `vulkan: OFF - software rendering only`, the package drops from 60 MB to 12 MB,
the eboot from 58 MB to 9.6 MB, and `radv`, `wsi/orbis` and `vk_icdGetInstanceProcAddr` are all
absent from the ELF. `/data/pkg/retroarchv-nomesa-20260828.pkg`.

    exits cleanly    the control holds, the archive is the variable, and the bisect is worth doing
    still crashes   the premise is stale and this hunt has been chasing the wrong difference
                    for a day - which is worth finding out in one install rather than in ten

⚠ **THE CLEAN WAS NOT OPTIONAL.** `HAVE_VULKAN` and `HAVE_OPENGLES` are `-D` defines and this
Makefile has no header dependencies, so objects built the other way are silently reused - recorded
here on 2026-08-23 as the trap that shipped a package with no cores in it.

## 2026-08-28 — ⚠ THE PREMISE WAS STALE, AND A DAY WENT INTO THE WRONG DIFFERENCE

The control was run and **it does not hold**. A frontend built with no `HAVE_VULKAN` and no
`HAVE_OPENGLES` - `vulkan: OFF - software rendering only`, 9.6 MB of eboot against 58, `radv`,
`wsi/orbis` and `vk_icdGetInstanceProcAddr` all absent from the ELF - ends **exactly the same way**:

    14:54:01.273  shutdown requested
    14:54:01.273  exit: main_exit returned, main() is about to return
    14:54:01.273  exit: atexit has begun
    14:54:01.273  exit: every atexit handler returned
    14:54:01.273  exit: .fini_array is running

then CE-34878-0.

**So Mesa is not the variable and never was.** Everything above in today's entries - util_queue's
atexit handler, `MESA_LOG_FILE`, `_Exit(0)`, the driver's teardown accounting, the hand-over to
mesa-ps4 - was measuring a difference that is not there. Each measurement is still true; the frame
around them was wrong.

⚠ **AND THE FRAME CAME FROM A RECOLLECTION, WHICH IS THE PART TO KEEP.** "A RetroArch built without
our Mesa exits with no error" was remembered from a build made weeks ago - before this port stopped
idling at Quit (`frontend_orbis_shutdown` used to hold the process on a heartbeat forever, and the
entry that changed it is in this file). So the clean exit being remembered was of a build that
**never returned from main() at all**. It was about a different exit path, not a different link
line, and nothing distinguished those two readings until the control was actually built. A control
that is remembered rather than run is not a control.

### Which leaves ps4_app.h's own sentence standing, and it was right from the start

    Termination: returning from main() on a retail console tears the process down outside the
    system's expected path and pops the error dialog, which reads as a crash (CE-34878-0).

That is the whole explanation, and it fits every measurement made today: the process survives
`main_exit`, every atexit handler, `.fini_array`, and `_Exit(0)` - because none of those is what is
wrong. **Ending without asking the system is.**

`sceSystemServiceLoadExec("exit", NULL)` is the path that asks. "exit" is the reserved argument
meaning hand control back to the system rather than replace this process with another title; the
SDK declares it with a real signature (`SystemService.h:63`) and `-lSceSystemService` has been on
this port's link line since the beginning. It is called from `frontend_orbis_shutdown`, after
`driver_uninit` has already torn everything down.

⚠ **IT IS NOT EXPECTED TO RETURN, AND THE FALLBACK IS THE OLD BEHAVIOUR RATHER THAN IDLING.** A
frontend that will not close is worse than one that closes with a dialog - that is the trade this
port already made once and it stands. The return code is logged either way, because "it refused"
and "it was never reached" must not look alike in a log.

⚠ **AND THE HAND-OVER TO mesa-ps4 HAS TO BE WITHDRAWN.** `ps4-mesa-docs` was given an entry today
saying the close-hang belongs to the driver. It does not. Correcting a record in another workshop
is part of the same job as writing it.

### Confirmed on hardware, and what the record now says

    15:02:41.450  shutdown requested

and nothing after it. No `sceSystemServiceLoadExec ... returned` line, none of the exit markers, no
dialog - the call did not return and the system took the process back. `("exit", NULL)` is the form
this firmware accepts; the second candidate the build carried was never reached and is gone.

**The markers stay.** They are now unreachable on a healthy exit and only appear if
`sceSystemServiceLoadExec` ever returns - which makes them free in the normal case and exactly the
record wanted in the abnormal one.

⚠ **AND THE ENTRY THIS PORT PUT IN ps4-mesa-docs HAS BEEN REVERTED** (`5a1021b`). It told that
workshop the close-hang was theirs to audit, and it was wrong. Leaving it would have cost somebody
a day looking for a leak that is not there. **Correcting a record in another workshop is part of
the same job as writing it.**

`ps4/RELEASE-NOTES.md` and the page `make-site.py` generates both said the dialog was expected and
explained it by a graphics teardown that never happened. Both now say Quit returns to the menu.
⚠ They still warn against *Close Application* from the console - that route has been seen to wedge
the machine and **has not been re-tested since the exit changed**, so it is written as unknown
rather than as fixed. It is the obvious next thing to measure and it costs one Quit's worth of
effort.

## 2026-08-28 — Close Application: a second failure, and this time the control was run first

Quit from inside the application is clean. **Closing from the console's own menu still is not**, and
it is a different fault with a different owner.

### The process is killed outright - nothing of this port runs

Two channels, same session, and both stop mid-sentence:

    UDP    the last line is an ordinary startup line. No "shutdown requested", no
           sceSystemServiceLoadExec, none of the exit markers.
    Mesa   /data/retroarch-mesa.log ends in the middle of a session at frame 768,
           with ZERO "scan-out down".

So `frontend_orbis_shutdown` is never called, the driver never tears down, video-out is never
closed and the scan-out buffers are never unregistered. ⚠ **The exit fix cannot apply here, because
none of this port's code runs.** The application is not told it is being closed.

### ⚠ AND THE CONTROL SEPARATES IT CLEANLY - SAME BINARY, ONE CONFIG LINE

    video_driver = "ps4"       software: video-out, 2 buffers, 16 MiB direct, no GPU   CLEAN
    video_driver = "vulkan"    RADV: video-out, swapchain images, GPU submissions      CRASH

Same eboot, Mesa linked either way, same `sceVideoOut*` API, same `sceKernelAllocateDirectMemory`.
**The only difference is whether RADV was running.** This time the control was measured before the
conclusion was written, which is the lesson the earlier half of today paid for.

⚠ **AND IT RULES OUT THE OBVIOUS READING.** "The display is scanning out of memory that vanishes"
cannot be the whole story: the software driver registers its own direct memory as scan-out buffers
and is killed exactly as abruptly, and the console comes back. What Vulkan adds on top is a live
GPU context with work in flight and swapchain images the GPU renders into.

### What the next experiment is, and what it costs

`wsi_orbis` registers the swapchain's own images as scan-out buffers when it can - zero copy,
`owns_buffers == false` - and falls back to allocating GARLIC buffers of its own with a memcpy per
frame. That fallback is much closer in shape to the software driver that survives.

    zero copy survives too    the display is not the variable; a live GPU context is
    zero copy is the one      the display scanning out RADV's render targets is what the
                              system cannot survive losing, and the copy path is a fallback
                              that could be selected when a title wants to be closable

⚠ **THERE IS NO ENV KNOB FOR IT.** `wsi_common_headless.c:862-893` takes the direct path whenever
every image is CPU-mapped and shares a pitch; `owns_buffers` follows from `addrs == NULL` at
`wsi_orbis.c:297`. Adding the knob is a few lines, but it costs a Mesa rebuild and a frontend
relink, so it is a decision rather than a try.

### ⚠ AND IT IS A DIALOG, NOT A WEDGE - WHICH SETTLES WHAT IT IS WORTH

Measured, not assumed: on the current build, Close Application on the Vulkan driver shows the error
dialog and **the console returns to its menu with no restart needed**.

That contradicts what this port has been telling users since v0.1.x - "that can leave the system
hung and needing a restart" - and the reason it was believed is worth keeping. The wedges that were
actually seen happened to a process that was ALREADY in a bad state: once when the system pthread
pool had been exhausted by the fragment-depth-write option, and once when the frontend still idled
forever at Quit so nothing ever ended. Neither was Close Application's own doing, and both got
written down as if they were.

So this is a cosmetic defect with a working alternative, not a hazard. **It does not justify a Mesa
rebuild on its own.** The zero-copy experiment above stays written down for whoever is in that tree
for another reason.

**Nothing here is a RetroArch change.** Quit is the route that works and the release notes say so.

## 2026-08-28 — v0.1.5, and the page that had been drifting for a day

Released. `retroarch-ps4-v0.1.5` at `ce0609cf1c`, `RetroArchV-PS4-v0.1.5.pkg`, 63 569 920 bytes.

    Quit returns to the console's menu with no dialog     sceSystemServiceLoadExec("exit", NULL)
    Framebuffer emulation works on Nintendo 64            patch 0008, the 639/640 width test
    Sound survives switching games                        one audio port for the life of the run
    No files written into another title's directory       the path anchor
    Stock Beetle PSX withdrawn from the core list         PS4_CORE_DROP

The cores run went green on all eight shards and the publish job with it. Verified by content
rather than by colour:

    .index-extended                                     101 lines
    mednafen_psx_libretro                               absent from the index
    http://cores.prx0.com/mednafen_psx_libretro.prx.zip 404 - the prune took it
    mednafen_psx_hw_libretro.prx.zip                    200 - the fork stayed
    mupen64plus_next  2026-08-28 e00be9e5               rebuilt with patches 0006-0008

### ⚠ THE PAGE IS NOT BUILT BY ANY WORKFLOW, AND THAT IS WHY IT SAID v0.1.4 ALL DAY

`ps4/make-site.py` is called from **nothing** - not `cores.yml`, not `frontend.yml`. The published
page is regenerated by hand, so it drifts from the moment a release is cut until somebody
remembers. Today it was a day behind: the cores run had already replaced `.index-extended` and the
zips, while `index.html` still advertised v0.1.4 and linked yesterday's all-cores bundle.

Regenerated by hand for this release and uploaded with wrangler, which is authenticated locally
under the same account CI uses. The old bundle was deleted explicitly - ⚠ **the prune step does not
touch it**, because it drops objects the index does not name and the index never names the bundle.
That is why `orbis-cores-2026-08-27.zip` was still being served after a run that superseded it.

**This should be a workflow step.** It needs the index, the shard manifests, the `.info` files, the
recipe, the icon and the release's URL and size - the publish job already holds all but the last
two. Until it is one, every release has this manual tail and every release can forget it.

⚠ **AND THE `.info` FILES ARE NOT IN THE SHARD ARTIFACTS.** Only the fork's is. The zips carry the
`.prx` alone, so the licence column - the entire point of the page for GPL purposes - came from
`~/.cache/ps4-cores/info-staged/` on the workshop machine. A workflow step would have to fetch them
from `libretro/libretro-core-info` rather than assume a directory that exists on one laptop. Two of
the 101 have no licence field at all: `bsnes_mercury` and `doublecherrygb`.

## 2026-08-28 — a USB keyboard works, and half the driver was already in the tree

`input/drivers/ps4_input.c` reads a USB keyboard. Measured on hardware against a Logitech ERGO
K860 on a Unifying receiver: the menu takes every key, and cap32 takes every key once Game Focus
is on.

⚠ **`rarch_key_map_ps4[]` HAD BEEN SITTING IN input/input_keymaps.c UNUSED SINCE THE ORBISDEV
PORT** - a complete HID-usage to `RETROK_` table, declared in the header, referenced by nothing,
for the whole life of this port. It is correct as it stands: `0x04 a`, `0x0e k`, `0x16 s`,
`0x1a w`, `0x20 3`, `0x26 9`, `0x33 semicolon` all check out against what the console reports.
Nothing had to be added to it. What was missing was loading the library and opening the device.

### ⚠ A SEVENTH ENTRY IN THE FAMILY, AND THE FIRST ONE THAT IS FATAL

`sceKeyboardInit()` on an unloaded `libSceKeyboard` **ends the process** - no message, no return,
no abort report. The first version of the probe called it directly and died twice, at exactly the
same line, on two boots.

    fcntl(F_SETFL)                    links, called, returns EACCES
    ioctl(FIONBIO)                    links, called, returns EACCES
    dup2 onto fd 2                    links, called, returns EPERM
    sceKernelGetCpuUsage              links, called, returns ENOSYS
    sceKernelQueryMemoryProtection    links, called, answers about something else
    access()                          links, called, returns EPERM against a stat() that works
    sceKeyboardInit, module unloaded  links, called, THE PROCESS ENDS

So "presence is not permission" grows a second half: **for a `.sprx` entry point, presence is not
even a call.** libSceKeyboard and libSceMouse are modules a title must load; the pad, audio and
video-out libraries are loaded for every title automatically, which is why this port had never
called `sceSysmoduleLoadModule` for anything and never noticed.

    sceSysmoduleIsLoaded(0x0106)     -> 0x805a1001   not loaded
    sceSysmoduleLoadModule(0x0106)   -> 0            the PUBLIC id is the one that works
    sceKeyboardInit                  -> 0
    sceKeyboardOpen(user, 0, 0, NULL)-> 0x011e0700   a handle, first try
    sceSysmoduleLoadModule(0x00A9)   -> 0            libSceMouse loads too

⚠ **EVERY FUTURE `-l<SceThing>` ON THIS PORT'S LINK LINE HAS TO BE PAIRED WITH THE MODULE LOAD**,
or the first call is a crash with nothing to read.

⚠ **AND THERE IS NO PRIVILEGE GATE.** `<orbis/Keyboard.h>`'s un-reversed list holds
`sceKeyboardSetProcessPrivilege` and `sceKeyboardSetProcessFocus`, which is what made a gate look
likely enough to probe for. Neither is needed: Open answered an ordinary GoldHEN-loaded title.

### Two details of the data that a driver written from the header would get wrong

⚠ **AN EMPTY STATE IS `nkeys == 1` WITH `keycodes[0] == 0`, NOT `nkeys == 0`.** A loop that treats
each of `nkeys` entries as a pressed key reports keycode 0 as held in every frame where nothing is
down.

⚠ **AND THE DIFF HAS TO WALK ALL 256 USAGES, NOT THE CURRENT REPORT.** A released key never appears
in `keycodes[]` again, so a loop over what arrived this frame can only ever press keys and never
let one go.

### ⚠ THE KEYBOARD THEN LOOKED HALF-BROKEN, AND THE FRONTEND WAS DOING IT ON PURPOSE

cap32 received `o j d . /` and nothing else while the menu took everything. Not a driver fault:
`input/input_driver.c:8624` drops any key bound to a RetroPad button or a hotkey before the core
sees it, and those five are simply the ones the default binds leave alone.

    if (down && !input_st->game_focus_state.enabled
        && BIT512_GET(input_st->keyboard_mapping_bits, code) ...)
        ... if (block_key_event) return;

The setting is **Settings → Input → "Auto Enable 'Game Focus' Mode"**, `input_auto_game_focus`,
and upstream defaults it to OFF. **This port now defaults it to Detect** (`config.def.h`, ORBIS
arm): on a desktop the escape is a Scroll Lock hotkey everyone knows, here there is no keyboard
hotkey bound and the setting is four menus deep.

⚠ **CHECKED BEFORE CHANGING IT, BECAUSE THE OBVIOUS WORRY IS BEING STRANDED IN A CORE.**
`game_focus_state.enabled` has exactly one reader in the whole input path - that keyboard filter -
plus a mouse grab that is a no-op here. Joypad hotkeys are untouched, so the pad combo that opens
the menu still works. Detect rather than On, so a pad-only session is unchanged.

### Still open

**`character` is always 0.** RETROK codes are right, so games and home-computer cores work, but
RetroArch's own text fields - the search box, network settings - read `character` and will take
nothing. `sceKeyboardGetKey2Char` exists with a real signature, but one of its parameters is
`bool unknown` and nobody has reversed what it means, so it is a probe rather than a guess.

**The mouse is not started.** Its module loads and `sceMouseInit()` returns without killing
anything, but every other `sceMouse*` entry point in this SDK is a name with no signature -
`void sceMouseOpen();` - so there is nothing to call without inventing an ABI.

## 2026-08-28 — two input defects the keyboard work uncovered, and neither was the keyboard

### ⚠ A SLEEPING PAD NEVER CAME BACK, AND THE FLAG WAS A LATCH

`ps4_joypad_poll` opened with `if (!ds_joypad_states[i].connected) continue;`, and the disconnect
branch set that flag false with **nothing anywhere setting it back**. A DualShock 4 that goes to
sleep reports `connected == 0` for one frame; from then on its slot was never polled again and the
pad was gone for the rest of the process. Only a restart brought it back.

The flag is now DERIVED from the read, every frame, and the two ways of saying nothing are told
apart - a distinction the old code did not make:

    scePadReadState fails       this frame said nothing. Buttons cleared, slot untouched.
                                Latching them is how a menu runs away on its own.
    data.connected == 0         a real disconnection. Flag down, analogue axes zeroed too -
                                they used to keep the last value, so a stick could stay
                                deflected after the pad fell asleep.

⚠ **AND THE PART THAT COULD NOT BE READ OUT OF THE CODE WAS MEASURED.** Whether the console hands
back the SAME handle on wake, or invalidates it and wants `scePadGetHandle` again, was a
prediction until the log said:

    19:10:57.947  [PS4] pad 0 went away.
    19:11:13.147  [PS4] pad 0 is back.

Same handle, fifteen seconds apart, nothing reopened. The two lines exist so that the next person
sees which half happened rather than only that the pad works.

⚠ **Forcing this takes seconds, not ten minutes of waiting**: hold PS on the pad for ~10 s until
the light bar goes out. Sleep and power-off both drop the BT link, so the driver cannot tell them
apart. A pad on a USB cable will do neither.

### ⚠ AND THE KEYBOARD HANDLE WAS THE AUDIO PORT AGAIN

Found by reading the log for the pad lines rather than by looking for it. RetroArch reinitialises
its input driver on every content load and every video-driver switch, and `ps4_kbd_open` was
opening a fresh keyboard each time:

    boot  9   11 opens
    boot 13    9 opens for 4 content loads
    handles   0x01210700, 0x01230700, 0x01250700 ... climbing by 0x20000, never reissued,
              monotonic across the whole day's fourteen boots

**That is the audio port's signature exactly** - a finite per-process resource, a close that
reports success, and a number that is not given back. Audio ran out after eight and went silent.
Nobody has found the keyboard's limit and the way to not find it is to stop asking.

Fixed the same way: one handle for the life of the process, later inits get the same one,
`ps4_kbd_close` no longer closes. A refusal is remembered too, so it is not retried on every
reinitialisation. Confirmed:

    before   boot 13:  9 opens, 4 content loads
    after    boot 15:  1 open,  3 content loads

⚠ **THIS IS THE SECOND TIME THIS SHAPE HAS COST SOMETHING, AND IT WILL NOT BE THE LAST.** Anything
this console hands out per process - audio ports, keyboard handles, and whatever comes next -
should be assumed non-returnable until measured otherwise, and held rather than reopened.

### What the keyboard does and does not do now

Working, measured on hardware: menu navigation, home-computer cores through Game Focus, hotplug of
the receiver while RetroArch runs, and the driver surviving with no receiver attached at all -
`sceKeyboardOpen` answers whether or not hardware is present.

Still open: **`character` is always 0**, so RetroArch's own text fields take nothing;
`sceKeyboardGetKey2Char` has a real signature but one parameter is `bool unknown`. And the
**mouse** is not started - every `sceMouse*` past `Init` is a name without a signature.

## 2026-08-28, close of day — where this stands, and PlayStation 2 next

### What is live

v0.1.5 is released and on `cores.prx0.com`: `RetroArchV-PS4-v0.1.5.pkg`, 63 569 920 bytes, 101
cores in the index, the page rebuilt for it. Quit returns to the console's menu with no dialog,
framebuffer emulation works on Nintendo 64, sound survives switching games, and files no longer
land in another title's directory.

### What is committed and NOT pushed

One commit per repository, both finished and both local:

    orbis-ports/RetroArch     cabc0080f5  (ps4) Keyboard, pad recovery, and the page in CI
    orbis-ports/ps4-mesa-docs ddce559     Record the keyboard, the pad latch, and the handle leak

⚠ **`802ad9ef8b`'s content is inside that RetroArch commit and it changes nothing until the next
FULL cores run** - it is the step that builds and uploads the download page from the publish job.
Until then the page is still whatever was uploaded by hand on 2026-08-28. And the ordering it
needs is written into the step: **tag the frontend, let frontend.yml cut the release, THEN run
cores** - the page reads the newest release for its version and package link.

### ⚠ THE WORKING RULES THIS SESSION HAD TO LEARN, WHICH ARE NOT ABOUT THE PORT

**One commit, amended.** Not a commit per finished piece. This branch reached 142 commits of which
75 touched nothing but a `.md`, and had to be rewritten to 30. Do not push without being asked.

**The narrative lives here, not in the code repository.** `ps4/RELEASE-NOTES.md` and
`ps4/CORE-STATUS.md` stayed in the RetroArch tree because the build reads them - a release's
`body_path` and the sharder's size table. Everything else moved.

**A control that is remembered is not a control.** A whole day went into the close-hang on the
belief that a build without Mesa exited cleanly. It did not; that recollection was of a build from
before the port stopped idling at Quit. Building the control settled it in one install.

### Open, cheapest first

    pcsx_rearmed        fails at LINK on lightrec_init_mmap - the same executable-memory problem
                        already solved for Beetle PSX. Purely local work, no console needed.
    keyboard character  input_keyboard_event gets character 0, so RetroArch's own text fields take
                        nothing. sceKeyboardGetKey2Char has a real signature; one parameter is
                        `bool unknown`, so it is a probe rather than a guess.
    PrBoom Load State   does NOT crash on the host - seven in-session loads and a cross-session
                        one, same commit, same WAD. On the console retro_serialize_size() returned
                        198200, which is exactly sizeof(extra) + 0x30000, the FLOOR, reached only
                        when thinkercap.next == NULL - a thinker list never initialised, not merely
                        empty (an empty one points at itself). So the core had no live level at the
                        moment of the load. One log line in the core settles it.
    GLideN64 clipping   `gl_Position.z /= 8.0` is never scaled back on this build. Now cheap to
                        test: patch 0008 added an `enableClipping` knob in /data/retroarch-gliden64.
    nestopia            loads, runs, exits cleanly, renders a green screen.
    coverage            61 of 101 built cores untested on hardware; 62 of 164 do not build.
    GL_TEXTURE_EXTERNAL_OES   `GL_INVALID_ENUM in glFramebufferTexture2D(unknown textarget 0x8d65)`,
                        seen on the host, pre-existing, nobody has looked.

## ⚠ NEXT: PlayStation 2, and what the recipe already says about it

The maintainer wants to see what can be got out of PS2 emulation here. Two cores exist in
`cores-linux-x64-generic`, and a third that is barely a project:

    pcsx2   https://github.com/libretro/pcsx2.git   CMAKE
    play    https://github.com/jpd002/Play-.git     CMAKE
    yaps2   https://github.com/yaps2/yaps2.git      CMAKE

⚠ **THE FIRST BLOCKER IS NOT PS2 AT ALL, AND FIXING IT PAYS FOR MORE THAN PS2.**
`ps4/build-cores.sh:500` skips every CMAKE core for want of a toolchain file - **seventeen cores**,
and the list is not a PS2 list:

    applewin arduous citra_canary dirksimple dolphin duckstation easyrpg flycast ishiiruka
    melondsds pcsx2 play swanstation thepowdertoy tic80 yaps2 trident

`duckstation` and `swanstation` are on it. Those are the two PlayStation cores this file has been
recording as "never attempted" since the Beetle work, and they would arrive as a side effect. So
the honest order is: write the CMake toolchain file first, see what falls out of seventeen cores,
and treat PS2 as one of the answers rather than the goal.

⚠ **AND TEMPER THE PS2 EXPECTATION WITH THIS FILE'S OWN ARITHMETIC.** Recorded earlier: PCSX2
"wants an order of magnitude more CPU than this machine has, so treat it as arithmetic rather than
porting." The measurements behind that are in this file and they are not encouraging by analogy -
Beetle PSX needs a recompiler and its own Vulkan renderer to hold 50 fps on one saturated Jaguar
core, and mupen64plus-next needed a GL context driver and the HLE RSP to reach 60. PS2 is a
generation past both.

**Play! is the one to try first, not PCSX2.** It is far lighter, has its own x86-64 recompiler, and
its renderer targets can be driven by the GL context driver this port already has. PCSX2 is worth a
build only to find out where it stops, and that answer should be written down rather than guessed.

⚠ **THE THREE THINGS THIS PORT ALREADY KNOWS THAT A PS2 CORE WILL MEET.** All of them cost days the
first time and are solved:

    executable memory   this kernel refuses PROT_EXEC at map time and grants it to mprotect after.
                        ps4/orbis_exec_mem.c, and beetle-psx's ps4/orbis_lightrec_mem.c.
    a GL context        gfx/drivers_context/orbis_gl_ctx.c, over Mesa's EGL and zink. GLES 3.1.
    the toolchain bugs  libc++'s ETIMEDOUT compiled against Linux's 110 on a FreeBSD target,
                        stderr going nowhere, and ps4/orbis_profile.c for measuring instead of
                        arguing. All three presented as a core crashing.

⚠ **AND ONE RULE THAT IS NEWER THAN MOST OF THIS FILE.** For a `.sprx` entry point, presence is not
even a call: `sceKeyboardInit()` on an unloaded module ENDS THE PROCESS rather than returning an
error. Any new `-l<SceThing>` has to be paired with `sceSysmoduleLoadModule`.

## 2026-08-28, later — the CMake toolchain file, and what the seventeen cores actually said

The previous entry said the first blocker was not PS2 but the missing CMake toolchain file gating
seventeen cores. That is now written, all seventeen have been attempted, and **swanstation - a
PlayStation core - builds.** The PS2 answer is in the last section and it is not the one this file
expected.

### Where the toolchain file lives, and why it is generated

`ps4/build-cores.sh` writes `$WORK/orbis-core.cmake` at the start of every run, from `$ORBIS_ARCH`
and `$C_INCLUDES`/`$CXX_INCLUDES` - the same arrays that already build `$CC_ORBIS` and
`$CXX_ORBIS` for the make path. There is nothing to keep in step, which was the whole point: two
descriptions of what this platform is would drift, and the second one would drift silently.

⚠ **IT IS NOT `orbis-compat/cmake/ps4-openorbis.cmake`, AND MERGING THEM WOULD BE WRONG.** That
file is for EXECUTABLES - Tempest, OpenGothic, VK-GL-CTS. It links `crt1.o` into the product,
puts the overlay on the link line with `--whole-archive`, forces `BUILD_SHARED_LIBS OFF`, and does
not pass `-nostdinc`. A libretro core is none of those things: its own link is EXPECTED to fail,
and build-cores.sh collects the objects and links them against `ps4/orbis-module.ld` itself.

`CMAKE_SYSTEM_NAME` is **FreeBSD**, not Generic. Generic leaves `UNIX` unset, and a libretro
CMakeLists routinely branches on `if(UNIX)` for threading, dynamic loading and endianness - taking
the other arm is not a build failure, it is a wrong build.

### ⚠ THE FIVE TRAPS, EVERY ONE OF WHICH REPORTED SOMETHING ELSE

**1. AN EMPTY TOOLCHAIN FILE IS NOT AN ERROR ANYWHERE, AND IT PASSES AS A GREEN CORE.**
The heredoc that writes the file is UNQUOTED - it has to be, that is how the flag arrays reach it -
so every backtick in it is live. A pair of backticks in one of its own comments turned the body
into a command substitution and left the file ZERO BYTES. CMake reads an empty toolchain file
happily, falls back to `/usr/bin/c++` and this desktop's headers, and builds a core for THIS
DESKTOP. ld.lld links it and create-fself accepts it, because host and console are both x86-64.
**The verdict was `OK` and the module was 1.5M.** The tell came one core later:

    undefined symbol: std::cerr     the reference was _ZSt4cerr - libstdc++'s mangling
                                    libc++.a defines _ZNSt3__14cerrE

There is now a `grep -q CMAKE_CXX_FLAGS_INIT` guard right after the heredoc that exits non-zero.
Anything written by an unquoted heredoc has to be looked at afterwards. The check that settles it
on any core is `readelf -h <obj> | grep OS/ABI` - it must say **UNIX - FreeBSD**.

**2. `CMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY` IS THE DOCUMENTED ESCAPE AND IT IS A TRAP.**
It is the standard answer for a target whose link cannot run, and it configures. It is wrong twice:

  * **Nothing links, so every `check_function_exists()` says yes.** A core asking whether
    `shm_open` exists is told yes because the reference compiled. It then configures a path this
    platform has no symbol for.
  * **CMake feeds the EXECUTABLE linker flags to the ARCHIVER.** The static library rule is
    `<CMAKE_AR> qc <TARGET> <LINK_FLAGS> <OBJECTS>` and llvm-ar reads a leading word as operation
    LETTERS, so yaps2's own compiler-flag probes came out as `llvm-ar: error: unknown option n`
    and `unknown option f` - reported as a COMPILE failure inside a try_compile.

So the toolchain file gives try_compile a REAL executable link line - the SDK's crt1.o, its
libraries, and orbis-compat's corrected `orbis-tls.ld`. Verified: a hello world links.

**3. A STALE `CMakeCache.txt` SURVIVES `git reset --hard` AND `git clean -fd`.** Every core
gitignores its build directory and clean does not touch ignored paths without `-x`.
`CMAKE_CXX_FLAGS_INIT` is only consulted on the FIRST configure, so a cache from a previous run
keeps the flags that run was given. Correcting the toolchain file and rebuilding changed nothing,
twice, with identical compiler errors each time - which reads as a fix that does not work rather
than a fix that was never applied. `rm -rf "$cbuild"` when `$KEEP -eq 0`, for the same reason the
patch loop resets before it patches.

**4. CMake LEAVES OBJECTS THAT DEFINE `main`.** Three kinds: `CMakeFiles/<version>/CompilerId*/`,
`CMakeScratch/` and `CMakeTmp/`, and - the one that cost a link - `check_ipo_supported()`, which
configures and builds AN ENTIRE SUB-PROJECT under `CMakeFiles/_CMakeLTOTest-C/` whose objects sit
in a perfectly ordinary-looking `boo.dir/`. swanstation: `duplicate symbol: main`. So "it is in a
`<target>.dir`" is NOT the test; the exclusions match a version number or an underscore-prefixed
component under `CMakeFiles/`.

**5. pkg-config IS A HOST PROGRAM AND ANSWERS WITH HOST PATHS.** `CMAKE_FIND_ROOT_PATH` has no say
over it. yaps2 printed `Found Freetype: /usr/lib/libfreetype.so` and `Found WebP: /usr/include` -
this desktop's, for a console build. That is the GLES-header accident one layer out, and it ends
in a core linking a host shared object. `PKG_CONFIG_LIBDIR` REPLACES the search path rather than
adding to it, so it is now pointed at the SDK.

Also, minor but universal: **CMake 4 removed compatibility with `cmake_minimum_required(<3.5)`**,
which most of these cores predate. `-DCMAKE_POLICY_VERSION_MINIMUM=3.5` is on the configure line.

### ⚠⚠ THE `__FreeBSD__` UNDEF IS DELIBERATE. DO NOT "FIX" IT. I DID, AND IT WAS WRONG.

`$OO_PS4_TOOLCHAIN/include/c++/v1/__config` line 12 is `#undef __FreeBSD__`. clang defines it as
12 for `--target=x86_64-pc-freebsd12-elf`, so the effect is that **a C++ TU on this platform sees
NO platform macro at all** - not `_WIN32`, not `__linux__`, not `__APPLE__`, not `__FreeBSD__` -
while every C TU in the same binary sees `__FreeBSD__ == 12`. The two halves of one program
disagree about what they are running on.

It looks exactly like a libc++ build artifact - a cmakedefine slot filled with a macro name it had
no business clearing - and I wrote a shim (`include/libcxx/__config`, ahead of libc++, handing
through with `#include_next` and putting the macro back), put it in orbis-compat, build-cores.sh
and Makefile.orbis, and it compiled clean including the std::abs check.

**It is load-bearing. It is how a libc++ built against musl is steered away from FreeBSD's arms.**
With `__FreeBSD__` restored, libc++ takes:

    __locale:35    #include <xlocale.h>          instead of  support/musl/xlocale.h
                   - and this SDK has no xlocale.h, so 13 fatal errors
    locale:220     #define _LIBCPP_GET_C_LOCALE 0   instead of  __cloc()

⚠ **THE SECOND ONE IS THE DANGEROUS ONE AND IT DOES NOT FAIL TO COMPILE.** The prebuilt
`libc++.a` was compiled with the macro undefined, so it uses `__cloc()`. Headers saying `0` would
disagree with the library across the ABI boundary, in `num_get`/`num_put`. The whole shim is
reverted; all three repositories are back to what they were.

**The right fix is per-core, and it is what the swanstation patches do:** `__ORBIS__` is always
defined here, so name it in the branch alongside `__FreeBSD__`.

### What the seventeen said

    OK        arduous       1.5M     thepowdertoy  4.3M     swanstation  6.3M (+2 patches)
    LINK      trident       349o  SDL_iconv_string_REAL      SDL2 built without its own iconv
              dirksimple     71o  luaopen_utf8               its bundled lua omits the utf8 lib
              melondsds     175o  duplicate symbol adler32_z FetchContent builds zlib TWICE, as
                                                             zlib.dir and zlibstatic.dir, and both
                                                             object sets are collected
    COMPILE   applewin           its libretro CMakeLists refuses to configure
              easyrpg            PlayerFindPackage - missing deps
              play               find_package(OpenGL) wants GLX, via deps/glew-2.0.0
              yaps2              find_package(plutovg 1.1.0), and more behind it
              flycast            find_package(OpenGL) wants GLX, CMakeLists.txt:238
              dolphin            DolphinLibraryTools.cmake:169 - "Requires LLVM_libc++ 150000 or
                                 higher". The SDK's libc++ is older, and that is an SDK question
                                 rather than a core one.
    LINK      ishiiruka     249o  glslang::InitializePoolIndex - it also wants SFML ("this
                                 operating system is not supported"), libusb, and 64 x
                                 'osreldate.h' file not found, which IS an overlay candidate:
                                 a FreeBSD header the SDK omits, reached from C where
                                 __FreeBSD__ is still defined.
    CLONE     duckstation        THE REPOSITORY IS GONE - see below
              pcsx2              THE REPOSITORY IS GONE - see below
              citra_canary       submodule externals/boost cloned twice, git aborts
              tic80              no branch 'master' - the recipe is out of date

**swanstation needed two patches, both the same one-line shape**, and both are consequences of the
`__FreeBSD__` section above rather than of anything about PlayStation:

    0001  memory_arena       named __ORBIS__ so the file compiles. The arena STAYS UNIMPLEMENTED
                             on purpose - its POSIX arm needs shm_open, which this SDK has in
                             neither its headers nor libc.a. Create() falls through to the
                             existing `return false`, a case the caller already handles: the arena
                             backs fastmem, and swanstation runs without it.
    0002  cpu_recompiler     `#error Unknown ABI.` x11, plus every register name behind that
                             branch reported as an undeclared identifier. Not a guess: the triple
                             is x86_64-pc-freebsd12-elf, so the convention IS System V.

⚠ **NOTHING ABOUT swanstation HAS BEEN RUN ON HARDWARE.** It links, it carries `retro_run`, and
create-fself accepted it. That is all that is known. It is also a second PlayStation core beside
`mednafen_psx_hw`, so before it is offered in the menu, read the `PS4_CORE_DROP` comment in
build-cores.sh about what a second name a letter apart costs a new user.

### ⚠ PLAYSTATION 2: THE CORE THE RECIPE NAMES DOES NOT EXIST ANY MORE

    git ls-remote https://github.com/libretro/pcsx2.git
      -> fatal: could not read Username for 'https://github.com'

That is what GitHub says for a repository that is deleted or private. **`libretro/duckstation` is
gone the same way.** Both are still in `cores-linux-x64-generic`, so the recipe is describing a
world that has moved. This was not knowable before the toolchain file existed, because both cores
were being skipped for their build type and never reached a clone.

So PS2 in RetroArch here is **Play!**, which is alive (`jpd002/Play-`, HEAD 04bde0d), and it stops
at `find_package(OpenGL)` inside `deps/Dependencies/glew-2.0.0` wanting GLX. That is the same wall
`flycast` hits, and it is a solvable one rather than a missing project: this port has GLES 3.1
through zink and a GL context driver, and what is missing is a CMake-level answer for
`find_package(OpenGL)`. Whoever takes it next should look at whether Play!'s libretro target needs
glew at all, since the recipe already passes `-DBUILD_PLAY=off`.

The earlier arithmetic in this file still stands and is worth repeating: PCSX2 "wants an order of
magnitude more CPU than this machine has". Play! being the only living option is not a downgrade
from the plan - it was already the one to try first.

## 2026-08-28, evening — Play! builds, and PS2 content is on the console

**`play` is `OK`, 8.3M, `04bde0d+3`.** A PlayStation 2 core. Nothing has been run on hardware.

Uploaded over lftp to 192.168.100.2:2121, so there is something to test with:

    /data/retroarch/system/ps2/          14 BIOS files
    /data/retroarch/roms/ps2/            Grand Theft Auto III, 4 698 767 360 bytes, byte-exact
    /data/retroarch/cores/               play_libretro.prx (8 625 264) and
                                         swanstation_libretro.prx (6 528 320)
    /data/retroarch/system/play/         created - PathUtils patch points here

### What Play! needed - four things, all in ps4/core-patches/play/

**`-DUSE_GLES=ON`, in core_cmake_flags().** `deps/Framework/build_cmake/FrameworkOpenGl` picks GLES
by platform NAME - Android, iOS, ARM, Emscripten - and this console is none of them, so it took the
desktop arm: `find_package(GLEW)`, then a bundled glew-2.0.0 doing `find_package(OpenGL REQUIRED)`
which wants GLX. USE_GLES is the truthful answer, not a workaround: this port has no desktop GL and
no GLX, it has GLES 3.1 through zink, which is the arm that variable selects.

**0001, zlib.** Play! does not use upstream zlib's CMakeLists - it has a minimal one that compiles
the sources directly and generates no `zconf.h`, so `Z_HAVE_UNISTD_H` is never defined, `gzguts.h`
skips `<unistd.h>`, and gzlib/gzread/gzwrite compile with lseek, read, write and close undeclared.
Latent on every platform; only fires on a libc that does not leak those declarations in through
another header. glibc does, musl does not.

**0002, PathUtils.** `getpwuid(getuid())`, the same `__FreeBSD__` mechanism as swanstation.
⚠ ADDING `__ORBIS__` TO THE LINUX ARM WOULD HAVE COMPILED AND THEN CRASHED - every function there
is `fs::path(getenv("HOME")) / …` and this console has no HOME and no XDG_*. It needed real paths.

**0003, and it is the interesting one: THE PROC-ADDRESS FUNCTION DOES NOT ONLY COME FROM glsm.**
`ps4/orbis_gl_forward.c` took it from `GLSM_CTL_PROC_ADDRESS_GET`, because every GL core so far
used libretro-common's glsm. Play! does not - it keeps its own `retro_hw_render_callback` and calls
the GLES core ABI directly. So the thunk table stayed null and the link wanted glTexImage2D,
glTexParameteri and glRenderbufferStorageMultisample. Three changes in this repository:

    orbis_gl_forward.c   orbis_gl_resolve_proc(getproc) split out; orbis_gl_resolve() is now a
                         wrapper that fetches it from glsm. Plus the three entry points -
                         ⚠ APPENDED, NEVER INSERTED: each thunk carries its slot as a LITERAL byte
                         offset (index * 8), so alphabetical order would renumber every thunk
                         after it. orbis_gl_slot grew 134 -> 137.
    orbis_weak_stubs.c   a WEAK glsm_ctl returning 0. An undefined weak symbol is exactly what
                         create-fself refuses - "missing library for symbol (glsm_ctl)" - even
                         though ld.lld is content. Weak means a core that DOES build glsm still
                         wins.
    core-patches/play    0003 calls orbis_gl_resolve_proc(g_hw_render.get_proc_address) from
                         retro_context_reset.

⚠ **THE glsm_ctl STUB WAS VERIFIED BY SYMBOL, NOT BY A GREEN VERDICT**, because getting it wrong
would be a black screen on Nintendo 64 rather than a build failure:

    mupen64plus_next.elf    t glsm_ctl   local, the core's real one, 393 lines of disassembly
    play.elf                W glsm_ctl   weak, this stub, as intended

### ⚠ TWO HARNESS BUGS FOUND HERE, AND THE FIRST ONE FAKED A WORKING PATCH

**`git reset --hard` AND `git clean -fd` DO NOT REACH INTO A SUBMODULE.** reset restores the
recorded COMMIT of a submodule, not the files in one; clean skips them. Play! keeps deps/Framework
and deps/Dependencies as submodules, so a hand edit inside either survives every reset - and
`git diff` in the superproject records such an edit as

    -Subproject commit 8a5f6b1…
    +Subproject commit 8a5f6b1…-dirty

AND NOTHING ELSE. So patch 0001 contained no change whatsoever, the build that "proved" it worked
was reading my dirty worktree, and git apply refused it the moment the tree was clean. The loop now
does `submodule foreach --recursive` reset and clean. A patch that touches a submodule has to be
generated from inside it:

    git -C deps/<sub> diff --src-prefix=a/deps/<sub>/ --dst-prefix=b/deps/<sub>/

**BUILD THE CORE'S TARGET, NOT `all`.** A CMake tree ships tools and test suites, and their entry
points get swept into the link: play gave `duplicate symbol: main` from CodeGenTestSuite and
NamcoSys147NANDTools, neither of which honoured the recipe's `-DBUILD_TESTS=no`. libretro CMake
cores name their target `<core>_libretro`, the convention this file already uses for the .prx
filename, so that target is built when it exists and `all` is the fallback.

⚠ **AND THE TARGET DETECTION WAS WRITTEN WITH `grep -q` IN A PIPELINE, WHICH THIS FILE ALREADY
WARNS ABOUT** two hundred lines above at the retro_run check. grep -q exits on first match, cmake
gets SIGPIPE, and under `set -o pipefail` the pipeline fails - so it answered `all` every time and
the duplicate `main` did not go away despite the fix being correct. Capture to a variable and match
with `[[ ]]`.

### Regression, checked rather than assumed

    swanstation  OK 6.3M   arduous OK 1.5M   thepowdertoy OK 4.3M   mupen64plus_next OK 5.5M

## ⚠⚠ 2026-08-28, night — THIS KERNEL DOES NOT DELIVER SA_SIGINFO, AND IT HAS BEEN EATING CRASHES

Loading content in Play! took the process down instantly. The dump named the instruction:

    # signal: 11 (SIGSEGV)      # reason: page fault (user read data, page not present)
    # fault address: 000000000000001a
    # rdi: 000000089c04b900     # rsi: 0000000000000002     # rdx: 00000007ee3c7880
    # rip: 0000000800a8bb40  ->  play_libretro.prx text 0x800864000, so + 0x227b40

    227b30 <CEeExecutor::HandleException(int, __siginfo*, void*)>
    227b39:  movq 0x529eb8(%rip), %rdi   # g_eeExecutor - loaded fine, rdi is a real pointer
    227b40:  movq 0x18(%rsi), %rsi       # sigInfo->si_addr   <- rsi is 2, and 2 + 0x18 = 0x1a

**The handler was entered correctly and handed a `siginfo_t*` of 2.** rdi = 11, a small integer in
rsi, and a valid stack address in rdx is FreeBSD's ORIGINAL signal handler signature -

    void (*)(int sig, int code, struct sigcontext *scp)

- and not SA_SIGINFO's `void (*)(int, siginfo_t *, void *)`. The flag is set in the sigaction the
core installs and does not reach the kernel.

⚠ **AND THIS PORT'S OWN CRASH REPORTER MAKES THE SAME ASSUMPTION.**
`orbis-compat/src/orbis_boot.cpp` installs `ps4SignalAction` with `SA_SIGINFO | SA_ONSTACK` and its
first act is `info != nullptr ? info->si_code : 0`. A `code` of 2 is not null, so the null check
passes and the read faults - inside the SIGSEGV handler, with `reentered` already set to 1, which
goes straight to `_Exit(2)`. **The process dies silently at the exact moment it was supposed to
explain itself.**

The evidence across every log ever captured in build-ps4-logs:

    "crash handlers installed ... sigaction rc=0"    present, repeatedly
    "fatal: signal ..."                              ZERO occurrences, in any log, ever

So the handler has been installed successfully and has never once produced a line. Every silent
death this port has investigated - and there have been several - had a crash reporter that could
not survive its own first statement.

⚠ **THIS IS THE HIGHEST-VALUE THING IN THIS FILE RIGHT NOW AND IT IS NOT FIXED.** The fix belongs
in orbis-compat, not here, and it is not a one-liner because the handler has to work out which
convention it was called with. A handler that logs its three raw arguments and nothing else would
settle the shape in one run; from the dump above the second argument is `code` and the third is a
`struct sigcontext *`, whose `sc_addr`/`sc_err` fields carry what si_addr was meant to carry.
Until then, assume ANY code in this port that reads `siginfo_t` in a signal handler is reading a
small integer as a pointer.

### Play! after that: `04bde0d+4`, uploaded

Patch 0004 takes Play!'s own escape - `DISABLE_PROTECTION`, which iOS, tvOS and 32-bit ARM builds
use - because the address the handler exists to read is the thing that never arrives, so there is
nothing in the handler to salvage. AddExceptionHandler becomes a no-op and SetMemoryProtected
becomes a nop.

⚠ **IT IS A REAL TRADE, NOT A FREE ONE.** Writes into already-translated code are no longer
detected by a fault, so a self-modifying game can run stale translated blocks. That is a
correctness risk rather than a crash, and it is the same one every iOS build of Play! carries.

### ⚠ IT RUNS. Grand Theft Auto III, on the console, at 3-5 fps

Confirmed on hardware 2026-08-28: `play_libretro.prx` at `04bde0d+4` boots GTA III and renders it.
That is the first PlayStation 2 content this port has ever run. It is not playable.

**What is already ruled out as the cause of 3-5 fps:**

    DISABLE_PROTECTION   NOT the cause, and worth stating because it looks like one. It makes
                         SetMemoryProtected a nop, and that call is only ever mprotect - so
                         translated blocks are invalidated LESS often, not more.
    executable memory    probably fine. deps/CodeGen/src/MemoryFunction.cpp maps its code with
                         mmap(PROT_WRITE | PROT_EXEC), which is exactly what this kernel refuses at
                         map time - and its assert() is a nop in Release, so a failure would be
                         memcpy() into MAP_FAILED and an instant crash. It renders instead, so the
                         mapping succeeded. ⚠ NOT PROVEN, only inferred - a core that fell back to
                         an interpreter would also render, slowly. Worth one log line.

**The next measurement costs nothing and needs no build.** The core exposes
`play_res_multi` (Resolution Multiplier; 1x|2x|4x|8x) and defaults to 1x, so the frame rate is not
being spent on extra pixels. Set it to 2x on hardware:

    frame rate barely moves   -> CPU-bound: EE/VU and the recompiler. Ask whether the recompiler
                                 is running at all before optimising anything.
    frame rate falls to 1-2   -> GPU/GS-bound: GSH_OpenGL through zink, a different problem.

Do that before touching code. This file has paid for guessing at performance before.

## ⚠ WHERE THE PS2 FRAME ACTUALLY GOES - MEASURED, SO NOBODY HAS TO GUESS AGAIN

Grand Theft Auto III, on hardware, `/data/retroarch-profile` present, core `04bde0d+6`:

    27 frames in 5138 ms = 5.25 fps
      ee             177.20 ms/f   93% of wall
      iop              8.11 ms/f    4%
      spu              2.14 ms/f    1%
      blockfactory     0.00 ms/f    0%      <- whole function, cache hit included

**EE is the frame. Everything else is noise.** Two things fall out of that and both close a
line of enquiry that was open for hours:

  * **THE RENDERER IS NOT THE PROBLEM.** GS/GL does not even appear. This matches the hardware
    observation that `play_res_multi` at 1x, 2x and 4x produced no measurable difference - the
    GPU is idle enough that four times the pixels costs nothing.
  * **THE BLOCK CACHE IS PERFECT.** `blockfactory` is 0.00 ms/f and it wraps the WHOLE function
    including the cache-hit path, which still hashes the block with XXH3 and copies every opcode.
    So the 178 ms is spent RUNNING translated code, not making it. Recompiler quality and cache
    lifetime are both off the table.

⚠ **AND THE OPTIMISER IS ON, WHICH WAS THE CHEAP THING TO CHECK BEFORE DRAWING CONCLUSIONS.**
`flags.make` for PlayCore carries `-msse -msse2 -mssse3 -O3 -DNDEBUG`. A toolchain file that set
only `CMAKE_<LANG>_FLAGS_INIT` could easily have lost `CMAKE_<LANG>_FLAGS_RELEASE`; it did not.

### ⚠ SO 5 fps IS ARITHMETIC, NOT A BUG, AND THE NUMBERS AGREE TOO WELL TO IGNORE

Play! needs roughly 13 ms/frame for this game on an ordinary desktop. A 1.6 GHz Jaguar is about
14x slower than that once clock and IPC are both counted. 13 x 14 = 182 ms. Measured: 177-190.

⚠ **AND Play! IS SINGLE-THREADED WHERE IT MATTERS.** `CPS2VM` starts exactly one `EmuThread`
(PS2VM.cpp:327) and runs EE, VU0, VU1 and IOP sequentially inside it. There is no VU thread and no
option to add one. The other five Jaguar cores cannot help without restructuring the emulator, so
there is no easy win available here - this is the ceiling of THIS emulator on THIS machine, and
saying otherwise would repeat the mistake this file already records twice.

**The one measurement still worth taking costs nothing: run a lighter PS2 title.** A 2D or
low-geometry game spends far less in EE, and if it runs acceptably then the port is fine and GTA
III is simply above the machine. That is a statement about the library, not about the port.

### PS2 IS PARKED, NOT ABANDONED - AND WHAT STAYS BEHIND IS DELIBERATE

Decision on 2026-08-28: stop here and revisit PlayStation 2 on PS5 hardware, where the CPU
arithmetic above stops being the whole story.

⚠ **THE DIAGNOSTIC PATCH IS GONE AND THE FIVE REAL ONES STAY.** `core-patches/play/0006` measured
where the frame went, the answer is written down two sections up, and a profiling hook has no
business in a shipped core - it was deleted and `/data/retroarch-profile` removed from the console.
`play` now rebuilds as `04bde0d+5`.

What remains is a Play! core that boots, renders, and does not crash, plus four findings that are
about THIS PLATFORM rather than about Play!, and which the next core to arrive will meet too:

    0001  a bundled zlib that never defines Z_HAVE_UNISTD_H - latent everywhere, fires on musl
    0002  no HOME, no XDG_*, no passwd database - path helpers need real paths, not a platform name
    0003  get_proc_address does not only come from glsm; orbis_gl_resolve_proc() exists for that
    0004  ⚠ THIS KERNEL DOES NOT DELIVER SA_SIGINFO - see the section above, it is the big one
    0005  one arena for compiled code, because a mapping per basic block runs this kernel out

⚠ **AND 0004 IS NOT A PS2 FINDING, WHICH IS WHY PARKING PS2 DOES NOT PARK IT.** The crash reporter
in orbis-compat makes the same assumption and has therefore never produced a single line in any log
this project has captured. That work is still open and still the highest-value item here.

### ⚠ OPEN, AND IT IS THE SECOND TIME THIS EXACT SYMPTOM HAS APPEARED

Left running, the PS2 core died after about a minute of steady play:

    22:07:35   38 frames in 5130 ms = 7.40 fps        <- 5-7 fps, steady, no drift
    22:07:50   [ScePthread/System] Internal Memory is running out.   x5
    22:07:51   10 frames in 15829 ms = 0.63 fps       <- ee only 8% of wall now
    22:07:51   libc++abi: terminating with uncaught exception of type
               std::__1::system_error: mutex lock failed: Out of memory

pthread_mutex_lock returned ENOMEM, std::mutex::lock threw, nobody caught it, abort(). The frame
rate did NOT decay on the way in - it was flat until the moment the pool emptied, so whatever ran
out did so in a step rather than a slope.

⚠ **THE ARENA FROM PATCH 0005 IS PROBABLY NOT THE CULPRIT, AND THE PROFILE IS WHY.**
`blockfactory` read 0.00 ms/f right up to the end, so almost no new blocks were being compiled and
the arena was not growing. It would be the obvious suspect and the measurement argues against it.
Suspect it again only with evidence.

⚠ **`[ScePthread/System] Internal Memory is running out` HAS BEEN SEEN BEFORE IN THIS PROJECT** -
on mupen64plus-next, when EnableFragmentDepthWrite exhausted the system pthread pool and took the
console down with it. Same family: something creates synchronisation objects or threads in a loop
and never gives them back. That is the third instance of this port's recurring shape - audio
ports, keyboard handles, and now whatever this is.

Finding it means counting what gets created per frame, which nobody has done. PS2 is parked, so
this is recorded rather than chased.

## ⚠ NEVER BUILD THE FRONTEND WITH A BARE `make -f Makefile.orbis`

It compiles clean, links, packages, installs and BOOTS - and it is silently crippled. Measured
2026-08-28 by shipping one to hardware:

    only RGUI, no XMB          HAVE_VULKAN=0, so XMB/Ozone/MaterialUI/widgets are all compiled out
                               - they draw through the video driver's texture path and RGUI is the
                               only menu that rasterises itself
    NO CORES AT ALL            HAVE_DYNAMIC=0, so the frontend never loads a .prx, and
                               HAVE_STATIC_DUMMY=1 links cores/dynamic_dummy.c in their place

⚠ **AND IT LOOKS EXACTLY LIKE THE CONSOLE LOST ITS CORE DIRECTORY**, which is what it was first
reported as. /data/retroarch/cores was untouched the whole time: 36 .prx, 316 .info,
libretro_directory correct. Nothing had been deleted. The BINARY could not load them.

The line CI uses, and the only one that should ever be typed by hand:

    make -f Makefile.orbis HAVE_VULKAN=1 HAVE_OPENGLES=1 \
         HAVE_STATIC_DUMMY=0 HAVE_DYNAMIC=1 \
         ORBIS_MESA_SRC=<mesa tree> -j<N> pkg

⚠ **THE PACKAGE SIZE IS THE TELL, AND IT IS ALREADY SITTING ON THE CONSOLE AS EVIDENCE.**

    ~63 MB   a correct package        every good one in /data/pkg is 63 504 384 bytes
    ~12 MB   Vulkan and Mesa missing  and /data/pkg ALREADY held retroarchv-nomesa-20260828.pkg
                                      at 12 517 376 - the control build from the close-hang work

A package that is a fifth of the expected size is not a smaller build of the same thing. Check the
byte count before uploading, and check the .elf: `llvm-nm retroarch_orbis.elf | grep -c xmb` must
be non-zero and `grep -c dynamic_dummy` must be zero.

## THE MOUSE WORKS, AND THE sceMouse ABI IS NOW ESTABLISHED

Confirmed on hardware 2026-08-28 with a Logitech receiver: cursor moves, left and right click,
wheel scrolls. `dosbox_pure` also builds and boots to a DOS prompt with the USB keyboard working.

⚠ **NONE OF THIS ABI IS IN THE SDK.** `<orbis/Mouse.h>` declares five entry points as
`void sceMouseOpen();` - an empty parameter list, which in C means "unspecified", not "none" - and
no data type at all. Both the prototypes and the struct were established on hardware by
`ps4/orbis_mouse_probe.c`, which dumps the raw bytes the call writes and watches which offsets
move. It is file-gated on `/data/retroarch-mouse-probe` and stays in the tree, like the keyboard's.

    sceMouseInit(void)                                    -> 0
    sceMouseOpen(userId, type, index, param)              -> handle 0x008b0700, FIRST TRY
    sceMouseRead(handle, data, count)                     -> EVENT COUNT
    ORBIS_SYSMODULE_MOUSE = 0x00A9, and it must be loaded first

The Open signature is the family's - scePadOpen and sceKeyboardOpen take the same four - and it
was right first time. The struct, measured over 1773 reads, all counts recorded:

    0x00  uint64 timestamp    rose ~16000 per read, i.e. microseconds
    0x08  uint32 connected    constant 1 with a receiver attached
    0x0C  uint32 buttons      0x00 x701, 0x01 x19, 0x02 x14, 0x03 x2 -> bit 0 left, bit 1 right
    0x10  int32  x            relative, +0x11..+0x28 right, -1..-0x40 left
    0x14  int32  y            relative
    0x18  int32  wheel        0x01 up x13, 0xffffffff down x9
    0x1C  int32  tilt         never moved

⚠ **THE MIDDLE BUTTON WAS NEVER PRESSED in that sample, so bit 2 is HID convention and not
measurement.** It is the one field here nobody has seen.

### ⚠⚠ THE ZERO FROM sceMouseRead MEANT THREE DIFFERENT THINGS TO ME, AND ALL THREE WERE WRONG

This one function cost four rebuilds, and every failure looked like a different bug:

    read as "0 == success"     `if (rc != 0) return;` threw away every read that CARRIED an event.
                               Handle opened, cursor drawn, nothing ever moved. The probe had only
                               logged its FIRST return - 0, because the mouse had not moved yet.
    read as "queue, ask for    Asking for 16 records and summing them made motion far coarser and
    more"                      FASTER, not smoother: the call fills the array with the same current
                               state, so extra records multiply the delta. Despite the name, this
                               is a snapshot like the keyboard's ReadState.
    read as "always has data"  Accumulating the buffer when rc == 0 re-applied the SAME delta on
                               every idle poll. A slow hand produces mostly idle polls, so slow
                               movement jumped while fast movement felt fine - which reads as a
                               sensitivity problem and is not one.

**rc is an event count. 0 means nothing happened and the buffer must be ignored.** Measured: over
a whole session it returned only ever 0 or 1, with every distinct value logged once.

### ⚠ TWO FRONTEND-SIDE TRAPS THE DRIVER ALONE COULD NOT FIX

**The menu does not ask with RETRO_DEVICE_MOUSE.** It asks with `RARCH_DEVICE_MOUSE_SCREEN`
(menu_driver.c:2016), which wants an ABSOLUTE pixel position, not a delta. Answering 0 pins the
pointer at the top-left corner while buttons and wheel work perfectly. This console only reports
deltas, so the absolute position is the driver's to keep - accumulated and clamped to the scan-out
(gfx/drivers_context/orbis_vk_ctx.c's 1920x1080). Both device ids are handled now; a core like
dosbox_pure reads the relative one.

**`video_fullscreen` was false, which silently disables the cursor.** Every GPU menu driver gates
it on the same expression - xmb.c:10382, ozone.c:12931, materialui.c -

    cursor_visible = menu_mouse_enable && (video_fullscreen || mouse_grabbed)

and this platform has no mouse-grab either, so a working driver moved the selection and drew no
pointer. ⚠ **"not fullscreen" IS NOT A STATE THIS CONSOLE CAN BE IN** - the application owns the
whole scan-out from sceVideoOutOpen and there is no compositor. Fixed in two places because one is
not enough: `config.def.h` for a fresh install, and `configuration.c` right after the config load,
because any console with an existing retroarch.cfg carries the old `false` forward and the menu
entry that would fix it sits behind Show Advanced Settings.

⚠ **AND RARCH_LOG DOES NOT REACH THE CONSOLE LOG ON THIS PORT.** A whole boot of this driver
produced no keyboard or mouse line while ps4_log output from the same run was there. That is why
"no mouse: open in the log" was misread as "the mouse did not open". **Anything meant to be
diagnosable on hardware must go through ps4_log.**

## ⚠ swanstation IS WITHHELD: IT BUILDS, IT DOES NOT RUN

Added to PS4_CORE_DROP on 2026-08-29. It builds cleanly and dies on the first game, which is the
worst thing a core list can contain - a name somebody picks over the one that works. The patches
stay in the tree; only the offering stops. `mednafen_psx_hw` remains this port's PlayStation core:
it holds 50 fps with a recompiler and its own Vulkan renderer.

### FOUR FAULTS WERE FIXED IN IT, AND ALL FOUR WERE THE SAME FAULT

⚠ **EVERY ONE WAS `__FreeBSD__` MISSING FROM A C++ TU** - this SDK's libc++ clears it on purpose
(see the section above) - AND EVERY ONE PRESENTED DIFFERENTLY:

    memory_arena.cpp    `#error Unknown platform.` and `use of undeclared identifier 'm_shmem_fd'`
    cpu_recompiler      `#error Unknown ABI.` x11, plus every register name reported undeclared
    jit_code_buffer     NO error at all - the chain fell through to `#else return false;`, so
                        JitCodeBuffer::Allocate quietly returned false and the throw surfaced as
                        `uncaught exception of type Xbyak::Error`, pointing at the wrong library
                        entirely. Xbyak is HANDED a buffer; it threw because it got none.
    memory_arena, again my own first patch only made it COMPILE and claimed in its comment that
                        "the arena backs fastmem, and swanstation runs without it". It does not:
                        bus.cpp:263 allocates the console's ENTIRE RAM through it. That comment was
                        wrong, on hardware, for a day - `ERROR: Failed to allocate memory`.

⚠ **AND THE jit_code_buffer ARM CARRIES A LATENT BUG FOR EVERY PLATFORM IT SERVES**, which is worth
lifting even though this core is parked: it maps with `PROT_READ|PROT_WRITE|PROT_EXEC` in one call
and then tests the result with `if (!m_code_ptr)`. mmap reports failure as MAP_FAILED, which is
(void*)-1, not NULL - so a refused mapping passes that test and travels on as a pointer.

### THE FIFTH, WHICH IS WHY IT IS PARKED - AND IT IS NOT A swanstation BUG

    reason: page fault (user write data, page not present)
    fault address: 0000000803f61000     rax: same     rdx: 0000000006440000
    CPU::CodeCache::AllocateFastMap  (swanstation_libretro.prx + 0x19b1b9)

`s_fast_map_pointers = std::make_unique<HostCodePointer[]>(num_slots)` - an ordinary `new[]` of
about 105 MB. make_unique would have thrown bad_alloc on failure and did not, so **the allocation
reported success and the pages are not there**. That is a level below the core: musl's malloc goes
through orbis-compat's mmap interposer, which suballocates from 128 MiB carve-outs.

⚠ **SO THIS IS WORTH CHASING FOR ITS OWN SAKE, NOT FOR swanstation'S.** An allocator that returns
addresses it has not backed would affect every title in this organisation, and it would look like
a different bug in each one. The next step is a standalone probe: allocate ~105 MB, touch every
page, and see where it stops.

## ⚠⚠ THE CRASH REPORTER WORKS. FIRST TIME EVER, 2026-08-29.

    fatal: signal 11 - SIGSEGV (bad address), si_code 2, fault address 0x803f61000
    signal - idle, close the title with the PS button

⚠ **AND MY DIAGNOSIS OF WHY IT NEVER WORKED WAS ONE THIRD RIGHT.** The evidence was sound - zero
`fatal: signal` lines in every log this project has captured - but I read one cause into it and
there were THREE, each sufficient on its own:

    1  SA_SIGINFO was Linux's       The SDK's bits/signal.h says SA_SIGINFO 4, SA_ONSTACK
                                    0x08000000. This kernel is FreeBSD's and reads
                                    SA_SIGINFO 0x0040, SA_ONSTACK 0x0001, and 4 as SA_RESETHAND.
                                    So `sa_flags = SA_SIGINFO` asked for a ONE-SHOT handler
                                    WITHOUT siginfo, and orbis_boot.cpp's first read of
                                    info->si_code faulted inside the SIGSEGV handler.
                                    ⚠ SA_ONSTACK was wrong the other way, which is why the boot log
                                    has been saying "NO alt stack" for months and blaming
                                    sigaltstack's return code rather than the flag value.
    2  RetroArch never called it    orbis::installCrashHandlers() has existed since the beginning
                                    and OpenGothic calls it. RetroArch did not. Every one of those
                                    zero lines from THIS title had the duller cause: nothing was
                                    listening. Only the log's own prefix gave it away -
                                    "crash handlers installed" appears as [opengothic], never
                                    as [retroarch].
    3  orbis_log had no sink        orbis_log.h states it: orbis_log() is "a no-op if nothing was
                                    registered" and orbis_log_fatal() writes "or NOWHERE".
                                    RetroArch logs through ps4_log, a different channel in the same
                                    library, and registered neither. So the second attempt
                                    installed the handlers correctly and STILL changed nothing
                                    observable.

⚠ **ONE SYMPTOM, THREE CAUSES, AND EACH FIX LOOKED LIKE IT HAD FAILED UNTIL THE LAST ONE LANDED.**
That is the shape to remember: absence of evidence from a channel that was never connected proves
nothing about the thing at the far end.

The fatal sink is ps4_rarch_err_v (klog + UDP), not ps4_rarch_log_v (UDP only), for the reason
orbis_log.h separates them: a datagram from a process the kernel is about to kill may never leave
the machine.

**Black screen after a crash is POLICY, not a fault.** orbis_fatal_action idles rather than exits -
a title that exits gets the console's error dialog and nothing else, one that idles has already
written its log. Close it with the PS button.

### ⚠ AND IT IMMEDIATELY PAID FOR ITSELF: si_code 2 IS SEGV_ACCERR

The allocator question (see the list) now has a fact it did not have. `si_code 2` is SEGV_ACCERR -
**mapped, but not permitted** - not SEGV_MAPERR, which would be "no such page". The console's own
klog called the same fault "page not present" and that phrasing sent me looking for missing pages.
The memory at 0x803f61000 EXISTS; writing to it is refused. That points at a PROT_NONE reservation
that was never promoted to read-write, rather than at an allocator handing back unbacked address
space. Whoever picks up that item should start there.

## 2026-08-29, evening — WHAT THE CRASH REPORTER COST AND WHAT IT BOUGHT

### The tools, which are the lasting part

    fatal: signal N          orbis-compat, working for the first time. See the section above for
                             the three separate causes of its silence.
    context dump             The kernel writes NO dump once the handler survives the signal, so the
                             fault address used to arrive with nothing to measure it against. The
                             handler now prints the raw context words.
    module base at load      libretro-common/dynamic/dylib.c prints each core's retro_run address.
                             Subtract its address in the .elf (llvm-nm) and you have the base:
                                 0x800ae4410 - 0x274410 = 0x800870000, confirmed against a kernel
                                 dump from the same core earlier the same day.
    ps4/orbis_alloc_probe.c  file-gated on /data/retroarch-alloc-probe. Allocates 1..256 MiB,
                             touches every page, reads it back.

⚠ **THE mcontext LAYOUT IS FreeBSD'S AND THE SDK'S HEADERS DESCRIBE musl'S.** Do not look up
mc_rip; measure it. From a real fault, with the context dumped as 64 words:

    ctx[24] = 0x001b00130000000c   -> mc_trapno 0xc (page fault), mc_fs 0x13, mc_gs 0x1b
                                      which puts mcontext at ctx[8]
    ctx[25] = mc_addr    = 0          matched the reported fault address
    ctx[28] = mc_rip     = 0x249303801
    ctx[30] = mc_rflags  = 0x10206    a plausible flags word
    ctx[31] = mc_rsp     = 0x7eeffa9c8

⚠ **AND backtrace() IS USELESS FROM A SIGNAL HANDLER HERE - DO NOT TRY IT AGAIN.** It returns
exactly one frame, ps4SignalAction itself: the kernel switches context to deliver the signal and
the frame-pointer chain does not survive. The context dump is the instrument; backtrace is not.

### ⚠ THE ALLOCATOR IS INNOCENT. That entry on the old list was wrong.

The probe passed every size - 1, 8, 32, 64, 105, 128, 160, 256 MiB - writing and reading back every
page with zero mismatches. malloc returns 0x2xxxxxxxx; the crash was at 0x803f61000, which is the
MODULE. Two different address spaces, and "page not present" in the console's klog is what sent a
day of work at the wrong one. **si_code told the truth and the klog did not.**

### swanstation: six faults found, six fixed, still does not run

Kept as PS4_CORE_DROP. The patches stay because five of the six are platform findings:

    0001  arena implemented (bus.cpp allocates the console's WHOLE RAM through it, not fastmem)
    0002  SysV ABI named
    0003  JIT buffer: platform named, RW-then-mprotect, and MAP_FAILED is -1 not NULL
    0004  ⚠ .bss OF A .prx IS NOT ALL WRITABLE. This module declares 50 MB of .bss against 259 KiB
          of file data; s_fast_map sits ~49 MB in and needed an explicit mprotect. Any core with a
          large .bss will meet this.
    0005  ⚠ A 4096-BYTE GUARD PAGE ON A 16 KiB-PAGE CONSOLE. mprotect rounds the guard UP to a
          granule, so the address the recompiler is handed lands 12 KiB INSIDE the guard. The 4096
          assumption is everywhere and harmless until a length is reused as an OFFSET.
    0006  fastmem teardown: InitializeFastmem returns early without calling UpdateFastmemViews,
          and BOTH callers discard the failure - `if (... && !InitializeFastmem()) { }`.

⚠ **0006 WAS WRITTEN AGAINST A MISREADING AND IS KEPT ONLY BECAUSE THE BUG IS REAL.** mc_rip was
outside every module, so it was called recompiler-generated code. The log says the core was in
GPU_HW_Vulkan initialisation at the time. The fix is correct; it is not this crash's cause, and
what is under 0x249303801 has not been established.

**Where it stands: a null dereference during Vulkan renderer setup, cause unknown.** The next step
is to find out what that address belongs to before changing anything - which is the step that was
skipped last time.

## ⚠ NEXT: CORES. tic80 FIRST, THEN THE CHEAP ONES, THEN ScummVM

The maintainer's direction, 2026-08-29: stop chasing platform faults for a while and widen the core
list. Take them in this order.

### 1. tic80 - one line, and it proves the mechanism

    tic80  CLONE  fatal: Nie znaleziono zdalnej gałęzi master

The recipe says `master`. Upstream renamed it: `git ls-remote https://github.com/nesbox/TIC-80.git`
lists clay, custom-menu, lovebyte, luajit, **main** - and no master. Nothing is wrong with the core.

⚠ **ps4/core-recipe-extra IS SEARCHED BEFORE THE RECIPE, SO IT CAN CORRECT A STALE LINE AND NOT ONLY
ADD A MISSING CORE.** That was built for dosbox_pure and has not yet been used to override
anything - tic80 is the first test of that half. Copy the recipe's line, change master to main:

    tic80 libretro-tic80 https://github.com/nesbox/TIC-80.git main YES CMAKE Makefile builddir \
      -DBUILD_PLAYER=OFF -DBUILD_SOKOL=OFF -DBUILD_SDL=OFF -DBUILD_DEMO_CARTS=OFF -DBUILD_LIBRETRO=ON

If it then builds, the sweep picks it up automatically: --all and shard-cores.sh both draw from
GENERIC + CMAKE across the recipe AND this file.

### 2. The rest of the cheap ones, in this order

    trident      LINK, 349 objects, undefined SDL_iconv_string_REAL. Its bundled SDL2 was
                 configured without iconv. Probably one -D; look at what the recipe passes.
    dirksimple   LINK, 71 objects, undefined luaopen_utf8. Its bundled lua omits the utf8 library -
                 either enable it or stub the one call.
    melondsds    LINK, 175 objects, duplicate symbol adler32_z. ⚠ HARNESS-SIDE, NOT CORE-SIDE:
                 FetchContent builds zlib TWICE (zlib.dir and zlibstatic.dir) and build-cores.sh
                 collects both object sets. Deciding which .dir wins would likely fix other
                 FetchContent cores too, so this one is worth more than one core.
    flycast      COMPILE, find_package(OpenGL) wants GLX. Play! hit the same wall and the answer
                 was core_cmake_flags() with -DUSE_GLES=ON. Look for flycast's equivalent switch
                 before assuming it has none.
    pcsx_rearmed LINK on lightrec_init_mmap - executable memory, already solved twice in this port
                 (ps4/orbis_exec_mem.c, beetle-psx's ps4/orbis_lightrec_mem.c).

### 3. ScummVM - wanted, and it needs a configure step this harness does not do

    scummvm  COMPILE  no objects; single-shot link?
    scummvm.log: Makefile:122: *** You need to run ./configure before you can run make.

⚠ **THIS IS A THIRD BUILD SHAPE, NOT A BROKEN CORE.** build-cores.sh knows two: GENERIC runs make
directly, CMAKE configures then builds. ScummVM is autotools-flavoured - it wants ./configure first,
and the recipe's subdir is backends/platform/libretro/build. Nothing in the harness runs a configure
script, so make stops on its own error message before compiling a single file.

The work is in build-cores.sh, and it is the same shape as the CMAKE arm added on 2026-08-28: a
build type that needs a preparation step, given the cross-compilation flags the arrays already
hold. ScummVM's configure takes --host= and honours CC/CXX, so $CC_ORBIS and $CXX_ORBIS should
reach it the same way they reach make.

⚠ **AND CHECK WHAT ScummVM WANTS BEFORE BUILDING IT**: it is a large tree with optional
dependencies, and the useful question is which subset configures at all here. A core that links is
worth more than a complete one that does not.

### THE LIST, as of 2026-08-29 after v0.1.6 and the crash-reporter work

⚠ **THE TWO ITEMS THAT USED TO HEAD THIS LIST ARE GONE.** SA_SIGINFO is fixed and working. The
allocator was never guilty - the probe cleared it, and that entry was mine and wrong.

**1. sigaltstack - diagnosed and repaired in the tree, NOT yet seen on hardware.** The SDK
declares `stack_t` in Linux's field order (`ss_size` and `ss_flags` exchanged against FreeBSD's),
so the kernel was reading `ss_size = 0` and `ss_flags = 0x10000` and answering EINVAL. Same header
and same provenance as the SA_* defect, one field over. `orbis-compat/include/signal.h` now carries
a layout-translating shim and `SS_DISABLE`; the boot line prints errno, a raw readback and a
legacy-layout probe so that ONE reboot settles it either way. See the 2026-08-30 section at the end
of this file for what each outcome will say. ⚠ Even when it works it covers the MAIN THREAD ONLY -
FreeBSD's alternate stack is per thread.

**2. Turn the context dump into one named line.** The mcontext layout was measured (mcontext starts
at ctx[8]; mc_rip is ctx[28] - see the section above). Reading it out by name would replace 16 log
lines with `fault at <rip>, module base <base>` and make every future crash a one-liner. Cheap, and
the measurement is already done.

**Input, small and known:**

    mouse middle button   bit 2 is convention, not measurement - it never appeared in the sample.
    keyboard character    input_keyboard_event gets 0, so text fields take nothing.
                          sceKeyboardGetKey2Char has a real signature; one parameter is
                          `bool unknown`, so it is a probe rather than a guess.

**Cores, cheapest first:**

    tic80          the recipe names a branch upstream renamed. ⚠ ONE LINE NOW:
                   ps4/core-recipe-extra is searched BEFORE the recipe and can correct stale entries.
    trident        SDL2 without iconv - SDL_iconv_string_REAL. Likely one -D.
    melondsds      FetchContent builds zlib twice; both object sets are collected -> duplicate
                   adler32_z. Harness-side, would help other FetchContent cores.
    dirksimple     bundled lua omits luaopen_utf8.
    flycast        find_package(OpenGL) wants GLX. Play! answered this with -DUSE_GLES=ON.
    pcsx_rearmed   LINK on lightrec_init_mmap - executable memory, solved twice already.

**Withheld on purpose:**

    swanstation    six faults fixed, still crashes - a null during GPU_HW_Vulkan initialisation.
                   ⚠ NEXT STEP IS TO ESTABLISH WHAT 0x249303801 BELONGS TO BEFORE CHANGING
                   ANYTHING. Last time that step was skipped and the patch missed.
    play           works at 4-12 fps. Parked for PS5.
    mednafen_psx   upstream Beetle with none of this port's work.

**Older, still open:**

    PrBoom Load State        one log line of thinkercap.next in the core settles it.
    GLideN64 clipping        gl_Position.z /= 8.0 never scaled back; cheap via the enableClipping
                             knob in /data/retroarch-gliden64.
    nestopia                 loads, runs, exits cleanly, renders a green screen.
    GL_TEXTURE_EXTERNAL_OES  GL_INVALID_ENUM in glFramebufferTexture2D, pre-existing.
    coverage                 61 of 104 published cores never run on hardware.
    pthread pool             "Internal Memory is running out" seen twice. Third instance of this
                             port's recurring shape after audio ports and keyboard handles.

**Fixed and pushed, waiting for the next release:**

    mesa-ps4 5db2def   -Dxmlconfig=disabled: RADV no longer opens the build machine's
                       DATADIR/drirc.d. Verified - `prefix/share` gone from the archive.
    orbis-compat       SA_* values, and the context dump.
    RetroArch          crash handlers wired up, module base logged at load.

---

## 2026-08-30 — the MAP_ANON myth, and what believing a header cost

A claim repeated in seven places across this port was false, and today it cost a whole task: an
agent was sent to write a `sys/mman.h` shim for a problem that does not exist.

**What was believed.** "The SDK's musl `sys/mman.h` says `MAP_ANON` is `0x0020`, and orbis-compat
sits ahead of it in the include path and corrects it to FreeBSD's `0x1000`." It appeared in
`ps4/orbis_exec_mem.c`, `ps4/build-cores.sh`, two applied core patches, `orbis-compat/include/sys/ioctl.h`,
`beetle-psx-libretro/ps4/orbis_lightrec_mem.c` and three places in this file. Every one of them
reached the RIGHT conclusion - large mappings and executable pages come from direct memory here -
for a reason that is not true.

**What is true, checked two independent ways.**

    $TOOLCHAIN/include/sys/mman.h:26      #define MAP_ANON 0x20     <- musl's Linux value
    $TOOLCHAIN/include/sys/mman.h:112     #include <bits/mman.h>    <- and then this
    $TOOLCHAIN/include/bits/mman.h:59-62  #undef MAP_ANON / #define MAP_ANON 0x1000

`bits/mman.h` corrects `MAP_SHARED`, `MAP_PRIVATE`, `MAP_FIXED` and the whole `PROT_*` set the same
way. **The SDK was already right.** And asked of the compiler under this port's own include order:

    clang --target=x86_64-pc-freebsd12-elf -nostdinc \
          -isystem $ORBIS_COMPAT/include -isystem $TOOLCHAIN/include -isystem $(clang -print-resource-dir)/include
    -> MAP_ANON 0x1000, MAP_PRIVATE|MAP_ANON 0x1002, PROT_READ|PROT_WRITE 3
    -> `int v = MAP_ANON;` lands in .data as 00 10 00 00

⚠ **orbis-compat has no `sys/mman.h` and no `bits/mman.h` at all** - its whole `include/bits/`
is one file, `alltypes.h`. Every comment describing the overlay as correcting these constants
described a mechanism that has never existed. The overlay's real corrections are declarations
(four pthread types, `sa_sigaction`'s macro) made by defining musl's `__DEFINED_<name>` guards
early, plus headers the SDK omits - which is why the include order still matters, just not for this.

**The disproof was sitting in the tree the whole time.** `orbis-compat/src/orbis_mmap.cpp:54` has
carried `static_assert(OwnedProt==3 && OwnedFlags==0x1002, ...)` since it was written. If `MAP_ANON`
were `0x20` that file could not compile, and it compiles in every build.

**The real reason large mappings need direct memory** - the conclusion the wrong premise was
propping up - is the **pool**, not a constant. Anonymous memory is FLEXIBLE memory: a separate,
much smaller per-process budget that measured **427008 KiB, about 417 MiB**, at the first
instruction of boot, shared with everything musl's malloc has ever grown into, while direct memory
had 4601856 KiB idle at the same instant. Executable pages are additionally refused at MAP time
(`sceKernelMapDirectMemory` with READ|EXECUTE returns `0x8002000d`, EACCES) and have to be promoted
with `sceKernelMprotect` afterwards.

⚠ **The lesson, and it is the one this file already gave once.** "Ask the compiler for constants
(`clang -dM`), not a header you found with grep" was written here on 2026-08-23 and then quietly
violated by every comment that quoted `sys/mman.h` without reading its last line. A header that
ends in `#include <bits/...>` has not finished speaking. All seven sites now say what is actually
true; nothing about the code around them changed, because the code was right.

---

## 2026-08-30, later — sigaltstack: the second half of the SA_* defect, one field over

`sigaltstack` has returned -1 on this console since the crash reporter was written, and the boot
line has blamed it for months. Fixing `SA_ONSTACK` (Linux's `0x08000000` where this kernel wants
FreeBSD's `0x0001`) did not change it, which ruled out the obvious explanation and is what sent
this search somewhere else. It did not have far to go: **the same header, the same provenance, the
next field along.**

    $TOOLCHAIN/include/bits/signal.h:91    struct sigaltstack { void *ss_sp; int ss_flags; size_t ss_size; };
    oracles/freebsd9/sys_sys_signal.h:358  typedef struct sigaltstack { char *ss_sp; __size_t ss_size; int ss_flags; } stack_t;

Twenty-four bytes either way, **`ss_size` and `ss_flags` exchanged**. `bits/signal.h` is musl's
Linux x86_64 copy verbatim - it is the file that also carries `SA_ONSTACK 0x08000000`,
`SA_SIGINFO 4`, a Linux `struct sigcontext` and a musl `mcontext_t`, every one of which this port
has already had to correct or work around. The struct was simply the piece nobody had looked at.

So `installCrashHandlers` filling in the obvious `ss_sp = buffer, ss_size = 65536, ss_flags = 0`
was handing the kernel:

    ss_size  = 0            (read out of the SDK's ss_flags plus its padding)
    ss_flags = 0x00010000   (the low half of the SDK's ss_size)

and FreeBSD's `kern_sigaltstack` rejects any bit outside `SS_DISABLE` with **EINVAL (22)** in a test
that runs *before* it ever looks at the size - so the errno is EINVAL, not ENOMEM. ⚠ The struct
layout, `SS_*`, `SA_*` and the errno numbers are **measured** from the FreeBSD 9 oracles in
`~/src/unemups4/oracles/freebsd9/`; the *order of those two checks* inside `kern_sig.c` is
**inferred** - that file is not among the oracles this port holds a copy of.

`SS_DISABLE` is wrong the same way: the SDK says `2` (Linux), FreeBSD tests for `0x0004`.
`SS_ONSTACK` is `1` on both. `SIGSTKSZ` is Linux's 8192 against FreeBSD's 34816 and is
**deliberately left alone** - the kernel enforces only `MINSIGSTKSZ`, which the SDK happens to have
right at 2048, so correcting it would only grow every `char buf[SIGSTKSZ]` in every consumer.

**Two things were verified along the way and are worth keeping.**

    libc.a's sigaltstack.lo has an EMPTY .text and no symbols at all - the SDK compiles a
    sigaltstack.c that produces no code. The symbol comes from libkernel.so, so there is no
    libc wrapper in between and nothing else to blame for the return value.

    errno IS libkernel's errno, not a musl-side copy. libc.a's __errno_location.lo is a single
    `jmp __error`, and libkernel.so exports __error. Reading errno after a libkernel-provided
    call is therefore meaningful - which is not something to assume on this platform.

**The repair is a translating shim, not a redeclared struct.** `stack_t` is typedef'd by the SDK
header and libc++ and every prebuilt archive were compiled against it, so the type stays exactly as
it is and only the twenty-four bytes that cross the syscall boundary are reordered
(`orbis-compat/include/signal.h`). It is a function-**like** macro so that `struct sigaltstack` as a
type name still means what it says; an object-like macro would have rewritten the tag too.
`test/sizes.c` now pins both layouts, and `test/declarations.c` calls it the way a caller does.

⚠ **THE CONSOLE HAS NOT CONFIRMED THIS YET, AND THE BOOT LINE IS BUILT SO THAT ONE REBOOT DOES.**
Three lines now, and they are diagnostic in every outcome:

    boot: crash handlers installed (... sigaltstack rc=<rc> errno=<n> <FreeBSD name> ...)
    boot: sigaltstack readback rc=.. errno=.. - raw <w0> <w1> <w2> - as FreeBSD stack_t: ... - <verdict>
    boot: sigaltstack legacy-layout probe ... rc=.. errno=..      (only printed if the first failed)

The readback is the evidence and it answers the layout question **whether or not the install
worked**, because `sigaltstack(NULL,&oss)` makes the kernel write twenty-four bytes and they are
printed raw, decoded by nobody:

    installed, FreeBSD order   w0 = the buffer   w1 = 0000000000010000   w2 = 0 or 1 (SS_ONSTACK)
    refused,   FreeBSD order   w0 = 0            w1 = 0                  w2 = 4 (SS_DISABLE)
    refused,   Linux order     w0 = 0            w1 = 2                  w2 = 0

The position of the one non-zero word names the layout even when nothing is installed. And the
legacy probe re-offers the same buffer in the SDK's byte order when the corrected one fails, so two
errnos side by side separate "the layout was the problem" from "this kernel refuses the call however
it is asked" - `ENOSYS (78)` in both would be the end of the road.

⚠ **AND AN ALTERNATE STACK IS PER THREAD HERE.** FreeBSD keeps it in `td_sigstk`; `sigaction`'s
disposition is per process. So even when this succeeds it covers the **main thread only**, and a
core overflowing on a worker thread still dies quietly. The next step is one more 64 KiB and one
more `sigaltstack` inside the thread start routine - deliberately not written yet, because nothing
has observed the working case.

The buffer is heap, not a static array, on purpose: a large static array on this console is a
page-permission question of its own, and there is no reason to put the one allocation that has to
work during a crash into the one region that needs promoting.

### dirksimple - built, and parked for want of the right video

The core builds and links (`d5d75f9+1`, one patch: upstream's CMakeLists omits `lutf8lib.c` from its
bundled Lua source list, so `luaopen_utf8` is undefined - invisible upstream because their output is
a `.so`). It has never been run, and will not be, because the content this port has is the wrong
shape.

DirkSimple is not an emulator. The game logic is reimplemented in Lua and ships inside the core
(`data/games/lair`, `data/games/cliff`); the only thing it needs from outside is an Ogg Theora
encode of the laserdisc footage, named `lair.ogv` or `cliff.ogv` - the filename is what selects the
script.

⚠ **AND IT MUST BE ONE CONTINUOUS VIDEO.** `game.lua` seeks by absolute time into a single stream:

    local start_time = scene_manager.current_sequence.start_time
    DirkSimple.start_clip(start_time)

223 KB of such markers, all calibrated against the original uncut footage. The CD-ROM release
(`DL_CDROM_V31.ISO`) stores the game as ~200 per-scene MPGs - `S35.MPG`, `S35B.MPG`, `S35D1.MPG` and
so on. Concatenating them would produce a video in which not one marker lands where the script
expects: the game would run and play the wrong scenes, which is worse than not running. Upstream's
README names the Digital Leisure DVD as the source, and that is a single continuous file.

Maintainer's call, 2026-08-30: not worth chasing for one game that is playable by other means. The
core stays built and unpublished until someone has the DVD encode.

## 2026-08-31, overnight - four results, and two briefs that were wrong

### A second eboot that speaks desktop OpenGL

`glcaps/eboot.bin` had been built months ago and never run. Run on hardware 2026-08-30:

    CEILING: OpenGL ES 3.1; desktop GL context CREATED
    OpenGL 4.6 / 4.5   eglCreateContext refused (0x3009)
    OpenGL 3.3         OK -> 3.3 (Core Profile) | GLSL 3.30 | 226 extensions
    OpenGL 2.1         OK -> 3.3 (Compatibility Profile) | 304 extensions
    OpenGL ES 3.1      OK | 157 extensions

⚠ **A laptop probe against a drm-shim had said 4.6.** Hardware says 3.3. The shim executes no GPU
work and is optimistic by a step - it also claimed GLES 3.2 where the console gives 3.1. Anything
measured on the shim is an upper bound.

`RTRG00001` / *RetroArchG* is now a separate product built from the same tree by flags alone
(`HAVE_OPENGL_CORE=1` instead of `HAVE_OPENGLES=1`), living in `/data/retroarch-glcore/` so the two
cannot fight over `retroarch.cfg`'s `libretro_directory`. Mesa needed no changes at all: `-Dopengl=true`
was already set and all 2312 desktop entry-point stubs were already inside the `libgallium-*.a` the
eboot links - they were simply unreachable by name.

⚠ **Two things in the brief were wrong and the work disproved them.** The frontend link gap is *zero*
symbols, not one: `gl2.c`'s `glGetTexImage` sits behind `#if defined(READ_RAW_GL_FRAME_TEST)`, which
nothing defines. And the core-side thunks belong to mupen64plus_next, not melonDS DS - six of them
(`glFinish`, `glGetFloatv`, `glGetString`, `glPolygonMode`, `glClearDepth`, `glDepthRange`), because
glsym covers GL 2.0+ and the GL 1.x entry points become plain link-time symbols on the desktop path.
melonDS DS needs none; it resolves its whole GL surface through `rglgen_resolve_symbols`.

⚠ **And a harness defect that would have silently eaten the thunks**: `liborbis-core-support.a` was
rebuilt on `[[ ! -f ]]` alone, so every cached core would have linked the old archive. It now rebuilds
when its sources change.

**Untested, and this is the honest risk**: a desktop context has never been *presented* here. glcaps
created one, read two strings and destroyed it without drawing a frame. Everything past
`eglMakeCurrent` is unexercised - zink's core-profile path, `gl3.c`'s SPIR-V filter chain over
`GL_ARB_gl_spirv`, kopper's swapchain under a non-ES context. GLideN64's 60.41 fps was measured on
the GLES *compatibility* path; the glcore build of that core is a different renderer and its frame
rate is an open question, not a carried-over result.

### An alternate signal stack on every thread, not just the main one

`sigaltstack` on the main thread was fixed and confirmed on hardware the same night (the SDK's
`struct sigaltstack` has `ss_size` and `ss_flags` swapped against FreeBSD's). FreeBSD keeps the stack
in `td_sigstk`, **per thread**, so worker threads were still dying silently.

Now every thread gets one, and ⚠ **including a core's own threads, which was not expected**:
`ps4/build-cores.sh` puts `-lorbis-compat` ahead of `-lc` on every core's link line, so a core that
calls `pthread_create` has an undefined reference and pulls the interposer in. Measured: **63 of 119
built cores define `T pthread_create` themselves**, flycast and melondsds among them. Each core gets
its own instance with its own counters and an unregistered log sink, so its installs are silent - but
they work, because what they set is the kernel's `td_sigstk` and the handler that runs on it is the
frontend's. `sigaction`'s disposition is per process; that cross-module split is why this works.

Cost: **zero new memory.** The 64 KiB is an array in the thread trampoline's own frame, carved out of
the 2048 KiB the interposer already reserves. Skipped below a 256 KiB stack.

The first worker thread now prints what it *inherited*, which settles a question the oracles cannot:
whether this kernel copies `td_sigstk` on thread creation. If `ss_sp` reads back as the main thread's
buffer, threads have been sharing one alternate stack all along.

### A crash dump was going out over UDP alone

    real SIGSEGV, 2026-08-30 23:27:04
      ps4-udp-*.log    4 lines of "fatal:"
      ps4-klog-*.log   0 lines, in a 14252-line file

`ps4_log()` stopped being a klog channel when orbis-compat introduced `klogWanted()`
(`s_frameKlog || orbis_netlog_ready()==0`, false on any console whose netlog came up), so both
branches of `ps4_log_emit` had been writing to the same channel. `include/ps4_app.h` still documented
`ps4_log` as "both channels", and this port's code was written against that stale comment. The dump
was legible that night only because `ps4_idle_forever()` held the process open - which is the exact
scenario the two-channel split exists for. Fatal lines now go through `ps4_log_fatal()`, which writes
`sceKernelDebugOutText` first and unconditionally. Bounded cost: 9 `[ERROR]` lines in one measured
run, 0 in another, plus 4-6 of dump. `[WARN]` stays on UDP - 55 lines a run would cost ~0.6 s.

### swanstation's 0x249303801 - it is the heap, and the answer was already in the log

⚠ **The instruction to identify the address before patching was right, and the identification needed
no new instrument.** When the shell killed the hung process 42 seconds after the fault, the kernel
wrote its own `# dynamic libraries:` dump - every image with exact extents - and nobody had read it.

    swanstation .prx   text 0x800870000 : 0x800e04000   data ... : 0x804068000
    eboot.bin          text 0x000400000 : 0x003bac000
    0x249303801        in NO image at all

What lives at `0x2xxxxxxxx` is stated by the frontend's own banner: the direct-memory carve-outs
`orbis_mmap.cpp` serves musl's anonymous mappings from. The arithmetic pins it 127 MiB into carve-out
0 - the heap. And `mc_err = 4` decodes as user-mode, **read**, page-not-present, with the
instruction-fetch bit **clear** and `mc_addr` = 0: the fetch at `0x249303801` succeeded. **Control had
already been transferred into a heap allocation and was running there** - a wild indirect branch
through a stale function pointer or vtable, not a call through null.

⚠ **And the recorded symptom is no longer trustworthy.** That run shows `ran 18 global constructor(s)`
**three times in 1.5 seconds**, with `vulkan: destroying the context` between the loads. Re-running 18
constructors over a live image re-initialises globals underneath pointers other globals still hold -
a mechanical route to exactly this fault. Both defects were fixed on 2026-08-30. Reproduce before
analysing further; the core builds clean as `7f69c19+6` against today's tree.

## 2026-08-31 - what caps this console at GL 3.3 is one bit, and it is ours

Measured by the extended `glcaps` probe on hardware:

    GL 4.0  MISSING 1 of 11 extensions:
            GL_ARB_tessellation_shader
    GL 4.1  all 9 requirements present
    GL 4.2  all 9 requirements present
    GL 4.3  all 16 requirements present
    GL 4.4  all 6 requirements present
    GL 4.5  all 7 requirements present
    GL 4.6  all 9 requirements present
    FIRST BLOCKER: GL 4.0. Everything below it is satisfied.

⚠ **Every rung above 4.0 is already complete.** Nothing else is missing anywhere in the ladder, so if
tessellation worked this port would go from 3.3 to **4.6** in one step - and from ES 3.1 to ES 3.2,
because tessellation is an ES 3.2 requirement too. One bit holds both ceilings.

The Vulkan side agrees and names it: `tessellationShader=0`, with `geometryShader=1`,
`shaderFloat64=1`, `multiDrawIndirect=1`, `fragmentStoresAndAtomics=1` and the rest of the 33
features the probe reads all set. 197 device extensions, including `VK_EXT_transform_feedback`,
`VK_KHR_draw_indirect_count` and `VK_KHR_maintenance2`.

⚠ **AND THE BIT IS THIS PORT'S OWN DECISION, NOT GFX7's LIMIT.** `src/amd/vulkan/radv_physical_device.c`:

    .tessellationShader = orbis_tessellation_available(),

which defaults to false and returns true only for `ORBIS_NO_TESS=0`. The comment above it says no run
has ever survived a pipeline that uses the stage on this silicon. So the ceiling is a documented
retreat from a fault nobody has since gone back to.

**What this makes worth doing**: find why the tessellation stage faults here. GFX7 has the hardware
and radeonsi drives it, so the suspicion is RADV's configuration under this kernel - LDS layout, the
HS/LS stage registers, or an initialisation Sony's own driver does differently. If that breaks, the
port gains GL 4.6 and ES 3.2 together.

⚠ **What is NOT worth doing is flipping the bit alone.** Advertising tessellation without fixing the
stage buys a version number and a fault in any pipeline that uses it - worse than the honest 3.3.

### A hypothesis this measurement killed

`shaderFloat64` was the coordinator's suspicion, on the strength of `zink_screen.c:1258` referencing
it. Wrong, and the source says so plainly: `zink_screen.c:877` sets `caps->doubles = true`
unconditionally, and `st_extensions.c:1699` derives `ARB_gpu_shader_fp64` from that cap - so the
Vulkan bit cannot gate it. `shaderFloat64` gates only the subgroup caps, which no GL 4.x rung
requires. The probe reports `shaderFloat64=1` on this hardware anyway.

## 2026-08-31 - swanstation runs. The bug was a header arm that does not exist here

**30-90 fps at 2x upscale with 2xMLAA**, measured by the maintainer. This core had been withheld from
every release (`PS4_CORE_DROP`) since the first sweep.

The whole failure was one `#if`. `src/common/page_fault_handler.cpp` defines `USE_SIGSEGV` for
`__linux__ / __ANDROID__ / __APPLE__ / __FreeBSD__` and nothing else. ⚠ **`__FreeBSD__` is undefined
in every C++ translation unit on this SDK** - libc++'s `__config` does `#undef __FreeBSD__`, which
this port discovered while building trident - and no arm names `__ORBIS__`. So `InstallHandler()`
reached `#else return false;` with no way to succeed, and from there:

    InitializeFastmem() fails
      -> Bus::UpdateFastmemViews() never runs
      -> m_fastmem_lut (a 16 MiB calloc) is NEVER ALLOCATED - not a failed allocation, never asked for
      -> Bus::GetFastmemBase() returns nullptr
      -> g_state.fastmem_base stays null
      -> but g_settings.cpu_fastmem_mode is STILL LUT, so the recompiler keeps emitting
         EmitLoadGuestRAMFastmem, whose RBX comes from that null base

Which is exactly what the crash instrument caught in the generated code:

    movl  $0x93C, %edi          ; guest address
    shrl  $12, %edi             ; page index -> 0
    movq  (%rbx,%rdi,8), %rdi   ; rip here, rbx = 0, fault address 0

⚠ **And patch 0006 had fixed the wrong variable.** It cleared `Bus`'s copy of the mode and re-derived
`g_state.fastmem_base` from it - setting to null a value that was already null - while the emitter
asks `g_settings`. That is why the identical crash survived it, and why the coordinator's note at the
time ("0006 fixes a real bug but not that crash") was right for the wrong reason.

Patch 0007 forces `cpu_fastmem_mode = Disabled` in `FixIncompatibleSettings`, two lines below the
`#ifndef WITH_MMAP_FASTMEM` downgrade upstream already does there, so it runs on the initial load and
on every core-option change, before the code cache is (re)initialised.

**Worth doing later, not needed now**: an `__ORBIS__` arm in `page_fault_handler.cpp`. `SA_SIGINFO`
now works here, and this handler only rewrites code and returns - it never modifies the ucontext,
which this kernel ignores anyway. Fastmem is plausibly recoverable as a performance project.

⚠ **swanstation is still in `PS4_CORE_DROP` and will not ship until that is changed.**

## 2026-08-31 - the internal-memory leak: what is established, and four coefficients that were not

⚠ **STATE: NOT FIXED. The spending is located, not named.** Read this whole section before touching it;
most of a day went into hypotheses that fit the data and were wrong, and they are recorded here so
nobody re-derives them.

### The symptom

`[ScePthread/System] Internal Memory is running out.` - thousands of lines, and on two occasions the
console-wide UI froze and needed a cold reboot. Seen four times before today (HANDOFF records audio
ports and keyboard handles); this is the first time it was measured.

### The meter

`sceKernelInternalMemoryGetAvailableSize` reads **14,013,728 bytes free at startup** and 96 bytes at
the failure. It is exported by the SDK's `libkernel.so` stub, undeclared in any header, so it is
called through a **weak** symbol with a dual-calling-convention guard (`ps4/orbis_watchdog.c`).
⚠ `[ScePthread/System]` is the logging *category*, not the owner: mutex, condvar, attr and thread
creation all keep working while this pool empties. The name led two rounds astray.

### What is established, all on hardware

- **9,280 bytes per frame, to the byte**, for ten consecutive 120-frame reports, stable across three
  different builds of the instrument. The same 9,280 also appears **once** at startup in every run
  this port has ever captured.
- ⚠ **It switches on.** Flat for the first several hundred frames, then never recovers. In `gl4.log`
  it began at the **sixth RADV device instance** - RetroArch tears the device down and recreates it
  on content load and on menu-driven driver reinit. Never observed before that point.
- **It is not page-granular** in either size: 9280/4096 = 2.27, 9280/16384 = 0.566. So it is heap
  bookkeeping, not address-space bookkeeping - which retires the whole `sceKernelInternalGetMapStatistics`
  family the export names had suggested.
- **The frame ledger locates it above this driver.** Ten marks, each segment bounded by exactly one
  call, time booked beside bytes:

      present:exit..vkAcquire (frontend between frames)   7190 B/frame  in 9764 us = 736 B/ms
      vkAcquire..vkQueueSubmit2 (DRAWS + ZINK + RADV REC) 1883 B/frame  in 9339 us = 201 B/ms
      everything inside our winsys                        0-73 B/frame            = 0-8 B/ms

  Every segment spans a comparable 8.0-9.8 ms and they differ ~100x in cost, so **the cost is not
  time-based** and 9,073 of the 9,280 bytes are above the winsys.
- **It is glcore-only.** RetroArch's Vulkan driver draws the same menu with a reset-per-frame buffer
  chain and pools that are never freed; its per-frame churn is ~0 and it does not leak on the same
  hardware. The glcore build draws everything through zink.

### ⚠ Four coefficients that fit the data and were all artifacts

Each was derived by dividing a window total by a correlated aggregate, each held for several runs:

    146 bytes per syncobj timeout   - the split counter halved the divisor and the coefficient
     73 bytes per syncobj poll      - poll rate fell 2.7x, per-frame loss did not move
    ~10 KB per frame                - two windows, same present count, 6.6x different loss
    73 bytes per clock_gettime      - 96 bytes over 2000 calls in isolation: noise

**The lesson is not a better divisor. It is to stop dividing** - which is what the ledger does.

### Measured at zero, on hardware, in isolation (2000 calls each)

`clock_gettime` MONOTONIC and REALTIME, `pthread_mutex` lock+unlock, `sceKernelUsleep(1)`,
`pthread_cond_timedwait` already-expired, **`sceVideoOutGetFlipStatus`**. ⚠ A clean sheet under a
tight loop from one thread is not innocence under the driver's conditions - but combined with the
ledger's B/ms it is now enough to exclude our own paths.

### Also established, and worth its own line

- **zink polls fences 60-162 times a frame** against **one `vkWaitForFences` per frame**. Those polls
  are driver-internal and their caller has never been named. The deadline they pass is
  microseconds in the past, not decades, so it is a caller with no patience - not a clock-base bug.
  `orbis-drm` now counts polls and real timeouts apart.
- **`sceKernelInternalHeapPrintBacktraceWithModuleInfo` produced nothing readable** when fired twice
  (before the leak and at a fifth of the pool). Either it writes somewhere other than klog or it is
  a stub here.

### Next

Naming the allocation now needs instrumenting **zink or the frontend**, not the winsys. Cheap tests
proposed and not yet run: a null core, or the menu with its shader chain disabled, to see whether the
9,280 tracks draw count rather than frames.

## 2026-08-31 - two build-system defects that cost four hardware rounds

⚠ **`Makefile.orbis` did not relink the frontend when Mesa changed.** `$(RADV_ARCHIVE)` reaches the
link through `$(LIBS)`, which make cannot see, so `make pkg` reported success and packaged the
**previous** driver. Four rounds of "the fix did not take" were four rebuilt Mesas that never reached
the console. Fixed in `149d4df1ef`; the ELF now depends on the archive.

⚠ **And ccache made the log agree with the wrong answer.** `ac_orbis_drm.c` prints `__DATE__ __TIME__`
as its build stamp; ccache served a cached object and the stamp stayed frozen at the old build, so the
console log confirmed a driver that was not running. **Two independent things saying "unchanged" is
what let this survive.** When checking which build ran, check a string that only the new code has.

⚠ **The two eboots were truncating each other's Mesa log.** Both read `/data/retroarch-env.txt`, so
both took the same `MESA_LOG_FILE`, and Mesa opens it with `fopen(path, "w")`. Whichever launched
last erased the other's log - which is how "the glcore build produces no Mesa output" was measured
when it had been producing plenty. glcore now reads `/data/retroarch-glcore-env.txt` (`e037d22651`).

⚠ **And a diagnostic that only speaks to a listener is a diagnostic for the runs that did not need
one.** A whole run's watchdog output was lost because the maintainer's log receiver had dropped, and
the netlog and klog captures come from one script - losing it loses both. `ps4/orbis_watchdog.c` now
writes every line to `/data/retroarch-watchdog.log` as well.
