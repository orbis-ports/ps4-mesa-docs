# Research 1 — `ac_drm_cs_submit_raw2`: amdgpu's submit onto Sony's

**Status: the unknown is nearly closed by reading. One question left, and it is answerable without
hardware.**

This was framed as "the one real research item — how does RADV's IB chaining map onto Sony's DCB/CCB".
Most of that dissolved on contact with the source.

## What a submit actually is

`ac_drm_cs_submit_raw2(dev, ctx_id, bo_list_handle, num_chunks, chunks, seq_no)` takes the kernel's raw
chunk array. RADV sends **seven** chunk kinds (`grep AMDGPU_CHUNK_ID_ src/amd/vulkan/winsys/amdgpu/`):

```
CHUNK_ID_IB                       the command buffers
CHUNK_ID_FENCE                    where to write completion
CHUNK_ID_BO_HANDLES               the resident set
CHUNK_ID_SYNCOBJ_IN / _OUT
CHUNK_ID_SYNCOBJ_TIMELINE_WAIT / _SIGNAL
```

And an IB chunk is **an address and a size**, nothing more:

```c
struct drm_amdgpu_cs_chunk_ib {
   __u32 _pad, flags;
   __u64 va_start;      /* virtual address to begin IB execution */
   __u32 ib_bytes;      /* size of submission */
   __u32 ip_type, ip_instance, ring;
};
```

`sceGnmSubmitCommandBuffers` takes **arrays** of DCB addresses and sizes. So N IB chunks become N array
entries. **The mapping is close to one-to-one**, which is what the framing missed.

## What the reading settled

* **IB chaining works on gfx7 and is switchable.** `info->can_chain_ib2 = info->gfx_level >= GFX7`
  (`ac_gpu_info.c:1153`), and `ws->chain_ib = !(debug_flags & RADV_DEBUG_NO_IB_CHAINING)`
  (`radv_amdgpu_winsys.c:254`) — on by default, with a debug flag that turns it off. So if a chained IB
  proves awkward against Sony's submit, **RADV can be told to emit a flat list instead**, and that is a
  documented knob rather than a patch.
* **`BO_HANDLES` can be ignored.** It is amdgpu's per-submit residency list; on the PS4 everything mapped
  is resident, so the chunk carries no information this platform needs. Ignoring it is a decision, not an
  omission — write it down at the call site.
* **The four syncobj chunks are deferrable.** A first light-up handles `IB` and `FENCE` and refuses the
  rest; see `research/02-syncobj.md` for which of them degrade cleanly.

## The one question left, and it is already answerable

**`ip_type`: GFX versus COMPUTE.** Sony's DCB/CCB submit is the graphics ring; compute queues go through a
different door (`sceGnmMapComputeQueue`, `sceGnmDingDong`). RADV builds `AMD_IP_COMPUTE` command streams
(`radv_cmd_buffer.c:1716`, `radv_cs.c:40` — `is_mec`).

**But the compute queue family is gated on a number we synthesise:**

```c
/* radv_physical_device.c:152 */
return pdev->info.ip[AMD_IP_COMPUTE].num_queues > 0 && ...
```

`num_queues` comes from `ac_drm_query_hw_ip_info`, which is one of the 54 functions this port writes. So
**reporting zero compute queues removes the second ring entirely** and every submission goes down the
graphics path — which Vulkan permits, since the GFX ring services graphics and compute and a driver may
expose a single queue family.

That turns the last unknown into a **choice with a known default**: start with one queue family, and add
the compute ring later only if a profile says the async path is worth it.

## Falsifiable first step, no console

1. Implement `ac_drm_cs_submit_raw2` to **decode and log** rather than submit: walk the chunk array, print
   each chunk's kind, and for `IB` print `va_start`, `ib_bytes`, `ip_type`, `ring`.
2. Run one `vkCreateComputePipelines` + a trivial dispatch under the **host** RADV with the drm-shim
   (`AMDGPU_GPU_ID=bonaire`), whose submit ioctl is already stubbed out.
3. **The log is the answer**: how many IBs per submit, whether they are chained, which chunk kinds arrive
   in what order, and whether `ip_type` is ever COMPUTE once the queue count says zero.

That runs on the laptop, needs no PS4, and produces the exact shape the PS4 arm has to accept — measured
rather than assumed. Sony's side is already proven by the GNM backend, which has been submitting
hand-rolled PM4 to this console for a year.

## What could still surprise

* Sony's submit has a **count limit** on the DCB/CCB arrays; our own backend has only ever passed one
  pair. If RADV routinely submits more IBs than that limit, the arm must batch. Checkable in the
  OpenOrbis headers and in `~/src/unemups4/oracles/`.
* `flags` on the IB chunk (`AMDGPU_IB_FLAG_*`) — preamble, preempt, CE. Worth enumerating which RADV sets
  on gfx7 before assuming they are all ignorable.
* IB alignment: `ib->ib_mc_address % ctx->ws->info.ip[ib->ip_type].ib_alignment == 0` is asserted, and
  `ib_alignment` is another field we synthesise. The bonaire shim says 32 for gfx7.
