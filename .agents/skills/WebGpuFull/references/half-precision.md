# Half precision

## Separate four choices

| Choice | Shader contract | Capability to check |
| --- | --- | --- |
| Typed `f16` arithmetic/buffers | `enable f16;`, explicit types | Granted `SHADER_F16` |
| Half-float texture | Loads/samples normally return `f32` | Format usages, filtering, sample counts |
| Half-float vertex attributes | `Float16x2`/`Float16x4` can feed `f32` | Vertex format and layout |
| Packed half pairs in `u32` | `pack2x16float`/`unpack2x16float` | Baseline; calculates in `f32` |

Choose storage compression separately from arithmetic precision. A half-float render target does
not require `SHADER_F16`. Enabling `SHADER_F16` does not make arbitrary texture formats filterable
or make every calculation faster.

## Enable at adapter, device, and shader

Inspect `adapter.features()` before requesting the device. Add `Features::SHADER_F16` to the
required features only when the selected path supports it. Verify `device.features()` before
creating an f16 shader. The device exposes only the features requested and granted.

For `wgpu 30.0.1`, use this optional precision negotiation. The caller supplies its other required
features and limits. If f16 is a hard requirement, reject an unsupported adapter instead of using
this optional fallback. Do not call `request_device` twice on the same adapter to probe a feature.

```rust
/// Request one device with an optional f16 shader path and the caller's requirements.
/// Select the shader variant from the returned device's granted features.
async fn precision_device(
    adapter: &wgpu::Adapter,
    required_features: wgpu::Features,
    required_limits: wgpu::Limits,
    prefer_f16: bool,
) -> Result<(wgpu::Device, wgpu::Queue), wgpu::RequestDeviceError> {
    let mut features = required_features;
    if prefer_f16 && adapter.features().contains(wgpu::Features::SHADER_F16) {
        features |= wgpu::Features::SHADER_F16;
    }
    adapter.request_device(&wgpu::DeviceDescriptor {
        label: Some("precision device"),
        required_features: features,
        required_limits,
        ..Default::default()
    }).await
}
```

Keep `enable f16;` at module scope before declarations in the selected f16 shader. There is no
WGSL preprocessor that makes an unsupported enable directive disappear at runtime. Use a separate
f32 module or deterministic source generation for the fallback. Make host layouts and bindings
match the selected variant; replacing `f16` with `f32` also changes buffer sizes and strides.

Example: store input pairs in half precision, then calculate and accumulate in f32.

```wgsl
enable f16;

@group(0) @binding(0) var<storage, read> pairs: array<vec2<f16>>;
@group(0) @binding(1) var<storage, read_write> energy: array<f32>;

// Read compact input, but keep the square and sum in f32.
// The output range can be shorter than the allocated input range.
@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
  let i = gid.x;
  if (i >= arrayLength(&pairs) || i >= arrayLength(&energy)) {
    return;
  }
  let p = vec2<f32>(pairs[i]);
  energy[i] = dot(p, p);
}
```

To use half arithmetic, explicitly convert operands to `f16` and keep the intended operations in
that type. Assignment to an f16 destination alone does not convert an earlier f32 calculation
into half arithmetic. Mixed expressions need explicit conversion where their types differ.

## Encode the actual bytes

Natural f16 scalar alignment/size is 2/2 bytes. `vec2<f16>` is 4/4, `vec3<f16>` is 8/6, and
`vec4<f16>` is 8/8. The natural array stride of `vec3<f16>` is 8, and `mat3x3<f16>` is 24 bytes.
Apply uniform address-space rules separately; smaller scalar types do not remove those rules.
Align buffer transfers to four bytes even when one shader element occupies two bytes.

Use an IEEE binary16 encoder, such as Rust's `half` crate. Do not cast a float numerically to a
`u16` and treat the integer as half-float bits. This helper matches one WGSL `vec2<f16>`:

```rust
/// Encode one half-float pair in the shader's little-endian byte representation.
/// Each value occupies two bytes; the pair satisfies four-byte upload granularity.
fn half_pair_bytes(x: f32, y: f32) -> [u8; 4] {
    let x = half::f16::from_f32(x).to_bits().to_le_bytes();
    let y = half::f16::from_f32(y).to_bits().to_le_bytes();
    [x[0], x[1], y[0], y[1]]
}
```

Keep current Rust immediate data free of f16: `wgpu 30.0.1` documents that exclusion even with
`SHADER_F16`. Verify storage/uniform paths instead of assuming all address spaces share support.

## Choose precision from an error budget

Expect binary16's limited range and precision: its largest finite value is 65504. Quantization,
cancellation, overflow, and subnormal handling can change results. Do not depend on subnormal
preservation without validating the selected shader rules and backend.

Start with compact inputs/intermediates whose error is bounded. Retain f32 for long reductions,
large coordinates, sensitive depth calculations, and repeated simulation state where tests show
error growth. Scaling or local coordinate systems can improve range, but change the algorithm's
contract and need explicit error tests.

Compare storage-only compression, mixed precision, and full half arithmetic independently. Measure
GPU time and bandwidth at equal output quality. Half precision may reduce memory traffic and
register pressure; conversion cost, compiler lowering, and hardware throughput can erase the gain.
Test extremes, cancellation, repeated updates, and representative inputs against an f32 reference.

## Verify

Use [WGSL f16][f16], [layout][layout], [packing built-ins][packing],
[WebGPU shader-f16][feature], and [Rust feature restrictions][rust].

[f16]: https://www.w3.org/TR/WGSL/#extension-f16
[layout]: https://www.w3.org/TR/WGSL/#memory-layouts
[packing]: https://www.w3.org/TR/WGSL/#pack2x16float-builtin
[feature]: https://www.w3.org/TR/webgpu/#shader-f16
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.Features.html#associatedconstant.SHADER_F16
