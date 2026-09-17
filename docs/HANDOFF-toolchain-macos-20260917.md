# Handoff — the toolchain builds on macOS, and the SDK ships as one bundle, 2026-09-17

**Two things that did not exist this morning exist now.** The whole chain runs on an Apple Silicon
Mac — clang, ld.lld, Mesa, `create-fself`, `PkgTool.Core`, a `.pkg` on the console — and the SDK,
the overlay and Mesa ship as one versioned tarball whose publication gate passes in CI. Both were
verified on hardware, not only in a build log.

Everything below is committed. `orbis-compat` is at `c301495`, its consumers are repinned, and every
CI job in the organisation is green.

---

## 1. What a person can do now that they could not before

```sh
scripts/orbis-new.sh --check        # every dependency, what it is, where to get it
scripts/orbis-new.sh mygame         # a project that builds and packages, one command
scripts/orbis-new.sh mygame --type vulkan
```

This exists because of a developer's own words, quoted in the script's header: *"it seems that the
mesa port is designed to work only with orbis-compat which seems to be a set of modifications for
openorbis but its unclear how to set it up"*. Every word of that was accurate. The pieces were all
documented and none of the documentation was where a person looks first.

`README.md` now opens with §0 — install, build, package, deploy, and what a working run prints —
before the thesis. §2's two hundred lines of measured evidence were not shortened: the problem was
the ordering, not the length.

Two worked examples, both run on a console:

| | what it proves | Mesa |
|---|---|---|
| `scripts/release/hello/` | the bundle's LAYOUT — corrected headers, `--whole-archive`, the linker script, `crt1.o`, `create-fself` | none, deliberately |
| `scripts/release/triangle/` | RADV comes up and a frame reaches the television | yes |

`hello` printed `orbis-sdk bundle: all checks passed`. `triangle` presented frames.

---

## 2. The macOS port of the toolchain

The Linux path is unchanged everywhere: every fix tries the GNU form first and adds a second.

| | was | now |
|---|---|---|
| file size | `stat -c%s` | GNU first, `-f%z` second. `deploy.sh:104` used it to VERIFY an upload, so on macOS the verification compared against nothing |
| job count | `nproc` | `getconf _NPROCESSORS_ONLN`. `orbis-env.sh` had a fallback and so never failed — it just built with four jobs on an 18-core machine |
| `create-fself` | `NAMES create-fself` | plus `create-fself-macos`. The miss was a WARNING, so the build carried on, produced no eboot, and failed later in whatever consumed it |
| meson compiler | literal `clang` | `@ORBIS_CC@`, defaulting to `clang` |

**The meson one is the interesting failure.** `build.sh` runs meson inside `nix develop nixpkgs#mesa`,
so a bare `clang` resolves to nix's cc-wrapper, and on aarch64-darwin that wrapper puts
`-mmacos-version-min=15.0` on a freebsd12 cross compile. meson probes with
`-Werror=unused-command-line-argument`, so **every probe failed**:

```
clang: error: argument unused during compilation: '-mmacos-version-min=15.0'
Check usable header "endian.h" : NO
```

168 NO against 66 YES. `config.h` lost `HAVE_ENDIAN_H`, `u_endian.h` fell past its `<endian.h>` arm
into the FreeBSD one and reached for a `<machine/endian.h>` nothing ships, and the build ended with
**1107 failed targets and 2847 compile errors, none of which named the cause**. With an unwrapped
clang: zero of both.

macOS also needs Rosetta — every tool in the SDK's `bin/macos` is x86_64.

---

## 3. What the console said

`hello` crashed on its first run with `SIGSYS`, and the diagnosis took one wrong turn worth
recording. The registers named the call site and I read the source instead:

```
# signal: 12 (SIGSYS)
# rip: 00000008000028bc      libkernel.sprx
# BrF: 000000000045a330      -> .plt slot 56 in our binary = _exit@plt
```

It had run **every** check and returned from `main()`. On this console that is `CE-34878-0` and reads
exactly like a crash — `optional/ps4_app.cpp` already said so and installs `ps4_idle_forever` for the
same reason. `hello` does not link `ps4_app` (README §3.2: a consumer that wanted a working `mmap`
should not inherit `-lSceNet`), so it now idles on its own.

**Resolve the address before reading the source.** `BrF` pointed at our own `.plt` and would have
answered it in the first minute.

---

## 4. The bundle, and what its gate found

`scripts/release/` cuts one tarball carrying the SDK, the overlay, Mesa, the licence ledger and a
toolchain file, with `bundle-gate.sh` refusing to call it publishable until something has been built
from an unpacked copy with the environment scrubbed.

It had never run. Its first six runs failed **in that machinery rather than in any bundle**:

| what | where |
|---|---|
| `sed 's/^_//'` truncated every Itanium-ABI C++ name, so `_Znwm` could never match | `verify-sdk-bundle.sh` |
| a dirty tree was FAILED not INCOMPLETE, so the gate refused every bundle it exists to fix | `verify-sdk-bundle.sh` |
| the hello guard sat above `project()`, where nothing the toolchain file sets exists yet | `hello/CMakeLists.txt` |
| the overlay archive was force-loaded twice, duplicating every symbol | `hello/CMakeLists.txt` |
| `grep -q` under `set -o pipefail` — SIGPIPE killed `llvm-nm`, pipefail took its 141, and a matching grep reported the overlay ABSENT from an image with 16 of its symbols | `bundle-gate.sh` |
| stage 5 let the port pick its own build directory, so run N compiled against run N-1's deleted sysroot | `bundle-gate.sh` |

