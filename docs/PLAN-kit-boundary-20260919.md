# Where the kit ends and the application begins — 2026-09-19

The owner's observation was that the responsibilities of the porting kit and of the applications are
starting to blur. This document states one boundary rule, applies it to every component of both
repositories, checks it against how other kits and named professional patterns draw the same line,
and derives from it a verdict on the proposed fifth platform macro. Every number is from a file read
or a command run on this machine today, with the source named; network reads are marked; the two
places where something is from memory say so.

---

## 0. The rule

**Hold the file and ask two questions. Would a second, unrelated program on this console need it
byte for byte? And could that program, or its user, notice it was there — does it own a thread, the
one audio port, a heap size, what happens after a crash, a path under `/data`? Needed unchanged and
unnoticeable: the kit's. Needed unchanged but noticeable: a backend or a policy, and it belongs to
whoever owns the resource — the middleware (SDL, the engine) for a backend, the program for a
policy. Everything else is the program's.**

Inside "the kit's" there are three roofs, and the trouble in both repositories is that only two of
them have names:

| roof | what it is | how it stops existing | today's home |
|---|---|---|---|
| **correction** | a libc, header or kernel behaviour the SDK gets wrong, reached from outside the port by the linker or the preprocessor | OpenOrbis fixes its SDK; the file is deleted | `orbis-compat` include/, src/, crt/ |
| **runtime library** | a function a program calls by name — `orbis_jit_alloc`, `orbis::ime_begin`, `find_data_root` — because the platform has the facility and the SDK has no wrapper | never; it is the platform's API, like libnx's `applets/swkbd.h` | split: `orbis-compat/src/` (jit, boot, paths) and `orbis-porting-kit/services/` |
| **tooling** | what a person runs or points a build at | never | `orbis-porting-kit` cmake/, scripts/, release/, patches/ |

The overlay's own README opens with *"This repository may shrink one day"* (`orbis-compat/README.md`,
quoted in the kit's README) — that is the correction roof's definition, and it is falsified by every
file under it that no SDK fix could ever remove. ⚠ Measured by `llvm-nm --defined-only` per member
of `build/liborbis-compat.a` (19 members, 161 external symbols): four members define **only**
libc/libc++ names (`orbis_wchar32.o` 73, `orbis_cv_fix.o` 5, `orbis_cxa_guard.o` 3,
`orbis_thread_atexit.o` 1) — pure corrections; six define **only** `orbis_*` names (`orbis_jit.o` 7,
`orbis_log.o` 7, `orbis_boot.o` 2, `orbis_env.o` 1, `orbis_sysconf.o` 1, `orbis_timer.o` 1) — a
runtime library under a roof named "compat"; nine are mixed. `orbis_mmap.cpp` contains no
`PROT_EXEC` and no reference to `orbis_jit` (grep), so the JIT arena is reachable by name only, and
nothing in the organisation outside `orbis-compat` references `orbis_jit` yet except three documents
(grep over every checkout, `.git` excluded).

The Yocto Project has the sharpest published form of the rule, as tests a layer must pass
(`yocto-check-layer`, read from `docs.yoctoproject.org/dev-manual/layers.html` today):
`common.test_signatures` — "BSP and DISTRO layers do not come with recipes that change signatures";
`bsp.test_bsp_no_set_machine` — "a BSP layer does not set the machine when the layer is added";
`bsp.test_machine_signatures` — "building for a particular machine affects only the signature of
tasks specific to that machine"; `common.test_patches_upstream_status` — "all the patch files
included in the layer contain a Patch Upstream Status". Translated: **adding the kit to a build may
change nothing the kit does not own.** ⚠ Our `--whole-archive` link of `liborbis-compat.a`
(`ps4-openorbis.cmake:230-241`) makes every `src/` member present in every consumer whether wanted
or not, so under this test every file in `src/` is a claim that no program could object to it. The
overlay already knows this: `optional/orbis_bigheap.c` says in its first lines that it "MUST NOT" be
in the archive because it is "a POLICY, not a correction". The rule was written; it was not applied
to the rest of the tree.

---

## 1. Every component, classified

`orbis-compat` (`find`, `wc -l`, today):

