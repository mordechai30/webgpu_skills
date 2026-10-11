# Render targets and textures

## Attachment lifetime and bandwidth

Choose format, resolution, sample count, usage, and lifetime from the next consumer.
Use the smallest format that preserves required quality and supported operations. Avoid allocating
an HDR target, depth/stencil aspect, or extra MRT channel without a consumer that needs it.

Use `LoadOp::Clear` when prior pixels are not needed and `LoadOp::Load` when they are.
Use `StoreOp::Store` when a later pass, readback, or presentation needs the result; use `Discard`
for an attachment that is dead after the pass. Load/store policy can have large bandwidth costs on
tile-based GPUs. A discard does not preserve history for a later load.

For pass-local depth or MSAA color whose contents never survive, consider Rust 30's
`TextureUsages::TRANSIENT_ATTACHMENT | RENDER_ATTACHMENT`. It can reduce memory and bandwidth
on tile-based hardware and can be a no-op elsewhere. Combine it with no other texture usages;
use clear/discard operations. Keep resolve output in a separate persistent texture. Do not use
this flag for history, sampled intermediates, readback, or an attachment loaded by another pass.

Combine compatible rendering into one pass when that avoids intermediate stores/loads and preserves
ordering. Split when a produced attachment must become sampled input, when usage hazards require it,
or when the next phase needs different attachments. Treat pass fusion as a measured tradeoff:
extra MRT bandwidth or shader/register pressure can outweigh fewer boundaries.

Reuse intermediate textures and views when their descriptors are stable. For sequential temporary
targets, pool compatible allocations only after their logical lifetimes no longer overlap. Do not
assume safe `wgpu` exposes arbitrary native memory aliasing. Rebuild dependent bind groups after a
texture handle changes. Keep persistent history separate from disposable presentation textures.

## Surface and multisampling

On Rust, choose surface format/present/alpha options from `surface.get_capabilities(&adapter)`.
On the browser API, use the preferred canvas format. Match the pipeline's format to the selected
attachment **view**, including an sRGB view when configured. Respect allowed `view_formats`.

Recreate size-dependent color, depth, MSAA, and history resources together after a real resize.
Skip zero-sized/minimized frames. Keep every attachment's dimensions and sample count coherent.
Do not rebuild format-compatible pipelines solely because dimensions changed.

For 4x MSAA, render into multisampled color/depth attachments with a matching pipeline. Resolve
color into a compatible single-sample target. Use `Discard` on the MSAA color attachment if only
the resolve result survives. Do not assume automatic depth resolve or ordinary filtered sampling of
an MSAA texture. Check format sample-count support; native extensions do not enlarge the portable
baseline silently.

## Color and alpha

Use sRGB formats for color assets authored in sRGB. Keep normals, roughness, IDs, and other numeric
data in linear formats. Perform lighting and blending in linear space. An sRGB sampled view decodes
RGB; an sRGB render-target view encodes RGB. Avoid applying another gamma conversion on that path.
Keep alpha treatment separate from RGB color conversion.

Use a linear float intermediate, often `Rgba16Float`, when HDR range is needed. Tone-map to the
chosen SDR output, or configure a verified HDR surface path with its color-space/luminance contract.
A float attachment alone does not prove HDR presentation. Native surface HDR settings and browser
canvas color management use different APIs; check each target path.

For straight-alpha output, blend source RGB with source alpha and destination RGB with
`1 - source_alpha`. For premultiplied output, use source RGB factor one. Use the corresponding alpha
equation and composition mode. Preserve transparent ordering and normally avoid writing depth for
ordinary alpha-blended geometry. Masked/cutout materials have a different depth policy.

## Texture capability contract

Create textures with the union of usages actually needed: sampled, storage, attachment, and real
copy paths. Check format usages, filterability, blending, sample counts, and storage access.
On Rust, inspect `adapter.get_texture_format_features(format)` and enable any required features.
When using native format capabilities beyond WebGPU guarantees, check the relevant Rust feature;
do not use adapter support as permission to bypass device negotiation.

Use `textureLoad` for exact texel access or integer/unfilterable data. Use `textureSample` for
filtered fragment-stage access with implicit derivatives. In compute or nonuniform derivative
control flow, use `textureSampleLevel` with an explicit level, `textureSampleGrad` where legal,
or `textureLoad`; do not suppress a uniformity error to conceal an invalid sampling design.

Match `texture_2d<f32>` to floating/normalized data, integer texture types to integer formats,
and depth/comparison bindings to depth sampling. `Rgba16Float` still uses a float texture interface,
not `texture_2d<f16>`. A half-float texture does not require `SHADER_F16` arithmetic.

For a writable storage texture, declare the exact format and access mode. Gate read/read-write
storage access and native format extensions on actual language/device support. Use sampled input
plus a separate output as a portable starting point. Restrict mip views to avoid accidental aliasing.

## Mips, compression, and uploads

Allocate enough mip levels **and populate them**; allocation is not mip generation.
For an ordinary color image, compare offline mips with a chain of downsample passes. Bind only the
source and destination levels, use nonoverlapping views, and size each level with `max(1, dim / 2)`.
Preserve odd edges and perform color averaging in linear space. Renormalize normal-map mips; use
appropriate reductions for masks/depth instead of blindly averaging every data texture.

Use mips for minified sampling when quality and texture-cache behavior benefit. Check explicit LOD
and sampler settings. For compressed assets, include a valid mip chain and select/transcode to a
supported BC/ETC2/ASTC path. Compression can reduce storage and bandwidth; benchmark quality,
transcode/upload cost, and target support. It does not suit every data or render-target texture.

Decode ordinary images off the render path and upload with `Queue::write_texture` or an encoder copy.
Use tightly packed rows for queue writes where valid; pad encoder-copy pitch to 256 bytes.
Apply block dimensions for compressed textures, and preserve channel order and numeric encoding.
On browser paths, external image/video APIs also have origin, color-conversion, and frame-lifetime
contracts; do not transplant those host object APIs into native Rust.

## Verify

Use [attachments][attachments], [formats][formats], [transient targets][transient], and
[texture sampling][sampling]. Check
[canvas configuration][canvas]. Use [Rust texture capabilities][rust] and [copy layout][copies].

[attachments]: https://www.w3.org/TR/webgpu/#render-passes
[formats]: https://www.w3.org/TR/webgpu/#texture-format-caps
[sampling]: https://www.w3.org/TR/WGSL/#texture-builtin-functions
[canvas]: https://www.w3.org/TR/webgpu/#canvas-configuration
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Adapter.html#method.get_texture_format_features
[copies]: https://docs.rs/wgpu/30.0.1/wgpu/struct.TexelCopyBufferLayout.html
[transient]: https://docs.rs/wgpu/30.0.1/wgpu/struct.TextureUsages.html
