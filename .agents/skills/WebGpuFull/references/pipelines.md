# Pipelines

## Layout and binding contract

Use an explicit `PipelineLayout` when pipelines share resources or switch frequently.
Reuse the same compatible layout objects to make the contract visible and avoid redundant binding.
Use `layout: None` for an isolated Rust pipeline when inference is useful. Treat bind groups made
from its inferred layout as specific to that pipeline.

Choose binding groups by reuse and update frequency. Put widely shared, infrequently switched data
at low group indices; place material and object selection later when that matches the workload.
Do not mandate fixed camera/material/object group numbers for every application.

Keep these properties equal across WGSL, `BindGroupLayoutEntry`, and `BindGroupEntry`:

- Group and binding indices; stage visibility; read-only versus writable access.
- Buffer minimum size, effective range, and dynamic-offset policy.
- Texture sample type, view dimension, multisampling, storage format/access, and sampler type.

Set `min_binding_size` for fixed records to validate size at bind-group creation rather than at each
draw. For variable storage arrays, specify a valid minimum and bind the intended range; distinguish
capacity from logical count. Avoid broad layouts with unused writable resources: they still affect
hazard validation and binding limits.

For unfilterable `Rgba32Float`, use `TextureSampleType::Float { filterable: false }` and a
`SamplerBindingType::NonFiltering` sampler when sampling. Enable `FLOAT32_FILTERABLE` only if the
selected path needs filtering and the adapter supports it. Inference can choose a filterable layout
that cannot bind the texture on the baseline path.

## Cache the whole pipeline identity

Create and warm expected variants outside the frame's critical path. Cache by all semantic inputs:

- Shader source/revision, selected entry points, and per-stage override values.
- Pipeline layout identity, including immediate data size.
- Vertex attribute formats, offsets, strides, and step modes.
- Primitive topology, strip index format, winding, culling, and depth clipping.
- Color view formats, blending, write masks, depth/stencil state, and sample count.
- Multiview state and other enabled extensions when used.

Invalidate the key on an ABI change. Bound the variant count. A quality slider usually belongs in a
runtime buffer; specialize only when a small set of variants can remove meaningful shader work.

Do not include target width/height in the key unless they change shader specialization or another
pipeline property. Resize attachments without rebuilding a format-compatible pipeline.

Separate an application map of live pipelines from `wgpu::PipelineCache`, which stores compilation
data. Use the latter only with the required feature and supported backend. Persist only cache data
returned by `wgpu`, with its device compatibility key and atomic file replacement. Do not accept
arbitrary bytes through the unsafe cache creation API.

## Match the render contract

Match each color target to its attachment **view** format and slot. Match depth/stencil format and
sample count to the pass. An MSAA resolve target has one sample; the pipeline matches the MSAA
attachment, not the resolve target. Match indexed strip format to the bound index format.

Keep vertex decoding separate from storage ABI. A packed `Float32x3` vertex stream can have stride
12; an `array<vec3<f32>>` storage stream has stride 16. Extra unused vertex attributes are not a
reason to invent a blanket validation failure; validate the inputs actually required by the shader.

Use a generated fullscreen triangle when no mesh attributes are needed. Compare vertex pulling to
fixed vertex fetch for real meshes: pulling offers arbitrary indexing and compute reuse, while
fixed fetch can decode compact normalized formats and perform better on some devices.

Do not translate JavaScript's `createRenderPipelineAsync` into a nonexistent Rust method.
Use the pinned Rust creation API and schedule expensive construction during loading or controlled
rebuilds. Keep workgroup memory initialization at its safe default; changing it requires proof that
every read follows a write and that the selected backend supports the intended semantics.

## Verify

Use [pipeline layouts][layouts], [default layouts][auto], and [pipeline validation][stages] for
WebGPU rules. Use [Rust pipeline descriptors][rust], [compilation options][options], and
[pipeline caches][cache] for exact Rust behavior.

[layouts]: https://www.w3.org/TR/webgpu/#pipeline-layout
[auto]: https://www.w3.org/TR/webgpu/#default-pipeline-layout
[stages]: https://www.w3.org/TR/webgpu/#gpuprogrammablestage
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.RenderPipelineDescriptor.html
[options]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineCompilationOptions.html
[cache]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineCache.html
