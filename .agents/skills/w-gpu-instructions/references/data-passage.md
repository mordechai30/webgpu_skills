# Data passage

## Host and shader layout

- Calculate member offsets, struct size, array stride, and address-space restrictions from WGSL.
- Use `repr(C)`, explicit padding, or a tested serializer; initialize every transferred byte.
- Store matrices by columns; pad `mat3x3<f32>` columns to 16 bytes, for 48 bytes total.
- Place a scalar after `vec3<f32>` at byte 12 when valid; keep the vector's array stride at 16.
- Keep packed `Float32x3` vertex stride 12 separate from `array<vec3<f32>>` storage stride 16.
- Apply uniform array/nested-struct padding unless relaxed uniform layout is verified on the target path.
- Encode flags as `u32`, not buffer `bool`; place runtime arrays only in legal storage positions.
- Pass logical counts when bound capacity exceeds live data; do not use `arrayLength` as that count.
- Decode packed vertex formats explicitly when reading them through storage.
- Give one invocation ownership of each packed word; avoid concurrent writes to its separate halves.

## Upload selection

- Initialize fixed buffers with `DeviceExt::create_buffer_init` or mapped creation.
- Use `Queue::write_buffer` for small updates; batch adjacent dirty ranges.
- Use `write_buffer_with` to generate bytes directly into staging; drop its view before submission.
- Compare `StagingBelt` for many recurring writes and mapped pools for sustained streaming.
- Keep GPU-produced data in GPU resources; reduce or compact before required CPU readback.
- Measure allocation/copy cost on the intended backend; do not prescribe fixed size thresholds.
- Upload separate ranges for separate commands. Queue writes precede commands in the next submission.
- Reclaim explicitly mapped slots only after their associated GPU use completes.

## Alignment and mapping

- Align buffer-copy and queue-write offsets/sizes to four bytes.
- Align mapping offsets to eight bytes and sizes to four bytes.
- Align binding offsets to device limits; order dynamic offsets by increasing binding number.
- Bind the intended record range, not the whole dynamic-uniform allocation.
- Use `MAP_READ | COPY_DST` for readback and `MAP_WRITE | COPY_SRC` for portable staging.
- Drop mapped views before unmapping; do not submit ordinary GPU use of mapped/pending-map buffers.
- Handle `get_mapped_range` / `get_mapped_range_mut` results in `wgpu 30.0.1`.
- Drive native callbacks with submission/polling; keep readback asynchronous in the application loop.
- Pad specified encoder texture-copy row pitch to 256 bytes; do not add that padding to queue writes.
- Calculate compressed pitches in texel blocks; remove copied row padding when reconstructing CPU images.

## Staging belt lifecycle

- For `wgpu 30.0.1`, construct `StagingBelt::new(device.clone(), chunk_size)`.
- Fill and drop views, call `finish`, submit the involved encoders, then call `recall`.
- Drive native progress so remapping completes; do not overwrite unfinished staging allocations.
- Use `finish_and_recall_on_submit` only with an encoder that will be submitted.

Sources: [WGSL layout][layout], [mapping][mapping], [queue writes][queue],
[staging belt][belt], [copies][copy].

[layout]: https://www.w3.org/TR/WGSL/#memory-layouts
[mapping]: https://www.w3.org/TR/webgpu/#buffer-mapping
[queue]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Queue.html#method.write_buffer
[belt]: https://docs.rs/wgpu/30.0.1/wgpu/util/struct.StagingBelt.html
[copy]: https://docs.rs/wgpu/30.0.1/wgpu/struct.TexelCopyBufferLayout.html
