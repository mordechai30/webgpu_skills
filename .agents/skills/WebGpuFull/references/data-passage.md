# Data passage

## Choose by producer and consumer

| Data path | Use | Performance decision |
| --- | --- | --- |
| CPU, fixed once | `DeviceExt::create_buffer_init` or mapped creation | Avoid repeated staging and uploads. |
| CPU, small updates | `Queue::write_buffer` | Start here; batch adjacent dirty records. |
| CPU, generated upload | `write_buffer_with` | Generate directly into staging to avoid a separate copy. |
| CPU, many recurring writes | `util::StagingBelt` | Reuse staging allocations when allocation cost matters. |
| CPU, large streaming | Mapped staging pool plus copies | Compare throughput and lifetime cost. |
| GPU to another GPU consumer | Shared resource or GPU copy | Avoid a round trip through the CPU. |
| GPU to CPU | Reduce, copy, map asynchronously | Bound bandwidth; accept result latency. |

Treat upload size thresholds as measurements, not universal constants. Native unified memory and
browser Wasm copies have different costs. Do not assume a mapped native path is also best on Wasm.

## Keep Rust and WGSL layouts equal

Calculate member offsets, total size, array stride, and address-space constraints from WGSL.
Use `#[repr(C)]`, explicit padding, and a byte-safe representation or a tested serializer in Rust.
Use `size_of` and `offset_of` checks; derive `Pod` only when all bytes are valid and initialized.
Rust arrays and math types do not automatically have shader alignment or column layout.

Use these natural layout facts for `f32`, `u32`, and `i32`:

| Shape | Alignment | Size |
| --- | ---: | ---: |
| Scalar | 4 | 4 |
| `vec2` | 8 | 8 |
| `vec3` | 16 | 12 |
| `vec4` | 16 | 16 |
| `mat3x3<f32>` | 16 | 48 |
| `mat4x4<f32>` | 16 | 64 |

Place each member at `round_up(member_alignment, previous_end)`. Round struct size to its maximum
member alignment. Use `round_up(element_alignment, element_size)` for natural array stride.
Store matrices by columns; `mat3x3<f32>` includes padding between its three columns.

A scalar may follow a `vec3<f32>` at byte 12 inside a struct. An array of that vector still has
stride 16. Do not require every uniform buffer or Rust structure to have a size divisible by 16.
Separate shader member alignment, API binding-offset alignment, and Rust CPU address alignment.

For portable uniform layout without `uniform_buffer_standard_layout`, align array strides to 16
and satisfy the additional nested-struct spacing rules. Use `@size`/`@align` or explicit vector
slots when needed. Use relaxed uniform layout only after verifying language support and validating
the selected Naga/browser path; publication of the feature alone does not prove Rust support.

Use `u32` for buffer flags, not Rust/WGSL `bool`. Runtime arrays belong at the end of a storage
record or as its top-level type. `arrayLength` describes the bound range, not the application's
logical count. Keep the logical count explicit when capacity exceeds live records.

For a vertex buffer also used by compute, match both contracts. Repack to a shared storage-friendly
layout, or read packed words/scalars and reconstruct attributes. `Float16x2`, normalized colors,
and other vertex formats perform decoding that a storage binding does not perform automatically.
Pack related sub-word fields into a word when concurrent writes could otherwise touch the same
memory word; avoid races between separate invocations updating halves of one packed record.

## Queue writes are not command snapshots

In native `wgpu`, `write_buffer` copies CPU bytes into staging now. Pending transfers execute before
the command buffers in the next `submit`. Encoding a draw records a reference to the GPU buffer.
If each object writes to the same offset before one submission, all draws can observe the final
record. Use separate ranges, indexed storage records, immediate data, or ordered encoder copies.

Queue ordering protects ordinary updates submitted after earlier uses. Do not add a whole-device
wait per frame. For an explicitly mapped staging pool, reclaim each slot only after the completion
associated with its use; a fixed number of elapsed frames does not prove GPU completion.

For `wgpu 30.0.1`, create `StagingBelt::new(device.clone(), chunk_size)`, fill its returned views,
drop all views, call `finish`, submit the involved encoders, and call `recall`. Poll native progress
so remapping can complete. Alternatively use `finish_and_recall_on_submit` with an encoder that
will actually be submitted. Older belt constructors and lifecycles can differ.

## Copy and mapping contracts

- Use 4-byte-aligned offsets and sizes for buffer copies and queue buffer writes.
- Align uniform/storage binding offsets to the corresponding **device** limit. Set the binding
  size to the record range; do not bind the full allocation when using dynamic offsets.
- Order multiple dynamic offsets by increasing binding number, not descriptor array order.
- Use mapping offsets divisible by 8 and mapping sizes divisible by 4. Drop mapped views before
  unmapping. A mapped or pending-map buffer is not available for ordinary GPU submission.
- Use `MAP_READ | COPY_DST` for portable readback and `MAP_WRITE | COPY_SRC` for reusable staging.
  Mapped-at-creation initialization is a separate path and does not require `MAP_WRITE`.
- For encoder texture/buffer copies, pad a specified row pitch to 256 bytes. Queue `write_texture`
  does not require that padding. Compute pitch and `rows_per_image` in texel-block rows for compressed
  formats. Remove padding when reconstructing CPU images.

Map after submitting the producing work, or use the pinned version's deferred mapping API.
In Rust 30, `get_mapped_range` and `get_mapped_range_mut` return `Result`; handle range errors.
On native `wgpu`, drive callback progress with submission/polling; browser WebGPU advances through
its event loop. Keep callbacks short and report completion to the application. Do not block the
browser loop or add `on_submitted_work_done` when mapping that buffer already provides the needed wait.

## Verify

Use [WGSL layout][layout], [buffer copies][copies], [mapping][mapping], [Rust queue writes][queue],
[Rust buffers][buffers], [staging belts][belt], and [texture copy layouts][texel].

[layout]: https://www.w3.org/TR/WGSL/#memory-layouts
[copies]: https://www.w3.org/TR/webgpu/#copies
[mapping]: https://www.w3.org/TR/webgpu/#buffer-mapping
[queue]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Queue.html#method.write_buffer
[buffers]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Buffer.html
[belt]: https://docs.rs/wgpu/30.0.1/wgpu/util/struct.StagingBelt.html
[texel]: https://docs.rs/wgpu/30.0.1/wgpu/struct.TexelCopyBufferLayout.html
