# notes/

Generated data, not hand-written. Regenerate after any Mesa bump:

    # what ac_linux_drm.c wants from libdrm - the symbols the empty stub does NOT resolve
    cd <mesa>/build-nm
    nm --undefined-only src/amd/vulkan/*.p/*.o src/amd/common/*.p/*.o \
      | awk '{print $NF}' | sort -u | grep -E '^(amdgpu_|drm)' > libdrm-symbols.txt

    # THE ac_drm_* SURFACE. src/amd/vulkan/ ALONE IS NOT ENOUGH - src/amd/common/ (ac_gpu_info.c,
    # ac_surface.c) calls another twenty, and those files are compiled into the driver too. Grepping
    # only amd/vulkan is how an earlier count said 39 instead of 54.
    cd <mesa>
    cat <(grep -rhoE 'ac_drm_[a-z0-9_]+' src/amd/vulkan/) \
        <(grep -rhoE 'ac_drm_[a-z0-9_]+' src/amd/common/*.c) | sort -u \
      | grep -vE '^ac_drm_(bo|device|fourcc|bo_import_result)$' > ac_drm-surface.txt

    # and what is live on gfx7: user queues are GFX11+
    grep -vE 'userq|uq_fw' ac_drm-surface.txt > ac_drm-surface-gfx7.txt

The grep -v drops four TYPE names that the pattern also catches - `ac_drm_bo`, `ac_drm_device`,
`ac_drm_fourcc`, `ac_drm_bo_import_result`. Counting them as functions is how an earlier note in the
Tempest backlog said 43 instead of 54.


## ac_drm-missing-link.txt - what the LINKER says is missing, 13 symbols

This is the one the plan wanted, and it is a *smaller* list than the 54, for a reason worth understanding:
with `-Dplatforms=orbis` the amdgpu winsys is out of the build, and **the winsys is what calls the other
41**. So this list is "what RADV needs when it has no winsys at all" - which is exactly the device-query
group, and exactly what a first light-up needs.

Regenerate from the built archive:

    cd build-orbis
    A=src/amd/vulkan/libvulkan_radeon.a
    llvm-nm --defined-only $A | awk '{print $NF}' | sort -u > /tmp/D.txt
    llvm-nm -u           $A | awk '{print $NF}' | sort -u > /tmp/U.txt
    comm -13 /tmp/D.txt /tmp/U.txt | grep '^ac_drm_'

`comm -13` rather than plain `nm -u`: an archive's members reference each other, so most "undefined"
entries are defined by a sibling object. Skipping the subtraction reports thousands.

The same command without the final `grep` is how the other two gaps were found: **`main`** (RADV was a
`shared_library`; on a platform with no ICD loader it has to be an archive) and **`thrd_current`**, which
OpenOrbis *declares* in `threads.h` and does not define in `libc.a`.


## radeon_info-orbis.txt and radeon_info-console.txt - the comparison this project exists to make

Both are `RADV_DEBUG=info` dumps of `radeon_info`: one from the host build of our own arm
(`./build.sh --host-orbis`), one from `ps4/radv` on real hardware. **They are identical - 164 fields, zero
differences.**

Regenerate the host one with `./build.sh --host-orbis`; the console one comes from the harness, which
relays it over UDP with a `radv-info|` prefix:

    sed -n 's/^[0-9:.]* radv-info| //p' build-ps4-logs/ps4-udp-*.log > notes/radeon_info-console.txt

⚠ The dump does NOT go through `mesa_log`, and on this platform it cannot go through `stdout` either -
`freopen` fails. The harness patches the one call site to `fopen` a file directly. Without that the file
comes back empty and looks like the driver never ran.
