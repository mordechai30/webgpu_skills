# Shader settings

## Select the input path

| Use | Select | Account for |
| --- | --- | --- |
| Shader invariant | WGSL `const` | Source/module change for a new value |
| Binding/type/algorithm structure | Generated WGSL | Module/pipeline creation and variant growth |
| Stable pipeline-time scalar | WGSL `override` | Cached specialization; possible removed work |
| Small runtime parameters | Uniform buffer | Upload bytes and binding-size limits |
| Many small command records | Dynamic uniform ranges | Shared binding; padded stride and offset checks |
| Dense object/material tables | Indexed storage | Tight packing; indexing and storage reads |
| Tiny per-command selection | Immediate data | Recorded bytes; feature and size limit |
| Regular vertex/instance attributes | Vertex/instance buffers | Fixed fetch and compact format decoding |
| Spatial or filtered data | Sampled textures | Format, filtering, and mip semantics |

- Select by lifetime and access pattern; measure uniforms against storage for the actual workload.
- Use built-ins for indices/invocation IDs already supplied by execution.
- Use inter-stage values for interpolation; match locations/types and use flat interpolation for integer IDs.
- Compare large varying records with recomputation or fragment-stage lookup.

## Constants and specialization

- Use `const` for true invariants; generate source only when shader structure must change.
- Use scalar `override` values for a bounded set of pipeline choices; include them in cache keys.
- Supply statically used overrides without defaults.
- With `@id(n)`, use the decimal ID key; otherwise use its name.
- Set `PipelineCompilationOptions::constants` as `&[(&str, f64)]` in `wgpu 30.0.1`.
- For `@id(0) override LANES: u32`, pass `&[("0", f64::from(lanes))]` after checking workgroup limits.
- Set values independently per programmable stage; keep shared vertex/fragment values equal.
- Calculate dispatch count from the specialized workgroup size.
- Keep time, transforms, colors, and other frequently changing values in runtime data.

## Dynamic uniforms and storage indexing

- Calculate allocation stride as `round_up(device.min_uniform_buffer_offset_alignment, record_size)`.
- Set `has_dynamic_offset: true`; bind `record_size` bytes and pass checked `u32` byte offsets.
- Keep allocation stride separate from shader array stride: an 80-byte record may occupy a 256-byte slot.
- Compare that padding and per-command binding cost with one dense storage table selected by an index.
- Use instancing for repeated geometry when it reduces commands; keep persistent table bindings stable.
- Split static and changing data only when fewer uploaded bytes outweigh additional bindings.

## Immediate data

- Check adapter `IMMEDIATES` and `max_immediate_size`; request the feature and required nonzero limit.
- Set `PipelineLayoutDescriptor::immediate_size`; supply the layout to the pipeline.
- Use `bind_group_layouts: &[Option<&BindGroupLayout>]` for the reviewed Rust version.
- Record bytes with pass/bundle `set_immediates(offset, bytes)`; align offset and byte count to four.
- Prefer tiny indices into persistent tables over copying complete object/material records.
- Initialize all accessible slots before execution; reset required values after a new pass or bundle.
- Use at most one statically accessed immediate variable per entry point; avoid arrays and array members.
- Keep immediate data free of f16 in `wgpu 30.0.1`; use uniforms/storage for those values.
- Provide a uniform/storage path when immediate support is unavailable.

Sources: [declarations][decl], [constants][constant], [immediates][immediate], [Rust options][rust].

[decl]: https://www.w3.org/TR/WGSL/#value-decls
[constant]: https://www.w3.org/TR/webgpu/#dom-gpuprogrammablestage-constants
[immediate]: https://www.w3.org/TR/webgpu/#programmable-passes-immediate-data
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineCompilationOptions.html
