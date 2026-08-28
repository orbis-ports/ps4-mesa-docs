# The RetroArch port's record

`HANDOFF.md` is the running account of the PlayStation 4 port of RetroArch — newest entry last,
every entry written from what the console said rather than from what was expected. `PLAN.md` is
the plan it executed. `CORE-CONTENT.md` lists the BIOS and game data each built core needs.

The code lives in `orbis-ports/RetroArch`, branch `ps4-support`. Nothing in the build reads these
files.

## ⚠ Two files stayed in the code repository, and not by oversight

They are **inputs to the build**, not documentation, and CI checks out only the code repository:

    ps4/RELEASE-NOTES.md   .github/workflows/frontend.yml passes it as `body_path`, so it is the
                           text of every GitHub Release. Moving it would make cutting a release
                           depend on a second checkout.
    ps4/CORE-STATUS.md     ps4/shard-cores.sh reads its first table as the size proxy that deals
                           164 cores into balanced shards. It is a data table that happens to be
                           readable prose.

`ps4/core-patches/README.md` stayed too: it is the README of a directory of patches, read by
whoever is editing them, next to what it describes.
