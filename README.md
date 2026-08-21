# The record of the PS4 RADV port

Everything here is an account of work, not a part of any build. Nothing in `~/src-ps4` reads these
files; deleting the lot would break no compilation and lose the whole reasoning behind the port.

Split out of `mesa-ps4/build-support/orbis/` on 2026-08-21, when `build-support/` was cut down to the
three things that actually build something: `build.sh`, `tools/` and a README.

    docs/HANDOFF.md          the running account, newest entry last. The single most useful file here
    docs/PLAN.md             the implementation plan, phases 0-7
    docs/TODO.md             the function-by-function work list the arm was written against
    docs/PLAN-opengothic-*   the title's own review and split
    docs/research/           five investigations that each settled one question
    notes/                   captured data - radeon_info dumps from console, laptop and a Bonaire
                             reference; the ac_drm_* surface lists; libdrm's symbol table
    cts.md                   the Vulkan CTS port: why it exists, how to run it, what it found

## Where the things these files talk about now live

    the driver            ~/src-ps4/mesa-ps4                  branch orbis, one staged change set
    the platform overlay  ~/src-ps4/orbis-compat              toolchain, vkloader, ps4-app, shims
    the CTS               ~/src-ps4/VK-GL-CTS   ps4-support    run config in targets/orbis/
    the engine            ~/src-ps4/Tempest     ps4-support
    the title             ~/src-ps4/OpenGothic  ps4-support
    ZenKit                ~/src-ps4/ZenKit      ps4-support

⚠ **A record goes stale in a way code does not**, because nothing fails when it is wrong. Two examples
found on the day this directory was created: `orbis-compat/include/signal.h` stated twice that the CTS
fork carried a patch it does not carry, and `cts.md`'s build recipe named a toolchain path and a
`-DORBIS_CTS_RUNTIME` file that had not existed for two days. Neither was visible to a build, a test or
a console run. Read these files against the thing they describe before trusting a detail.

⚠ **Mesa upstreaming is closed**, decided 2026-08-21: they will not take Orbis as a target and the
process is not worth the maintainer's time. `docs/PLAN.md` §8 and the "upstreamable list" in
`docs/HANDOFF.md` are kept as history but describe work that will not happen. The six vetted Mesa bugs
in that list are real, and their commits survive only via `refs/backup/orbis-267-commits` in
`~/src-ps4/mesa-ps4` and on the branch in the backup checkout at `/home/mikolaj/src/mesa-ps4`.

ZenKit and OpenGothic are a different matter - separate projects, with pull requests already open.
