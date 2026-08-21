# Plan — the OpenGothic PS4 diff, reviewed, 2026-08-20

The 18 staged paths in `~/src-ps4/OpenGothic` (3658 insertions, 16 files) were read end to end
before the collective commit was restored. Twenty findings. This plan orders them the way the
maintainer asked for: **the cheapest first, then the ones that matter, then the rest if we decide
they are worth it.**

**Status: CLOSED 2026-08-21. All three tiers applied, 23 items, every one run on hardware**, plus
`VfsMountMode::LAZY`, which this review did not find and which mattered more than any of them — see
the section before "Suggested order".

Two things are deliberately unfinished: **2.6 is unverified** (nobody has held a trigger and pressed
a masked input) and **lives in the Tempest fork, uncommitted**. Everything else sits in OpenGothic's
throwaway WIP commit, unsigned, waiting for the maintainer's `difit .` and his own signature.

---

## How to read the tiers

**Tier 1** — mechanical. No design decision to make, each is a deletion or a one-line change, and
each can be verified on the laptop without a console trip. Doing all of Tier 1 in one pass and
one build is reasonable.

**Tier 2** — changes behaviour. Each needs a decision and most need a console run to confirm.
Do these one at a time, with a run between them.

**Tier 3** — real but arguable. Listed so the decision is deliberate rather than forgotten.

Column meanings: **Where** is the site, **Verify** is what proves it, **HW** means a console run
is needed to confirm.

---

# TIER 1 — the cheap ones — DONE 2026-08-20

**Applied, built clean, exit 0.** Eleven items: the ten planned plus 1.9, which the plan did not
have because reading the file did not find it.

    7 files, +90 -192            unstaged on top of the 18 staged paths
    build                        /home/mikolaj/.cache/opengothic-ps4-tier1, exit 0
    warnings                     3, all pre-existing in Tempest, none from these changes
    content id                   IV0000-TMPS10021_00-TEMPESTOPENGOTHI.pkg - unchanged, as intended

⚠ **The old build directory was stale and is not a control.** `~/.cache/opengothic-ps4/build`
referenced `/home/mikolaj/src/forks/OpenGothic`, a path that no longer exists — so every binary in
it predates the move to `~/src-ps4` and a section-size comparison against it measures the move, not
this work. It was left untouched and a fresh work directory used instead. (Sizes did grow, by 573 KB
of text; that is the two trees differing, not Tier 1, and it is stated here rather than quietly
dropped.)

The one measurement that *is* a control: `probeLargestArchiveMap` was 1 symbol before and is 0
after. Unlike `ps4_capture` — which was static and whose removal changed nothing — this one was
non-static and actually reached the binary.

---

## 1.1 The two C++20 shims collide — `ps4/cxx20shim/`

`concepts` and `ps4_cxx20_shim.h` carry **the same 21 concept definitions under different include
guards**. Any TU that does `#include <concepts>` gets both and fails to compile — 21 redefinition
errors that will read as a libc++ problem.

    clang++ -std=c++20 -fsyntax-only -Ips4/cxx20shim \
            -include ps4/cxx20shim/ps4_cxx20_shim.h tu.cpp
    → error: redefinition of 'same_as'   (×21)

Latent only because nothing in the tree includes `<concepts>` directly today — and ZenKit, which
is the reason the shim exists at all, is exactly what would start.

**Fix:** `concepts` becomes one line, `#include "ps4_cxx20_shim.h"`, plus its own guard. The
definitions live in one place.
**Verify:** the repro command above must exit 0 after the change. Then a full PS4 build.

## 1.2 `census()` calls `ps4_log` while holding the audio mutex — `og_sound_orbis.cpp:600`

`census()` is called from `mixBlock():427` inside the `lock_guard`. `ps4_log` writes klog, and
`orbis-compat/include/ps4_app.h` states the measurement: **klog is 8-15 ms a line on this console**.
The audio block period is 5.33 ms.

So every ~2 s the mixer stalls 8-30 ms holding the lock: an underrun in the audio, and the render
thread blocked in `setListenerPosition`/`add`/`remove` for the same window.

