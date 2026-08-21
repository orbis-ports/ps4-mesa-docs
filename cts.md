# The Vulkan CTS on the PlayStation 4

The port itself lives in the CTS fork — `~/src-ps4/VK-GL-CTS`, branch `ps4-support`, one commit.
What is left here is the part a fork cannot hold: how to run it, and what it has found.

    deqp-args.txt   canonical dEQP arguments. Copy to /data/deqp-args.txt on the console
    deqp-env.txt    canonical environment.    Copy to /data/deqp-env.txt
    qpa-status.py   read a result file, including a killed run

⚠ **There is no install step any more, and the files it used to copy are gone from here.** The
platform file, the target file and every patch are in the fork, and `external/fetch_sources.py`
applies the one third-party patch (amber's) by itself. This directory used to hold drifted copies of
`tcuOrbisPlatform.{cpp,hpp}` and `orbis.cmake` that were OLDER than the fork's, plus an `install.sh`
that would have copied them over it — deleted 2026-08-21.

## Why this exists

Every measurement this driver has ever taken came from one game. That is a real risk and the
maintainer named it before the tooling did: a queue bound written as `count-3` worked because
OpenGothic asks for five swapchain images and would have deadlocked anything asking for three. A
driver that only works for the shape it was written against is a fixture.

The CTS is the answer that does not depend on taste: tens of thousands of named tests, each with a
verdict.

## Configuring

From the fork's root. ⚠ The authoritative copy of this recipe is the header comment of
`targets/orbis/orbis.cmake` IN THE FORK — it sits next to the variables it names, so it cannot go
stale the way the version that used to be here did (it still pointed at `~/src/Tempest` and at an
`ORBIS_CTS_RUNTIME` file that no longer exists).

    cmake -S . -B build-orbis -G Ninja -DDEQP_TARGET=orbis -DDE_OS=DE_OS_UNIX \
          -DCMAKE_BUILD_TYPE=Release \
          -DCMAKE_TOOLCHAIN_FILE=$HOME/src-ps4/orbis-compat/cmake/ps4-openorbis.cmake \
          -DORBIS_COMPAT_DIR=$HOME/src-ps4/orbis-compat \
          -DORBIS_VKLOADER_DIR=$HOME/src-ps4/orbis-compat/vkloader \
          -DORBIS_RADV_LIB=$HOME/src/mesa-ps4/build-orbis/src/amd/vulkan/libvulkan_radeon.a \
          -DORBIS_MESA_INCLUDE=$HOME/src/mesa-ps4/include

⚠ `-DDE_OS=DE_OS_UNIX` cannot be moved into the target file: delibs decides before the target is
read. `targets/orbis/orbis.cmake` refuses rather than letting it be wrong silently.

⚠ **Before ever running `external/fetch_sources.py`, check every external for local work**, because
it does `git reset --hard HEAD` on each one:

    for d in external/*/src; do [ -d "$d/.git" ] && \
        echo "$d $(git -C "$d" status --porcelain | wc -l)"; done

On 2026-08-21 that check found TWO undocumented modifications to third-party trees. `amber` was
load-bearing and is now a tracked patch; `jsoncpp` turned out to be dead (measured: its own library
and the only other consumer, Vulkan SC's `vksJson.cpp`, both build clean without it) and was
reverted. Anything that survives a fetch has to be wired through `GitRepo(..., patch = "...")`.

## What was already there, which is most of it

* **A CMake toolchain for the console** — `orbis-compat/cmake/ps4-openorbis.cmake`, proven by the
  whole title building with it. ⚠ It links `liborbis-compat.a` with `--whole-archive` for EVERY
  consumer, which is how the `pthread_create` interposer that raises thread stacks to 2 MB gets into
  deqp-vk without the target file mentioning it.
* **`tcu::StaticFunctionLibrary`** — in the framework since 2014. Every other platform dlopen()s
  `libvulkan.so.1`; here RADV is an archive with `ps4-vkloader` in front of it, so the five
  global-level entry points are ordinary symbols and their addresses go straight into a table.
* **A console-only platform shape** — the `vanilla` platform and the `vulkan_headless` target. No
  window system is a case upstream already supports; nothing is being faked.

So the port is a platform file and a target file, not a fork.

## The traps, in the order they were hit

**`DE_OS` was `DE_OS_VANILLA`.** The toolchain sets `CMAKE_SYSTEM_NAME=Generic` because CMake has no
PlayStation, and delibs then selects a target with no files, no clock and no threads. Wrong about
this console: it is a FreeBSD kernel with a musl libc and pthreads, mmap and file I/O all work.

**`__FreeBSD__` IS DEFINED AND MEANS NOTHING HERE — and worse, it depends on the language.** The
triple is `x86_64-pc-freebsd12-elf` so clang defines it. But this toolchain compiles C++ with a
forced `-include stdlib.h`, and something in OpenOrbis's header chain **undefines it**. Verified with
`-dM -E`: present without that include, gone with it. The C flags carry no such include, so:

    C   translation units:  __FreeBSD__ defined
    C++ translation units:  __FreeBSD__ NOT defined

Third-party code keying on it therefore takes different branches in C and C++ in the same build.
`__PS4__` is passed on the command line and cannot be taken away; it is the marker to use.

**musl is not FreeBSD's libc.** `malloc_usable_size` lives in `<malloc.h>` here, not where the
FreeBSD arm looks. `execinfo.h` does not exist at all, so there are no backtraces in the crash
handler. POSIX interval timers deliver through `SIGEV_THREAD`, and `struct sigevent` here has no
`sigev_notify_function` — deqp's own thread-based timer is the right arm and it is a real
implementation, not a stub.

**`memfd_create` does not exist**, so `dEQP-VK.memory.map_placed.*` reports NotSupported rather than
testing `VK_EXT_map_memory_placed` a way this console cannot set up.

## Two findings that are not the CTS's fault

Both are worked around in `orbisCtsRuntime.c` so that making the CTS build cannot break the title,
and both belong somewhere else:

1. **`ps4-vkloader` has no `vkEnumerateInstanceVersion`.** `vkthunks.c` generates a thunk for every
   entry point it knows about and this one is missing. It is the function an application calls to
   learn whether the loader supports Vulkan 1.1 *before* creating an instance; RADV answers it
   perfectly well and only the forwarding is absent. Any application that asks gets an undefined
   symbol. **The fix belongs in `Tempest/ps4/vkloader`.**

2. **This libc has no `__cxa_thread_atexit_impl`.** libc++ needs it for `thread_local` objects with
   non-trivial destructors; the title has never declared one and dEQP does. The stub here registers
   nothing, so those destructors never run — a leak that is acceptable in a test binary and would
   not be acceptable in a driver, which is why it lives in a file only the CTS links.

## Packaging and running

    cd <cts>/build-orbis/external/vulkancts/modules/vulkan
    OO_PS4_TOOLCHAIN=~/.local/opt/openorbis ~/.local/opt/openorbis/bin/linux/create-fself \
        -in=deqp-vk -out=deqp-vk.oelf --eboot eboot.bin --paid 0x3800000000000011
    mkdir -p pkgout && ~/src/Tempest/scripts/ps4/make-pkg.sh \
        --eboot eboot.bin --out-dir pkgout --title-id TMPS10022 --title "Vulkan CTS"

**107 MB**, against 48 MB for the whole game. Stripping debug information saves 0.7 MB of it, so that
is real code: every test, glslang and SPIRV-Tools in one binary.

⚠ **There is no command line on a console.** `tcuMain.cpp` reads `/data/deqp-args.txt` instead, one
argument per line, when the process is given no argv — the same shape as the title's environment
file, and for the same reason: a run is configured by putting a file on the console over FTP.

⚠ **`--deqp-log-filename=/data/...` is not optional.** deqp writes `TestResults.qpa` into the working
directory, which is not writable here; without it the run produces nothing anyone can read.

## It runs

    dEQP-VK.api.smoke.*   Passed: 6/6 (100.0%)
    deviceName: PlayStation 4 Liverpool (RADV ORBIS) (RADV LIVERPOOL)

Six tests, six passes: `triangle`, `asm_triangle`, `asm_triangle_no_opname`, `create_sampler`,
`create_shader`, `unused_resolve_attachment`. Real draws through the real driver on the real console,
with a result file to read.

## The last thing in the way was a four-byte type

⚠ **`sizeof(pthread_mutexattr_t)` is 4 on this console, and the implementation behind it writes
more.** Creating one mutex cleared 24 bytes of an unrelated heap block 4704 bytes away; the identical
four calls written out in the caller's own stack frame did no damage at all. Same code, different
frame, different victim - which is what a small stack overflow looks like and nothing else does.

`deMutexUnix.c` gives the attribute a padded home and the corruption stops. That was posted as a
prediction before the run rather than as a patch that happened to work: open-coding the mutex in the
caller would also have "worked" and would have taught nothing.

⚠ **This is not dEQP's problem.** Anything on this console that puts a `pthread_mutexattr_t` on the
stack has four bytes of somebody else's frame at risk, and that includes Mesa - `u_mutex` uses
attributes for recursive mutexes. OpenGothic has never hit it because its mutexes are static
initialisers; the CTS is the first thing in this port to call `pthread_mutexattr_init` at all.

**Five wrong diagnoses preceded it**, each refuted by measurement rather than by argument: file
permissions, the libc heap (512 MiB free), the XML writer, an allocator handing out overlapping
blocks, and the mutex's own allocation overflowing. The error message said "Unable to create output
XML writer to file" and was about none of those.

## The first full run: 6590 tests

    Passed:        5339/6590 (81.0%)
    Failed:         443/6590  (6.7%)
    Not supported:  808/6590 (12.3%)

⚠ **Every one of the 443 failures is in TWO families**, and both are this port claiming something it
cannot do. Nothing else failed at all.

### 1. Device memory does not come back zeroed — 433 failures

    dEQP-VK.memory.zero_initialize_device_memory.image_transition   369
    dEQP-VK.memory.zero_initialize_device_memory.clear_buffer        64
    "Memory type 2 failed / Memory type 5 failed"

`radv_physical_device.c:1626` sets `.zeroInitializeDeviceMemory = true` **unconditionally**, for every
driver. On amdgpu that is safe because the KERNEL clears a page before it reaches another process.
There is no kernel doing that here: the arena is one allocation this process carves up and hands out
again, so a new buffer can read the previous tenant's contents.

⚠ **And it matters less than the first version of this paragraph claimed.** I wrote that it was a
disclosure the moment anything else shared the console; the maintainer pointed out that nothing else
shares this console, and he is right. That framing is borrowed from PC drivers where processes really
do share a GPU, and it does not apply here.

What is left is smaller and still true: **Vulkan does not promise zeroed memory by default.** The
promise exists only when an application asks for this extension, and nothing that runs on this
console asks. So the practical impact today is nil, and this is a conformance failure rather than a
broken program.

That decides the fix. Clearing every range as it is handed out costs a memset on a path that runs
thousands of times, paid by everything, for a promise nobody here collects. Making the advertisement
conditional costs one line. **Stop claiming it** - the same answer as `has_userptr` below, and as
the queue priorities earlier.

### 2. Host pointers cannot be imported — 10 failures

    dEQP-VK.memory.external_memory_host.*   VK_ERROR_INVALID_EXTERNAL_HANDLE

⚠ **The driver code here is already correct and says so.** `ac_orbis_drm.c:6575` refuses any pointer
outside the arena and its comment explains exactly why this fails:

> Refusing loudly matters more than usual: radeon_info reports `has_userptr = 1`, derived by
> `ac_gpu_info` from the DRM version rather than probed, so RADV believes this works.

So the refusal is right and the advertisement is wrong. `EXT_external_memory_host` is gated on
`pdev->info.has_userptr`, which nothing ever measured. Setting it to 0 removes the extension and the
ten failures with it - the same shape as the queue-priority fix: **stop claiming, do not start
faking**.

## After both claims were dropped: zero failures

Same 6363 tests, like for like:

| | before | after |
|---|---|---|
| Pass | 5112 | 5105 |
| NotSupported | 808 | 1258 |
| **Fail** | **443** | **0** |

⚠ **And the prediction was not quite right, which is the interesting part.** I expected the 443
failures to become NotSupported and nothing else to move. 450 moved: the 443, plus **seven tests that
used to PASS**.

All seven are `zero_initialize_device_memory.image_transition.*` on 1x1 images - inside the very
family that was turned off. They were passing by luck: a 1x1 allocation is small enough to land on
arena pages nothing had used yet, so the memory happened to be zero. The driver was not keeping the
promise for them either; it was being lucky, and a lucky pass is indistinguishable from a real one
until something moves.

Nothing outside the two families changed at all, which is what the full re-run was for.

## What the CTS cost and what it found

Six wrong diagnoses before the first test ever ran, every one killed by measurement rather than
argument: file permissions, the libc heap (512 MiB free at the time), the XML writer, overlapping
allocations, a mutex overflowing its own block, and a self-test of mine that reported OK on a broken
futex because it checked that a thread FINISHED rather than that the calls WORKED.

Then, in the runs themselves:

| finding | status |
|---|---|
| context priorities advertised on a console with one submission path | fixed |
| BO table fixed at 4096 | grows |
| overlap table fixed at 8192, and it had gone silently blind | grows |
| `pthread_mutexattr_t` is 4 bytes and the implementation writes more | padded |
| `_umtx_op` is ENOSYS, so every contended `simple_mtx` in Mesa was a spin | replaced |
| nonexistent instance layer names accepted | open |
| allocation callbacks not called for descriptor set layouts | unattributed |
| device memory not zeroed | fixed - claim dropped |
| host pointer import advertised and impossible | fixed - `has_userptr` was derived, now false |

Two of those - the futex and the mutex attribute - had been wrong since the first day of the port and
were invisible because nothing it ran was contended or multithreaded.

## Still to do

* **Scope.** The full CTS is tens of thousands of tests and this is a 1.6 GHz Jaguar. Start with
  `dEQP-VK.api.smoke.*`, which answers the only question worth asking first — does it start, create
  an instance and run a test. Then `dEQP-VK.api.*` and `dEQP-VK.memory.*`: that is where a driver
  doing its own memory management, descriptors and queues will fail first.
* **Test data.** Some groups read files from the CTS data directory. `--deqp-archive-dir=` points at
  it; nothing has needed it yet because the smoke tests do not.

## The multithreaded family: 134/134, and it took four defects

`dEQP-VK.api.object_management.multithreaded_*` is **134 tests, all passing**. That family had never
completed once: `multithreaded_per_thread_device.descriptor_pool` hung on its first attempt and was
cut from the case list, and getting it back cost four separate defects and eleven runs.

**1. The arena's once-guard set its flag before doing the work.**

    static bool done = false;
    if (done) return orbis_va_end > orbis_va_base ? 0 : -ENOMEM;
    done = true;
    ... take direct memory, map it, set orbis_va_base and orbis_va_end ...

No lock, no atomic. Two threads in `vkCreateDevice`: the first claims the flag and starts building,
the second sees it and returns immediately on a window that is still zero. Now a proper once-init.

**2. The default thread stack on this console is 65536 bytes.** Measured, printed by `tcuMain`.
`radv_graphics_shaders_compile`'s frame alone is about 72 KB, and it dies inside a `memset` of 12864
bytes with the write faulting 0x323f in. Linux gives 8 MB. `deThread` never asked for a size, so it
took the default. ⚠ **This is not a dEQP bug** - anything here that creates a thread and compiles a
pipeline on it has the same cliff, including any multithreaded title.

**3. `_umtx_op` is ENOSYS**, so every contended `simple_mtx` in Mesa was a spin - recorded above.

**4. The protect-freed scheme cannot take this workload.** `ORBIS_FLAT_ARENA=1` is what walked the
run past two of the walls, and it is **not a fix**: the knob's own message says stale accesses go
silent again. With zero submissions the retire drain's guard `(int32_t)(label - newest) < 0` is never
true, so it runs `sceKernelMprotect` on every unmap instead of about once a frame. The repair is to
stop making one syscall per range - batching, or draining once per submit. **Still open.**

⚠ **AND THREE OF THE ELEVEN RUNS WERE READ AS HANGS THAT WERE CRASHES.** The klog receiver had been
dead since the console was switched off overnight - its *process* was alive, so it looked healthy,
and it had received nothing for twelve hours. The rule in the handoff is "check the receiver is
GROWING"; checking that it exists is not the same test. Three separate diagnoses were built on
frozen logs before the receiver came back and named the third wall in one line.

⚠ **AND ONE INSTRUMENT WAS BUILT ON A WRONG READING OF ANOTHER.** The watchdog's silence was
attributed to a jammed libc `FILE` lock, and a whole package went on giving it a private `write(2)`
descriptor. The descriptor was worth having - it is what proved the freeze was not the logging - but
the process had simply died. An instrument built for the wrong reason can still be the right
instrument, and that is luck rather than method.

**The instruments that survived**, in `ORBIS_WATCHDOG=<seconds>`: the arm's eight locks record their
holders; a breadcrumb names where the holder of `orbis_map_lock` or `orbis_kernel_mem_lock` is; and
the futex shim registers every thread asleep in `futex_wait`, which covers every contended
`simple_mtx` in Mesa without touching a single call site.

⚠ **None of them named a culprit.** Each said "not here", which is what narrowed the search - but
the defect that mattered was found by splitting the test family three ways and noticing that only
the device-per-thread variant hung. **The experiment did the work; the instruments kept it honest.**

## synchronization: 1899 run, 711 pass, 1185 not supported, 3 fail

First run of this family. The three failures are all in **synchronization2** and the sharpest
evidence is a matched pair:

    synchronization.basic.event.device_set_reset     Pass     (the v1 entry point)
    synchronization2.basic.event.device_set_reset    Fail     (the same test, v2)

    synchronization2.basic.event.none_set_reset            Fail
    synchronization2.basic.binary_semaphore.none_wait_submit  Fail

Both event failures say `Event should be in signaled state after set` - `vkGetEventStatus` does not
see an event that the GPU set. The v1 path does it correctly, so this is `vkCmdSetEvent2` and not
events in general.

⚠ **The mechanism is not established.** A NONE-stage-mask explanation is tempting because two of the
three are `none_*`, and it is weakened by the run itself: `synchronization2.none_stage` is 138 pass,
78 not supported, **zero failures** - NONE masks in image barriers work.

⚠ **AND THE FIRST ATTEMPT NEVER GOT PAST TEST ONE.**
`synchronization.basic.binary_semaphore.chain` exhausted a fixed 1024-entry syncobj table, returned
VK_ERROR_OUT_OF_HOST_MEMORY out of vkCreateFence, and dEQP - which treats a ResourceError as fatal
to the session - ended 1899 tests after one. That table now grows, and it is the **third** fixed
ceiling this suite has walked into after the BO table (4096) and the live-mapping table (8192).
Every one of them was a number large enough for a game.

**A prediction written before the run held exactly**: `global_priority_transition` is 396/396
NotSupported. This driver refuses a non-zero context priority with -EINVAL because there is nothing
to set on this console, and the refusal reaches the layer that answers the query - so the driver
claims nothing it does not do. Had any of the 396 passed, that would have been the same defect shape
as the two capability claims that produced 443 failures in `memory.*`.

**Left out of this run and why**, because a partial pass reads as a clean bill of health otherwise:
`op.*` (22k, the bulk, wants its own run), `cross_instance.*` (62k, needs external memory and
semaphores this port does not have), `timeline_semaphore` (5.7k, the sync provider leaves
timeline_wait NULL on purpose so the extension is not advertised), `win32_keyed_mutex`,
`signal_order.*`.

## The survey: what does not work, as of the 810 tests that ran

A 25613-test sweep across twenty small and medium packages, to produce a LIST rather than a fix. It
reached 810 and the console killed the process with a GPU timeout. Everything below is from those
810, which is 3% of the sweep and a fifth of a percent of the suite - so this is a first page, not a
map.

**1. Conditional rendering does not suppress work.** 48 failures, and the split is the finding:

    expect_execution    75 pass,   0 fail
    expect_noop         24 pass,  48 fail

Zero failures where work must run; 48 where it must be SKIPPED. That is the exact signature of
predication having no effect at all - with no predication, everything executes, so every
"expect_execution" passes for the wrong reason and every "expect_noop" fails.

⚠ This sits on known ground rather than new: `SET_PREDICATION` with `PREDICATION_OP_BOOL64` is
undefined on GFX6/7 and this port already narrowed it once. VK_EXT_conditional_rendering is
advertised; on this evidence it should either work or not be advertised.

**2. `conditional_rendering.draw.*.draw_indexed_indirect_count` hangs the GPU.**

    exception: 0xa0d0c00a (CPU_FAULT_SUBMITDONE_TIMEOUT_IN_SUSPEND_ASYNC)
    (Async exception from GPU/system. No thread information.)

Not a CPU fault and not a lock: a submission that never completed and a system that timed it out.
An indirect draw with a count buffer is the one thing that test does which the passing variants of
the same family do not.

**3. dEQP's 1719 data files were never on the console**, and the gap read as a driver defect:

    Failed to open file: './vulkan/dynamic_state/VertexFetch.vert'   ->   ResourceError

⚠ **A ResourceError here has meant three different things in one day**: the arena exhausted by
max_concurrent, a 1024-entry syncobj table exhausted by binary_semaphore.chain, and now a missing
file. Reading the status without the text would have put a driver defect on this list that does not
exist. The archive now lives at /data/deqp-data (1719 files local, 1719 there, counted) and the run
takes --deqp-archive-dir.

**Predictions that held**, written before the runs: `global_priority_transition` 396/396
NotSupported, so the driver's refusal of context priority reaches the query layer; `tessellation`,
`protected_memory` and `drm_format_modifiers` NotSupported throughout.

**Still unrun**: 24803 of the sweep, and the giants entirely - api (267k), binding-model (150k),
transform-feedback (134k), fragment-shading-rate (110k), robustness (99k), synchronization2 (82k),
renderpasses (81k), and the rest.

## The whole conditional-rendering family, as a baseline

906 tests - `api.smoke` as a control plus every `dEQP-VK.conditional_rendering.*` case in the
mustpass except the 130 `indirect_count` variants, which are left out because they hang the GPU and
one hang ends the session. **All 906 carried a verdict**: the family completes on this driver, which
the survey never got to see because it died inside it.

| | Pass | Fail | NotSupported |
|---|---|---|---|
| `api.smoke` | 6 | 0 | 0 |
| `conditional_ignore` | 80 | 0 | 78 |
| `clear_attachments` | 12 | 12 | 36 |
| `dispatch` | 64 | 58 | 158 |
| `draw` | 56 | 56 | 144 |
| `draw_clear` | 62 | 77 | 0 |
| `transform_feedback` | 0 | 7 | 0 |

And the split that is the actual measurement:

    expect_execution   128 pass,   0 fail
    expect_noop         24 pass, 104 fail

Zero failures where predicated work must RUN, 104 where it must be SKIPPED. The same shape the
survey found at 75/0 and 24/48, now over the whole family instead of the part of it a dying sweep
reached.

⚠ **This run measured the OLD behaviour on purpose-by-accident, and the log is what said so.**
`d07de333fac` split the suppression knob per caller and never called the new function; the original
whole-function gate sat untouched below it. The console printed

    radv/orbis: SET_PREDICATION is SUPPRESSED (the default here)

where the new path prints `predication mode is COND`. One line of driver log separated "the change
did not work" from "the change did not run", and only the second is true. Wired in by `199ccac361b`.

**Two traps for the file on the way there**, both mine, both cheap to avoid next time:

* **dEQP's shader cache is ENABLED by default** at the relative path `shadercache.bin`.
  `vkPrograms.cpp` then asks whether its parent directory exists, gets `"."`, and calls
  `createDirectoryAndParents(".")` - whose loop walks up until the path stops changing, which for
  `"."` is immediately, so `DE_CHECK_RUNTIME_ERR` kills the run at the FIRST test. Every run needs
  `--deqp-shadercache=disable` in `/data/deqp-args.txt`; reconstructing that file from memory
  without it cost a console run.
* **Point `MESA_LOG_FILE` at a new name per run.** The previous run's log is the baseline for the
  comparison being made, and the default name overwrites it.

## Conditional rendering: 210 failures to 21, in three runs

Same 906 tests each time, one variable changed per run.

| | `none` | `cond` | `condexec` | `condexec` + PFP sync |
|---|---|---|---|---|
| Pass | 280 | - | 369 | **469** |
| Fail | 210 | - | 121 | **21** |
| `expect_execution` | 128 / 0 fail | - | 128 / 0 fail | **128 / 0 fail** |
| `expect_noop` | 24 / 104 fail | - | 76 / 52 fail | **128 / 0 fail** |

`cond` has no numbers because it hung on the first predicated test - that is its entry.

**1. SET_PREDICATION hangs this CP, for conditional rendering exactly as for meta.** The run stopped
on `clear_attachments.condition_host_memory_expect_execution`, the first test in the family that
predicates anything, with the six unpredicated `api.smoke` controls ahead of it passed. Five samples
over three minutes had the result file, the driver log and the trace identical to the byte.
⚠ The test that hung is an **expect_execution** one - the predicate says RUN. A CP mis-evaluating a
condition draws the wrong thing; one that does not implement the opcode stops.

**2. COND_EXEC is the way round it, and a better fit than SET_PREDICATION ever was.** It skips a
counted run of dwords when the value at an address is zero, and the Vulkan predicate is 32-bit - so
the whole BOOL64 widening that upstream needs on pre-GFX9 parts falls away: no upload allocation, no
COPY_DATA, no PFP sync on the begin path. `radv_cs_emit_compute_predication()` already did this for
MEC, inverted conditions included, because the compute queue never had predication either.

**3. Then the PFP was reading the inverted predicate before the ME wrote it.** 85 of the 121
remaining failures were the inverted cases - every one of them and almost nothing else. The
inversion writes a scratch dword with COPY_DATA, which the **ME** executes, and reads it back with
COND_EXEC, which is a **PFP** packet, because skipping dwords is something only the stage that
fetches them can do. `PFP_SYNC_ME` between the two closes the gap. ⚠ WR_CONFIRM makes that write
reliable, not early - it was already there and did not help.

### What is left, in three groups that do not overlap

**`dispatch.condition_size.*` - 6.** dEQP's own comment says the group exists to verify "the
condition is interpreted as a 32-bit value", and it splits: `0x00000001` executes and passes;
`0x00000100`, `0x00010000` and `0x01000000` are all skipped. ⚠ **Four data points and no
documentation** - nothing in the Mesa tree defines COND_EXEC's fields beyond the opcode - so "it
tests bit 0" is the reading that fits rather than a citation, and the low byte fits equally. Any
predicate of 0 or 1 is unaffected, which is everything else in the family.

**`draw_clear.clear.depth.*discard*` - 8.** All depth, all with a discard, and both the `invert` and
`no_invert` variants fail, so the inversion is not it. The colour equivalents all pass. A depth
clear that does not go through a draw packet would not be covered by any of the wrapping above.

**`transform_feedback.*` - 7, the whole group.** ⚠ **Not a predication failure at all**: it was 0
pass / 7 fail in every one of the three runs, including the baseline where nothing predicated.
Whatever this is, it was there before and none of this work touched it.

## Multiview: the feature works, three of its combinations do not

946 tests, all with a verdict - 769 pass, 63 fail, 114 NotSupported. Reached by taking the blockers
out one at a time, which took five runs and is the whole story of the family.

**Multiview itself is fine: 703 passing.** What fails is what it is combined with, and each was
separated by pairing the combination against the same feature on its own:

| combination | result | the control that separated it |
|---|---|---|
| multiview alone | 703 pass | - |
| + geometry shader | **63 fail, 0 pass** | plain geometry is in use in retail captures |
| + indirect draw | **20 fail, 0 pass** | direct-draw multiview passes 703 |
| + multisample | **halts the run** | plain multisample: 60 pass, 0 fail |
| + tessellation | NotSupported | task #21, and the gate below |

⚠ **Every one of the 63 failures mentions geometry, and nothing else does.** Not a mixed bag with a
theme - a single cause with no exceptions: `index.geometry_shader`, `input_attachments_geometry` and
`secondary_cmd_buffer_geometry`, across `dynamic_rendering`, `renderpass2` and the default path
alike. `multiviewGeometryShader` is advertised true and is left that way deliberately: geometry
shaders themselves are not in question on this silicon.

### Four runs were spent on halts, and three of them were configuration

**Two runs halted on the same tessellation test** and neither said so - it had to be read out of the
last unterminated block, and the first reading of it was wrong. The cause was our own switch:
`ORBIS_NO_TESS` turned `.tessellationShader` off and left `.multiviewTessellationShader` true, and
dEQP gates `multiview.*.tessellation_shader.*` on the second and nothing else. ⚠ **A feature that
depends on another has to follow it, or the switch only looks like one** - the same shape as the
predication workaround being wider than its evidence, in the other direction.

The giveaway was in what the results did NOT contain: zero tessellation cases reported
NotSupported. A refusal that never reaches the query layer is not a refusal.

⚠ **The other two halts were a missing line in a hand-rebuilt config** - the shader cache and the
log filename. That is why `deqp-args.txt` and `deqp-env.txt` now live in this directory with the
reason for each line next to it, and why the args carry `--deqp-crashhandler` and `--deqp-watchdog`:
neither lets a run continue, but both make it write down which test killed it instead of leaving
that to be inferred.

## The multiview combinations, read on the laptop

**Geometry (63 failures).** ⚠ **The likeliest explanation is not multiview at all: OpenGothic has
ZERO `.geom` shaders**, so the geometry path in this driver has in all probability never executed on
this console. The multiview tests are simply the first thing the CTS ran that uses it.

⚠ **And that corrects a claim made in this repository yesterday.** "Geometry shaders are not in
question on this silicon - retail captures show ES/GS/VS in use" is evidence about the HARDWARE, not
about this driver's path through it. Those are different claims and they were run together.

What the path involves on GFX7, which is the legacy one: an ES/GS pair writing through the ESGS and
GSVS rings, plus a **GS copy shader** that reads the ring back. `radv_emit_view_index` handles that
copy shader explicitly (radv_cmd_buffer.c:11229), so the multiview-specific part looks right; the
rings themselves are upstream RADV with nothing orbis-specific anywhere near them.

**The decisive test is one case list**: plain `dEQP-VK.geometry.*` - 200 tests, no multiview
involved. If they fail too, the finding is "geometry shaders do not work on this port", which is
much larger than a multiview footnote and belongs at the top of the list rather than in it.

**So `multiviewGeometryShader` stays advertised for now.** Dropping a claim is the right answer three
times over on this port, but only after the thing has been measured on its own - and twice already
something was disabled here that turned out to be fine.

**Multisample (halts the run).** Not investigated on hardware; plain multisample passes 60/0, so the
combination is the target. ⚠ The suspect worth naming first is **surface layout for layered MSAA
images**, because the one layout defect this port has already found was exactly a layered surface -
the fog LUT, a 3D image whose slices landed in the wrong place, still worked around by
`ORBIS_3D_LINEAR`. Multiview writes array layers; samples > 1 adds FMASK and CMASK on top; and every
addrlib input on this port is synthesised, two of them marked UNCITED in the arm itself.

That one can be narrowed **without a console**: `build-support/orbis/tools/tilecheck.cpp` and the
drm-shim Liverpool entry exist to compare what addrlib computes here against what the shim says, and
an MSAA array image is a shape neither has been pointed at yet.

## Geometry shaders: 32 failures to one, and the defect was a number nobody had measured

`dEQP-VK.geometry.*` is **206 tests, 186 pass, 1 fail, 19 NotSupported** - a complete run, no halt.
Before the fix the same family had 32 failures and the run stopped partway.

**The GSVS ring was half the size the hardware needs.** The path to that, four runs, one variable
each:

| | |
|---|---|
| a GS emitting 3 vertices | draws its triangle correctly |
| `basic.output_10`, `output_128` | draw NOTHING - black against an 80% white reference |
| ring x2 | `geometry.basic` 3 pass / 11 fail -> 14 / 1 |
| ring x4 | no better than x2, test for test |
| `ORBIS_NUM_SE=4`, ring x1 | IDENTICAL to ring x2, test for test |

⚠ **Two readings of this failure were wrong before the images settled it**, and both were built on
the same run. "Every case that completes a primitive fails" was an artefact of POSITION IN THE RUN -
those cases pass in isolation. "Tests expecting an empty image pass" died when a passing emit test
turned out to have drawn a triangle. The images cost one four-minute run and ended a day of theories.

⚠ **And the experiment meant to name the term does not name it.** It was built to separate "the
topology number is wrong" from "the hardware wants the size PER ENGINE and we hand it the total",
and both predict the same doubling: a total of 2X across two real engines IS X per engine. So the
tree scales the ring rather than changing `max_se`, because that factor is right under both readings
while `max_se` also feeds `pa_sc_raster_config`, `cu_bitmap` and every occupancy number RADV
derives. **SQ_WAVE_HW_ID is what settles it** - a compute kernel ORing one bit per (SE, SH, CU)
reads the real topology off the hardware - and no further CTS run can.

**It also explains the halts.** Failures that appeared only in long runs - `emit.points_emit_1_end_1`
and friends, passing in isolation - are gone. One undersized ring, reallocated when the first large
emit arrived, was breaking everything after it. There was never a second state-dependent defect.

### The one that survives

`basic.output_vary_by_texture` draws nothing against a reference of three coloured polygons. Its GS
decides how many vertices to emit from a **texture fetch**. ⚠ It is narrow rather than general:
`output_vary_by_uniform` and `output_vary_by_attribute` pass, and so does
`output_vary_by_texture_instancing` - the same test with instancing. Only the non-instanced texture
variant fails, and no theory here is worth more than that sentence until it is measured.

### ⚠ And the prediction written here was wrong

This file said to "expect the 63 multiview + geometry failures to be largely gone with this". The
confirming run says **63 of 72, exactly as before** - the ring fix changed nothing for them. (It also
said 72/72 in conversation, which was wrong twice over: nine of them passed before and nine pass
now.)

**But the failure MODE changed, and that is the finding.** The images are four quadrants, one per
view. Before, geometry failures were black frames - nothing drawn. Now:

    reference:   dark green | green          result:   WHITE    | WHITE
                 olive      | mauve                    olive    | mauve

**Two of the four views are correct and two come out white.** So multiview + geometry is a real,
separate defect about which VIEW gets what, not about whether geometry draws at all - and it was
hidden underneath the ring defect until that one was fixed.

Where to look: on GFX7 the legacy geometry path needs the view index in the **GS copy shader** as
well as the GS, and `radv_emit_view_index` handles that copy shader through a separate branch
(radv_cmd_buffer.c:11229) from the one that walks the active stages.

## Multiview + geometry is the same defect again: ring capacity, multiplied by the view count

`ORBIS_GS_RING_SCALE=8` takes multiview + geometry from **15 pass / 63 fail to 78 / 0**. Every one
of the 63 flipped.

What made this findable was measuring all 63 images instead of looking at one:

| view mask | n | missing | extra | wrong colour |
|---|---|---|---|---|
| 15, 15_15_15_15, 1_2_4_8, 5_10_5_10, max_view_count | 39 | **100%** | 0% | 0% |
| 8_1_1_8 | 9 | 66.7% | 0% | 0% |
| 8 | 9 | 50.0% | 0% | 0% |

⚠ **Not one extra pixel and not one wrong colour, anywhere in 63 failures.** That is what killed the
view-index and layer-routing hypotheses in one step: a wrong view index puts content in the WRONG
view, which shows up as extra as well as missing. Nothing was ever drawn wrongly here - it was only
ever not drawn.

⚠ **Five readings of this defect were wrong before that, and every one came from a sample.** "The
first complete primitive fails" (position in a 754-test run, not vertex count). "Tests expecting an
empty image pass" (a passing test had drawn a triangle). "Two of four views are white" (contradicted
by 0% extra pixels across all 63). "The GS copy shader needs the view index" -
`radv_nir_export_multiview.c` injects the LAYER store before every `emit_vertex_with_counter`, so
the GS carries it through the ring and the copy shader needs nothing. And "the view mask is the
axis" - which 3 of 9 pass varies from mask to mask, with no structure.

**The measurement that ended it took no console time at all**: the images were already in the .qpa
from the previous run, and classifying every failing pixel as missing / extra / wrong-coloured is
twenty lines of numpy.

### What is still open: the term

Scale 8 works, scale 2 does not. The factor is somewhere in (2, 8]. `radv_get_esgs_gsvs_ring_size`
has **no view-count input at all** - it cannot, it never sees the view mask - and multiview draws
one draw per view.

⚠ **Prediction, written before the run that tests it:** if the factor is the number of views in the
render pass, then `ORBIS_GS_RING_SCALE=4` fixes every mask with four views or fewer and leaves
`1_2_4_8_16_32` (six) and `max_multi_view_view_count` failing. If instead 4 fixes everything, the
requirement is not the view count and this reading is wrong too.

## A combined image sampler read from the geometry stage takes the console down

Looking for a control that would separate "the texture fetch is wrong" from "the coordinate reaching
it is wrong" in `geometry.basic.output_vary_by_texture`, the obvious one turned out to exist:
`binding_model.shader_access.*.geometry.*` binds resources to a named stage, 588 of them samplers
and images. The run stopped on the FIRST of them:

    dEQP-VK.binding_model.shader_access.primary_cmd_buf.bind
        .combined_image_sampler_immutable.geometry.descriptor_array.1d

The .qpa carries that test's own log - "Descriptors are accessed in { geometry } stages." - and then
the block never closes. Two size samples a minute apart are byte-identical.

⚠ **This is not the same defect as output_vary_by_texture, and the difference matters.** That test
draws black and finishes with a verdict; this one ends the process. Both sample in a geometry
shader. The names carry the visible difference: a **descriptor array**, and **1D**.

⚠ **And the crash handler wrote nothing again.** `--deqp-crashhandler=enable` was added precisely so
a run's ending would name itself; this is the second halt it has stayed silent through, after the
tessellation one. Until that is understood it is not a mechanism to rely on - the ending still has to
be read out of an unterminated block, which is how the tessellation halt was misread the first time.

**So the question moved.** It is no longer "why does one test draw black" but "why can a sampler in
the geometry stage kill the console", with 588 cases waiting behind it that cannot run until it does
not. That is a bigger and better-defined target than the one it replaced.

## It is the STAGE, not the descriptor shape: a sampler in geometry kills the console

    fragment   147 / 147 PASS   including 43 descriptor_array cases
    geometry     0 verdicts     dies on the FIRST one it reaches, whichever shape that is

The run excluded `descriptor_array` from the geometry set, on the strength of the previous halt
landing there. The halt simply moved to the next shape in the queue,
`multiple_arbitrary_descriptors.2d`.

⚠ **So "the visible difference is in the name - a descriptor array, and 1D", written here yesterday,
was wrong.** That was the name of the alphabetically first test, not a property of the defect. It
came from one sample, which is the seventh time in this sequence. The fragment control was in the
run precisely to catch that, and it earned its place: the identical shapes pass in another stage.

**A sampler bound to the geometry stage takes the console down, at any descriptor shape.** That is
narrow, reproducible, and the same class as the three defects fixed today - a configuration this
hardware does not execute and the driver advertises anyway.

It probably also explains `geometry.basic.output_vary_by_texture`, which samples in a GS and draws
black instead of dying - a milder form of the same thing rather than a separate item. ⚠ Probably:
that is one observation, and it cannot be checked until the killing stops.

**Where to read**: how RADV programs descriptors for the GS stage on GFX7. The legacy path is ES and
GS as separate hardware programs plus a copy shader - three sets of user SGPRs where fragment has
one.

## The GS rings, and where eight runs of tuning actually landed

Two independent faults, each mistaken for the whole story at some point:

| | symptom | answered by |
|---|---|---|
| the size register is too small | the ring wraps early and overwrites itself - **wrong images** | scaling the computed size |
| the allocation is too small | the hardware writes past the size it was told - **GPU fault, process dies** | padding the allocation |

⚠ **Neither is understood.** The formula in `radv_get_esgs_gsvs_ring_size` is identical in RADV and
radeonsi, term for term, and both run on real GFX7 under amdgpu. The topology it depends on is right:
the published spec for this APU - 18 CUs, 72 TMUs, 32 ROPs, HD 7850 based - agrees with every number
this port synthesises, and `ORBIS_NUM_SE=4` was falsified on hardware. So the formula is right, its
inputs are right, and this part still overruns. **The multipliers are margins, not derivations.**

### What ships, and what it costs

`scale 2, pad 16x, total capped at 512 MiB` - 606 tests, **523 pass, 64 fail, no device loss**:

    binding_model geometry   328 / 328   the console-killer, closed
    geometry.*               180 / 181   only output_vary_by_texture left
    multiview + geometry       9 /  72   STILL BROKEN, deliberately

⚠ **Multiview + geometry stays broken on purpose.** Scale 4 fixes it - 78/78 - but at scale 4 the
largest ring reaches upstream's clamp of `63.999 MB * num_se`, the padding then asks for half a
gigabyte on top of that, and the arena loses the device. Measured uncapped, at 512 MiB and at
256 MiB: all three die. **63 wrong images beat a device loss that takes 328 passing tests with it.**

### The eight configurations, so nobody repeats them

| scale | pad | cap | result |
|---|---|---|---|
| 1 | 16 | none | 496 pass, 91 fail |
| 2 | 16 | none | 334 pass, DeviceLost at a 1 GB ring |
| **2** | **16** | **512 MiB** | **523 pass, 64 fail** ← ships |
| 4 | 1 | - | dies on the first geometry test |
| 4 | 16 | none | DeviceLost, 2 GB for one ring |
| 4 | 16 | 512 MiB slack | 514 pass, 1 fail, then DeviceLost |
| 4 | 16 | 512 MiB total | DeviceLost |
| 4 | 16 | 256 MiB total | 234 pass, 353 fail |

**The arena limit sits between 536 MB (survived) and 646 MB (lost the device)**, which is the only
hard number this search produced.

### Where the answer actually is

Not in another multiplier. The hardware writes past a size it was explicitly given, which points at
`VGT_GSVS_RING_ITEMSIZE` / `VGT_GS_VERT_ITEMSIZE` - the per-vertex strides the hardware computes
offsets from - or at on-chip versus off-chip GS mode. That is reading, and comparing against what
radeonsi programs for the same generation, rather than another run.
