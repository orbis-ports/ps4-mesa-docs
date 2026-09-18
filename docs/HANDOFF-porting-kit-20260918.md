# Handoff — the porting kit exists and ships, 2026-09-18

**orbis-compat split in two.** Everything a person runs or points a build at moved to
**orbis-ports/orbis-porting-kit**, which is public, tagged, and publishes a bundle. The overlay kept
include/, src/, crt/ and the archive — nothing with a user-facing surface. Every consumer in the
organisation builds against both and all CI is green.

---

## 1. Where things are now

```
orbis-compat        9173267   master        include/ src/ crt/ optional/ test/ licenses/ + build.sh
orbis-porting-kit   0aff500   main          cmake/ vkloader/ services/ scripts/ release/ examples/
mesa-ps4            1a59cc1   orbis         release orbis-mesa-1a59cc147b54
OpenGothic          f73d184   ps4-support
RetroArch           2c32571   ps4-support
sonic3air           5cecd68   main          (ps4-support carries the port)
```

Kit tags: `v1.0.0 … v1.4.0`, with `v1` moving. Bundle: **`orbis-sdk-v1`**, 92 MB, `gate=pass`.

Consumers take the toolchain through one published action:

```yaml
- uses: orbis-ports/orbis-porting-kit/.github/actions/setup-orbis@v1
  with:
    orbis-compat-ref: <sha>
    mesa-release:     orbis-mesa-<sha>
```

or, for a shipped build, one string instead of three pins:

```yaml
  with:
    sdk-bundle: orbis-sdk-v1
```

Both modes end at the same export step: `OO_PS4_TOOLCHAIN`, `ORBIS_COMPAT_DIR`, `ORBIS_KIT_DIR`,
`ORBIS_MESA_SRC`, `ORBIS_MESA_BUILD`.

---

## 2. What moved, and the rule that decided it

**If a person runs it or points a build at it, it is the kit's.** By that rule: `cmake/`,
`vkloader/`, `scripts/ps4/`, `scripts/orbis-new.sh`, both examples, and the whole release machinery
(cut, offline verify, publication gate, 27 tests, mutation harness).

`sdk-licenses.sh` stayed in the overlay, and that is not an exception: it **writes** that
repository's `licenses/`, `NOTICE.md` and `LICENSING.md`. The cut calls it where it lives and ships
it in the bundle.

⚠ **Four private runtime shims came the other way, out of the ports and into the overlay**, because
each works by definition order — an object on the link line before `-lc` and `-lc++`:

| | was | why it is a correction |
|---|---|---|
| `orbis_wchar32.c` 1515 | sonic3air's build dir | libc.a was built with a **16-bit wchar_t**; clang and libc++ use 32. `std::wstring` went through a libc reading half of each character — SIGSEGV in `wmemcpy` **before main()** |
| `orbis_cxa_guard.c` 329 | RetroArch **and** sonic3air, drifted | libc++abi aborted with "recursive initialization" where there was none |
| `orbis_thread_atexit.c` 109 | both | libc++ calls it, this libc never defines it; our stub leaked by design |
| `orbis_abort_report.c` 171 | both | assert/abort_message write to stderr, which goes nowhere here; `dup2` onto fd 2 returns EPERM |
| `orbis_cv_fix.cpp` 114 | both | condition variables |

⚠ **`wchar32` has a second half that must never be separated from it.** `regcomp`, `regexec` and
`vfscanf` stay as libc.a members and call `mbtowc` with a **two-byte** `wchar_t` on their own stack;
`regcomp` keeps a saved register right behind that slot. `build/libc16/` holds renamed copies so
those three keep a 16-bit `mbtowc`, and `ps4-openorbis.cmake` puts them on every executable's link
line. **Mesa's libgallium references `regcomp`** for driconf, so this reaches every port with a
graphics stack. Shipping the 32-bit members without libc16 is live stack corruption.

New headers, same family, both found by a build failing: `include/sys/endian.h` and
`include/sys/dirent.h`. The target says `__FreeBSD__ 12`, portable code takes its FreeBSD arm, and
the SDK ships only the glibc spellings.

---

## 3. Traps this cost a day to find

