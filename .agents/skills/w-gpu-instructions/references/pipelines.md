# Pipelines

## Layouts and bindings

- Use explicit `PipelineLayout` objects when pipelines share resources. Reuse compatible layouts.
- Use `layout: None` for isolated pipelines; keep inferred bind groups specific to their pipeline.
- Match group/binding indices, stage visibility, buffer access/range, texture type, and sampler type.
- Set `min_binding_size` for fixed records to move size checks to bind-group creation.
- Bind the intended storage range; keep capacity separate from logical count.
- Avoid unused writable entries that enlarge hazard scopes or consume binding limits.
- For unfilterable float textures, use `Float { filterable: false }` and nonfiltering samplers.
- Request `FLOAT32_FILTERABLE` only when filtering is needed and supported.

## Pipeline identity and reuse

- Include shader revision, entry points, stage overrides, and layout identity in cache keys.
- Include vertex formats/strides, primitive state, blending, depth/stencil, samples, and multiview.
- Invalidate cached pipelines after shader ABI or device changes; bound the number of variants.
- Exclude target dimensions unless they affect specialization or another pipeline property.
- Separate live-pipeline reuse from `PipelineCache` compilation data.
- Gate `PipelineCache` by feature/backend support; persist only compatible data returned by `wgpu`.
- Do not pass arbitrary cache bytes to unsafe cache creation.

## Render and compute contracts

- Match color targets to attachment view formats/slots and depth/stencil state to the pass.
- Match pipeline sample count to the multisampled attachment, not the single-sample resolve target.
- Match indexed-strip format to the bound index format.
- Use fixed vertex fetch for compact format decoding; compare pulling when arbitrary indexing is needed.
- Use a generated fullscreen triangle when the draw needs no mesh attributes.
- Keep workgroup-memory initialization enabled unless every read-after-write requirement is proved.

Sources: [layouts][layout], [pipeline validation][stage], [Rust descriptors][rust], [pipeline caches][cache].

[layout]: https://www.w3.org/TR/webgpu/#pipeline-layout
[stage]: https://www.w3.org/TR/webgpu/#gpuprogrammablestage
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.RenderPipelineDescriptor.html
[cache]: https://docs.rs/wgpu/30.0.1/wgpu/struct.PipelineCache.html
