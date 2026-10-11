# Half precision

## Enable f16

- Check `adapter.features().contains(wgpu::Features::SHADER_F16)` before device creation.
- Add `SHADER_F16` to `DeviceDescriptor::required_features` with the other required features.
- Verify `device.features()` before creating the selected f16 shader.
- Put `enable f16;` at module scope before declarations; explicitly type/convert intended f16 operations.
- Use a separate f32 shader variant when f16 is optional; reject unsupported adapters when required.
- Match host bytes, binding sizes, and array strides to each variant.

## Choose storage and arithmetic separately

| Need | Select | Check |
| --- | --- | --- |
| Typed half arithmetic/buffers | WGSL `f16` | Granted `SHADER_F16` and numerical error |
| Half-float texture | `Rgba16Float` or suitable format | Format capabilities; use f32 texture interfaces |
| Compact vertex attributes | `Float16x2` / `Float16x4` | Vertex layout; f32 inputs when suitable |
| Half pairs without typed f16 | `pack2x16float` / `unpack2x16float` | Word layout; f32 calculations |

- Keep texture/vertex compression independent of `SHADER_F16` arithmetic support.
- Convert compact f16 inputs to f32 before sensitive calculations or accumulation.
- Do not infer half arithmetic from an f16 destination; keep intended operand operations in f16.
- Keep immediate data free of f16 in the reviewed Rust version.

## Encode and validate

- Encode IEEE binary16 bits with a tested converter such as `half::f16`; do not numerically cast to `u16`.
- Use natural alignment/size 2/2 for f16, 4/4 for vec2, 8/6 for vec3, and 8/8 for vec4.
- Use stride 8 for `array<vec3<f16>>` and size 24 for `mat3x3<f16>`; apply uniform rules separately.
- Align transfers to four bytes even when shader elements occupy two.
- Check range, quantization, cancellation, and overflow; keep finite magnitudes within 65504.
- Do not depend on subnormal preservation without verifying the selected backend's shader rules.
- Retain f32 for long reductions, large coordinates, sensitive depth, or demonstrated simulation error growth.
- Compare storage-only, mixed-precision, and half-arithmetic paths at equal quality.
- Measure conversions, bandwidth, register pressure, and GPU time; do not assume half is faster.
- Test extremes and repeated updates against an f32 reference and explicit error tolerance.

Sources: [f16][f16], [layout][layout], [packing][pack], [Rust feature restrictions][rust].

[f16]: https://www.w3.org/TR/WGSL/#extension-f16
[layout]: https://www.w3.org/TR/WGSL/#memory-layouts
[pack]: https://www.w3.org/TR/WGSL/#pack2x16float-builtin
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Features.html#associatedconstant.SHADER_F16
