# Constant settings and shader inputs

## Select the lifetime first

| Mechanism | Use when | Performance tradeoff |
| --- | --- | --- |
| WGSL `const` | Shader invariant | Constant evaluation; change source to change value. |
| Generated source | Structural changes | Rebuild modules/pipelines; bound variant growth. |
| WGSL `override` | Pipeline-time scalar | Reuse module; cache specialized pipelines. |
| Uniform buffer | Small runtime parameters | Reuse pipeline and binding; upload changed bytes. |
| Dynamic uniform range | Many small command records | Share binding; padded stride and offset validation. |
| Indexed storage buffer | Dense object tables | Tight packing; indexing and storage reads. |
| Immediate data | Tiny command parameters | Record bytes; gate feature, limit, and API version. |
| Vertex/instance buffer | Regular attribute streams | Fixed fetch and compact format decoding. |
| Sampled texture | Spatial or filtered data | Format, filtering, and mip semantics. |

Do not equate shader read-only data with compile-time constants. A uniform cannot set a workgroup
size or specialize a pipeline. A function-local `let` stores a runtime value; it is not an API
input or necessarily a compile-time value.

Use built-ins for values already supplied by execution, such as vertex/instance indices and
invocation IDs. Use vertex-to-fragment outputs for interpolated data; match locations/types and
interpolation. Use flat interpolation for integer IDs. Compare varying bandwidth with recomputation
or a fragment-stage lookup when passing a large record; do not interpolate object IDs as floats.

## Constants and overrides

Use `const` for true invariants and small fixed tables. Use generated WGSL for structural changes
that overrides cannot express. Keep deterministic source assembly, stable binding assignments,
and source locations for diagnostics. A Rust `include_str!` shader file is a valid default;
JavaScript string interpolation advice does not mandate embedding WGSL in Rust strings.

Use `override` for legal scalar pipeline-time values: `bool`, `i32`, `u32`, `f32`, or enabled `f16`.
Provide a value when a statically used override has no default. With `@id(n)`, use the decimal ID
string in `PipelineCompilationOptions::constants`; without it, use the identifier. Do not pass both
spellings or copy older tutorials that allow the name of an ID-bearing override.

Set constants independently for each programmable stage. If vertex and fragment must agree, provide
the same values to both stage descriptors. Include stage values in the pipeline cache key.

Specialize workgroup size or a stable algorithm option when it can remove significant work or improve
occupancy. Do not create a new pipeline for every time value, color, object transform, or frame.
Compiler optimization of specialized branches is useful, but not a portable speed guarantee.

For `wgpu 30.0.1`, compile a kernel that declares `@id(0) override LANES: u32 = 64u` and
`@workgroup_size(LANES)` with this function. The caller must also size dispatches from `lanes`.

```rust
/// Create one compute variant with a legal one-dimensional workgroup size.
/// Cache the result; the shader uses override ID 0 for its lane count.
fn specialize(
    device: &wgpu::Device,
    module: &wgpu::ShaderModule,
    layout: Option<&wgpu::PipelineLayout>,
    lanes: u32,
) -> wgpu::ComputePipeline {
    let limits = device.limits();
    assert!(lanes > 0 && lanes <= limits.max_compute_workgroup_size_x);
    assert!(lanes <= limits.max_compute_invocations_per_workgroup);
    device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
        label: Some("specialized compute"),
        layout,
        module,
        entry_point: Some("main"),
        compilation_options: wgpu::PipelineCompilationOptions {
            constants: &[("0", f64::from(lanes))],
            ..Default::default()
        },
        cache: None,
    })
}
```

## Uniform records versus storage indexing

Use one frame uniform for data common to all commands. Split static from changing records when
that reduces actual uploaded bytes without excessive bindings. Keep stable buffer handles.

For dynamic uniforms, use `round_up(device.min_uniform_buffer_offset_alignment, record_size)` for
allocation stride. Bind exactly `record_size` bytes, set `has_dynamic_offset: true`, and pass a
checked `u32` byte offset to `set_bind_group`. The GPU struct's array stride is a separate quantity.

Example: this record is 80 bytes. Its allocation stride may be 256 on a selected device; do not
declare a WGSL object array with that stride unless the shader is explicitly designed for it.

