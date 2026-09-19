# Minimal port diff: what the kit must own so ports stop carrying it — 2026-09-19

The question was how other porting kits keep the changes inside the ported software near zero, and
whether what we build generalises past the five ports in this organisation. The reframing offered
with it survives testing and needs one sharpening: **the goal is zero lines this organisation
maintains**, and there are three ways a line stops being ours, not one. It can be absorbed by the kit
(nobody writes it), it can be accepted upstream (somebody else keeps it), or it can be written in the
shape upstream already uses for other consoles so that the port carries it as one directory instead
of as a fork. Every kit examined below uses all three; the plan ranks our four levers by which of the
three each one actually delivers, measured against the files.

Every number here comes from a file read or a command run on this machine on 2026-09-19, with the
source named. Network reads are marked; nothing is from memory except where a sentence says so.

---

## 0. The baseline: what this organisation maintains today

Measured with `git diff --stat <merge-base> HEAD` in each checkout under `~/src/orbis-ports`;
"pre-existing" means `--diff-filter=M`, the lines inside files upstream already had, which is the
intrusive part of a port. "Names a PS4 macro" is the count of added lines matching
`__ORBIS__|__PS4__|PS4`.

| repository (branch) | upstream base | whole diff | pre-existing files | CMake | notes |
|---|---|---|---|---|---|
| OpenGothic (ps4-support) | origin/master 17e889d3, 2026-08-17 | 40 files, +4205/−81 | 25 files, +675/−81; 22 of 484 added lines in game/ and lib/ name a PS4 macro | CMakeLists.txt +177, all under `if(PS4)` | ps4/ directory: boot 762, sound 1075+430, IME 279 |
| Tempest (ps4-support) | origin/master 4ccbc71d | 9 files, +674/−10 | 5 files, +79/−10 | Engine/CMakeLists.txt +35/−10 under `if(PS4)` | ps4input.cpp 420, ps4api.cpp 129 |
| ZenKit (ps4-support) | origin/main 083cc5f5 | 8 files, +753/−44 | all 8 | +29 (lazy VFS option, pread probe) | ⚠ nothing in it is PS4-specific, and GothicKit/ZenKit main has no `ZK_ENABLE_LAZY_VFS` (raw CMakeLists fetched): this is an upstream feature we are keeping in a fork |
| sonic3air (ps4-support vs main) | 5cecd683 | 56 files, +6354/−20 | 39 files, +779, 80 lines name a PS4 macro (excluding the vendored SDL2) | build/_ps4/CMakeLists.txt, new file, mirrors upstream's _android/_emscripten layout | vendored SDL2: 41 lines in 10 upstream files + 3 new driver files; glew_orbis_gl11.c 2067 |
| RetroArch (ps4-support) | c59b1833, 2026-08-21 | 122 files, +21887/−1368 | outside ps4/ and .github/: 41 files, +3236/−750, 98 lines name a PS4 macro | Makefile.orbis exists upstream at the merge-base | ps4/ holds the harness, 30 core patches, and the frontend's own compat objects |
| Panda3DS (ps4-support) | 69095ee6 | 17 files, +310/−50 | 14 files; the 162 added lines in renderer_gl.cpp sit under three `#ifdef __ORBIS__` | third_party/glad/CMakeLists.txt +2 (`elseif(ORBIS)`) | .gitmodules repoints dynarmic, fmt and SDL2 at orbis-ports forks (3 pins) |
| 3dsTrident (ps4-support) | b97b0bef | 2 files, +2/−2 | .gitmodules + submodule pin | — | exists only to point at the Panda3DS fork |
| dynarmic (ps4-support vs master) | — | 2 files, +7/−6 | see §4 | src/dynarmic/CMakeLists.txt +7 (`elseif (ORBIS)` at :483) | ⚠ the −6 is a stale duplicate of a fix upstream already has, §4 |
| fmt (ps4-support vs main) | — | 1 file, +5 | include/fmt/base.h | — | a clang ≥ 22 workaround, not a platform change; fmt master's core.h has no such arm (fetched) |
| SDL2 (orbis-2.32 vs release-2.32.10) | — | 17 files, +2264/−2 | 10 upstream files, 41 lines | orbis/CMakeLists.txt, 152 lines, no `install()` or `export()` | ⚠ the handoff's "~8 lines touch upstream files" is the three bootstrap arrays alone; the fork's own header says 39, the diffstat says 41 |
| RetroArch/ps4/core-patches | — | 30 `.patch` + 2 `.patch.parked` + README | — | 5 patches touch a CMake file | ⚠ across **8** core directories, not 9: dirksimple 1, flycast 3(+2 parked), melondsds 4, mupen64plus_next 8, pcsx_rearmed 1, play 5, swanstation 7, tic80 1; the ninth, trident, was deleted when it became a fork (ps4/core-recipe-extra says so) |

Two corrections to the brief fall out of this table before any lever is discussed. The kit toolchain
**does** set a CMake platform variable: `set(PS4 TRUE)` at `orbis-porting-kit/cmake/ps4-openorbis.cmake:37`,
and 14 lines of OpenGothic's CMakeLists, 5 of Tempest's, 4 of sonic3air's `_ps4` file and both kit
examples test it. What neither toolchain sets is `ORBIS`, which is the name the two fork patches
chose. And build-cores.sh's generated toolchain sets neither, so the same `if(PS4)` that works for a
title is false for a core.

---

## 1. Lever 1 — make the platform look POSIX (the overlay), and its limits

What the overlay can absorb is exactly what the linker and the preprocessor let it reach from
outside the port's sources, and nothing else:

1. **A symbol the SDK's libc, libc++ or libkernel defines wrongly or not at all.** An object ahead of
   `-lc`/`-lc++` on the link line wins over an archive member, and `--whole-archive` on
   `liborbis-compat.a` (`ps4-openorbis.cmake`, the `CMAKE_EXE_LINKER_FLAGS_INIT` block) makes the
   interposers present without anything referencing them. 21 files in `orbis-compat/src/` are this.
