# Splitting the OpenGothic diff: upstream, overlay, fork

The PS4 branch of OpenGothic carries **3770 insertions across 24 paths**. Most of it is genuinely
this title on this console. Some of it is not, and that part should leave: eight lines belong
UPSTREAM because they are wrong on every platform, ~135 belong to **orbis-compat** because they
correct the SDK rather than the game, and ~40 can be deleted outright because the overlay already
does the job.

Every item below carries the evidence that established it, so none of it has to be re-derived.

---

## A. Upstream — send to OpenGothic. No PS4 knowledge required to review any of it.

### A1. `game/gamemusic.cpp` — a buffer overflow, 2 lines

```cpp
void renderSound(int16_t *out, size_t n) override {
  if(!isEnabled()) {
-   std::memset(out, 0, n * sizeof(int32_t) * 2);
+   std::memset(out, 0, n * sizeof(int16_t) * 2);
```

Two providers, same mistake. The buffer is `int16_t`; the memset writes **twice** the promised
bytes whenever music is disabled. Platform-independent. This is the highest-value item here.

### A2. `game/resources.cpp` — 2.69 GB of I/O thrown away, 1 line DELETED

```cpp
- auto in = zenkit::Read::from(i.name);
```

`in` is never used - but that alone would not settle it, and the first version of this entry claimed
"delete it and nothing else changes" without checking. What settles it: `mount_disk` on the NEXT LINE
opens the same path by the same mechanism (`Mmap` under `_ZK_WITH_MMAP`, otherwise an `ifstream` sized
by `tellg()`), so every failure this call could surface - a missing file makes `(size_t)-1` and the
allocation throws - reaches the same two catch blocks from `mount_disk` anyway. The call guards nothing.

What it costs is a full read of every archive, discarded: 2.69 GB across fifteen files. Free while
`_ZK_WITH_MMAP` made it a mapping; **112 s measured** once it became a read.

⚠ The rest of that hunk (`VfsMountMode::LAZY` under `#ifdef __PS4__`) is FORK, not upstream.

### A3+A4. `compatibility/cpu32.h`, `compatibility/mem32.h` — a missing include, 1 line each

`#include <unordered_map>`. Both files use `std::unordered_map` (twice each, checked) without
including it. libstdc++ pulls it in transitively; libc++ does not. The code was always incorrect and
compiled by luck.

### A5. `compatibility/directmemory.cpp` — a C++20-only constructor, 2 lines

```cpp
- std::string_view(reinterpret_cast<const char*>(ptr), reinterpret_cast<const char*>(ptr)+size)
+ std::string_view(reinterpret_cast<const char*>(ptr), (size_t) size)
```

`basic_string_view(It first, End last)` is C++20 and needs `contiguous_iterator`. Verified absent
from this SDK's `<string_view>` (zero hits for `_It __begin, _End __end` and for
`contiguous_iterator`). The length form is C++17 and denotes exactly the same view.

### A6. `game/graphics/sceneglobals.cpp` — template deduction, 1 line

`std::max(std::log2(float(...)), 0.f)` cannot deduce when `log2` returns `double`. The added
`float(...)` is a no-op where it already compiled.

### A7. `shader/ssao/ssao.comp` — 30 lines, a real optimisation, and a SEPARATE conversation

A locally-declared array indexed by a run-time value (`pyramidId`) cannot live in registers on GCN,
so the compiler spills it: **112 B of scratch per thread, 25 spill stores**, and `ssao.comp` cost
**19.9 ms of a 63 ms GPU frame**. Moving it to shared memory changes where the 100 bytes live, not
what is computed — bit-identical output.

⚠ NOT a free win elsewhere: 64 threads x 25 floats x 4 B = **6400 B of LDS per group**, which can
cost occupancy on hardware that was not spilling in the first place. Send it as a measurement and a
question, not as a silent PR. Four amdgpu compiler flags were tried first and none changed a byte -
the decision is made when the local array is lowered, not in the scheduler.

---

## B. orbis-compat — corrections to the SDK, not to the game

### B1. Interpose `sysconf(_SC_NPROCESSORS_ONLN)` — removes ~60 lines from an upstream file

`std::thread::hardware_concurrency()` returns **1** on this console. Measured, printed to the
driver's log and read back. Consequence: every `parallelFor` in the title ran SERIALLY on six
available cores for the whole life of the port - animation alone was 27.2 ms of a 61.6 ms frame
while the process burned 0.87 cores.

