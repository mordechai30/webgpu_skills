# Render targets and textures

## Attachment lifetime

- Select the smallest format/resolution/sample count that preserves required quality and operations.
- Use `Clear` when prior contents are unnecessary and `Load` when they are needed.
- Use `Store` for later consumers and `Discard` for dead contents; account for tile-memory bandwidth.
- Use `TRANSIENT_ATTACHMENT | RENDER_ATTACHMENT` only for pass-local clear/discard targets.
- Combine transient usage with no other usages; keep resolve output in a separate persistent texture.
- Fuse compatible rendering when it saves stores/loads; measure added MRT bandwidth and shader pressure.
- Split when an attachment becomes sampled input or usage/attachment requirements change.
- Pool compatible temporary targets only when logical lifetimes do not overlap.
- Match pipeline formats to attachment views; respect allowed `view_formats`.
- Select surface format/present/alpha modes from `surface.get_capabilities(&adapter)`.
- Resolve multisampled color into a compatible single-sample target; discard dead MSAA contents.
- Do not assume automatic depth resolve or filtered sampling of multisampled textures.

## Texture access and capability

- Request only needed usages; check filtering, blending, sample counts, and storage access.
- Inspect `adapter.get_texture_format_features(format)`; grant required device features before use.
- Use `textureLoad` for exact/integer access and `textureSample` for derivative-based fragment sampling.
- Use explicit LOD or gradients where legal when implicit derivatives are unavailable/nonuniform.
- Do not suppress uniformity diagnostics to conceal invalid sampling control flow.
- Match sampled type, dimension, multisampling, and sampler type to the binding layout.
- Use `texture_2d<f32>` for normalized/float formats, including `Rgba16Float`.
- Declare exact storage format/access; gate read/read-write and extended-format support.
- Restrict source/destination views to required subresources to avoid aliasing.

## Color, alpha, and mips

- Use sRGB views for sRGB color assets; keep normals, masks, IDs, and numeric data linear.
- Perform lighting/blending in linear space; let sRGB views decode/encode RGB without duplicate conversion.
- Keep alpha separate from RGB conversion; match straight/premultiplied blending and composition.
- Preserve transparency order; normally disable depth writes for ordinary alpha blending.
- Use float intermediates for HDR range; verify the output color-space/luminance path separately.
- Populate allocated mip levels; size each with `max(1, dimension / 2)` and preserve odd-edge contributions.
- Average color in linear space; renormalize normal maps and use suitable mask/depth reductions.
- Select supported BC/ETC2/ASTC formats; compare bandwidth gains with quality and transcode/upload cost.
- Decode images off the render path; preserve channel order, encoding, and compressed block layout on upload.

Sources: [attachments][pass], [format rules][format], [sampling][sampling], [Rust texture usages][rust].

[pass]: https://www.w3.org/TR/webgpu/#render-passes
[format]: https://www.w3.org/TR/webgpu/#texture-format-caps
[sampling]: https://www.w3.org/TR/WGSL/#texture-builtin-functions
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.TextureUsages.html