2. **A header the SDK omits** (`execinfo.h`, `pthread_np.h`, `sys/{cpuset,dirent,endian,ioccom,param,sysctl}.h`)
   or **a declaration it gets wrong** (the four pthread types, corrected through musl's own
   `__DEFINED_` guards because the overlay's include directory is searched first).
3. **A kernel behaviour the port reaches only through a libc call** — `mmap`, `stat`, `pthread_create`,
   `timer_create`, `sysconf`. These are the four interposers of README §2.5 and the thread and timer
   work of PLAN.md.

The classes that can **never** be absorbed this way, each with the patches that prove it exists:

- **A platform chain inside the port's own translation units.** `#elif defined(__linux__) || defined(__FreeBSD__)`
  cannot be reached from a header the overlay ships, and on this target it is not merely unreachable
  but wrong twice: ⚠ the SDK's libc++ does `#undef __FreeBSD__` at `include/c++/v1/__config:12`
  (inside its `_LIBCPP_CONFIG_SITE` block, libc++ 11), so every C++ TU sees neither `__linux__` nor
  `__FreeBSD__`, and the overlay's include directory sits *behind* libc++'s by the load-bearing
  `std::abs` order in `ps4-openorbis.cmake`, so it cannot shadow `__config`. Eight of the thirty
  patches are this and nothing else: swanstation 0001/0002/0003, play 0002, flycast 0001,
  melondsds 0004, and the `|| defined(__ORBIS__)` hunks of swanstation 0001 (four chains in one file).
  ⚠ Every one of the 13 core-patch files that mention FreeBSD mention **the C macro or the kernel**;
  `grep -n -E '\bUNIX\b|CMAKE_SYSTEM_NAME' -r RetroArch/ps4/core-patches` returns nothing. The only
  lines that remove this class are port-side `|| defined(__ORBIS__)`, which is what a port written
  for Sony's own SDK would already contain, and which upstream projects with console arms accept.
- **A feature the kernel lacks but the SDK links.** `check_symbol_exists(sem_timedwait)` answered
  `1` under all three toolchain variants in the probe (§2), because `libc.a` has the symbol; the
  console returns `EINVAL` from every `sem_*` call (melondsds 0003, SDL2 `orbis/CMakeLists.txt`'s
  reason for `thread/generic/SDL_syssem.c`). ⚠ This one is *partly* absorbable and is the one new
  overlay candidate this plan proposes (§5, rank 6): a `sem_*` interposer over mutex+condvar would
  delete melondsds 0003 and SDL2's source-list exception, if the SDK's `sem_*` are not also called
  from inside other `libc.a` members — the `regcomp`/`mbtowc` trap in the handoff is exactly what has
  to be ruled out first.
- **Executable memory with the port's own arena type.** `orbis_jit` (src/orbis_jit.c, 32 KB, the
  handoff's item 1) can serve a port that calls `mmap(PROT_EXEC)` or `mprotect`; it cannot serve
  swanstation's `MemoryArena`, play's `MemoryFunction` or mupen's `new_dynarec`, which allocate
  through their own classes and need an arm that names the platform (swanstation 0001/0003/0004/
  0005, play 0005, mupen 0001/0003, pcsx_rearmed 0001, flycast 0002/0003).
- **Signal-context layout.** The kernel writes FreeBSD's amd64 mcontext into a musl `ucontext_t`
  and calls handlers with the old `(sig, code, scp)` convention unless `SA_SIGINFO` carries
  FreeBSD's value; the overlay's `signal.h` corrects the constants, but a port that reads `rip` out
  of the context (flycast 0001/0004, play 0004, swanstation 0006/0007) reads it in its own code.
- **Build-system decisions** — lever 2. **Product and performance decisions** — §6.

So lever 1 is finished as a *class*: what remains absorbable is one interposer family (`sem_*`) and
the JIT arena for the `mmap` callers, and everything else the overlay could ever take it has taken.

---

## 2. Lever 2 — platform identity in the build system, measured

### 2.1 What CMake actually does (probe, cmake 4.4.3 at /opt/homebrew)

A throwaway project (`~/.claude/jobs/67187e43/tmp/cmtest/`) with `project(probe C CXX)`, a real
`add_executable`, an `add_library(SHARED)`, two `check_symbol_exists`, and five toolchain files that
each `include()` the kit's `ps4-openorbis.cmake` and then override the system name. All five
configured, compiled and linked the test executable against the SDK (exit 0).

| variant | CMAKE_SYSTEM_NAME | WIN32 | APPLE | UNIX | BSD | PS4 | ORBIS | `if(WIN32)/elseif(APPLE)/elseif(UNIX)/else()` | TARGET_SUPPORTS_SHARED_LIBS | CMAKE_DL_LIBS |
|---|---|---|---|---|---|---|---|---|---|---|
| kit as-is | Generic | — | — | — | — | TRUE | — | **else** | FALSE (author warning: SHARED became STATIC) | dl |
| build-cores.sh's | FreeBSD | — | — | 1 | FreeBSD | TRUE¹ | — | **unix** | TRUE | "" |
| Orbis + `Platform/Orbis.cmake` (no UnixPaths) | Orbis | — | — | — | — | TRUE | TRUE | **else** | FALSE (set by the module) | dl |
| Orbis + module with `include(Platform/UnixPaths)` | Orbis | — | — | 1 | — | TRUE | TRUE | **unix** | FALSE (set by the module) | dl |
| Orbis, no module | Orbis | — | — | — | — | TRUE | — | else | TRUE | dl; "System is unknown to cmake" printed five times |

¹ PS4 is TRUE here only because the probe wrapper included the kit file; the real generated
`orbis-core.cmake` (build-cores.sh:396) sets no platform variable.

The mechanism, from the modules on this machine: `CMakeSystemSpecificInitialize.cmake` does
`unset(APPLE) unset(UNIX) unset(CYGWIN) unset(MSYS) unset(WIN32) unset(BSD) unset(LINUX) unset(AIX)`
and then `include(Platform/${CMAKE_SYSTEM_NAME}-Initialize OPTIONAL)`; `UNIX` comes back only from
`Platform/UnixPaths.cmake:18`, which `Platform/FreeBSD.cmake` includes and `Platform/Generic.cmake`
does not; `BSD` only from `Platform/FreeBSD-Initialize.cmake:1`. ⚠ So the claim "a custom platform
makes WIN32, APPLE and UNIX false" is true and **already true of Generic today** — a Platform/Orbis
module does not change which arm upstream's chain takes. What it uniquely adds is (a) `ORBIS` as a
variable every consumer sees, so `-DORBIS=ON` is never hand-passed and a stranger's `elseif(ORBIS)`
is meaningful; (b) the choice of `UNIX` decoupled from claiming to be FreeBSD (so `BSD` is not set
and `CMAKE_SYSTEM_NAME MATCHES "FreeBSD"` arms are not taken); (c) a `Platform/Orbis-Initialize.cmake`
that is the one place identity lives — which is exactly devkitPro's layout (§7). `CMAKE_MODULE_PATH`
appended inside the toolchain file is honoured for the Platform lookup (the probe's module was found;
the path shows twice because CMake reads a toolchain file twice, harmless).

### 2.2 The blast radius among the 30 patches: zero, in both directions

Patches that exist only to undo the consequences of claiming FreeBSD in CMake: **none**. Patches that
would break if the system name changed: **none**. The 13 files the brief counted match `__FreeBSD__`
the preprocessor macro (swanstation 0001/0002/0003, play 0002, flycast 0001, melondsds 0004) or the
word in prose about the kernel (flycast 0004/0005 parked, melondsds 0002/0003, play 0004,
swanstation 0006/0007); none contains `UNIX` or `CMAKE_SYSTEM_NAME`. The five patches that touch a
CMake file at all (dirksimple 0001 lutf8lib, melondsds 0001 libretro-common sources, 0002 a header
directory, 0004 FetchContent, play 0001 zlib's `HAVE_UNISTD_H`) test no platform identity.

### 2.3 Where the system name is load-bearing: the cores' own CMakeLists

The recipe (libretro-super `recipes/linux/cores-linux-x64-generic`, fetched, 184 lines) has 17
CMAKE-type cores; the seven this port has flags or patches for are dirksimple, flycast, melondsds,
play, swanstation, tic80 and trident. Their own CMake files were fetched by partial clone at today's
HEAD (flycast dd7a5f0, melonds-ds bc4e4b6, Play- 83700b2, swanstation b6c30a7, TIC-80 dc8ad33,
DirkSimple d5d75f9; Panda3DS local) and every `if()` naming UNIX, WIN32, APPLE, ANDROID, LINUX,
FREEBSD, BSD or CMAKE_SYSTEM_NAME was read in context. The result per core, under the three
candidate identities:

| core | arms reached | FreeBSD (today) | Orbis, UNIX unset (= Generic) | Orbis, UNIX=1 |
|---|---|---|---|---|
| **flycast** | `:327 if(UNIX AND NOT ANDROID AND NOT APPLE)` runs **`getconf PAGESIZE` on the host** and bakes it in as `PAGE_SIZE`; `:241 CMAKE_SYSTEM_NAME MATCHES "FreeBSD…"` → `find_package(OpenGL REQUIRED)` (dead under `-DUSE_OPENGL=OFF`); `:75`, `:558`, `:1433` are under `if(NOT LIBRETRO)` | ⚠ **PAGE_SIZE = 16384 on this Mac** (`getconf PAGESIZE` → 16384, arm64), 4096 on ubuntu CI. flycast's `core/stdclass.h:25` defaults to 4096 and the parked patch 0005 records that `common_linux_setup()` verifies it equals `sysconf(_SC_PAGESIZE)`, which answers 4096 on the console. A Mac-built flycast is wrong today and nothing in build-cores.sh or the patches addresses it | correct by accident (arm skipped) | same as today; needs the one-line upstream fix `AND NOT CMAKE_CROSSCOMPILING`, which is a genuine bug for every cross build of flycast |
| **melondsds** | `ConfigureFeatures.cmake:137 HAVE_DYNAMIC ← TARGET_SUPPORTS_SHARED_LIBS`; `:157` direct-mode networking needs `(WIN32 OR UNIX) AND HAVE_DYNAMIC`; `libslirp.cmake:58 if(UNIX) -DUNIX elseif(BSD) -DBSD`; `src/libretro:258` suffix `.so` under UNIX | HAVE_DYNAMIC on, libslirp gets `UNIX` | ⚠ HAVE_DYNAMIC off, libslirp compiled with **neither** `UNIX` nor `BSD` — a different core, probably a broken one | today's configuration, provided the module sets `TARGET_SUPPORTS_SHARED_LIBS TRUE` |
| **swanstation** | only `CMAKE_SYSTEM_NAME STREQUAL "Linux"` arms (`:141`, `src/common:144`) | indifferent | indifferent | indifferent |
| **tic80** | `:28` BUILD_STATIC default list (`ANDROID OR EMSCRIPTEN OR NINTENDO_3DS OR NINTENDO_SWITCH OR BAREMETALPI`) — why build-cores.sh:906 hand-passes `-DBUILD_STATIC=ON`; `:110 if(UNIX AND NOT APPLE AND NOT EMSCRIPTEN AND NOT ANDROID) set(LINUX TRUE)`, then `naett.cmake:12 if(LINUX) find_package(CURL)` (not found under `FIND_ROOT_PATH … ONLY` → `USE_NAETT FALSE`, correct), `quickjs.cmake:60 if(LINUX)` adds `_GNU_SOURCE _POSIX_C_SOURCE=200112` and links `m dl pthread`, `core.cmake:137` links `m dl` | builds as "Linux" | ⚠ `LINUX` unset → `USE_NAETT` stays TRUE and `naett.c` has arms for WIN32, LINUX and APPLE only; quickjs loses `_GNU_SOURCE` under musl. Would break | today's configuration |
| **dirksimple** | `:177 if(NOT CMAKE_SYSTEM_NAME STREQUAL "Windows")` | indifferent | indifferent | indifferent |
| **trident / Panda3DS** | glad `:7 elseif(ORBIS)` ahead of `elseif(NOT APPLE)` (else GLX and `<X11/X.h>`); dynarmic `:483 elseif (ORBIS)` ahead of `elseif (UNIX)`; `Panda3DS:802 if(LINUX OR FREEBSD)` is inside the Qt block (off); `Panda3DS:188 add_subdirectory(third_party/SDL2)` runs **SDL's own** CMakeLists | needs `-DORBIS=ON` (build-cores.sh:871); dynarmic arm load-bearing | glad arm needed, dynarmic arm dead; ⚠ **SDL's own CMakeLists fails to configure under Generic**: "Threads are needed by many SDL subsystems and may not be disabled" (`cmake/macros.cmake:36` via `CMakeLists.txt:3036`, measured with `-DSDL_TEST=OFF -DSDL_SHARED=OFF`) — so trident cannot be built with the kit toolchain today | ORBIS from the module, `-DORBIS=ON` deleted; dynarmic arm stays load-bearing; SDL configures (measured) — and generates its own `SDL_config.h` with **zero orbis drivers** (dummy video and audio, hidapi/virtual joystick, `SDL_LOADSO_DLOPEN`), `-DUSING_GENERATED_CONFIG_H`, so the fork's backend is not in the trident build under either name |

⚠ The titles were written against the opposite assumption. OpenGothic's CMakeLists.txt:28 says in
its own words that Generic leaves `UNIX` false "so none of the `-rdynamic` / `-lpthread -ldl` /
windows branches fire" (`:184`, `:251`); Tempest's `:357 elseif(UNIX)` (X11, Xcursor) is safe only
because `elseif(PS4)` precedes it. The SDK ships 8-byte empty `libpthread.a`, `libdl.a`, `libm.a`
and `librt.a` (measured, `~/.local/opt/openorbis/lib`), so `-lpthread -ldl` links either way;
`-rdynamic` becomes `--export-dynamic` on the eboot and its effect on `create-fself` is the one
unmeasured item in the title-side flip.

**Conclusion.** Neither Generic nor FreeBSD is the right answer. Generic breaks tic80, melondsds and
SDL-in-Panda3DS; FreeBSD sets `BSD`, takes `MATCHES "FreeBSD"` arms that mean "a desktop with X11
and libusb" (flycast :241, :506), and leaks the host page size. The measured target is one
`Platform/Orbis.cmake` that includes `Platform/UnixPaths` (Emscripten's choice, `Emscripten.cmake:48`
"good enough Linux/Unix emulation", read from the fetched file), sets `ORBIS 1` in
`Platform/Orbis-Initialize.cmake`, sets `CMAKE_DL_LIBS ""` (as FreeBSD.cmake and Emscripten.cmake
both do), and sets `TARGET_SUPPORTS_SHARED_LIBS TRUE` because this platform does load `.prx`
modules; the kit toolchain keeps `BUILD_SHARED_LIBS OFF FORCE` for titles as it does now. dynarmic's
seven lines remain load-bearing under it — that is the honest count, and §4 says where they go.

### 2.4 Migration path, not a flip

**Step 0 — ship the module, change nothing.** Add `cmake/Platform/Orbis.cmake` and
`cmake/Platform/Orbis-Initialize.cmake` to the kit; in `ps4-openorbis.cmake` add
`list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}")` (devkitPro's
`dkp-initialize-path.cmake` does exactly this with `${DEVKITPRO}/cmake`) and make line 31 read
`set(CMAKE_SYSTEM_NAME "${ORBIS_SYSTEM_NAME}")` with `ORBIS_SYSTEM_NAME` defaulting to `Generic`,
resolved like `OO_PS4_TOOLCHAIN` (`-D`, then environment, then default). Do the same three lines in
build-cores.sh's heredoc at :396 with the default `FreeBSD`. Nothing changes for anybody; the
falsifier is a consumer whose configure output differs with the module merely present (it should
not: the Platform file is only read for the name that selects it).

**Step 1 — titles and examples (the kit toolchain).** Flip `ORBIS_SYSTEM_NAME=Orbis` for `hello`,
`triangle`, `vkloader`, `services`, `orbis-compat/optional`, sonic3air `_ps4`, SDL2 `orbis/` — all
of which test only `PS4` (counts measured: UNIX 0 in each). Then OpenGothic and Tempest, where the
two `UNIX` arms above become reachable: measure `llvm-readelf --dyn-syms | wc -l` on
`Gothic2Notr.elf` before and after, run the bundle gate (which builds OpenGothic from an unpacked
copy), and boot the package once. If `--export-dynamic` changes the loader's verdict, the fix is
`AND NOT PS4` on OpenGothic:184 — two words in a file that already carries 14 `PS4` tests.
Falsifier for the whole step: any object list (`find build -name '*.o' | sort`) that differs
between the two configures of the same tree.

**Step 2 — cores, one at a time.** With build-cores.sh's default still FreeBSD, build each CMake
core with `ORBIS_SYSTEM_NAME=Orbis` and compare three things against the FreeBSD build of the same
upstream commit: the configure summary, the object count, and `llvm-nm --defined-only <core>.prx | wc -l`.
Expected from the table: swanstation and dirksimple identical; tic80 and melondsds identical (UNIX
is 1 in both); flycast identical on Linux CI and *different* on a Mac only through `PAGE_SIZE`,
which is the bug, not the migration; trident configures without `-DORBIS=ON` (delete build-cores.sh:871's
first flag) and its SDL2 subtree still contains no orbis driver. Per-core divergence beyond
`BSD`, `CMAKE_DL_LIBS`, `CMAKE_EXE_EXPORTS_C_FLAG` and `CMAKE_LINK_GROUP_USING_RESCAN` in a
`CMakeCache.txt` diff falsifies the table for that core and is the reason this is per-core.

**Step 3 — flip both defaults, then converge the two toolchains.** build-cores.sh generates its own
file for reasons that stay valid (`-nostdinc`, the try_compile link line, a module link it discards);
the end state is that its heredoc `include()`s the kit's Platform directory rather than duplicating
identity. Out of scope here; the migration above does not depend on it.

**What lever 2 buys, honestly.** One hand-passed flag (`-DORBIS=ON`), the ability to build trident
with the kit toolchain at all, the end of "System is unknown to cmake", and — the real reason — a
platform variable that upstream patches can be written against. SDL2's own CMakeLists keys its
console arms on variables the toolchains set (`if(VITA OR PSP OR PS2 OR N3DS)` at `SDL2/CMakeLists.txt:361`,
`VITA` set at `vita.toolchain.cmake:104`, fetched; devkitPro's Switch patch keys on
`NINTENDO_SWITCH`, fetched), and tic80's console list is such a variable list. Without `ORBIS` there
is nothing upstreamable to write; with it, tic80's `-DBUILD_STATIC=ON` becomes a one-word upstream
change and the glad arm becomes a two-line one.

---

## 3. Lever 3 — ship dependencies prebuilt

### 3.1 What the ports actually consume (grep of every CMakeLists and .cmake, vendored trees excluded)

- **SDL2**: sonic3air vendors a copy under `framework/external/sdl/SDL2` and its `_ps4` CMakeLists
  compiles it from a hand-written source list — the same ~60-line glob list as the fork's
  `orbis/CMakeLists.txt`, written twice. Panda3DS takes it as a submodule and runs SDL's own build
  (`CMakeLists.txt:188`), or, with `USE_SYSTEM_SDL2`, does `find_package(SDL2 CONFIG REQUIRED)` and
  links `SDL2::SDL2` (`:182-183`) — ⚠ so Panda3DS already contains the zero-line consumer of a
  config package. OpenGothic and Tempest reach it only through openal-soft's `find_package(SDL2 QUIET)`
  (audio is forced off on PS4). flycast, dirksimple and tic80 look for it only on non-libretro paths.
- **zlib**: Tempest `add_subdirectory("thirdparty/zlib")` (`Engine/CMakeLists.txt:84`) and libpng at
  `:93`; sonic3air vendors it; play patches its bundled copy (play 0001); ⚠ sonic3air's `_ps4` build
  already reuses **Mesa's** zlib by globbing `${ORBIS_MESA_BUILD}/subprojects/zlib-*/libz.a`
  (`build/_ps4/CMakeLists.txt:73`) — a prebuilt zlib exists in every bundle today with no package
  around it.
- **libpng**: only Tempest, vendored, and it is `find_package(ZLIB REQUIRED)` inside that vendored
  tree that a ZLIB package would satisfy. **freetype**: no consumer in the organisation. **fmt**:
  Panda3DS `add_subdirectory(third_party/fmt)`; the fork exists for clang ≥ 22 and is compiled into
  Panda3DS's own TUs, so a prebuilt fmt removes nothing.

So the evidence orders them SDL2, then zlib (formalising what sonic3air already does by glob), and
stops. libpng, freetype and fmt have no consumer that a package would change.

### 3.2 What `find_package(SDL2)` and `sdl2-config` need, and who can produce it

For a stranger's `find_package(SDL2 CONFIG)` and `pkg-config --libs sdl2` / `sdl2-config --static-libs`
to work unchanged, a prefix must contain `include/SDL2/*.h` (including `SDL_config_orbis.h`, which
`SDL_config.h` selects on `__ORBIS__`), `lib/libSDL2.a`, `lib/cmake/SDL2/{SDL2Config,SDL2ConfigVersion,SDL2staticTargets}.cmake`,
`lib/pkgconfig/sdl2.pc`, `bin/sdl2-config`, and — ⚠ the part a generic recipe would miss — the
three `PUBLIC` compile definitions `SDL_ORBIS_ENABLE_{AUDIO,JOYSTICK,VIDEO}` and the `INTERFACE`
link libraries `-lSceAudioOut -lSceUserService -lScePad -lSceVideoOut -lSceGnmDriver -lSceSystemService`
that `orbis/CMakeLists.txt` attaches to the target, which must appear in `INTERFACE_COMPILE_DEFINITIONS`
and `INTERFACE_LINK_LIBRARIES` of `SDL2::SDL2-static`, in `sdl2.pc`'s `Cflags:` and `Libs.private:`,
and in `sdl2-config --cflags` / `--static-libs`. A consumer TU that includes `SDL.h` without those
three defines compiles against a different SDL than it links (the fork's own comment says so).

Can SDL2's own install rules produce it? **Not today, from either side.** Upstream's CMakeLists has
the machinery (`configure_package_config_file` at `:3647`, `install(EXPORT SDL2staticTargets)` at
`:3681`, `sdl2.pc` at `:3730`, `sdl2-config` at `:3747`) but, measured under both FreeBSD and
Orbis+UNIX, it configures a dummy SDL with none of the orbis drivers and its own generated
`SDL_config.h`. The fork's `orbis/CMakeLists.txt` builds the right library and has no `install()`
at all. Two ways to close the gap:

- **(a) Kit-owned, now:** ~50 lines added to `orbis/CMakeLists.txt`: `install(TARGETS SDL2-static EXPORT SDL2staticTargets)`,
  `install(EXPORT …)`, a hand-written `SDL2Config.cmake` of the shape emscripten ships
  (`tools/ports/sdl2/sdl2-config.cmake`, 20 lines, INTERFACE IMPORTED targets; fetched), and
  `configure_file` of upstream's own `sdl2.pc.in` and `sdl2-config.in` with `SDL_STATIC_LIBS` set to
  the six Sce libraries. The toolchain gains `list(APPEND CMAKE_PREFIX_PATH …/portlibs)` — needed
  because `CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY` (`ps4-openorbis.cmake:357`) confines package search
  to `CMAKE_FIND_ROOT_PATH` — and `set(ENV{PKG_CONFIG_LIBDIR} …/portlibs/lib/pkgconfig)`
  (Emscripten does this at `Emscripten.cmake:275`; devkitPro ships a wrapper script
  `aarch64-none-elf-pkg-config` in `portlibs/switch/bin` instead, from `switch/pkg-config/PKGBUILD`).
  The bundle stages one more directory next to `sdk/ orbis-compat/ mesa/ toolchain/`
  (`make-sdk-bundle.sh:360`); verify-sdk-bundle checks paths by name, so an addition is safe.
- **(b) The upstream shape, later:** an `if(ORBIS)` block in SDL's own CMakeLists.txt plus
  `SDL_config.h.cmake` entries, so upstream's install rules do everything. This is what devkitPro
  carries for the Switch — an 88-line CMakeLists hunk and 57 lines of `#cmakedefine` inside a
  26,628-line patch on 41 files, applied at package build (`switch/SDL2/SDL2-2.28.5.patch`, fetched).
  It requires lever 2's `ORBIS` variable, and it is the form SDL upstream already accepts
  (`src/video/{vita,psp,ps2,n3ds}` and `src/audio/{vita,psp,ps2,n3ds}` are in the 2.32 tree). ⚠ vitasdk's
  `packages/sdl2/VITABUILD` (fetched) builds SDL 2.32.8 from the pristine upstream tarball with
  **no patch at all**, because the Vita backend is upstream; that is the only known way to make the
  2264 lines cost this organisation nothing.

Consumers after (a): Panda3DS `-DUSE_SYSTEM_SDL2=ON` and zero lines; sonic3air's `_ps4` CMakeLists
drops its source list (about 150 lines) for `find_package(SDL2 CONFIG REQUIRED)`; the `SDL2`
submodule pin in Panda3DS's `.gitmodules` reverts to upstream.

---

## 4. Lever 4 — patches belong to the kit

### 4.1 What is actually in the two forks

**dynarmic.** `orbis-ports/dynarmic` master is at `01eeffcb Fix lift_sequence`, and the GitHub API
confirms that commit is **upstream's** (Panda3DS-emu/dynarmic, author wheremyfoodat, 2026-07-11):
it adds the `std::integer_sequence` partial specialisation, +6 lines. ps4-support (`f010527c`)
solved the same compile error a month earlier by *replacing* the generic template (−2/+2), so
`git diff origin/master origin/ps4-support` now shows `lift_sequence.hpp | 6 ------`: ⚠ the "6
deleted lines" in the brief are ps4-support undoing upstream's fix, not a port change. The compile
error the brief reproduced is real against the pre-July tree and gone against master. Rebased, the
fork is exactly `src/dynarmic/CMakeLists.txt +7`, the `elseif (ORBIS)` arm — which stays
load-bearing under the §2 target (UNIX=1) and is the shape upstream would take if it takes anything.

**fmt.** `+5` in `include/fmt/base.h`: `FMT_USE_CONSTEVAL 0` when `FMT_CLANG_VERSION >= 2200`,
because clang 22+ rejects fmt's compile-time format check (the fork's own comment). This machine
runs Homebrew clang 23.1.1. fmt master has restructured (`base.h` is now a compatibility header for
`core.h`) and `core.h:118-136` has no clang-version arm; the guard chain has no `#ifndef`, so it
cannot be pre-empted by `-D`. This is a host-compiler defect report, not a platform patch.

**Panda3DS.** Three `.gitmodules` pins (dynarmic, fmt, SDL2 → orbis-ports), two glad lines, and
~300 lines of `__ORBIS__`-guarded renderer, shader-JIT and scheduler changes with hardware
measurements in their comments — upstream-shaped (the `config.hpp` line the fork edits already
carries `__ANDROID__ || __APPLE__`). **3dsTrident** is two lines that exist to point at that fork.

### 4.2 Whether a kit registry lets the forks stop being forks

Yes for three of the four, and the fourth shrinks to one kind of content:

- dynarmic → `patches/dynarmic/0001-orbis-generic-exception-handler.patch` (7 lines) applied to
  upstream's submodule; Panda3DS's `.gitmodules` pin reverts (−1 line). Fork deleted.
- fmt → `patches/fmt/0001-runtime-format-check-on-clang-22.patch` (5 lines) plus an upstream issue;
  pin reverts. Fork deleted.
- SDL2 → not a patch (2264 lines) but consumed as a portlib (§3), so the pin reverts; the branch
  `orbis-2.32` remains the library's home until (b) lands upstream. Fork stays, consumers don't see it.
- Panda3DS → glad's two lines become a registry patch or an upstream PR once `ORBIS` exists; what
  remains is the guarded renderer work, which is a PR to Panda3DS-emu or a fork that carries port
  code and nothing else. 3dsTrident's fork and the `trident` correction in `ps4/core-recipe-extra`
  then disappear together.

### 4.3 What the registry has to be to serve a project that is not a libretro core

`RetroArch/ps4/core-patches/` already has the right rules (its README: platform arms, header
spellings, dlopen assumptions; never harness bugs; every `--update` re-applies) and the right
provenance (`<commit>+<n>` in the manifest). It lacks four things a stranger's project needs:

1. **A key that is the upstream, not the frontend's name for it**: `patches/<project>/NNNN-<slug>.patch`
   with a `SERIES` file naming the upstream URL, the base ref, and per patch an `upstream-status:`
   of `local`, `sent <url>` or `merged <sha>` — so a merged patch is deleted by a script, not by
   memory. dynarmic's stale six lines are what happens without this field.
2. **An apply step reachable from CMake, not only from shell.** ⚠ melondsds 0004 shows the hole:
   FetchContent'd sources exist only after configure, so build-cores.sh's "reset, then apply" cannot
   reach them and the patch has to ride on `FetchDependencies.cmake`. A kit function
   `orbis_patch_source(<dir> <project>)` usable as a FetchContent `PATCH_COMMAND` and as an
   `ExternalProject` step, and a `scripts/orbis-patch.sh` for Makefile and submodule consumers, are
   the same registry seen from both sides.
3. **A CI check that every patch still applies to the ref its `SERIES` names**, which is what turns
   "a cost carried at every `--update`" into a red build instead of a surprise.
4. **A place**: `orbis-porting-kit/patches/` with `libretro/<core>/` underneath for the thirty that
   exist; build-cores.sh reads `$ORBIS_KIT_DIR/patches/libretro` instead of `ps4/core-patches`
   (one variable), and cores.yml already pins the kit ref through `setup-orbis`, so patches and
   harness version together as they do now.

---

## 5. Ranked recommendation — cheapest and safest first

**1. Delete what is already upstream or already dead.** Rebase `orbis-ports/dynarmic` ps4-support
onto master (drops the −6), which leaves 7 lines; file the fmt clang-22 defect upstream with the
fork's five lines as the proposed fix. Files: the two forks' branches, Panda3DS `.gitmodules`.
Blast radius: 3 repositories, 11 lines. Falsifier: `lift_sequence.hpp` from Panda3DS-emu/dynarmic
master failing to compile under this SDK's libc++ 11 — one `cmake --build` of trident with the pin
moved says so in a minute.

**2. Move `core-patches/` into the kit and give it the four things in §4.3.** Mechanical, no
behaviour change, and it is the precondition for retiring the dynarmic and fmt forks. Files:
`RetroArch/ps4/core-patches/**` → `orbis-porting-kit/patches/libretro/**`; build-cores.sh's patch
path; a `SERIES` per project; one CI job. Blast radius: 30 patches unchanged in content, one
harness variable. Falsifier: cores.yml unable to see the kit checkout — it cannot be, `setup-orbis`
exports `ORBIS_KIT_DIR` for every consumer already (handoff §1).

**3. Platform/Orbis, opt-in, then the migration of §2.4.** Files: kit `cmake/Platform/Orbis.cmake`,
`cmake/Platform/Orbis-Initialize.cmake`, `ps4-openorbis.cmake:31` and one `list(APPEND CMAKE_MODULE_PATH)`,
build-cores.sh:396 heredoc (+3 lines), then `:871` (drop `-DORBIS=ON`). Measured blast radius: 0 of
30 patches; of 7 CMake cores, 2 indifferent, 2 unchanged only because UNIX stays 1, 1 that starts
configuring at all (trident), 1 with a pre-existing host leak (flycast) that the migration neither
causes nor cures, 1 whose 7-line arm stays (dynarmic); 2 title-side arms (OpenGothic :184, :251)
become reachable and must be measured once on hardware. Falsifier: a `CMakeCache.txt` diff between
the FreeBSD and Orbis configures of any core showing more than the four variables named in §2.4 step 2.

**4. SDL2 as a portlib with a config package, `.pc` and `sdl2-config`, path (a) of §3.2.** Files:
`SDL2/orbis/CMakeLists.txt` (+~50), `ps4-openorbis.cmake` (+2: `CMAKE_PREFIX_PATH`,
`PKG_CONFIG_LIBDIR`), `make-sdk-bundle.sh` (+1 staging directory), then Panda3DS `-DUSE_SYSTEM_SDL2=ON`
and sonic3air `_ps4` (−~150). zlib follows the same shape, replacing sonic3air's glob of Mesa's
`libz.a`. Falsifier: a consumer that compiles `SDL.h` in its own TUs and gets a different
`SDL_config_orbis.h` than the library saw — the three `SDL_ORBIS_ENABLE_*` defines missing from
`INTERFACE_COMPILE_DEFINITIONS` or from `sdl2.pc`'s `Cflags` is the exact failure to test for.

**5. Send upstream what is already upstream-shaped**, in this order of acceptance likelihood:
flycast `CMakeLists.txt:327` (`AND NOT CMAKE_CROSSCOMPILING`; a real bug for every cross build, no
console knowledge needed); ZenKit's lazy VFS (753 platform-independent lines currently kept in a
fork); OpenGothic A1/A2 from `PLAN-opengothic-split.md`; tic80 `:28 OR ORBIS` and Panda3DS glad
`elseif(ORBIS)` once rank 3 exists; dynarmic `elseif (ORBIS)`; Panda3DS's guarded renderer work;
and, as the long project, the SDL2 backend (SDL2 is in maintenance; SDL3 is where a new backend
lands, which the handoff already schedules). Falsifier per item: a maintainer's no, which costs a
PR and returns the line to rank 2's registry with `upstream-status: sent`.

**6. Two overlay candidates, with a measurement before each.** A `sem_*` interposer (mutex+condvar)
deletes melondsds 0003 and SDL2's `SDL_syssem.c` exception — after `llvm-nm libc.a` shows no other
member calls `sem_*` internally, the `regcomp`/`mbtowc` shape. `orbis_jit` for callers of
`mmap(PROT_EXEC)`/`mprotect` (handoff item 1) — after counting which of the nine JIT patches go
through libc rather than through a port-owned arena (§1 says at least four do not).

---

## 6. What cannot be made zero-diff — stop chasing it

- **Preprocessor platform chains in the port's own sources**, eight patches, while libc++ undefines
  `__FreeBSD__` (`__config:12`). The only fix that is not a port-side `|| defined(__ORBIS__)` is
  rebuilding the SDK's libc++ without that line, which reopens every musl-versus-FreeBSD arm inside
  libc++ itself. Port-side, and upstreamable.
- **Kernel-truth code**: signal contexts (flycast 0001/0004, play 0004), fastmem handlers
  (swanstation 0006/0007, melondsds 0004), port-owned JIT arenas (swanstation 0001/0003/0004/0005,
  play 0005, mupen 0001/0003, pcsx_rearmed 0001, flycast 0002/0003). Fourteen of the thirty patches.
- **Product and performance decisions**: mupen 0002/0004–0008, Panda3DS's renderer and JIT lines,
  sonic3air's 779 engine lines and its 2067-line GL 1.1 shim, OpenGothic's 675 modified lines plus
  its `ps4/` directory, RetroArch's 3236 frontend lines (input, joypad, dylib, net). These are what
  a port *is*; the kit's job is that nothing else sits beside them.
- **The port's presence in the build**: sonic3air `build/_ps4/`, OpenGothic's `if(PS4)` block,
  Tempest's `elseif(PS4)`. Upstream sonic3air models platforms as exactly such directories
  (`_android`, `_emscripten` exist), so these are already in the accepted shape.
- **Compiler-version workarounds** (fmt): not the platform's, and ours until upstream takes them.

---

## 7. What other kits do that we have not considered

All from files fetched today unless marked; nothing else is installed on this machine
(`/opt/devkitpro`, `~/emsdk`, `/usr/local/vitasdk` absent; no `emcc`, `arm-vita-eabi-gcc`,
`aarch64-none-elf-gcc` on PATH).

**devkitPro** (`devkitPro/pacman-packages`, files `cmake/switch/{Switch,NintendoSwitch}.cmake`,
`cmake/common-utils/{Generic-dkP,dkp-toolchain-common,dkp-initialize-path,dkp-rule-overrides}.cmake`,
the PKGBUILDs, `switch/mesa/OpenGLConfig.cmake`):

- Toolchain and platform are two files installed under one prefix: `Switch.cmake` sets
  `CMAKE_SYSTEM_NAME NintendoSwitch` and includes `devkitA64.cmake`; `Platform/NintendoSwitch.cmake`
  includes `Platform/Generic-dkP` (no UnixPaths, so `UNIX` unset — the opposite of Emscripten),
  sets `NINTENDO_SWITCH TRUE`, `TARGET_SUPPORTS_SHARED_LIBS FALSE`, `CMAKE_EXECUTABLE_SUFFIX .elf`,
  and defines `nx_create_nro()` and friends — our `ps4_create_eboot()` is the same idea, in the
  toolchain file instead of the platform file.
- ⚠ `CMAKE_FIND_PACKAGE_PREFER_CONFIG TRUE` and `CMAKE_SYSTEM_PREFIX_PATH` pointed at the portlibs
  prefix, so a stranger's `find_package(ZLIB)` finds the portlib's config package before CMake's
  `FindZLIB` goes looking in the sysroot. We have neither; §3.2 adds the prefix.
- A `<triplet>-cmake` wrapper (3 lines: `exec env CMAKE_TOOLCHAIN_FILE=… cmake "$@"`) installed in
  `portlibs/switch/bin`; Emscripten's `emcmake` is the same 40 lines in Python. We have
  `scripts/orbis-new.sh`; a `ps4-cmake` wrapper would let `cmake -S . -B build` in any stranger's
  tree be the whole instruction.
- A `-pkg-config` wrapper per platform (its own package, `switch-pkg-config`).
- `DKP_PLATFORM_BOOTSTRAP` with `CMAKE_TRY_COMPILE_PLATFORM_VARIABLES` to build the libc itself with
  `STATIC_LIBRARY` try_compile *only when asked* — the trap build-cores.sh's comment at :368
  documents, handled as a mode rather than a rule.
- Rule overrides: `-O2` for Release instead of `-O3`, `.o` instead of `.cpp.o`, `-g` everywhere.
- Ports are PKGBUILDs building from upstream tarballs with `make install`, one patch per library,
  76 for the Switch; a hand-written `OpenGLConfig.cmake` (35 lines) for mesa, whose upstream ships
  none — the model for §3.2(a). The SDL2 port is a 26,628-line patch on 41 files kept in the
  registry and applied at build, keyed on `NINTENDO_SWITCH` in SDL's own CMakeLists — the model for
  §3.2(b).

**Emscripten** (`cmake/Modules/Platform/Emscripten.cmake`, `tools/ports/sdl2.py`, `emcmake.py`):
one file is both toolchain and platform module (it appends its own directory to
`CMAKE_MODULE_PATH` at :73 so CMake finds it as `Platform/Emscripten`); `set(UNIX 1)` on purpose;
`CMAKE_DL_LIBS ""`; ports built on demand by the compiler driver, with a 20-line hand-written
`sdl2-config.cmake` of INTERFACE IMPORTED targets and a generated `.pc`. ⚠ It sets
`CMAKE_CROSSCOMPILING_EMULATOR` to node (`emcmake.py:37-40`, `Emscripten.cmake:376`) so `try_run`
and `ctest` work under cross-compilation. Nobody has proposed that here; the handoff says `unemups4`
loads the plain ELF directly, which is exactly the shape of an emulator entry. Speculative, and
worth one afternoon: `CMAKE_CROSSCOMPILING_EMULATOR=<unemups4 launcher>` would run a stranger's
test suite from `ctest`.

**vitasdk** (`vita-toolchain/cmake_toolchain/vita.toolchain.cmake`, `vita.cmake`,
`packages/sdl2/VITABUILD`, package tree): `CMAKE_SYSTEM_NAME Generic` plus `set(VITA True)`;
136 package recipes (vdpm), built from pristine tarballs; SDL2 with **zero patches** because the
backend is upstream; helper functions `vita_create_self()` etc. in a separate included file, as our
`ps4-package.cmake` is. libretro-super already carries `recipes/playstation/{ps3,psp,vita,vita-cross}`
(tree listing), so `recipes/playstation/ps4` is the upstream home for `ps4/core-recipe-extra`'s
corrections, and RetroArch upstream already has `Makefile.orbis` at our merge-base.

**Not examined:** the same tree listing shows `cmake/wiiu/{CafeOS,WiiU}.cmake`, `cmake/3ds/`,
`cmake/wii/`, `cmake/gamecube/` — the layout repeats per console — but none of those files was
opened. PSL1GHT (PS3) was not looked at at all; anything said about it here would be from memory,
so nothing is.

---

## 8. Contradictions with the brief, collected

- Thirty patches across **8** cores, not 9 (`find RetroArch/ps4/core-patches -type f`).
- The kit toolchain sets a platform variable, `PS4`, at `ps4-openorbis.cmake:37`; it is `ORBIS` that
  nobody sets.
- The dynarmic mcl change is not unavoidable: upstream fixed it on 2026-07-11 (`01eeffcb`, in
  Panda3DS-emu/dynarmic) and the fork's master already carries that commit; ps4-support's −6 undoes it.
- SDL2's touch on upstream files is 41 lines in 10 files, not ~8; the fork's own header says 39.
- The FreeBSD name has a measured cost the brief did not list: flycast bakes the **host's** page size
  into the core under `if(UNIX)` (16384 on this Mac), and `check_symbol_exists` cannot tell that
  `sem_*` are broken because they link.
- A custom Platform module does not by itself change which arm upstream chains take; Generic already
  makes WIN32, APPLE and UNIX false. Its value is `ORBIS`, the decoupling from the FreeBSD name, and
  the choice of `UNIX` — and the measured right choice is `UNIX=1`, which two cores and SDL's own
  CMakeLists require and which the titles were written to avoid.
- SDL's own build under any non-Generic name produces a dummy SDL with none of the orbis drivers, so
  the trident core's SDL2 submodule is a dummy library today and the fork's backend reaches only
  sonic3air and whatever links `orbis/CMakeLists.txt` explicitly.