```rust
/// Hold one object transform and tint in the shared shader layout.
/// WGSL uses a vec4 followed by a column-major mat4x4, for 80 bytes total.
#[repr(C, align(16))]
#[derive(Clone, Copy, bytemuck::Pod, bytemuck::Zeroable)]
struct ObjectParams {
    /// Linear RGBA multiplier at byte offset 0.
    tint: [f32; 4],
    /// Transform columns at byte offset 16.
    transform: [[f32; 4]; 4],
}

/// Upload distinct records into aligned uniform slots with one dynamic binding.
/// Pass the returned stride times the object index as the draw's dynamic offset.
fn object_binding(
    device: &wgpu::Device,
    queue: &wgpu::Queue,
    objects: &[ObjectParams],
) -> (wgpu::BindGroupLayout, wgpu::BindGroup, u32) {
    assert!(!objects.is_empty());
    let record_size = std::mem::size_of::<ObjectParams>() as u64;
    let alignment = u64::from(device.limits().min_uniform_buffer_offset_alignment);
    let stride = record_size.div_ceil(alignment) * alignment;
    let total = stride.checked_mul(objects.len() as u64).expect("uniform size overflow");
    assert!(total <= device.limits().max_buffer_size);
    let last_offset = stride * (objects.len() as u64 - 1);
    assert!(last_offset <= u64::from(u32::MAX));
    let stride = u32::try_from(stride).expect("dynamic stride exceeds u32");
    let buffer = device.create_buffer(&wgpu::BufferDescriptor {
        label: Some("object uniform slots"),
        size: total,
        usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
        mapped_at_creation: false,
    });
    for (index, object) in objects.iter().enumerate() {
        queue.write_buffer(&buffer, index as u64 * u64::from(stride), bytemuck::bytes_of(object));
    }
    let layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
        label: Some("object uniform layout"),
        entries: &[wgpu::BindGroupLayoutEntry {
            binding: 0,
            visibility: wgpu::ShaderStages::VERTEX | wgpu::ShaderStages::FRAGMENT,
            ty: wgpu::BindingType::Buffer {
                ty: wgpu::BufferBindingType::Uniform,
                has_dynamic_offset: true,
                min_binding_size: wgpu::BufferSize::new(record_size),
            },
            count: None,
        }],
    });
    let group = device.create_bind_group(&wgpu::BindGroupDescriptor {
        label: Some("object uniform binding"),
        layout: &layout,
        entries: &[wgpu::BindGroupEntry {
            binding: 0,
            resource: wgpu::BindingResource::Buffer(wgpu::BufferBinding {
                buffer: &buffer,
                offset: 0,
                size: wgpu::BufferSize::new(record_size),
            }),
        }],
    });
    (layout, group, stride)
}
```

For dense object tables, compare that padded allocation and per-draw binding cost to one read-only
storage array selected by `instance_index` or a tiny command parameter. Batch repeated geometry
with instancing when possible. An indexed uniform table can also work for small tables, subject to
uniform array padding and binding-size limits. Neither uniforms nor storage are universally faster.

## Immediate data and older push constants

For `wgpu 30.0.1`, check `adapter.features().contains(Features::IMMEDIATES)`, check
`adapter.limits().max_immediate_size`, enable the feature in `DeviceDescriptor::required_features`,
and request the required `Limits::max_immediate_size`. The Rust default is zero. Union these
requirements with other features/limits before the single device request.

Set `PipelineLayoutDescriptor::immediate_size` to the allocated byte count. In this version,
`bind_group_layouts` is a slice of `Option<&BindGroupLayout>`. Supply that layout to the pipeline.
Call `RenderPass::set_immediates(offset, bytes)`, `ComputePass::set_immediates`, or the bundle
encoder equivalent. Offset and byte count must be multiples of four.

Use a small index pair rather than copying whole object/material records into each command:

```wgsl
// Select persistent storage records for one command without another buffer binding.
struct DrawSelection {
  object_index: u32,
  material_index: u32,
}
var<immediate> selection: DrawSelection;
```

```rust
/// Record object and material indices for the following draw.
/// The enabled immediate feature and pipeline layout must provide at least 8 bytes.
fn select_draw(pass: &mut wgpu::RenderPass<'_>, object: u32, material: u32) {
    pass.set_immediates(0, bytemuck::cast_slice(&[object, material]));
}
```

Initialize all accessible slots before a draw/dispatch, again after a new pass or bundle execution.
Padding slots need not be initialized. Values recorded in a bundle are fixed; use a referenced
buffer or indirect data for values that must change without re-recording it.

For portable WGSL immediate data, use at most one statically accessed immediate variable per entry
point and avoid arrays or structs containing arrays. Respect the actual granted limit; do not turn
the published WebGPU limit into a universal native hardware capacity.

Older `wgpu` versions use `PUSH_CONSTANTS`, `max_push_constant_size`, `push_constant_ranges`,
`var<push_constant>`, and stage-specific setters. These are Rust/native extensions, not the same
published browser API. Check that exact version; retain a uniform/storage fallback where needed.
Do not mix old push-constant fields with the immediate-data API. Keep current Rust immediates free
of `f16`; `wgpu 30.0.1` documents that restriction separately from the WGSL specification.

## Verify

Use [WGSL declarations][wgsl], [API constants][constants], [immediate data][immediate],
[stage interfaces][io], [Rust compilation options][options], and [Rust layout fields][layout].

[wgsl]: https://www.w3.org/TR/WGSL/#value-decls
[constants]: https://www.w3.org/TR/webgpu/#dom-gpuprogrammablestage-constants
[immediate]: https://www.w3.org/TR/webgpu/#programmable-passes-immediate-data
[io]: https://www.w3.org/TR/WGSL/#stage-inputs-outputs
[options]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineCompilationOptions.html
[layout]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineLayoutDescriptor.html