**A tool that exists is not a tool that runs.** `bin/linux/create-fself` is present on a Mac and
passes `[[ -x ]]` — it is a Linux ELF. A name-ordered search picked it. Candidate lists are ordered
by `uname` now, in `Makefile.orbis`, `ps4/build-core.sh` and `ps4/build-cores.sh`.

**A harness that guesses invents facts about other people's code.** With create-fself unable to run,
build-cores.sh reported `undefined weak: sceKernelInternalMemoryGetAvailableSize` — a heuristic
filling an empty column. It reads the tool's own message first now.

**An empty archive reported success.** `llvm-ar` is keg-only on macOS, so `build.sh` fell back to
Apple's `ar`, which refuses ELF objects; it wrote a 96-byte archive with no members and every check
passed, because none of them read the archive. `build.sh` counts members now.

**`yaml.safe_load` accepts duplicate keys.** Two `if:` keys in one step parsed clean and the runner
refused the file. Validate action YAML with a loader that rejects duplicates.

**Reproducing without the toolchain file reproduces something else.** `find_file()` obeys
`CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY`, so it searched the sysroot and could not see a script
beside the build. It passed locally because the local check had no toolchain file in it.

**One stale line in `/data/orbis-env.txt` changes every image on the console.** `ORBIS_THREAD_STACK=0`,
left from one test the day that file was added, gave every thread Sony's 64 KiB; a core died as
SIGILL with a write eight bytes under `%rsp`. The knob is **removed** — it could only point down.

---

## 4. Verified on hardware, 2026-09-18

`hello`, `triangle`, OpenGothic and RetroArch (with melonds) all built **locally on an Apple Silicon
Mac** and run on the console. The bundle's gate builds OpenGothic from an unpacked copy with the
environment scrubbed.

⚠ **Open, and not today's doing:** trident's **core options** screen runs at ~330 ms/frame against
18 ms normally, with the GPU idle 0% of the window and `retro_run` never entered. Confirmed on a
frontend built against **yesterday's** overlay, so it predates all of the above. 19 options, no long
value lists. `ps4/orbis_profile.c` exists and **nothing instruments the menu** — one slot answers
this in five seconds instead of an hour of inference.

---

## 5. Open, in the order I would take them

1. **`orbis_jit`.** 9 of the 30 core patches in RetroArch's `ps4/core-patches/` are one missing
   executable-memory arena. The specification is already written twice: `ps4/orbis_exec_mem.c`
   (promotes pages the core already owns; rel32 needs ±2 GiB of the module's text) and
   beetle-psx's `orbis_lightrec_mem.c` (maps the buffer). Panda3DS, dynarmic and 3dsTrident are
   waiting in the org untouched.
2. **SDL2.** The orbis backend — video 888, joystick 534, audio 326 — is vendored inside sonic3air.
   Plan agreed: fork SDL2 at its newest tag, carry the backend there, substitute games' copies.
   Only ~8 lines touch upstream files (three bootstrap arrays). SDL3 is a later project.
3. **Wire one `orbis_profile` slot into the menu**, then answer the trident question.
4. `sigaltstack` returns -1, so a stack overflow dies with no line from us — today's SIGILL was
   exactly that shape.
5. Small debts: `PLAN.md` §13 is stale; kit `release/README.md` still describes commit-drift
   pairing; `bundle-gate.sh` stage 0 cannot find `llvm-nm` where macOS keeps it; `cores.yml` is
   repinned but has not run since.
6. Delete the three product paths from `orbis_env.cpp` — waits on a released package of OpenGothic
   and RetroArch that writes `/data/orbis-env.txt`.

---

## 6. Development console

`192.168.50.230` is the Mac, `192.168.50.231` the PS4, one subnet. `dnsmasq` from
`unemups4/scripts/ps4link.sh wifi` answers the console and blackholes 108 Sony update domains;
`get.net.playstation.net` is one of them, so **the console's own internet test will always fail** —
Download Cores is the real test. The resolver binds one address, so give the Mac a DHCP reservation.

Build locally before pushing: the SDK is at `~/.local/opt/openorbis`, `/opt/homebrew/opt/llvm/bin`
must be on PATH, and `ORBIS_KIT_DIR` plus an unpacked `orbis-sdk-v1` are enough to build every
target on this machine — frontend, cores, examples and packages.
