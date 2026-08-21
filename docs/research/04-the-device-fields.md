# Research 4 — the 39 device fields, and which of them actually decide anything

**Status: the phase-2 unknown is much smaller than it looked, because the field group that would have been
hardest to get right turns out not to come from the chip identity at all.**

## The list

`ac_gpu_info.c` reads 39 fields off `drm_amdgpu_info_device`:

    grep -oE 'device_info(\.|->)[a-z0-9_]+' src/amd/common/ac_gpu_info.c | sort -u

Roughly ten are GFX11+ and legitimately zero on gfx7 — `mall_size`, `tcp_cache_size`, `num_sqc_per_wgp`,
`sqc_data_cache_size`, `sqc_inst_cache_size`, `gl1c_cache_size`, `gl2c_cache_size`, `userq_ip_mask`,
`enabled_rb_pipes_mask_hi`, and `pa_sc_tile_steering_override` (gfx10). Zero there is the right answer, not
a gap.

## Liverpool is not in Mesa's chip table, so the identity is a CHOICE

```c
/* ac_gpu_info.c:646 - ac_identify_chip */
switch (device_info->family) {
   case FAMILY_CI: identify_chip(BONAIRE);  identify_chip(HAWAII);  break;
   case FAMILY_KV: identify_chip2(SPECTRE, KAVERI);  identify_chip2(KALINDI, KABINI);  ...
```

`identify_chip` is `ASICREV_IS(device_info->external_rev, <asic>)`, from addrlib's
`amdgpu_asic_addr.h`. **There is no Liverpool arm**, because it is a semi-custom part that never shipped a
Linux driver. So this port picks `family` and `external_rev`, and whatever it picks is what RADV believes
the silicon is.

That sounded like the most dangerous decision in the phase. It is not, for a reason worth writing down.

## What the identity does NOT decide: addressing

I expected the chip enum to select addrlib's pipe configuration, which would have put it in direct conflict
with something already **measured** on Liverpool — that it takes the 8-pipe branch of the CIK tile table
with `PIPE_CONFIG P8_32x32_16x16` (`backlog/docs/gnm-tiling.md`, H5/H6/H7). It does not:

```c
ac_gpu_info.c:443   info->gb_addr_config = amdinfo->gb_addr_cfg;                        /* a register */
ac_gpu_info.c:448   info->num_tile_pipes = 1 << G_0098F8_NUM_PIPES(info->gb_addr_config);
ac_surface.c:828    regValue.pMacroTileConfig = info->cik_macrotile_mode_array;         /* the tables */
```

`gb_addr_cfg`, `gb_tile_mode[32]` and `gb_macro_tile_mode[16]` all arrive through
**`ac_drm_query_gpu_info`** — that is, through us. Addressing is driven by registers this port supplies,
not by the name it claims. Three consequences:

1. **The tiling values the fork has already measured go straight in.** This is the one field group where
   the work is done rather than pending.
2. **It becomes a cross-check rather than a risk.** Feed addrlib the measured tables and its addresses must
   agree with what the GNM backend's own tiling code produces for the same surface. Two independent
   implementations of CIK addressing, one of which has passed on hardware — that is a far better test than
   either alone, and it runs on the laptop.
3. The 8-pipe measurement stops being a constraint on the identity choice, which frees the choice to be
   made on the grounds that actually matter, below.

## What the identity DOES decide: which hardware bugs RADV works around

```c
ac_gpu_info.c:325   out->has_lds_bank_count_16 = info->family == CHIP_KABINI || ...
ac_gpu_info.c:411   out->has_primid_instancing_bug = info->gfx_level == GFX6 && info->max_se == 1;
ac_gpu_info.c:364   out->smaller_tcs_workgroups = !info->has_distributed_tess && info->max_se > 1;
```

So the choice is: **which generation-mates' bug list should Liverpool inherit?** That is a tractable
question, because every flag is a falsifiable statement about the silicon rather than a label.

`has_lds_bank_count_16` is the interesting one, and it is not idle: this project has an **open question
about `ds_read_b128` / `ds_write_b128` on gfx700** and an amdllpc pin named `out-ds128` that may have
disabled both. A flag that says "this chip's LDS has 16 banks instead of 32" is adjacent to that, and
claiming CHIP_KABINI would switch it on while CHIP_BONAIRE leaves it off. **Neither should be chosen by
resemblance.** The honest sequence is: pick the identity whose flags are all either measured or inert,
and treat any flag that is neither as a hypothesis with a test attached.