**Fix:** `ps4_log_frame` (UDP only) instead of `ps4_log`, which is what the same file's own comment
says the frame path is for. Or format into a buffer under the lock and emit after it.
**Verify:** read the mixer's own census line spacing in a console log; HW to confirm the stall is
gone, but the change is correct without it.

## 1.3 `OG_FOG_DIAG` has no reader — `CMakeLists.txt:246-248`

43 lines of comment describing a four-rung experiment ladder (`0|1|3|9|5|15`, "3 versus 9 versus 5
is the experiment"), a cache variable, `add_definitions(-DOG_FOG_DIAG=15)` on every target, and a
`message(STATUS)` printed on every configure.

**Nothing reads it.** Not a `.cpp`, not a `.h`, not a shader — checked across the tree. The fog-LUT
question it was built for was settled by `ORBIS_3D_LINEAR` (task #22) and the diagnostic went with
the patch queue.

⚠ And it sits **outside `if(PS4)`**, so the define and the status line fire on Linux and Windows
builds of upstream OpenGothic too. It is the only thing in this diff that leaks past the platform
guard.

Same shape as the GNM tail cut out of Tempest the same morning: a knob nobody reads, with an
elaborate comment that makes it look load-bearing.

**Fix:** delete all three lines and the comment block.
**Verify:** `grep -rn OG_FOG_DIAG` returns nothing; section sizes unchanged (the define reached no
code, so they must be).

## 1.4 `probeLargestArchiveMap` is ~70 lines of dead code — `og_ps4_boot.cpp:249,354-423`

Declared in `og_ps4_boot.h:142` with a six-line docstring in the present tense — "Map the largest
archive whole, the way ZenKit will ... this is the 300 MB case that actually carries the fonts".
The only call site is commented out at `:350`, with a correct note explaining that it cost 37 s and
689 MiB of I/O on every boot and had to go.

The removal was right. What was left behind is a function nobody calls and a header that says it
runs.

**Fix:** delete `probeLargestArchiveMapImpl`, `probeLargestArchiveMap`, both forward declarations
and the header entry. Keep the `NOTE:` block at `:341-353` — that is the measurement (mmap
populates eagerly here) and it is worth more than the code.
**Verify:** symbol table before/after. This one is non-static, so unlike `ps4_capture` the count
will actually move.

*(1.4 also disposes of F3: `probeLargestArchiveMapImpl` ignores `fstat`'s return and would index
`p[SIZE_MAX]` if `len` came back 0.)*

## 1.5 `OrbisMixer::announceOnce` is never called — `og_sound_orbis.cpp:358`

Dead. The null backend's namesake at `og_sound_null.cpp:72` is called eight times; this one zero.

**Fix:** delete.

## 1.6 Two adjacent comments state opposite facts — `og_sound_orbis.cpp:626`

The block above `int32_t port` says the census "is restated periodically, **from
SoundDevice::process() which OpenGothic calls every frame on the render thread**".

Twenty lines above, `census()`'s own comment retracts exactly that, with its evidence:

> the first version called it from SoundDevice::process() on the strength of a comment that said
> OpenGothic calls that per frame. It does not call it AT ALL — `grep -rn '\.process()' game/`
> finds nothing

The retraction is the true one. The stale half is the paragraph that explains *why the census
exists*, so it is the one a reader trusts.

**Fix:** rewrite the `int32_t port` block to say the mixer thread is the pump, keeping the reason
the census repeats (the netlog receiver started after the title, so a one-shot startup line was
unobservable).

## 1.7 The first census line always lies — `og_sound_orbis.cpp:621`

`censusAt` starts at 0 and `census()` returns early only on `blocks < censusAt`. The first call
comes from `mixBlock()` before `++blocks`, so `blocks == 0` and the branch fires:

> sound: census - the port is open and NO BLOCK has been output: the mixer thread is not running

Printed **from the mixer thread**. Every run's first census line is false.

And its sibling is unreachable: `if(port<=0)` can never be true here, because the constructor
returns before spawning the thread when the port fails — so the one diagnostic written for
"there is no sound because the port refused" never prints.

**Fix:** skip the first census (`censusAt = 1`), or move `++blocks` above `mixBlock()`. Delete the
`port<=0` arm from `census()` — the constructor already logs that case — or make the constructor
the place that repeats it.

## 1.8 Small ones, one line each

| # | Where | What |
|---|---|---|
| a | `og_ps4_boot.cpp:775` | `compare(0,6,"/app0/")==0 \|\| accepted=="/app0/"` — second disjunct is subsumed by the first |
| b | `og_sound_orbis.cpp:607` | comment says "NOT under the lock"; it *is* under the lock. It means "does not take the lock" — say that |
| c | `CMakeLists.txt:110` | "Not a strong-symbol trick like **the three above**" — one source precedes it; the three moved to orbis-compat, as the comment ten lines earlier says |
| d | `CMakeLists.txt` | `CONTENT_LABEL TEMPESTOPENGOTHIC` is 17 chars and the packager silently truncates to 16 — `IV0000-TMPS10021_00-TEMPESTOPENGOTHI.pkg` on disk. Declare 16 |
| e | `og_ps4_ime.cpp` `toUtf8` | walks `g_text` until a NUL with no bound; a loop over `MaxText` costs nothing |
| f | `og_sound_orbis.cpp:262` | ADPCM `perBlock` underflows when `blockAlign < channels*4` — only `==0` is guarded. Contained today (`length_error` → `SoundDevice::load`'s `catch(...)`), but it is one condition |

## 1.9 Six includes that outlived the file they were written for — the two sound backends

**Not in the plan: found while editing, by the editor's own diagnostics.** Both sound backends
opened with

    #include "../../Engine/sound/sounddevice.h"      (and sound.h, soundeffect.h)

From `OpenGothic/ps4/`, `../..` is one level **above the repository** and `Engine/` is not there.
The path was correct when these files lived at `lib/Tempest/ps4/opengothic/`, where `../..` really
is the Tempest root; the move left it behind.

It kept compiling because a quoted include falls back to the `-I` list, and
`-I<root>/lib/Tempest/Engine/include` happens to sit **exactly two levels** below `lib/Tempest`, so
`<that>/../../Engine/sound/sounddevice.h` resolves. A two-level coincidence, in six places. Tempest
moving its public include directory would have broken all six with "file not found" and nothing
pointing at a file move as the cause.

**Fixed:** `<Tempest/SoundDevice>`, `<Tempest/Sound>`, `<Tempest/SoundEffect>` — which are one-line
forwards to those same three headers, so it is identical code through the interface Tempest exports.

⚠ **Worth its own note: a comment audit could not have found this.** The include was not a
statement about the code, it *was* code, and it compiled. What found it was the editor parsing the
file on its own terms and saying the header was not there — the same class of instrument as a
compiler, pointed at a file the compiler was reaching by a lucky path.

---

---

# TIER 2 — the ones that matter — DONE 2026-08-21

All six applied and run on hardware: 4.5 minutes in-game, 51073 audio blocks, 223 sounds
decoded, 0 refused, 16 voices. **2.5 is the one the log proves outright**: the boot section went
from 104 lines to 55, only the vdf probe and its stat-interposition VERDICT remain, and the four
heavy probes report zero occurrences.

⚠ **2.6 is applied but NOT VERIFIED, and it lives in Tempest, not here.** Nobody has held a
trigger and pressed a masked input. The hunk is in `lib/Tempest/Engine/system/api/ps4api.cpp` and
is uncommitted in that fork.

⚠ **2.1 is armour, not a repair.** The hang it prevents has never been seen; exercising it on
demand would mean forcing the framework to report STATUS_NONE, which nothing here can do.

Six items. Each changes behaviour; each wanted its own console run.

## 2.1 ⚠ The IME can hang the save dialog with no way out — `og_ps4_ime.cpp:214`

    if(st==ORBIS_DIALOG_STATUS_RUNNING) return ImeState::Running;
    if(st!=ORBIS_DIALOG_STATUS_STOPPED) return ImeState::Running;   // NONE lands here

`ORBIS_DIALOG_STATUS_NONE` is 0 and falls into the second arm, which returns `Running`
**with no bound**. And `gamemenu.cpp`'s `SavNameDialog` swallows *every* event while `imeUp` —
`keyUpEvent` returns early on all keys including `K_ESCAPE`, `mouseUpEvent` likewise.

So if the panel ever reports NONE — a system abort, or an `Init` that returned 0 for a dialog that
never came up — the dialog is **unclosable and the game is stuck**. No timeout, no escape hatch,
and the swallowing is deliberate and correct for the normal case.

The `NONE before the first status` reading is right; what is missing is that NONE is also what you
get *after* the framework has let go.

**Fix, in order of preference:**
1. Count polls in `NONE` and give up after a bound (a few seconds at frame rate), returning
   `Cancelled` and logging that the panel went away. The dialog then closes with the old name.
2. Let `K_ESCAPE` through even while `imeUp`, calling `imeEnd()` — a manual escape the player can
   reach. Weaker on its own: it does not help if input is not reaching the dialog either.

Do both. (1) is the fix; (2) is the thing that would have made this cheap to find.
**HW:** yes — and the hang itself has not been observed, so this is armour, not a repair.

## 2.2 ⚠ Every `SoundEffect` setter races the mixer thread

`play():921`, `pause():934`, `setPosition():969`, `setMaxDistance():980`, `setVolume():985` all
write `Voice` fields with no lock. The mixer reads every one of them under `sync` in
`mixBlock():427`. `add`/`remove` take the lock; the field updates do not.

Not merely formal UB. The concrete failure:

> `play()` on a finished voice sets `finished=false; cursor=0; playing=true`. An in-flight
> `mixStatic` on the same voice — started before, still running — then writes
> `finished=true; playing=false` at end-of-sample. **The sound that was just started never plays.**

Intermittent, unreproducible, and indistinguishable from "the sound file failed to load". Gothic
plays footsteps and hit sounds from pooled `SoundEffect` slots, so `play()`-on-finished is the
common path, not the rare one.

`setPosition` writing three floats while the mixer reads them is the milder sibling: a half-updated
position for one block, i.e. a click.

**Fix — decide which:**

| Option | Cost | Note |
|---|---|---|
| Setters take `sync` | one `lock_guard` each | Simplest. The lock is held across `renderSound()` in `mixBlock`, so a setter can block for a block period. Probably fine — the mix is "trivially cheap next to the frame" by this file's own note |
| `Voice` fields become `std::atomic` | more churn | Avoids blocking the game thread; `cursor` is a double and `pos` is a Vec3, so those two still need care |
| A per-voice command queue | most work | Correct and non-blocking, and overkill for this |

Recommend the first. If it costs, the second.
**Also:** `SoundDevice::setGlobalVolume` writes `*data->gain` unsynchronised while `mixVoice` reads
it. Same pass.
**HW:** yes — and the symptom is exactly the kind that a run cannot *prove* gone.

## 2.3 ⚠ The example env file produces the crash its own warning describes

`ps4/tempest-env.example.txt:40-46`:

> ⚠ TWO OF THESE ARE NOT OPTIONS. `ORBIS_3D_LINEAR=1` and `ORBIS_NO_TESS=1` are the configuration
> this port runs on, and BOTH are off in the driver unless this file turns them on. […] the title
> then loads and **crashes on entering 3D** — measured 2026-08-19, twice, same binary, only this
> file different.

The file's active content is one line — `ORBIS_DUMP_SUBMIT=7` (`:54`). `ORBIS_3D_LINEAR=1` appears
**only commented out** at `:61`. `ORBIS_NO_TESS` appears nowhere as a line at all.

Copied verbatim to `/data/tempest-env.txt`, this file crashes. And the one knob it *does* enable is
a submission hexdump — the diagnostic-on-by-default inversion that `main.cpp`'s own block argues
against at length.

**Fix:** the example ships the two mandatory knobs uncommented and nothing else. The diagnostics
stay documented above and commented out below. That is what the file says it is for.

Also worth a line while in there: the doc cites `radv_physical_device.c:1079` for `ORBIS_NO_TESS`,
which is correct today — but `:1149` reads the same variable for `.multiviewTessellationShader`,
and RADV's own comment there says *"ORBIS_NO_TESS HAD BEEN HALF A SWITCH, and two runs died on the
half it did not cover."* Cite both or cite neither.

**HW:** the fix is confirmed by the run that already established the crash. No new trip needed.

## 2.4 Every sound is held twice, and a comment promises otherwise

`Source::pcm`'s comment: *"shared with Tempest's Sound::Data so the samples outlive the SoundEffect
if the same Sound is played twice."*

The code at `:892` makes a **fresh** `make_shared<vector<int16_t>>(n)` per effect and memcpy's into
it. Nothing is ever shared. So:

* `Sound::Data` holds one full copy (`unique_ptr<char[]>`, `implLoad:812`),
* each `SoundEffect` holds another,
* N slots playing the same sound hold N copies.

Gothic ships 968 sound files and 6000 speech lines. On a console this is worth the change, and
`Sound::Data` is *already* a `shared_ptr<Data>` — the sharing the comment describes is one refactor
away, not a redesign.

**Fix:** `Sound::Data` carries the `shared_ptr<vector<int16_t>>` directly (it already carries the
channel count in `format`, so the layout is already ours), and `SoundEffect` aliases it. The
comment then becomes true.
**Verify:** memory census before/after with several voices on the same sound. **HW:** useful, not
required.

## 2.5 The boot probes cost most of a second on every launch

`boot()` runs `probeDirent`, `probeVdfRead`, `probeDataInventory`, `orbis::probeCtype` and
`probeSavePaths` unconditionally, every time. That is roughly 40-60 `ps4_log` lines — one per file
in `Data/` — at 8-15 ms of klog each: **0.5-0.9 s added to every boot**.

`probeSavePaths` also has side effects that outlive it: `mkdir("/data/OpenGothic")` on every launch,
and five create/write/stat/read/unlink round trips — one of them **into the player's own Gothic
installation** (`root + "probe.sav"`). If the unlink fails, a stray file is left in their game data
and only the log says so.

Each probe earned its place by convicting something specific. The question is whether they should
still run when nothing is being diagnosed.

**Fix — decide:**
* Gate the lot behind a key in `/app0/opengothic.cfg` (`probes=1`), default off. The regression
  armour argument for `probeVdfRead`'s VERDICT line is real, so that one may want to stay.
* Or keep them and drop the per-file inventory lines to a single summary, which is most of the
  count.
* Drop the `root + "probe.sav"` candidate regardless — writing into someone's game install to
  learn something we already know is not worth it.

**HW:** measurable directly from the log timestamps in an existing run.

## 2.6 L2 and R2 mask almost every bind while held — `ps4/tempest-pad.cfg`

The file explains that a modifier is matched exactly and that "an input is excluded from its own
modifier mask", so `L2` still fires its own jump. What it does not say is the other half: **while
L2 or R2 is held, the mask is non-empty, so every bare bind becomes an unbound `L2+X`.**

    L2 = LAlt    jump          holding it masks: Cross Circle Triangle Square DUp DDown DLeft DRight
    R2 = LCtrl   attack        same, and turns L1+X into L1+R2+X

So no attack while jumping, and no screen (inventory, status, log, map) reachable while attacking.
R2-as-attack and L2-as-jump are held actions in normal play, not taps.

This was already on the handoff's leftover list; the review confirms it from the file rather than
from memory.

**Fix — decide:** either an input bound to a key is excluded from the mask entirely (the general
form of the rule the file already states for its own bind), or L2/R2 stop being modifiers and the
camera-distance layer moves to a button that is not also an action.
**HW:** yes, and it is judged by feel rather than by a log.

---

# TIER 3 — the rest — DONE 2026-08-21

All six applied. Run on hardware: reached `exec`, five minutes of play, 56065 blocks, 21 voices,
221 sounds decoded, 0 refused. The maintainer's verdict was "wszystko działa jak działało", which
is the correct outcome for a tier that is almost entirely hygiene.

**3.1 was proved rather than assumed.** A control build with `--sound-null` contains no
`sceAudioOutOpen`; the default build contains it. The knob genuinely selects the other backend.

**3.5 followed the rule from the comment audit — keep the FACT, drop the POINTER:**

    "Measured on the console by ps4/gapi-suite, twice"  ->  "Measured on the console twice"
    "og_ps4_stat.h has the full decode"                 ->  "the overlay's orbis_stat.h has ..."
    "Under unemups4 the same lock returned success"     ->  "Under an emulator the same lock ..."

One reference was kept deliberately: `lib/Tempest/ps4/opengothic/` in og_sound_orbis.cpp. That is
not a dead pointer, it is the history that explains why the include in 1.9 resolved by accident.
Deleting it would turn an explanation into a puzzle.

Real, but each was arguable and none was blocking.

## 3.1 `OG_SOUND_NULL` has no knob — `CMakeLists.txt:114-125`

Both sound files are compiled and each guards its whole body on the macro, which is a good design.
But nothing sets it: no `option()`, no `set()`, no `-D` in `ps4/build.sh`. The control rung needs a
hand-edited compile flag.

Exactly inverted against 1.3, which is a knob with no code. Fix is one `option(OG_SOUND_NULL ...)`
plus a `target_compile_definitions`, and `--sound-null` in `build.sh`.

## 3.2 `workers.cpp` reaches into RADV for a log channel — `game/utils/workers.cpp:7`

    extern "C" void ac_orbis_note(const char *msg);

A hand-written declaration of a driver-internal symbol, no header, from game code — while the whole
rest of the port logs through the overlay's `ps4_log`, which `workers.cpp` could include with one
line. If Mesa's signature ever changes there is no diagnostic, only a link error at best.

Trivial to fix; listed here because it is a layering question, not a bug.

## 3.3 `decodeWav` allocates from an untrusted length

`raw.resize(sz)` where `sz` is a `uint32_t` read straight from the file header — up to 4 GiB. Gothic's
own archives are the only input today, so this is a mod-and-corruption question, not a live one.
Contained by `SoundDevice::load`'s `catch(...)`.

## 3.4 `imeBegin`'s ladder is unreachable if the first panel fails — `og_ps4_ime.cpp`

If the very first `sceImeDialogInit` returns non-zero, the function returns false **without
incrementing `g_opened`**. Every later attempt therefore takes the `g_opened==0` "panel 1" path
again — a plain Init — and the four-teardown ladder is unreachable for the process's lifetime.

The ladder is the thing that would catch the 0x80bc0008 restriction coming back, so it is worth
being reachable. One line: walk the ladder whenever a plain Init refuses, not only when
`g_opened>0`.

## 3.5 Dead cross-references in this diff

The handoff already carries "26 files reference eight things that no longer exist". These are the
ones inside the 18 staged paths, so they are cheap to take while the files are open:

    og_ps4_boot.h / .cpp   backlog/docs/session-state.md ×2, scripts/ps4/run-tests.sh,
                           ps4/gapi-suite, unemups4, og_ps4_stat.h / og_ps4_stat.cpp ×3
                           (moved to orbis-compat), ps4/opengothic/… ×3 (the directory is ps4/)
    CMakeLists.txt         backlog/docs/session-state.md, ps4/opengothic/og_ps4_boot.h

The rule from the comment audit applies: keep the fact, drop the pointer.

## 3.6 `currentTime()` is meaningless for a producer — `og_sound_orbis.cpp`

`SoundEffect::currentTime()` returns `cursor * 1000 / rate`, but for a producer voice `cursor`
indexes the *pull buffer* and is rebased to near zero on every refill (`mixProducer`:
`v.cursor -= double(i0)`). So a music or video effect reports a time near zero forever.

**Checked: nothing in OpenGothic calls it.** `grep -rn 'currentTime()' game/ lib/Tempest/Engine/`
finds only the declaration and Tempest's own definition — no call site anywhere in the title. So it
is defined because the header declares it, like `SoundDevice::process()` beside it.

That keeps it in Tier 3, and lowers it: the honest fix is a comment saying the value is only
meaningful for a static sound, not a rewrite. Worth one line if the file is open for 2.2 anyway.

---

# THE ONE THE REVIEW DID NOT FIND — `VfsMountMode::LAZY`

Not a review finding. It came out of the failure of 2026-08-20, and it is worth more than any
single item above.

    game/resources.cpp   mount_disk(i.name, VfsOverwriteBehavior::OLDER);   // two arguments

ZenKit's third argument defaults to `VfsMountMode::FULL`, which mmaps each archive WHOLE and keeps
the mapping alive in `Vfs::_m_data_mapped` for the life of the process. **PS4 mmap POPULATES
EAGERLY** — 37 s to touch three bytes of a 722 MB archive. That is ~2.69 GiB of Gothic II faulted
into a process with 387 MiB of flexible memory.

⚠ **We introduced the default ourselves.** ZenKit fork commit `39bf1344` (2026-08-18, "feat(vfs):
Support for lazy loading VDF files") added `VfsMountMode` and chose `FULL`. `resources.cpp` was
never updated to ask for `LAZY`. The branch has been compiled in the whole time — `_ZK_WITH_PREAD=1`
in every build — and nothing ever called it. Before that commit the two-argument form did the pread
thing unconditionally, so a feature commit turned a guarantee into an opt-in and nobody opted in.

    FULL, cold          console dies at Speech2.vdf — no pad, no log, reboot
    FULL, warm          20.2 s
    FULL, very warm      1.9 s   (an outlier, not the norm)
    LAZY                 0.90 s  <- faster than FULL's best case, all 15 archives

Speech1.vdf, 722 MB: 0.119 s. Speech2.vdf, the one that killed the console: 0.129 s.

**Why the review missed it.** Every item in this document came from reading the diff — and this
line is not in the diff. `resources.cpp` was untouched by the port; the defect was in what the call
did NOT say. A review that reads changes cannot see a default that quietly changed underneath one.

⚠ **And why the bisect that followed was worthless:** the failure is not deterministic. Two
functionally identical binaries — same symbols, same objects but for embedded paths — gave opposite
outcomes, and the same binary took 1.9 s and 20.2 s for the same work. Eight console runs were spent
before the maintainer asked "to my nie robimy lazy?" and one argument settled it. Recorded in
memory as [[one-console-run-is-not-a-verdict]].

# What the order turned out to be

The plan's order was followed and it held. Recorded as run, with what each cost:

    Tier 1, eleven items, one pass, one build   the tenth was 1.9, which the plan did not have
    Tier 2, six items, one build                2.3 needed no console trip, as predicted
    Tier 3, six items, one build                plus one control build for 3.1

    THE FAILURE IN BETWEEN                      eight console runs, six dead hypotheses,
                                                and a defect the review could not have found

⚠ **The plan said "do Tier 2 one at a time, with a run between them." It was not followed** — all
six went in one package, and Tier 3 likewise. That was the right call only because the changes are
in different files and nothing broke; had something broken, attribution would have cost as much as
the night before. The rule stands for anything touching a shared path.

---

# What this review is, and what it is not

Every finding above came from reading the code, not from running it. Three were checked against
something outside the file — the SDK's own `ime_dialog.h` for the status enum, `ps4_app.h` for the
klog cost, a one-command compile for the shim collision — and those three are the ones stated as
facts rather than as readings.

Two categories showed up that the comment audit of the same morning was built to catch, and did
not, because they are not comments that stopped being true:

* **a knob with no code** (1.3) and **code with no knob** (3.1), which are the same mistake seen
  from either end;
* **a diagnostic that reports the opposite of the truth** (1.7) — the first census line says the
  mixer thread is not running, and prints it from the mixer thread.

The third shape is new and worth naming: **a document that contradicts its own warning** (2.3). The
env example spends six lines saying two knobs are mandatory and then ships neither. Nothing in the
tree could have caught that — it is not code, it does not compile, and the warning and the payload
are forty lines apart in one file.