⚠ **The mechanism is now known.** `nm -u` on `thread.cpp.o` inside the SDK's `libc++.a` shows
`U sysconf` - so `hardware_concurrency` is `sysconf(_SC_NPROCESSORS_ONLN)` (84 in this SDK's
`unistd.h`), and the SDK answers 1. An interposer in the overlay - the same mechanism as
`clock_gettime`, `stat`, `mmap` and `pthread_create` - fixes it for the title, the CTS and anything
else, instead of `#if defined(__PS4__)` in `game/utils/workers.cpp`.

The real answer comes from `scePthreadGetaffinity` + `__builtin_popcountll`: not how many cores the
chip has, but how many this process may run on.

⚠ **AND SIX WAS NOT BETTER THAN ONE.** Animation fell 27.2 -> 9.5 ms, but the frame went 61.6 ->
77.2 because the GPU's own time grew ~15 ms: six Jaguar cores compete with the GPU for the
cache-coherent bus, which is the one every surface this port touches sits on. So whatever ships must
keep a knob (`OG_WORKERS`) and the useful measurement is the CURVE - 1,2,3,4,6 - not the endpoint.

### B2. `ps4/cxx20shim/` -> the overlay — 75 lines

libc++ 11 ships a synopsis-only `<concepts>`: the header exists, declares nothing. That is an SDK
gap, and every C++20 consumer on this toolchain hits it - not just this title. It sits beside
`bits/alltypes.h`, `errno.h` and the rest of what orbis-compat already corrects.

### B3. While there: `ac_orbis_note`

`workers.cpp` reaches for `ac_orbis_note` (exported by the driver) to log its affinity finding. If
B1 moves the logic into the overlay, it reports through `orbis_log` like every other correction and
the title stops needing a driver symbol.

---

## C. Delete now — the overlay already does this, so there is nothing to replace it with

### C1. `game/utils/workers.{cpp,h}` — the `pthread_create` rewrite, ~40 lines

`trampoline`, `struct Start`, `thStarted[]`, and the change of `th[]` from `std::thread` to
`pthread_t`, all to ask for a **1 MB** stack because `std::thread` cannot.

⚠ **It has been redundant since the overlay's thread interposer landed.** `orbis_thread.cpp` raises
any request below the floor, and the floor is the MAIN THREAD's stack - **2048 KiB**, confirmed on
hardware today (`thread probe: floor is 2048 KiB`). 1 MB is below 2 MB, so the explicit request is
raised to 2 MB anyway; a plain `std::thread` gets the same 2 MB by the same path. The rewrite buys
nothing and costs ~40 lines of `#if defined(__PS4__)` in a file upstream owns.

Verify the same way today's removals were verified: rebuild and diff the symbol table. The only
symbols that may move are `Workers::trampoline` and the `std::thread` machinery.

---

## D. Stays in the fork — this title, on this console

    ps4/og_ps4_boot.{cpp,h}     1001   game-data discovery, the boot probes
    ps4/og_sound_orbis.cpp      1002   the sceAudioOut mixer
    ps4/og_sound_null.cpp        428   the control rung
    ps4/og_ps4_ime.{cpp,h}       323   the on-screen keyboard
    game/main.cpp                186   PS4 boot, the env file, idle-instead-of-return
    CMakeLists.txt               183   the PS4 block
    game/ui/gamemenu.cpp          98   wiring IME into the save dialog
    ps4/tempest-{env,pad}         195   configuration
    ps4/build.sh                  99   the cross build
    game/resources.cpp                 the LAZY arm - FULL freezes the machine, measured
    .gitmodules, lib/ZenKit            fork pointers

⚠ **`lib/Tempest`'s gitlink is stale**: the index says `61b58f71`, the fork's HEAD is `251ed657`.
A symlinked submodule cannot be updated with `git add`; it needs
`git update-index --cacheinfo 160000,<sha>,lib/Tempest`.

---

## Order, and why

1. **C1 first.** It is the only item that subtracts without adding, and it is verified by a symbol
   diff rather than by a console run.
2. **A1-A6 next.** Eight lines, no PS4 knowledge, two of them real bugs. Sending them is what makes
   the fork smaller permanently rather than just tidier.
3. **B1.** The largest single reduction of `#if defined(__PS4__)` in a file somebody else owns, and
   it hands the CTS a correct core count as a side effect - the same shape as the clock interposer.
4. **B2** whenever convenient. No urgency; it is already working where it sits.
5. **A7** last, as a measurement and a question rather than a patch.