## What `num_shader_engines` reaches

```c
ac_gpu_info.c:1194  info->max_se = device_info->num_shader_engines;
ac_gpu_info.c:1226  for (int i = 0; i < info->max_se; i++)     /* the cu_bitmap walk */
```

Load-bearing for `smaller_tcs_workgroups` and the CU-mask walk, but **not** for addressing. So getting it
wrong is a performance and occupancy error rather than a corruption one — which is still the class of
defect that is hardest to notice, hence **task-94**: measure `SE_ID`/`SH_ID`/`CU_ID` out of
`SQ_WAVE_HW_ID` rather than assert them.

## The reference dump, and why it is worth building the host for

`RADV_DEBUG=info` makes Mesa print the whole `radeon_info` struct. Two runs — one against the drm-shim's
`bonaire` entry, one against our arm — and the diff is the list of fields still wrong, on a laptop.

**Captured: [`notes/radeon_info-bonaire.txt`](../notes/radeon_info-bonaire.txt), 165 fields.**

    LD_PRELOAD=build-host/src/amd/drm-shim/libamdgpu_noop_drm_shim.so \
    AMDGPU_GPU_ID=bonaire \
    VK_DRIVER_FILES=build-host/src/amd/vulkan/radeon_devenv_icd.x86_64.json \
    RADV_DEBUG=info vulkaninfo --summary

⚠ **The variable is `RADV_DEBUG`, not `AMD_DEBUG`** — `AMD_DEBUG` is radeonsi's, and with it the dump is
silently absent rather than refused. Two earlier notes in this repo said `AMD_DEBUG`; both are corrected.
And the count is **165**, not the 245 an earlier note claimed — that number had no source.

The identity decision from research above is now concrete, because the dump prints what Bonaire reports:

    family = 54        (CHIP_BONAIRE, radeon_family ordinal)
    family_id = 120    (addrlib FAMILY_CI)  <- this is device_info->family
    chip_external_rev = 21                  <- this is device_info->external_rev
    chip_rev = 1

So `family_id = 120` with `external_rev = 21` is exactly what makes RADV say CHIP_BONAIRE, and
`has_lds_bank_count_16 = 0` follows. That is the pair this port would report to inherit Bonaire's bug list.

And the dump makes the legitimate differences explicit rather than leaving them to be discovered in a diff:

| | Bonaire (reference) | Liverpool |
|---|---|---|
| `num_rb` / `max_render_backends` | 4 | **8**, measured (`gnm-tiling.md` H5) |
| `num_cu` | 14 | not citable — task-94 |
| `num_se` | 2 | not citable — task-94 |
| `num_cu_per_sh` | 7 | not citable — task-94 |
| `IP COMPUTE queues` | 4 | **0** by choice (research/01) |
| `max_gpu_freq` | 1075 MHz | ~800 MHz (`gnmtune.h:138`) |

**Bonaire is a structural reference, not a value one.** It is real gfx7 whose `hw_ip`, firmware versions
and `mmr_regs` were dumped from hardware, so it says what a *well-formed* answer looks like for about 165
derived values. It is not Liverpool, and it does not have Liverpool's CU count or its 8 RBs. A diff line is
therefore not automatically a defect — but a field that is wrong in a way Bonaire's is not, is.

## What could still surprise

* `-Dplatforms` must not be shared between the two builds. Passing `orbis` to the host build makes
  `dep_libdrm` a `null_dep`, so the drm-shim compiles without libdrm's own `-I` and dies on
  `xf86drm.h`'s `#include <drm.h>`. Cost one build round.
* `external_rev` is matched by `ASICREV_IS`, which is a **range** test in addrlib, not an equality. A value
  chosen to hit one arm may sit inside another's range too, and `identify_chip` has no `break` — the last
  matching arm wins. Worth reading the macro rather than assuming.
* `ac_gpu_info.c:449` asserts that the pipe-interleave field of `gb_addr_config` matches
  `info->pipe_interleave_bytes`. Our measured 256 B has to be encoded in the register the way the macro
  expects, or the driver asserts during init rather than rendering wrongly — which is the good failure.
