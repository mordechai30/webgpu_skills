# Profiling, diagnostics, and lifecycle

## Measure the limiting work

Measure CPU preparation, command encoding, GPU execution, transfer cost, and presentation latency
separately. Warm pipelines and allocations before steady-state comparisons. Keep resolution,
geometry, iteration count, and output quality fixed. Test a target GPU and backend, not only a
single fast development machine. Use frame distributions as well as averages.

Gate timestamps with the granted `TIMESTAMP_QUERY` feature. Use pass timestamp writes for a
portable timing path. In Rust, multiply timestamp differences by `Queue::get_timestamp_period()`
to obtain nanoseconds. Native writes inside passes/encoders have separate feature requirements;
do not copy those methods into a baseline browser path.

Resolve queries into `QUERY_RESOLVE | COPY_SRC`, then copy into `MAP_READ | COPY_DST`. Align the
resolve destination offset to 256 bytes. Reuse query/readback slots only after their results are
available. Read asynchronously several submissions later to avoid serializing every frame.
Reject invalid or unavailable samples; never interpret zero or stale values as a speedup.

Compare pass timings with whole-frame captures. Indirect command validation, implicit clearing,
copies, and backend work can sit outside the timed pass. On Metal, use an Xcode GPU capture to
inspect those costs when available. A faster shader can still produce a slower frame.

Treat batching, bundles, dynamic uniforms, storage indexing, pass fusion, f16, and larger workgroups
as alternatives to measure. Do not prescribe branchless code, maximum occupancy, or a fixed upload
threshold without a workload and evidence. Preserve correctness and numerical quality first.

## Capture errors around the operation

Label resources, passes, and pipelines with purpose and variant identifiers. Keep WGSL source
locations for generated shaders. Install an uncaptured-error handler and a device-loss callback.
Use narrow error scopes for recoverable shader/pipeline/resource creation failures.

The Rust 30 API differs from older examples: `push_error_scope` returns a guard, and the guard's
`pop` produces the future. Pop before awaiting it; keep nested scopes in reverse creation order.
Do not carry scopes across unrelated work or async task reordering.

```rust
/// Capture validation failure for one shader creation without leaking the error scope.
/// The caller must reject the returned module when the captured error is present.
async fn checked_shader(
    device: &wgpu::Device,
    source: &str,
) -> (wgpu::ShaderModule, Option<wgpu::Error>) {
    let scope = device.push_error_scope(wgpu::ErrorFilter::Validation);
    let module = device.create_shader_module(wgpu::ShaderModuleDescriptor {
        label: Some("checked WGSL"),
        source: wgpu::ShaderSource::Wgsl(source.into()),
    });
    let error = scope.pop().await;
    (module, error)
}
```

In this version, `on_uncaptured_error` takes `Arc<dyn UncapturedErrorHandler>`.
`set_device_lost_callback` takes a `Send + 'static` closure receiving the reason and message.
Browser promise examples illustrate the lifecycle, but do not supply these Rust signatures.
Drive native mapping/completion with submission or `Device::poll`; browser polling is automatic.
Avoid blocking a UI/event-loop thread merely to obtain timing data.

## Own persistent state and recover derived resources

Keep CPU descriptors, asset source data or reload paths, and shader sources sufficient to rebuild
GPU state. Separate those from device-owned resources and pass-local command state. Cache keys
must include the creating device identity or generation; old handles cannot migrate to a new device.

Use safe `wgpu` handle ownership for ordinary lifetimes. The implementation retains resources
needed by submitted work. Do not implement frame-count-based deferred deletion to fix an assumed
safe-handle use-after-free. Explicit mapping pools, raw HAL access, and intentional destruction
have separate completion and validity requirements. Do not destroy a resource needed by future work.

Suspend surface acquisition/configuration for zero-sized windows. On resize, recreate size-dependent
targets/views and update configuration; keep pipelines unless their format/sample contract changes.
Drop outstanding acquired textures before reconfiguring. Check surface capabilities for the actual
adapter; present/alpha modes and formats are not universal constants.

For `wgpu 30.0.1`, `Surface::get_current_texture()` returns `CurrentSurfaceTexture`:

- `Success(texture)`: render, submit, then call `queue.present(texture)`.
- `Suboptimal(texture)`: the texture is usable; present it and schedule configuration repair.
- `Timeout` or `Occluded`: skip that frame without a blocking retry loop.
- `Outdated`: reconfigure after releasing outstanding surface textures.
- `Lost`: recreate the surface unless the device itself is lost.
- `Validation`: report the captured/uncaptured validation error and fix its cause.

Do not copy an older `Result<SurfaceTexture, SurfaceError>` match or `texture.present()` into this
version. A lost device requires a new device/queue and all device-owned resources. Recheck features
and limits during recovery, select compatible shader variants, and invalidate all GPU caches.

## Verify

Use [queries][queries], [device loss][loss], [Rust error scopes][device],
[surface acquisition][surface], and [Rust presentation][queue].

[queries]: https://www.w3.org/TR/webgpu/#queries
[loss]: https://www.w3.org/TR/webgpu/#dom-gpudevice-lost
[device]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Device.html
[surface]: https://docs.rs/wgpu/30.0.1/wgpu/enum.CurrentSurfaceTexture.html
[queue]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Queue.html#method.present