The last is README §9 trap 7, inside the script whose header quotes that trap as its reason to exist.

**The pairing rule changed.** It compared commit ids, so any commit invalidated the pair — a README
typo, a comment, a change to the gate itself. Measured: a bundle refused over `cc75949..4f61d6f`,
three commits whose `git diff -- include/` is empty. `include/` is the whole of what Mesa compiles
from this repository, so that is now what refuses. The commits are still recorded; they are no longer
the criterion. `verify` cannot re-derive that diff without a git history, so the cut records
`orbis-compat-include-sha256` and `pairing-basis`, and verify says plainly which half it checked.

⚠ **The strictness that matters is unchanged.** `sys/umtx.h` carries ten `static inline` functions
that compile INTO Mesa. A change there moves no symbol, so the import-list assertion cannot see it —
its own comment says *"presence, not meaning"* — and refusing the combination is the only cover.

---

## 5. Licences — what moved

* `cmake/orbis-tls.ld` is **GPL-3.0-only**, not MIT. It is the SDK's `link.x` with two match patterns
  added; a derivative of a GPL-3.0 file cannot be relicensed by the person deriving it. `LICENSE` now
  names both non-MIT files (the other is `include/sys/ioccom.h`, BSD-3-Clause).
* **`crt1.o` was never the problem.** It, `crti.o`, `crtn.o` and `crt_dyn.o` come from
  `OpenOrbis/musl` (`crt/ps4/crt1.c`, `arch/ps4/crt_arch.h`) — MIT. Only `crtlib.o` is GPL-3.0, and
  it reaches `.prx` modules only.
* `crt/` exists anyway, because the SDK's `crtlib.o` has a real defect: it resolves
  `__init_array_start`/`__init_array_end` into its own `.bss` (measured `0xc030`/`0xc038` against a
  real `.init_array` at `0x4000`), so `module_start` walks one zeroed entry and calls through NULL.
  **Static constructors in a `.prx` never run.** Tentative definitions that were COMMON under
  `-fcommon` and are ordinary objects under clang 18.
* `sdk/lib/*.so` has **two readings, not none** — `LICENSING.md` §5.1 said "no licence exists" and
  was analysing one chain. They arrive inside a GPL-3.0 release with no per-file headers; they are
  also generated from `ps4libdoc`, whose terms are unstated. And they are link-time, so an eboot
  carries their imports rather than their bytes. The question is about OpenOrbis's terms, not Sony's.

---

## 6. Open, in the order I would take them

1. **`scripts/release/` has no test.** 1800 lines, run only by a human after a push. Three of the
   eight CI failures today were in it. `build.sh` has tests and its one failure landed exactly where
   they do not reach — the crt compile flags.
2. **The libssl probe belongs in `make-pkg.sh`.** `PkgTool.Core` dlopens `libssl.so.1.1` and runners
   have OpenSSL 3. The workaround is now copied into **three** workflows — OpenGothic's says "(Same
   step as RetroArch.)" and the bundle's is the third. Moving the probe into the script deletes all
   three, and the script is in the bundle, so everyone gets it.
3. **Mesa's `manifest.txt` should record the include/ hash**, so `verify` checks the pairing itself
   instead of trusting a verdict carried from the cut.
4. **One message to OpenOrbis.** It settles `crtlib.c`'s licence, the stubs, and whether the private
   BSD-libc toolchain can be seen. Material is drafted at `scratchpad/outreach/` — situation with
   quotes and dates, what we bring, ranked questions, a message to paste, and a risk note.
   ⚠ That directory is a scratchpad and does not survive; copy it somewhere real before relying on it.
5. **`PLAN.md` items 10-12**, added today: the env-file list that names its consumers, the `ORBIS_*`
   switches that could not be set on a console, and `crtlib.o`'s remaining licence question.

---

## 7. State

```
orbis-compat          c301495   master        11 commits + 6 CI fixes
mesa-ps4              4c934eb   orbis         release orbis-mesa-4c934ebf9cda
OpenGothic            6e0c3f8   ps4-support
RetroArch             0ac249e   ps4-support
VK-GL-CTS             46a992a   ps4-support   ps4/build.sh only
beetle-psx-libretro   1657e2f   ps4-support
unemups4              6774735   feat/ue4-...  scripts/ps4link.sh only
```

All CI green: bundle `gate=pass` (91.5 MB artifact), OpenGothic, RetroArch frontend, RetroArch cores
10/10 shards.

**Left uncommitted deliberately, none of it mine:** `orbis-compat/scripts/ps4/logs.sh` (a `case`
reformat), VK-GL-CTS's tessellation work and `cases-*.txt`, `OpenGothic/ps4/tempest-env-tess.txt`,
unemups4's `crates/libs/src/libkernel/*.rs`.

**Host setup, for the next machine:** `brew install llvm lld cmake ninja glslang`, both keg-only bin
directories on `PATH`, `softwareupdate --install-rosetta`, SDK v0.5.4 asset unpacked at
`~/.local/opt/openorbis` (sha256 `3c7cd5bb593ca74fa1c13fd59f3938dc0fc07985167f7275063019e63abe4526`).

⚠ **That asset is the v0.5.3 tree.** `kernel.h` in it is byte-identical to the v0.5.3 tag and still
says `typedef mode_t OrbisKernelMode`; the `uint16_t` fix from PR #278 is in the v0.5.4 **tag** only.
Everyone pins the asset, so nobody has the fix, and `orbis_stat.h`'s citation of `kernel.h:177` is
correct for what people actually build against.
