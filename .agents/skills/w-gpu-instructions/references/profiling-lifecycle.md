# Profiling and lifecycle

## Measure the workload

- Separate CPU preparation/encoding, GPU execution, transfers, and presentation latency.
- Warm pipelines/allocations; compare fixed workloads and quality on supported target backends.
- Measure distributions and whole-frame cost; do not infer speed from branch removal or workgroup size.
- Grant `TIMESTAMP_QUERY` for pass timestamps; gate inside-pass/encoder writes separately.
- Multiply timestamp differences by `Queue::get_timestamp_period()` for nanoseconds.
- Resolve into `QUERY_RESOLVE | COPY_SRC`, then copy into `MAP_READ | COPY_DST`.
- Align resolve destination offsets to 256 bytes; map asynchronously and reclaim completed slots.
- Reject stale/invalid samples; include hidden indirect validation and copies in whole-frame analysis.
- Use backend captures when needed to explain costs outside timed passes.

## Errors and ownership

- Label GPU objects by purpose/variant; preserve source locations for generated WGSL.
- Install uncaptured-error and device-loss handlers; capture recoverable creation failures in narrow scopes.
- For Rust 30, retain the guard from `push_error_scope`, call `guard.pop()`, then await the result.
- Pop nested scopes in reverse order; do not retain them across unrelated work or task reordering.
- Pass `Arc<dyn UncapturedErrorHandler>` to `on_uncaptured_error`.
- Give `set_device_lost_callback` a `Send + 'static` closure receiving reason and message.
- Use safe handle ownership for submitted resources; do not add fixed-frame deferred deletion.
- Use completion tracking for explicit mapping pools/raw access.
- Do not destroy resources needed by future work.
- Retain descriptors, shader/asset sources, and reload paths sufficient to rebuild GPU state.

## Surface and device recovery

- Suspend acquisition/configuration for zero-sized windows; recreate size-dependent targets on resize.
- Release acquired textures before reconfiguration; retain pipelines with compatible format/sample contracts.
- Handle Rust 30 `CurrentSurfaceTexture` outcomes as follows:

| Outcome | Action |
| --- | --- |
| `Success(texture)` | Render, submit, then call `queue.present(texture)`. |
| `Suboptimal(texture)` | Present usable contents and schedule reconfiguration. |
| `Timeout` / `Occluded` | Skip the frame; avoid a blocking retry loop. |
| `Outdated` | Release outstanding textures and reconfigure. |
| `Lost` | Recreate the surface; check whether the device was lost too. |
| `Validation` | Report and fix the captured/uncaptured validation cause. |

- After device loss, request a new device/queue and rebuild all device-owned resources.
- Recheck capabilities, select compatible variants, and invalidate caches tied to the old device.

Sources: [queries][query], [device loss][loss], [Rust device][device], [surface outcomes][surface].

[query]: https://www.w3.org/TR/webgpu/#queries
[loss]: https://www.w3.org/TR/webgpu/#dom-gpudevice-lost
[device]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Device.html
[surface]: https://docs.rs/wgpu/30.0.1/wgpu/enum.CurrentSurfaceTexture.html