| component | lines | roof | verdict |
|---|---|---|---|
| `include/` musl corrections, `sys/*.h`, `execinfo.h`, `pthread_np.h`, `signal.h`, `errno.h`, `stdlib.h`, `unistd.h` | 2501 (1105 of them the `orbis_*.h` API headers, 108 `ps4_app.h`) | correction | stays |
| `src/` interposers: wchar32, cxa_guard, cv_fix, thread_atexit, abort_report, stat, mmap, thread, timer, sigev, clock, sysconf, backtrace, mem | 4538 (src/ minus the six files listed separately below) | correction | stays |
| `src/orbis_log.c`, `src/orbis_env.cpp` | 62 + 185 | substrate both roofs call | stays; the runtime library depends on the overlay, never the reverse (PLAN.md §13 already says this) |
| `src/orbis_paths.cpp` | 259 | correction (relative `open` is EINVAL, `getcwd` is ENOSYS — `orbis_paths.h:11-12`; rewriting is a correction of kernel behaviour reached through libc, 5 libc names interposed) | stays; ⚠ it once hard-coded `/data/OpenGothic/` "compiled into an overlay that three other titles now link" and "RetroArch has been creating `/data/OpenGothic/` on every boot" (`orbis_paths.h:48-50`) — the exact failure the rule exists to prevent, already paid for once |
| `src/orbis_jit.c` + `orbis_jit.h` | 778 + 143 | **runtime library** (7 `orbis_jit_*` names, 0 libc names, no interposition) | moves under the runtime-library roof; the precedent is libnx `kernel/jit.h` (fetched: `jitCreate`, `jitTransitionToWritable`, `jitTransitionToExecutable`, `jitClose`, two `JitType_` strategies) — the OS library owns the executable-memory primitive, and honours the platform's actual rule (libnx toggles W^X because that kernel has it; ours is RWX for life because `sceKernelMapDirectMemory` refuses EXEC at map time, `orbis_jit.h:22-24`) |
| `src/orbis_boot.cpp` | 552 | runtime library (crash handlers, ctype probe; 2 `orbis::` names) | moves with jit |
| `src/orbis_mem.cpp` | 463 | mixed: `operator new` interposition (9 libc++ names) plus `orbis_mem_*` reporting | stays; the reporting half is what makes it noticeable and is log-only |
| `crt/` | 662 | correction (a toolchain's CRT) | stays |
| `optional/orbis_netlog.cpp` | 85 | runtime library — libnx puts the dev-host link in `runtime/nxlink.h` (tree listing fetched) | moves under the runtime roof; ⚠ its default `NETLOG_TAG` is `"tempest"` (`orbis_netlog.cpp:17`), a product name in a shared file |
| `optional/ps4_app.cpp` | 294 | two things: the log tee (runtime library) and the termination policy + `/app0/ps4-run.cfg` reader (policy — its own header says "THIS IS POLICY", `ps4_app.h:3`) | split: tee to the runtime roof, termination and run-cfg become an **example** the program copies |
| `optional/orbis_bigheap.c` | 38 | policy (the SDK's own idiom is that the *application* defines `sceLibcHeapSize`; "OpenGothic deliberately does NOT link it") | an example snippet, not a target; it is already neither in the archive nor a target (`optional/CMakeLists.txt:12-14`) |
| `scripts/release/sdk-licenses.sh`, `licenses/`, `LICENSING.md`, `NOTICE.md` | 544 + ledger | tooling that writes this repository | stays (handoff §2's argument holds) |
| `test/`, `build.sh` | — | the overlay's own | stays |

`orbis-porting-kit`:

| component | lines | roof | verdict |
|---|---|---|---|
| `cmake/ps4-openorbis.cmake`, `ps4-package.cmake`, `orbis-compat.cmake`, `orbis-tls.ld`, `orbis.ini.in` | 907 | tooling (toolchain) | stays; this is devkitPro's `Switch.cmake` + `NintendoSwitch.cmake` + `switch_rules` in one file |
| `cmake/OrbisPatch*.cmake`, `scripts/orbis-patch.sh`, `patches/*/SERIES`, `patches.yml` | 113 + 163 + 8 SERIES + 125 | tooling (the registry *mechanism*) | stays |
| `patches/libretro/**` contents | 30 patches, 8 cores | see §2.2 | 16 stay as a draining staging area; 7 kernel-truth stay; **7 go back to RetroArch** |
| `vkloader/` | 9772 (9274 of it generated thunks) | tooling in the Khronos sense: it *is* the loader for a platform with one static ICD | stays; `vkloader.h:11-15` already refuses to own the per-consumer thunk list — the right shape |
| `services/audio` | 179 | **backend** (owns a thread and the one port) | out of the `orbis::` API — §2.1 |
| `services/ime` | 350 | runtime library (libnx `applets/swkbd.h`, 64 functions; PSL1GHT `sysutil/osk.h`; vitasdk `psp2/ime_dialog.h` — all fetched tree listings) | stays, under the runtime roof |
| `services/data` | 251 | runtime library (libnx `runtime/env.h`; the `.pkg`-has-no-argv fact is every title's) | stays, under the runtime roof |
| `examples/hello`, `examples/triangle` | 684 | tooling | stays |
| `release/` | 3120 | tooling (distribution) | stays |
| `scripts/ps4/{deploy,logs,make-pkg,log-receiver,peerfilter,orbis-env,gen-icon0,check-wchar-link}` | 1071 | tooling (host side of the dev link — libnx's `nxlink` and devkitPro's `elf2nro` are the same two things) | stays |
| `scripts/orbis-new.sh` | 408 | tooling | stays |
| `.github/actions/setup-orbis` | 470 | tooling | stays |

The one structural change this table asks for is a **named runtime-library roof** — `services/`
renamed to say what it is, holding ime, data, jit, boot, netlog and the log tee — so that the overlay
is corrections and substrate only, and "it is a runtime shim, so it lands in `src/`" (PLAN.md §13,
item 1, the sentence that put `orbis_jit` where it is) stops being a valid reason to put a named API
under `--whole-archive`. Whether that roof is a directory in the kit or a third repository is a
packaging choice; devkitPro installs libnx, its `switch_rules` and the toolchain into one prefix, and
consumers pin one thing, so a directory in the kit is the precedent. ⚠ Two stale claims sit in the
way and should go with the rename: the kit README names `scripts/check-copies.sh` twice
(`README.md:10`, `:55`) and the file does not exist (`ls scripts/`; PLAN.md §13 says it "expired");
`services/ime/orbis_ime.h:38-39` says the four Sce libraries "are linked by the toolchain file", and
`ps4-openorbis.cmake` contains no `Sce` at all (grep) — `services/CMakeLists.txt:44-45` links them
itself, which is correct and is the sentence the header should say.

---

## 2. The four questions

### 2.1 An audio mixer and an IME wrapper in a porting kit

They are not the same case, and the rule separates them.

**IME and data-root are runtime library.** Every homebrew kit examined owns the on-screen keyboard
and the "where am I, where is content" question: libnx `applets/swkbd.h` (64 entry points, fetched),
PSL1GHT `sysutil/osk.h` and `sysutil/game.h`, vitasdk `psp2/ime_dialog.h` and `psp2/appmgr.h` (tree
listings). None of them owns a save-dialog flow, and neither does ours: `orbis_data.cpp:1-2` says the
"game-specific half … stayed in the title where it belongs". These two pass both questions.

**Audio is a backend and fails the second question.** `orbis_audio.cpp` owns a `std::thread` that
loops on `sceAudioOutOutput` (`pump()`) and the one port; `orbis_audio.h:31-33` states
in its own words that a title with several audio devices "cannot give each one a port … Mix in
software, then hand blocks to this." The thread and the port are the resource; the resource has one
owner per process; the kit is not that owner. What the precedents own is narrower: libnx
`services/audout.h` is 31 service wrappers (`audoutOpenAudioOut`, `audoutAppendAudioOutBuffer`,
`audoutWaitPlayFinish`, …) with no thread and no callback; the thread lives in SDL's
`src/audio/switch` or in the program. Godot's public tree has the same split at directory level:
`platform/` holds `os_linuxbsd.cpp`, `detect.py`, `platform_config.h`, the crash handler; `drivers/`
holds `alsa`, `coreaudio`, `pulseaudio`, `wasapi`, `xaudio2` (contents API, fetched) — the audio
*driver* is never in the platform directory.

⚠ **The conflict the brief names is measured, and wider than two.** `grep -rl sceAudioOutOpen` over
the organisation finds four owners of the main port: `RetroArch/audio/drivers/ps4_audio.c` (436
lines), `OpenGothic/ps4/og_sound_orbis.cpp` (still built, kit README: "OpenGothic still builds its own
copies"), `orbis-porting-kit/services/audio/orbis_audio.cpp`, and `SDL2/src/audio/orbis/SDL_orbisaudio.c`
(332 lines), whose header at lines 45-48 says a title "must drive it from ONE place: either this
driver or `orbis::audio_start()`, never both".

⚠ **And the kit's API encodes a hardware fact the other two owners measured differently.**
`orbis_audio.h:27-28`: "Every Open in all three ladders returned the same handle, 0x20000007, for
every rate, grain and format, and after every Close." `RetroArch/audio/drivers/ps4_audio.c:174-179`:
"eight opens, eight closes, EVERY close returning 0x00000000 — and the ninth open failing with
PORT_FULL. The handles counted down 0x20000007, 0x20000006 … 0x20000000 and were never reissued."
`SDL_orbisaudio.c:33-35` says the same as RetroArch and caches the handle for the life of the
process. The kit's `audio_stop()` calls `sceAudioOutClose(g_port)` and `audio_start()` reopens
(`orbis_audio.cpp:92-102`, `:62`). Under RetroArch's measurement, the ninth start/stop pair in a
process is silent for the rest of that process, and nothing in the kit says so. Which measurement is
right is a console question; that the kit's public API rests on the one that has two contradicting
measurements beside it in the same organisation is the design question, and it is answered by the
rule: the kit should not have had a `stop` to get wrong.

⚠ **And the API has no consumer.** `grep -rn "orbis::(audio|ime|data)_|orbis::services|orbis-services"`
over OpenGothic, Tempest, sonic3air, RetroArch, Panda3DS, the kit's examples, cmake and README, and
orbis-compat finds only `.github/workflows/kit.yml:99-100`, which archives the library and counts its
symbols. The brief's "kit API that games call" is an API no game calls. The CI assertion — "compiled
for the console, with no engine around them" — proves the extraction compiles, not that it is reused.

**What moves.** `orbis_audio.h` keeps its measured facts as constants and a comment and loses its
four functions; `orbis_audio.cpp` is deleted; the two backends that exist — the SDL2 driver for SDL
titles, OpenGothic's own `SoundDevice` for Tempest — are the audio path, and the kit's contribution
to a third engine is the facts, not a thread. If a reference pump is wanted, it is an `examples/`
file, not an `orbis::` symbol.

### 2.2 Thirty application patches in the kit

The precedent is uniform and the brief has it right: devkitPro's 76 Switch patches are on the
libraries it packages, Emscripten's ports are libraries, vitasdk builds SDL 2.32.8 from a pristine
tarball (PLAN-minimal-port-diff §7, verified yesterday). Yocto's `common.test_patches_upstream_status`
is the same idea made a test: a layer may carry patches, and each one must say where it is going.
The registry's `exit:` field is that test, written before the precedent was found, and
`scripts/orbis-patch.sh debt` (run today) prints: 30 patches, 16 with an exit, 14 `none`; by route
7 `overlay:orbis_jit`, 3 `upstream (candidate)`, 1 `portlib:zlib`, 1 `portlib:lua (candidate)`,
1 `overlay:sem_*`, 1 `overlay:orbis_jit (partly)`, 1 `overlay:net/if_dl.h + tls_init`,
1 `overlay:getcwd (candidate)`.

So the category error is not the registry — a staging area that names each line's exit is the kit's
tooling, and the CMake entry point (`OrbisPatch.cmake`, needed because melondsds fetches at
configure time) cannot live in a shell harness. The error is inside the 14 `none`. Read from each
`SERIES`'s own `exit-why`:

- **Kernel truth, the platform's, and legitimately the kit's** — the same lines would be needed by a
  standalone build of the same core with no RetroArch anywhere: flycast 0001 ("this kernel does not
  deliver a usable segfault context"), flycast 0002 (section placement), play 0004 ("does not deliver
  SA_SIGINFO"), swanstation 0001 (shared memory, "no shm_open here"), 0005 (16 KiB page), 0006, 0007
  (fastmem off where its handler cannot exist). Seven.
- **A frontend's own decision, wearing the platform's name** — melondsds 0001: "which of the frontend
  objects this link may use", a fact about RetroArch's link, not about the console. One.
- **Product and performance decisions** — mupen64plus_next 0002 ("renderer selection, a product
  decision"), 0004 ("frame instrumentation across five files"), 0005 ("a performance decision"), and
  0006/0007/0008, which record what this GL stack does with a depth attachment: those three are
  reports about mesa-ps4's behaviour that belong beside mesa-ps4's known issues, carried as a patch
  because a core was the place they were noticed. Six.

Seven of thirty therefore move back to `RetroArch/ps4/` — or, for the three depth patches, to a
mesa-ps4 issue with the core patch kept in RetroArch until the driver answers. The kit keeps what any
consumer of that core on this console would need. Patches whose exit is `overlay:` or `portlib:` are
the kit's until the exit is taken, and the weekly `patches.yml` dry-apply (Mondays 05:23 UTC,
`patches.yml:30`) is the mechanism Yocto's test asks for. ⚠ The seven `overlay:orbis_jit` exits
are not yet exercised by anything: no core patch, harness or source in the organisation references
`orbis_jit` today (grep, §0); until one does, the arena is a specification with a header.

### 2.3 The seam between the repositories, and `orbis_jit`

"Anything a person runs or points a build at" (handoff §2) is a correct test for tooling and says
nothing about runtime code, which is why five runtime shims went *into* the overlay by a second rule
(definition order) and `services/` went *into* the kit by a third (extracted from a game). Three
rules made two repositories with a runtime library split across both. The seam should be the roofs
of §0: the overlay is corrections plus the two substrate files; the kit is tooling plus the runtime
library. The definition-order shims stay in the overlay under that seam — they define libc and
libc++ names (§0's nm count), which is the definition of a correction.

`orbis_jit` in a libc overlay is therefore the wrong roof and the right repository is a packaging
choice. What decides it is not "is it runtime" but "does a program call it by name": it does
(`orbis_jit_alloc`, `_alloc_at`, `_free`, `_protect`, `_state`, `_reachable`, `_release_all`, from
`orbis_jit.h`), and nothing reaches it through `mmap` (§0). The one argument for the archive — that a
port calling `mmap(PROT_EXEC)` would be served without naming us — is not implemented, and the
patches say most JIT callers would not be served that way anyway (PLAN-minimal-port-diff §1: swanstation's
`MemoryArena`, play's `MemoryFunction`, mupen's `new_dynarec` allocate through their own classes).

### 2.4 The macro — see §4.

---

## 3. How others draw the line

### 3.1 The homebrew kits, by layer (files fetched today unless marked)

**devkitPro / libnx.** Four directories, each a layer: `kernel/` (svc, jit, virtmem, mutex, shmem,
tmem, thread), `services/` (audout, audren, fs, hid, applet, nifm, … — one header per system
service), `applets/` (swkbd, error, web, psel — system dialogs), `runtime/` (nxlink, env, pad,
hosversion, diag — the dev link and the boot environment). Audio: service wrappers only; input:
`hid` plus `runtime/pad.h`; filesystem and save data: `services/fs.h` wrappers; nothing that owns a
thread, mixes, or decides. The toolchain rules (`nx/switch_rules`, 82 lines) are installed *with
libnx*, not with the compiler. ⚠ The preprocessor side, unverified yesterday, is now verified:
`__SWITCH__` is a **build-system define, never a compiler one** — libnx's own `nx/Makefile:35`
(`-D__SWITCH__ -DLIBNX_NO_DEPRECATION`), the application template in switch-examples
(`templates/application/Makefile:55`, `-D__SWITCH__`), and `pacman-packages/cmake/switch/NintendoSwitch.cmake:15`
(`NX_COMMON_FLAGS "-ffunction-sections -fdata-sections -D__SWITCH__"`). libnx has no compile-time
version macro in `switch.h` (an umbrella of includes) or `nx/Makefile` (grep `VERSION`: none);
`runtime/hosversion.h` is the *firmware* version at run time.

**PSL1GHT** (PS3). `ppu/include/` has the same shape — `lv2/`, `sys/`, `audio/audio.h`, `io/pad.h`,
`sysutil/{osk,save,msg,game,video}.h`, `rsx/`, `net/`; `ppu_rules` (94 lines) has no `-D` at all
(`MACHDEP` at line 21 is codegen flags). The kit's identity macro is defined by the consumer:
RetroArch's `Makefile.psl1ght:40` is `MACHDEP := -D__PSL1GHT__ -D__PS3__ -mcpu=cell`.

**vitasdk.** `vita-headers/include/psp2/` is Sony's namespace verbatim (`audioout.h`, `ctrl.h`,
`ime_dialog.h`, `appmgr.h`, `common_dialog.h`, `io/`, `kernel/`, …), 136 recipes for libraries, and
no wrapper layer. Its identity macro is the compiler's: `__vita__` comes from
`buildscripts/patches/gcc/0001-vita-target.patch` (GitHub code search), and
`tests/toolchain-contract/run.sh:47` asserts `#define __vita__ 1` and the float ABI (`:52-54`) —
the macro is treated as a **contract with a test**.

**Emscripten.** `__EMSCRIPTEN__` is defined by clang for the triple
(`clang/lib/Basic/Targets/OSTargets.h:1037`, fetched); the beyond-POSIX API lives under one prefix in
`system/include/emscripten/` (`html5.h`, `webaudio.h`, `wasmfs.h`, `fetch.h`, `eventloop.h`,
`threading.h`). The platform's own API is named after the platform and nothing else is.

**What every one of them refuses:** a mixer, a resampler, a save-game format, a content locator for
one title, a termination policy, and any file whose default carries a product's name. What every one
of them owns: system-service wrappers, system dialogs, the dev link, the packaging tools, and the
toolchain rules.

### 3.2 The named patterns

- **BSP layer model (Yocto).** Hardware layer, distro (policy) layer, software layers, each a
  repository; the tests in §0 are the boundary. Our overlay is the BSP layer; policy files in it
  (`ps4_app`'s termination, `bigheap`, a `"tempest"` tag) are what `bsp.test_bsp_no_set_machine` is
  written to catch — a layer that decides something merely by being added. Best practice 3.2 of the
  same manual: "do not copy an entire recipe into your layer and then modify it. Rather, use an
  append file" — the argument against forks and for the patch registry, stated by someone else.
- **Backend / driver model (SDL, Godot).** SDL's `SDL_config_orbis.h` selects drivers; the driver
  owns the device and its thread; the application never sees the port. Godot: `platform/` versus
  `drivers/`, §2.1. In this model the audio thread is *always* the middleware's.
- **Loader / ICD (Khronos).** `LoaderInterfaceArchitecture.md`: "The loader is responsible for
  discovering available Vulkan drivers"; layers are "optional components" and "any function a layer
  does not hook is simply skipped"; `LoaderDriverInterface.md:807`: "The Vulkan symbols exported by
  a driver must not clash with the loader's." `vkloader/` is a loader with one static ICD and a
  generated dispatch — the pattern is recognisable, and its refusal to ship the thunk list is the
  loader refusing to know the application.
- **HAL (SDK wrappers).** vitasdk and PSL1GHT are pure HALs: the vendor's names, no policy above
  them. Our `services/` names are ours (`orbis::ime_begin`), which is the libnx choice, not the
  vitasdk one; both are accepted, but a wrapper that renames must not also decide.
- **Engine platform extensions (Unreal, Godot).** *From memory, not verified today:* Unreal keeps
  console platforms as `Engine/Platforms/<Name>/` directories distributed under NDA outside the
  public tree, and Godot's console ports are done by third parties outside the repository. What is
  measured: Godot's public `platform/` contains `android ios linuxbsd macos visionos web windows` and
  no console. The pattern: the engine defines the extension point; the platform owner fills it; the
  engine's tree never contains the platform's product decisions.

### 3.3 The "two implementations" precedent is older than this platform

The PS4 is not the first console with an official SDK and a homebrew one, and the canonical
multi-platform application already shows how the distinction is drawn — measured in
`RetroArch` today:

| platform | official SDK's macro | homebrew kit's macro | who defines the kit's | files testing kit's / official's |
|---|---|---|---|---|
| PS3 | `__CELLOS_LV2__` | `__PSL1GHT__` | the application (`Makefile.psl1ght:40`) | 31 / 1 |
| Switch | (not in tree) | `HAVE_LIBNX`, `__SWITCH__` | the application (`Makefile.libnx:22,134`) | 75 / — (`__SWITCH__` 6) |
| Vita | `__psp2__` | `__vita__` | the compiler (vitasdk gcc patch) | 5 / 0 |
| PS4 | `__ORBIS__` | — | — | `ORBIS` 42, `__ORBIS__` 0 |

SDL3's `SDL_platform_defines.h:468` accepts both Vita spellings, `defined(__vita__) || defined(__psp2__)`,
and has **no PS4 arm at all** (grep `PS4|ORBIS`: 0 of 519 lines, fetched from `main`) — Sony's own
SDL port is not upstream, so a public PS4 arm has nothing to be compatible with. The homebrew kit's
identity is, in every case with a second implementation, a *second* macro beside the platform's,
defined by whoever links the kit.

---

## 4. The macro

Derived from §0: the kit's runtime library is the thing a program reaches **by name**, and names are
enforced by the linker, not the preprocessor. Measured: the overlay defines 0 `sce*` symbols and 45
`orbis*` ones (`llvm-nm --defined-only build/liborbis-compat.a`); Sony's W^X facility `sceKernelJit*`
is exported by `libkernel_sys.so` (4), `libkernel_jvm.so` (4), `libkernel_ps2emu.so` (4),
`libkernel_psmkit.so` (3) and by `lib/libkernel.so` **not at all** (0), while every link line here is
`-lc -lkernel -lc++` (`ps4-openorbis.cmake:346`) and nothing in the organisation links `kernel_sys`
(grep); the SDK declares them as `void sceKernelJitCreateSharedMemory();` with no parameters
(`include/orbis/libkernel.h:285-291`). A port written against Sony's JIT fails at link on this kit;
a port written against ours fails at link on Sony's. The boundary already exists and it is linkable.

`__ORBIS__` is not a promise this kit can make. It is defined by **clang itself** for the triple
`x86_64-scei-ps4` (`OSTargets.h:626-634`, `PS4OSTargetInfo`), on top of a base that defines
`__FreeBSD__ 9`, `__SCE__`, `unix`, and — the part that matters — `WCharType = UnsignedShort`
(`OSTargets.h:583-599`): a 16-bit `wchar_t`, which is what the SDK's `libc.a` was built with
(handoff §2) and what the overlay's 1524-line `orbis_wchar32.c` exists to override for our
`x86_64-pc-freebsd12-elf` triple. ⚠ Upstream RetroArch's `Makefile.orbis` at the merge-base
(`git show c59b1833:Makefile.orbis`, line 121) builds with OpenOrbis on Sony's triple:
`--target=x86_64-scei-ps4 -DORBIS -D__ORBIS__ -D__PS4__`; this organisation's fork moved it to
`x86_64-pc-freebsd12-elf` (`Makefile.orbis:471`). So `-D__ORBIS__` on our command line asserts a
platform whose compiler ABI we deliberately left. It still earns its place — it is the spelling that
makes upstream code take its PS4 arm, which is the only reason any of the four exist — but it is a
request to portable code, not a contract with Sony's SDK, and "compatibility with the official SDK"
should be stated as exactly that: *a port's `#ifdef __ORBIS__` arms compile here*, never *the kit's
API is Sony's*.

⚠ The brief's count needs one correction before the verdict: the kit toolchain defines **three**
macros, not four — `ps4-openorbis.cmake:99` is `-D__PS4__ -DPS4 -D__ORBIS__` and the bare `ORBIS`
appears only in `build-cores.sh:306` and `Makefile.orbis:471` (grep `-DORBIS ` in the toolchain
file: 0). And the bare one is the one RetroArch actually tests — 42 files against 0 for `__ORBIS__`
— so a title built with the kit toolchain sees a different macro set than a core built with the
harness. The organisation's own census (c/cpp/h across OpenGothic, Tempest, sonic3air, Panda3DS,
SDL2, orbis-compat, the kit, mesa-ps4/src/amd, ZenKit; vendored SDL and RetroArch excluded): `PS4`
65 files, `__PS4__` 24, `__ORBIS__` 19, `ORBIS` 2.

**Verdict.** A fifth *platform* macro on the command line is the wrong instrument: it would mark
nothing the linker does not already mark, and every kit with a second implementation (§3.3) puts the
kit's identity in the *consumer's* build or in the compiler, never in a fourth spelling of the
platform. What is right, and small, is a **feature macro provided by a kit header**, the way
`HAVE_LIBNX` is a feature and `__PSL1GHT__` a library: an `orbis_compat.h` (or the existing
`orbis_prefix.h`, which is already `-include`d) defining `ORBIS_COMPAT_VERSION` and, once the roof
exists, `ORBIS_RUNTIME_VERSION`, so that code which must compile against both Sony's SDK and ours —
of which there are **zero files today** — can say `#if defined(__ORBIS__) && __has_include(<orbis_jit.h>)`
and be honest about what it links. vitasdk's contract test is the model for keeping it true: one
line in `orbis-compat/test/declarations.c` asserting the macro's presence and value. What should not
happen is the thing the four macros already show: another define that every command line must carry
and that two of the three build entry points will eventually disagree on.

---

## 5. Anti-patterns, and which ones we are doing

1. **The kit grows into an engine.** Doing, in one file: `services/audio` owns a thread. The
   extraction refused the mixer, resampler and decoder (`orbis_audio.h:4-8`) and kept the pump,
   which is the resource owner. The rest of `services/` and the overlay do not do this.
2. **The kit owns product decisions.** Doing, in three places: six mupen64plus_next patches whose
   own `exit-why` says "product decision" and "performance decision" (§2.2); `ps4_app`'s
   termination policy and run-cfg in the overlay (labelled policy, still shipped by the overlay's
   CMake as a target); `NETLOG_TAG "tempest"`. Did, and fixed: `/data/OpenGothic/` in `orbis_paths`.
   The handoff's item 6 (three product paths in `orbis_env.cpp`) is the same class and is still open.
3. **The kit hides a real constraint behind an API that cannot honour it.** Doing: `audio_stop()`
   closes a port that two other measurements say is spent on close (§2.1). Adjacent, not ours:
   `sem_*` link and return EINVAL (SDK); `check_symbol_exists` cannot see it.
4. **Two owners of one resource.** Doing: four `sceAudioOutOpen` callers in the organisation, two of
   them in code the kit ships or forks.
5. **An API with one origin and no consumer.** Doing: `orbis::services` has zero callers; the kit's
   own README says "A copy with a test against it is a bridge. A copy without one is the above", and an extraction nobody
   links is a copy with a new name. The same holds for `orbis_jit` until the first patch is deleted
   by it.
6. **Claiming another SDK's identity.** Doing, mildly and for a good reason: `-D__ORBIS__` under a
   different `wchar_t` (§4). Acceptable as long as nobody writes "Sony-SDK compatible".
7. **A layer that decides by being present.** Doing, structurally: `--whole-archive` of every
   `src/` member into every consumer, including the CTS and Mesa, which is why `optional/` had to be
   invented. The Yocto test for this is `bsp.test_bsp_no_set_machine`.
8. **Documentation of a tree that does not exist.** Doing: `check-copies.sh` (twice in the README),
   "linked by the toolchain file" in the IME header.
9. **A kit whose boundary is defined by three different rules.** Doing — it is the cause of the
   owner's observation, and §0 is the replacement.

Not doing: forking instead of patching (the registry exists and forks are being retired); patching
the harness's bugs into the software (the registry README forbids it and says why); shipping a
package format or an installer as the kit's — `make-pkg.sh` follows the SDK's own sample rule and
says so.

---

## 6. Where the brief was wrong, collected

- "Kit API that games call": zero callers outside the kit's own CI (§2.1).
- "Evidenced by upstream SDL2's `SDL_platform.h`": the `#if defined(__ORBIS__) || defined(PS4)` arm
  is in the **SDK's vendored SDL 2.0.9** (`~/.local/opt/openorbis/include/SDL2/SDL_platform.h:163-166`,
  `SDL_version.h:60-62`); upstream `release-2.32.x` `SDL_platform.h` has 0 matches for `ORBIS|PS4`
  and the fork's copy has 0. The evidence that Sony's SDK sets `__ORBIS__` is clang's
  `OSTargets.h:633`, which is stronger.
- "Four platform macros on every command line": three on the kit's, four on RetroArch's (§4).
- "One audio port `0x20000007` per process": the kit's header and RetroArch's driver disagree on what
  `Close` does to it (§2.1), and the kit's `stop` follows the header.
- "Two implementations": at the toolchain level there are three identities in play — Sony's triple
  with Sony's SDK, Sony's triple with OpenOrbis (upstream RetroArch's choice), and ours.
- `orbis_jit` "added yesterday" is referenced by no code anywhere yet; the seven `overlay:orbis_jit`
  exits are promises.
- Confirmed as stated: 780 service lines (179 + 350 + 251); 30 patches over 8 cores; 16/14;
  0 `sce*` and 45 `orbis` defined symbols; `sceKernelJit*` in `libkernel_{sys,jvm,ps2emu,psmkit}.so`
  (4/4/4/3) and not in `libkernel.so`.
