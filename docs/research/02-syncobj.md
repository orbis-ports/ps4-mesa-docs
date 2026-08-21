# Research 2 — syncobj: a vtable to fill, not an emulation to design

**Status: the unknown collapsed. This was called "a subsystem, not nine functions"; it is one struct with
fourteen function pointers, nine of which may be left NULL.**

## It is a vtable, and Mesa already expects a second implementation

```c
/* src/util/u_sync_provider.h */
struct util_sync_provider {
   int (*create)(p, uint32_t flags, uint32_t *handle);
   int (*destroy)(p, uint32_t handle);
   int (*handle_to_fd)(p, handle, int *out_obj_fd);
   int (*fd_to_handle)(p, int obj_fd, uint32_t *handle);
   int (*import_sync_file)(p, handle, int sync_file_fd);
   int (*export_sync_file)(p, handle, int *out_sync_file_fd);
   int (*wait)(p, uint32_t *handles, unsigned n, int64_t timeout_nsec, unsigned flags,
               uint32_t *first_signaled);
   int (*reset)(p, const uint32_t *handles, uint32_t count);
   int (*signal)(p, const uint32_t *handles, uint32_t count);
   int (*timeline_signal)(p, handles, uint64_t *points, count);
   int (*timeline_wait)(p, handles, points, n, timeout_nsec, flags, first_signaled);
   int (*query)(p, handles, points, count, flags);
   int (*transfer)(p, dst_handle, dst_point, src_handle, src_point, flags);
   void (*finalize)(p);
   struct util_sync_provider *(*clone)(p);
};
```

`util_sync_provider_drm(int drm_fd)` is the existing one, and it is declared **under
`#if HAVE_LIBDRM` with a stub `#else` arm** — so the place for a non-DRM provider is already carved out.
`ac_drm_device_get_sync_provider(dev)` is how RADV reaches it, and it is one of the 54.

## Nine of the fourteen can be NULL, and one NULL is load-bearing

```c
/* ac_gpu_info.c:1595 */
info->has_timeline_syncobj = ac_drm_device_get_sync_provider(dev)->timeline_wait != NULL;
```

Leave `timeline_wait` NULL and RADV simply does not advertise the features built on it:

```c
/* radv_physical_device.c */
.KHR_timeline_semaphore = pdev->info.has_timeline_syncobj,   /* :809  */
.KHR_present_wait       = pdev->info.has_timeline_syncobj,   /* :767  */
.timelineSemaphore      = pdev->info.has_timeline_syncobj,   /* :1119 */
```

**That is a clean degradation, not a failure** — the extensions vanish from the device's list and
applications that need them see an honest absence. It takes `timeline_signal`, `timeline_wait`, `query`
and `transfer` off the critical path.

The four fd-based entries — `handle_to_fd`, `fd_to_handle`, `import_sync_file`, `export_sync_file` — are
cross-process sharing, which has no meaning for one homebrew process. NULL, and the extensions built on
them go with it. Plus `clone`, needed only when a provider is duplicated across devices.

## Five must exist, and the machinery for them is already on the console

```
create    destroy    signal    reset    wait
```

**`has_syncobj` is hardcoded, not probed** (`ac_gpu_info.c:1065`: `info->has_syncobj = true;`) — so RADV
assumes basic syncobj works and these five cannot be refused. They are what Vulkan binary semaphores and
fences ride on.

All five are counter operations over the machinery the GNM backend already has on hardware:

| provider entry | what it becomes |
|---|---|
| `create` | allocate a slot in the label page; return its index as the `uint32_t handle` |
| `destroy` | free the slot |
| `signal` | write the label from the CPU — the host-signal path |
| `reset` | zero the label |
| `wait` | `GnmDevice::waitFence`'s loop: poll the label with `sceKernelUsleep`, honouring `timeout_nsec` and reporting `first_signaled` |

The GPU-side signal is not a provider entry at all: it arrives as `CHUNK_ID_SYNCOBJ_OUT` on the submit,
so the arm turns that chunk into the end-of-pipe timestamp write the backend already emits
(`dcb.timestampEop`). **That is the join between the two research items** and the reason submit should be
implemented first.

## Falsifiable first step, no console

Write the provider with the five entries over a **plain host implementation** — an array of `uint64_t`
counters, `signal` incrementing, `wait` spinning with a timeout — and run the host RADV with the drm-shim
against it. A `vkQueueSubmit` + `vkWaitForFences` on the shim (whose submit is a no-op, so every fence is
trivially signalled) exercises the whole path. If RADV initialises, allocates semaphores and returns from
a wait, the shape is right; only the *backing store* then changes on the console, from an array to the
label page.

## What could still surprise

* **`wait` semantics.** `flags` carries `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL` and `_WAIT_FOR_SUBMIT`. The
  second one means "block until the syncobj even has a fence attached", which on the console has no
  analogue and needs a decision rather than a translation.
* **`first_signaled`** is an out-parameter RADV uses for wait-any. Cheap, but easy to leave unwritten and
  then wonder why a wait-any returns the wrong index.
* **Timeouts are absolute nanoseconds** in DRM's convention, and `GnmDevice::waitFence` takes
  milliseconds — the conversion is the sort of thing that turns a 1 ms wait into a 1 000 000 ms hang.
* Whether RADV needs `has_syncobj` to be *true* for anything beyond these five. It is hardcoded, so the
  flag says nothing about what it enables; the way to find out is to run and see what it calls.
