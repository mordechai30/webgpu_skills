<a id="main-content"></a>

<a id="crate-wgpu"></a>

# Crate wgpu

<a id="search-button"></a>

[Search](https://docs.rs/wgpu/latest/wgpu/index.html?search=)

[Settings](https://docs.rs/wgpu/latest/settings.html)

[Help](https://docs.rs/wgpu/latest/help.html)

[Source](https://docs.rs/wgpu/latest/src/wgpu/lib.rs.html#1-355)

Expand description

`wgpu` is a cross-platform, safe, pure-Rust graphics API. It runs natively on
Vulkan, Metal, D3D12, and OpenGL; and on top of WebGL2 and WebGPU on wasm.

The API is based on the [WebGPU standard](https://gpuweb.github.io/gpuweb/), but is a fully native Rust library.
It serves as the core of the WebGPU integration in Firefox, Servo, and Deno.

<a id="getting-started"></a>

### [§](#getting-started)Getting Started

The main entry point to the API is the [`Instance`](https://docs.rs/wgpu/latest/wgpu/struct.Instance.html "struct wgpu::Instance") type, from which you can create [`Adapter`](https://docs.rs/wgpu/latest/wgpu/struct.Adapter.html "struct wgpu::Adapter"), [`Device`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html "struct wgpu::Device"), and [`Surface`](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html "struct wgpu::Surface").

If you are new to `wgpu` and graphics programming, we recommend starting with [Learn Wgpu](https://sotrh.github.io/learn-wgpu/).

Additionally, [WebGPU Fundamentals](https://webgpufundamentals.org/) is a tutorial for WebGPU which is very similar to our API, minus differences between Rust and Javascript.

We have a [wiki](https://github.com/gfx-rs/wgpu/wiki) which has information on useful architecture patterns, debugging tips, and more getting started information.

There are examples for this version [available on GitHub](https://github.com/gfx-rs/wgpu/tree/v30/examples#readme).

The API is refcounted, so all handles are cloneable, and if you create a resource which references another,
it will automatically keep dependent resources alive.

`wgpu` uses the coordinate systems of D3D and Metal. Depth ranges from [0, 1].

| Render | Texture |
| --- | --- |
| ![render_coordinates](<uindexhtml/9a2e058eb6a1e62ca2cee982.png>) | ![texture_coordinates](<uindexhtml/2d90af2cc269b1c786ccbd11.png>) |

`wgpu`’s MSRV is **1.87**.

<a id="extension-specifications"></a>

### [§](#extension-specifications)Extension Specifications

While the core of `wgpu` is based on the WebGPU standard, we also support extensions that allow for features that the standard does not have yet.
For high-level documentation on how to use these extensions, see documentation on [`Features`](https://docs.rs/wgpu/latest/wgpu/struct.Features.html "struct wgpu::Features") or the relevant specification:

🧪EXPERIMENTAL🧪 APIs are subject to change and may allow undefined behavior if used incorrectly.

- 🧪EXPERIMENTAL🧪 [Ray Tracing](https://github.com/gfx-rs/wgpu/blob/v30/docs/api-specs/ray_tracing.md).
- 🧪EXPERIMENTAL🧪 [Mesh Shading](https://github.com/gfx-rs/wgpu/blob/v30/docs/api-specs/mesh_shading.md).

<a id="shader-support"></a>

### [§](#shader-support)Shader Support

`wgpu` can consume shaders in [WGSL](https://gpuweb.github.io/gpuweb/wgsl/), SPIR-V, and GLSL.
Both [HLSL](https://github.com/Microsoft/DirectXShaderCompiler) and [GLSL](https://github.com/KhronosGroup/glslang)
have compilers to target SPIR-V. All of these shader languages can be used with any backend as we handle all of the conversions. Additionally, support for these shader inputs is not going away.

While WebGPU does not support any shading language other than WGSL, we will automatically convert your
non-WGSL shaders if you’re running on WebGPU.

WGSL is always supported by default, but GLSL and SPIR-V need features enabled to compile in support.

To enable WGSL shaders, enable the `wgsl` feature of `wgpu` (enabled by default).
To enable SPIR-V shaders, enable the `spirv` feature of `wgpu`.
To enable GLSL shaders, enable the `glsl` feature of `wgpu`.

<a id="feature-flags"></a>

### [§](#feature-flags)Feature flags

<a id="backends"></a>

#### [§](#backends)Backends

- **`dx12`** *(enabled by default)* — Enables the DX12 backend on Windows.
- **`metal`** *(enabled by default)* — Enables the Metal backend on macOS & iOS.
- **`vulkan`** *(enabled by default)* — Enables the Vulkan backend on Windows, Linux, and Android.
- **`gles`** *(enabled by default)* — Enables the OpenGL/GLES backend on Windows, Linux, Android, and Emscripten.
- **`webgpu`** *(enabled by default)* — Enables the WebGPU backend on WebAssembly.

<a id="conditional-backends"></a>

#### [§](#conditional-backends)Conditional Backends

- **`angle`** — Enables the GLES backend on macOS only for use with [ANGLE](https://github.com/google/angle).
- **`vulkan-portability`** — Enables the Vulkan backend on macOS & iOS only for use with [MoltenVK](https://github.com/KhronosGroup/MoltenVK).
- **`webgl`** — Enables the GLES backend on WebAssembly only.

<a id="backend-features"></a>

#### [§](#backend-features)Backend features

- **`drm`** — Support creating textures from DRM display/window handles.
- **`noop`** — Enables the noop backend for testing.

  This backend allows creating resources such as buffers and textures,
  but performs no computation.
  Because it lacks basic functionality, it is only actually used if explicitly enabled
  through `NoopBackendOptions`.

**Note:** In the documentation, if you see that an item depends on a backend,
it means that the item is only available when that backend is enabled *and* the backend
is supported on the current platform.

<a id="shading-language-support"></a>

#### [§](#shading-language-support)Shading language support

These features enable support for that input language on all platforms.
We will translate the input language to whatever the backend requires.

- **`spirv`** — Enable accepting SPIR-V shaders as input.
- **`glsl`** — Enable accepting GLSL shaders as input.
- **`wgsl`** *(enabled by default)* — Enable accepting WGSL shaders as input.
- **`naga-ir`** — Enable accepting naga IR shaders as input.

<a id="assertions-and-serialization"></a>

#### [§](#assertions-and-serialization)Assertions and Serialization

- **`strict_asserts`** — Apply run-time checks, even in release builds. These are in addition
  to the validation carried out at public APIs in all builds.
- **`serde`** — Enables serialization via `serde` on common wgpu types.
- **`trace`** — Allow writing of trace capture files. See [`Adapter::request_device`](https://docs.rs/wgpu/latest/wgpu/struct.Adapter.html#method.request_device "method wgpu::Adapter::request_device").

<a id="external-libraries"></a>

#### [§](#external-libraries)External libraries

The following features facilitate integration with third-party supporting libraries.

- **`static-dxc`** — Enables statically linking DXC.

  Normally, to use the modern DXC shader compiler with wgpu, the final application
  must be shipped alongside `dxcompiler.dll` (min v1.8.2502) (which can be downloaded from [Microsoft’s GitHub](https://github.com/Microsoft/DirectXShaderCompiler)).
  This feature statically links a version of DXC so that no external binaries are required
  to compile DX12 shaders.

<a id="other"></a>

#### [§](#other)Other

- **`counters`** — Internally count resources and events for debugging purposes. If the counters
  feature is disabled, the counting infrastructure is removed from the build and
  the exposed counters always return 0.
- **`fragile-send-sync-non-atomic-wasm`** — Implement `Send` and `Sync` on Wasm, but only if atomics are not enabled.

  WebGL/WebGPU objects can not be shared between threads.
  However, it can be useful to artificially mark them as `Send` and `Sync`
  anyways to make it easier to write cross-platform code.
  This is technically *very* unsafe in a multithreaded environment,
  but on a wasm binary compiled without atomics is a definitionally single-threaded environment.
- **`web`** *(enabled by default)* — Use web-specific libraries on WASM

  Those libraries (wasm-bindgen, web-sys, js-sys) can only be used when there is a JavaScript
  context around the WASM VM, e.g., when the WASM binary is used in a browser.
- **`std`** *(enabled by default)* — Enables use of the standard library within `wgpu` and its dependencies.

  This can allow for better error reporting and for improved multithreading
  support.
- **`parking_lot`** *(enabled by default)* — Uses `parking_lot` as the implementation for locking primitives.

  This is a recommended feature for most users and should only be disabled when
  required, e.g., for `no_std` support.
  If disabled, either `std::sync::Mutex` or `core::cell::RefCell` will be used,
  based on whether `std` is enabled or not.
- **`exhaust`** — Enables `exhaust` method for `TextureFormat` of `wgpu-types`.

  This is relevant for `wgpu-info`.

<a id="feature-aliases"></a>

#### [§](#feature-aliases)Feature Aliases

These features aren’t actually features on the crate itself, but a convenient shorthand for
complicated cases.

- **`wgpu_core`** — Enabled when there is any non-webgpu backend enabled on the platform.
- **`naga`** — Enabled when target `glsl` or `spirv` input is enabled, or when `wgpu_core` is enabled.

<a id="surface-color-spaces-and-hdr-output"></a>

### [§](#surface-color-spaces-and-hdr-output)Surface color spaces and HDR output

A surface can present in different color spaces to get HDR or wide-gamut
output onto the screen. This is configured through
[`SurfaceConfiguration::color_space`](https://docs.rs/wgpu/latest/wgpu/type.SurfaceConfiguration.html#structfield.color_space "field wgpu::SurfaceConfiguration::color_space"), and the rest of this section is a
concept primer on how it works.

<a id="hdr-output-in-a-nutshell"></a>

#### [§](#hdr-output-in-a-nutshell)HDR output in a nutshell

By default a surface is standard dynamic range (SDR) with the sRGB gamut: the
value `1.0` is the brightest white, anything above it clips, and only colors
within sRGB are expressible. Other [`SurfaceColorSpace`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html "enum wgpu::SurfaceColorSpace")s opt into a **wider
gamut** (for example [`DisplayP3`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.DisplayP3 "variant wgpu::SurfaceColorSpace::DisplayP3"), still SDR
but with more saturated colors than sRGB), **high dynamic range** (values
above `1.0` drive brighter-than-white output), or both (for example
[`Bt2100Pq`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.Bt2100Pq "variant wgpu::SurfaceColorSpace::Bt2100Pq"), aka HDR10), on platforms that
support it.

Three ideas carry most of the weight:

- **Reference white and headroom.** Brightness is measured in *nits*
  (cd/m²). On HDR-capable monitors, SDR reference white (plain white, `(1.0, 1.0, 1.0)` in the
  extended color spaces) sits *below* the display’s peak output on purpose, so
  highlights have room above it. That gap is the display’s *headroom*.
- **The transfer function is a round-trip.** Your shader applies an
  encoding transfer function (the OETF) to turn the light it computed into a
  stored signal, and the display applies the inverse decoding transfer
  function (the EOTF) to turn it back into light. Choosing a color space
  chooses which transfer functions both ends use.
- **Who applies the encoding transfer function.** wgpu applies it for you
  **only** when you render to an `*Srgb` texture view format, where the GPU
  runs the sRGB OETF when a value is stored to a texture. For every other
  color space (linear extended sRGB, encoded extended sRGB or P3, PQ, HLG)
  **the values your shader writes to the surface texture must already be
  encoded by you**, along with any gamut conversion; in a typical renderer
  this happens in a final tone-mapping or post-processing pass. wgpu hands
  the signal to the compositor unchanged; getting this wrong produces a
  wrong image with no error.

wgpu does **not** tonemap or gamut-map for you. It gives you the surface
and, through [`DisplayHdrInfo`](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHdrInfo.html "struct wgpu::DisplayHdrInfo"), the display’s advisory capabilities;
*choosing* and *applying* a tone curve is your application’s job.

<a id="the-practical-path"></a>

#### [§](#the-practical-path)The practical path

1. **Query capabilities.** Call `Surface::get_capabilities`. To use HDR or
   wide-gamut output, read [`SurfaceCapabilities::format_capabilities`](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceCapabilities.html#structfield.format_capabilities "field wgpu::SurfaceCapabilities::format_capabilities") (each
   format and the [`SurfaceColorSpaces`](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceColorSpaces.html "struct wgpu::SurfaceColorSpaces") it supports), **not**
   [`SurfaceCapabilities::formats`](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceCapabilities.html#structfield.formats "field wgpu::SurfaceCapabilities::formats"): the latter lists only formats usable
   with [`Auto`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.Auto "variant wgpu::SurfaceColorSpace::Auto"), which never selects HDR.
   [`SurfaceCapabilities::color_spaces`](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceCapabilities.html#method.color_spaces "method wgpu::SurfaceCapabilities::color_spaces") is a convenience lookup for one
   format.
2. **Optionally query the display.** Call `Surface::display_hdr_info` for the
   current [`DisplayHdrInfo`](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHdrInfo.html "struct wgpu::DisplayHdrInfo") (peak and SDR-white nits, EDR headroom,
   primaries, and a coarse dynamic-range/gamut bucket). Use it to pick a
   tone-map target ([`DisplayHdrInfo::tone_map_headroom`](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHdrInfo.html#method.tone_map_headroom "method wgpu::DisplayHdrInfo::tone_map_headroom")); *whether* HDR is
   worthwhile is the capability question from step 1, not this live value.
   Every field is advisory and optional; `None` means “cannot tell here”,
   never “SDR”.
3. **Choose a format and color space.** Intersect what you want with what
   step 1 advertises, in your own preference order (for example HDR10, then
   linear extended sRGB, then encoded extended sRGB, then SDR
   [`Srgb`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.Srgb "variant wgpu::SurfaceColorSpace::Srgb")). Keep an SDR fallback for when nothing
   HDR is advertised, such as when OS HDR is off.
4. **Configure the surface.** Set [`SurfaceConfiguration::color_space`](https://docs.rs/wgpu/latest/wgpu/type.SurfaceConfiguration.html#structfield.color_space "field wgpu::SurfaceConfiguration::color_space") and
   `format`. [`Auto`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.Auto "variant wgpu::SurfaceColorSpace::Auto") (the default) reproduces
   wgpu’s historical behavior and never picks HDR; any other value must be in
   that format’s advertised set or configuration fails validation.
5. **Encode what you write to the surface texture.** For an `*Srgb` format,
   output linear and the hardware encodes for you. Otherwise the values your
   shader writes to the surface texture must already carry the encoding the
   chosen color space expects (sRGB, extended sRGB, PQ, or HLG) **and** any
   gamut conversion (for example outputting in the BT.2020 gamut for HDR10);
   in a typical renderer you do this in a final tone-mapping or
   post-processing pass. See the table below.
6. **Present** as usual. If OS HDR is toggled mid-run, you’ll see it on the
   next `Surface::display_hdr_info` poll; re-query `Surface::get_capabilities`
   and re-run the steps above.

The standalone [HDR surface example](https://github.com/gfx-rs/wgpu/tree/v30/examples/standalone/03_hdr_surface) implements every step, including the
encoding transfer function for each color space.

<a id="what-to-output-from-your-fragment-shader"></a>

#### [§](#what-to-output-from-your-fragment-shader)What to output from your fragment shader

What the values your shader writes to the surface texture must contain for
each color space, and whether wgpu applies the transfer function for you:

| Color space | Typical format | You write | wgpu encodes? |
| --- | --- | --- | --- |
| `Srgb`, `*Srgb` format | `{Rgba,Bgra}8UnormSrgb` | linear | **yes** (hardware sRGB OETF on store) |
| `Srgb`, non-srgb format | `{Rgba,Bgra}8Unorm` | sRGB-encoded | no; apply the sRGB OETF yourself or use `*Srgb` instead |
| `ExtendedSrgbLinear` (scRGB) | `Rgba16Float` | linear, `1.0` = SDR white | no, but no encoding is necessary |
| `ExtendedSrgb` | `Rgba16Float` | extended sRGB-encoded | no; apply the extended sRGB OETF yourself |
| `DisplayP3` | `Bgra8Unorm` | sRGB-encoded, P3 primaries | no; apply the sRGB OETF (after gamut-mapping to P3) |
| `ExtendedDisplayP3` | `Rgba16Float` | extended sRGB-encoded, P3 primaries | no; apply the extended sRGB OETF (after gamut-mapping to P3) |
| `Bt2100Pq` (HDR10) | `Rgb10a2Unorm` | PQ-encoded, BT.2020 primaries | no; apply the PQ OETF (after gamut-mapping to BT.2020) |
| `Bt2100Hlg` | `Rgb10a2Unorm` | HLG-encoded, BT.2020 primaries | no; apply the HLG OETF (after gamut-mapping to BT.2020) |

In short, wgpu applies the transfer function for you only when you render to
an `*Srgb` format. In every other case the values your shader writes to the
surface texture must already carry both the transfer function and any gamut
conversion. The [HDR surface example](https://github.com/gfx-rs/wgpu/tree/v30/examples/standalone/03_hdr_surface) implements every encoder in WGSL.

<a id="glossary"></a>

#### [§](#glossary)Glossary

- **Chromaticity** — a color’s hue and saturation independent of its
  brightness, given as an `(x, y)` coordinate on the CIE 1931 diagram.
- **Primaries / gamut** — the chromaticities of the red, green, and blue a
  color space addresses, and so the range of colors it can express. [BT.709](https://www.itu.int/rec/R-REC-BT.709)
  is the sRGB gamut, [Display P3](https://en.wikipedia.org/wiki/DCI-P3#Display_P3) is wider, and [BT.2020](https://www.itu.int/rec/R-REC-BT.2020) is wider still.
- **White point** — the chromaticity of `R = G = B` (what “white” looks
  like). Every color space here uses [D65](https://en.wikipedia.org/wiki/Standard_illuminant#D65_values), standard daylight.
- **Transfer function (OETF / EOTF)** — how stored values map to light. The
  *OETF* is the encoding transfer function your application applies; the
  *EOTF* is the inverse decoding transfer function the display applies.
- **SDR / HDR** — standard dynamic range clips at `1.0` (reference white);
  high dynamic range lets values above `1.0` drive brighter-than-white
  output.
- **Nits and reference white** — a *nit* (cd/m²) is a unit of brightness;
  *reference white* (also called *paper white*, especially on Windows) is the
  brightness of SDR plain white, set below the panel’s peak so highlights have
  room above it.
- **Headroom (EDR)** — how much brighter than current SDR white the display
  can go right now, as a multiplier (`1.0` means none). Dynamic; see
  [`DisplayHdrInfo::tone_map_headroom`](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHdrInfo.html#method.tone_map_headroom "method wgpu::DisplayHdrInfo::tone_map_headroom").
- **PQ / HLG** — the two HDR transfer functions: [PQ](https://en.wikipedia.org/wiki/Perceptual_quantizer) (SMPTE ST 2084,
  HDR10) encodes absolute luminance, [HLG](https://www.itu.int/rec/R-REC-BT.2100) (BT.2100) encodes relative
  luminance.
- **scRGB / extended-range sRGB** — sRGB extended past 0.0..=1.0 for
  HDR: [scRGB](https://en.wikipedia.org/wiki/ScRGB) is *linear*
  ([`ExtendedSrgbLinear`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.ExtendedSrgbLinear "variant wgpu::SurfaceColorSpace::ExtendedSrgbLinear")), while
  [`ExtendedSrgb`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html#variant.ExtendedSrgb "variant wgpu::SurfaceColorSpace::ExtendedSrgb") is the same range but
  sRGB-*encoded* (gamma), the web’s HDR path.

<a id="reexports"></a>

## Re-exports[§](#reexports)

`pub extern crate wgpu_hal as hal;`

`pub extern crate wgpu_core as wgc;`

`pub extern crate wgpu_types as wgt;`

<a id="reexports-1"></a>

## Re-exports[§](#reexports-1)

<a id="reexport.naga"></a>

`pub use ::wgc::naga;``naga` or `wgpu_core`

<a id="reexport.rwh"></a>

`pub use raw_window_handle as rwh;`

<a id="modules"></a>

## Modules[§](#modules)

[custom](https://docs.rs/wgpu/latest/wgpu/custom/index.html "mod wgpu::custom")`custom`
:   Provides wrappers custom backend implementations

[util](https://docs.rs/wgpu/latest/wgpu/util/index.html "mod wgpu::util")
:   Utility structures and functions that are built on top of the main `wgpu` API.

<a id="macros"></a>

## Macros[§](#macros)

[include\_spirv](https://docs.rs/wgpu/latest/wgpu/macro.include_spirv.html "macro wgpu::include_spirv")`spirv`
:   Macro to load a SPIR-V module statically.

[include\_spirv\_raw](https://docs.rs/wgpu/latest/wgpu/macro.include_spirv_raw.html "macro wgpu::include_spirv_raw")
:   Macro to load raw SPIR-V data statically, for use with [`Features::PASSTHROUGH_SHADERS`](https://docs.rs/wgpu/latest/wgpu/struct.Features.html#associatedconstant.PASSTHROUGH_SHADERS "associated constant wgpu::Features::PASSTHROUGH_SHADERS").

[include\_wgsl](https://docs.rs/wgpu/latest/wgpu/macro.include_wgsl.html "macro wgpu::include_wgsl")
:   Load WGSL source code from a file at compile time.

[vertex\_attr\_array](https://docs.rs/wgpu/latest/wgpu/macro.vertex_attr_array.html "macro wgpu::vertex_attr_array")
:   Macro to produce an array of [`VertexAttribute`](https://docs.rs/wgpu/latest/wgpu/struct.VertexAttribute.html "struct wgpu::VertexAttribute").

<a id="structs"></a>

## Structs[§](#structs)

[Adapter](https://docs.rs/wgpu/latest/wgpu/struct.Adapter.html "struct wgpu::Adapter")
:   Handle to a physical graphics and/or compute device.

[AdapterInfo](https://docs.rs/wgpu/latest/wgpu/struct.AdapterInfo.html "struct wgpu::AdapterInfo")
:   Information about an adapter.

[AllocatorReport](https://docs.rs/wgpu/latest/wgpu/struct.AllocatorReport.html "struct wgpu::AllocatorReport")
:   A report that can be generated for informational purposes using `Allocator::generate_report()`.

[BackendOptions](https://docs.rs/wgpu/latest/wgpu/struct.BackendOptions.html "struct wgpu::BackendOptions")
:   Options that are passed to a given backend.

[Backends](https://docs.rs/wgpu/latest/wgpu/struct.Backends.html "struct wgpu::Backends")
:   Represents the backends that wgpu will use.

[BindGroup](https://docs.rs/wgpu/latest/wgpu/struct.BindGroup.html "struct wgpu::BindGroup")
:   Handle to a binding group.

[BindGroupDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupDescriptor.html "struct wgpu::BindGroupDescriptor")
:   Describes a group of bindings and the resources to be bound.

[BindGroupEntry](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupEntry.html "struct wgpu::BindGroupEntry")
:   An element of a [`BindGroupDescriptor`](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupDescriptor.html "struct wgpu::BindGroupDescriptor"), consisting of a bindable resource
    and the slot to bind it to.

[BindGroupLayout](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupLayout.html "struct wgpu::BindGroupLayout")
:   Handle to a binding group layout.

[BindGroupLayoutDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupLayoutDescriptor.html "struct wgpu::BindGroupLayoutDescriptor")
:   Describes a [`BindGroupLayout`](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupLayout.html "struct wgpu::BindGroupLayout").

[BindGroupLayoutEntry](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupLayoutEntry.html "struct wgpu::BindGroupLayoutEntry")
:   Describes a single binding inside a bind group.

[Blas](https://docs.rs/wgpu/latest/wgpu/struct.Blas.html "struct wgpu::Blas")
:   Bottom Level Acceleration Structure (BLAS).

[BlasAabbGeometry](https://docs.rs/wgpu/latest/wgpu/struct.BlasAabbGeometry.html "struct wgpu::BlasAabbGeometry")
:   Definition for an axis-aligned bounding box geometry group for a bottom level acceleration structure.

[BlasAsyncError](https://docs.rs/wgpu/latest/wgpu/struct.BlasAsyncError.html "struct wgpu::BlasAsyncError")
:   Error occurred when trying to asynchronously prepare a blas for compaction.

[BlasBuildEntry](https://docs.rs/wgpu/latest/wgpu/struct.BlasBuildEntry.html "struct wgpu::BlasBuildEntry")
:   Builds the given sets of geometry into the given [Blas](https://docs.rs/wgpu/latest/wgpu/struct.Blas.html "struct wgpu::Blas").

[BlasTriangleGeometry](https://docs.rs/wgpu/latest/wgpu/struct.BlasTriangleGeometry.html "struct wgpu::BlasTriangleGeometry")
:   Definition for a triangle geometry for a Bottom Level Acceleration Structure (BLAS).

[BlendComponent](https://docs.rs/wgpu/latest/wgpu/struct.BlendComponent.html "struct wgpu::BlendComponent")
:   Describes a blend component of a [`BlendState`](https://docs.rs/wgpu/latest/wgpu/struct.BlendState.html "struct wgpu::BlendState").

[BlendState](https://docs.rs/wgpu/latest/wgpu/struct.BlendState.html "struct wgpu::BlendState")
:   Describe the blend state of a render pipeline,
    within [`ColorTargetState`](https://docs.rs/wgpu/latest/wgpu/struct.ColorTargetState.html "struct wgpu::ColorTargetState").

[Buffer](https://docs.rs/wgpu/latest/wgpu/struct.Buffer.html "struct wgpu::Buffer")
:   Handle to a GPU-accessible buffer.

[BufferAsyncError](https://docs.rs/wgpu/latest/wgpu/struct.BufferAsyncError.html "struct wgpu::BufferAsyncError")
:   Error occurred when trying to async map a buffer.

[BufferBinding](https://docs.rs/wgpu/latest/wgpu/struct.BufferBinding.html "struct wgpu::BufferBinding")
:   Describes the segment of a buffer to bind.

[BufferSlice](https://docs.rs/wgpu/latest/wgpu/struct.BufferSlice.html "struct wgpu::BufferSlice")
:   A slice of a [`Buffer`](https://docs.rs/wgpu/latest/wgpu/struct.Buffer.html "struct wgpu::Buffer"), to be mapped, used for vertex or index data, or the like.

[BufferTextureCopyInfo](https://docs.rs/wgpu/latest/wgpu/struct.BufferTextureCopyInfo.html "struct wgpu::BufferTextureCopyInfo")
:   Information about a copy between a buffer and a texture.

[BufferTransition](https://docs.rs/wgpu/latest/wgpu/struct.BufferTransition.html "struct wgpu::BufferTransition")
:   A buffer transition for use with `CommandEncoder::transition_resources`.

[BufferUsages](https://docs.rs/wgpu/latest/wgpu/struct.BufferUsages.html "struct wgpu::BufferUsages")
:   Different ways that you can use a buffer.

[BufferUses](https://docs.rs/wgpu/latest/wgpu/struct.BufferUses.html "struct wgpu::BufferUses")
:   Similar to `BufferUsages`, but used only for `CommandEncoder::transition_resources`.

[BufferView](https://docs.rs/wgpu/latest/wgpu/struct.BufferView.html "struct wgpu::BufferView")
:   A read-only view of a mapped buffer’s bytes.

[BufferViewMut](https://docs.rs/wgpu/latest/wgpu/struct.BufferViewMut.html "struct wgpu::BufferViewMut")
:   A write-only view of a mapped buffer’s bytes.

[Color](https://docs.rs/wgpu/latest/wgpu/struct.Color.html "struct wgpu::Color")
:   RGBA double precision color.

[ColorTargetState](https://docs.rs/wgpu/latest/wgpu/struct.ColorTargetState.html "struct wgpu::ColorTargetState")
:   Describes the color state of a render pipeline.

[ColorWrites](https://docs.rs/wgpu/latest/wgpu/struct.ColorWrites.html "struct wgpu::ColorWrites")
:   Color write mask. Disabled color channels will not be written to.

[CommandBuffer](https://docs.rs/wgpu/latest/wgpu/struct.CommandBuffer.html "struct wgpu::CommandBuffer")
:   Handle to a command buffer on the GPU.

[CommandBufferDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.CommandBufferDescriptor.html "struct wgpu::CommandBufferDescriptor")
:   Describes a [`CommandBuffer`](https://docs.rs/wgpu/latest/wgpu/struct.CommandBuffer.html).

[CommandEncoder](https://docs.rs/wgpu/latest/wgpu/struct.CommandEncoder.html "struct wgpu::CommandEncoder")
:   Encodes a series of GPU operations.

[CompilationInfo](https://docs.rs/wgpu/latest/wgpu/struct.CompilationInfo.html "struct wgpu::CompilationInfo")
:   Compilation information for a shader module.

[CompilationMessage](https://docs.rs/wgpu/latest/wgpu/struct.CompilationMessage.html "struct wgpu::CompilationMessage")
:   A single message from the shader compilation process.

[ComputePass](https://docs.rs/wgpu/latest/wgpu/struct.ComputePass.html "struct wgpu::ComputePass")
:   In-progress recording of a compute pass.

[ComputePassDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.ComputePassDescriptor.html "struct wgpu::ComputePassDescriptor")
:   Describes the attachments of a compute pass.

[ComputePassTimestampWrites](https://docs.rs/wgpu/latest/wgpu/struct.ComputePassTimestampWrites.html "struct wgpu::ComputePassTimestampWrites")
:   Describes the timestamp writes of a compute pass.

[ComputePipeline](https://docs.rs/wgpu/latest/wgpu/struct.ComputePipeline.html "struct wgpu::ComputePipeline")
:   Handle to a compute pipeline.

[ComputePipelineDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.ComputePipelineDescriptor.html "struct wgpu::ComputePipelineDescriptor")
:   Describes a compute pipeline.

[ContextBlasAabbGeometry](https://docs.rs/wgpu/latest/wgpu/struct.ContextBlasAabbGeometry.html "struct wgpu::ContextBlasAabbGeometry")
:   Context version of [BlasAabbGeometry](https://docs.rs/wgpu/latest/wgpu/struct.BlasAabbGeometry.html "struct wgpu::BlasAabbGeometry").

[ContextBlasBuildEntry](https://docs.rs/wgpu/latest/wgpu/struct.ContextBlasBuildEntry.html "struct wgpu::ContextBlasBuildEntry")
:   Context version see [BlasBuildEntry](https://docs.rs/wgpu/latest/wgpu/struct.BlasBuildEntry.html "struct wgpu::BlasBuildEntry").

[ContextBlasTriangleGeometry](https://docs.rs/wgpu/latest/wgpu/struct.ContextBlasTriangleGeometry.html "struct wgpu::ContextBlasTriangleGeometry")
:   Context version of [BlasTriangleGeometry](https://docs.rs/wgpu/latest/wgpu/struct.BlasTriangleGeometry.html "struct wgpu::BlasTriangleGeometry").

[CooperativeMatrixProperties](https://docs.rs/wgpu/latest/wgpu/struct.CooperativeMatrixProperties.html "struct wgpu::CooperativeMatrixProperties")
:   Describes a supported cooperative matrix configuration.

[CopyExternalImageDestInfo](https://docs.rs/wgpu/latest/wgpu/struct.CopyExternalImageDestInfo.html "struct wgpu::CopyExternalImageDestInfo")
:   View of a texture which can be used to copy to a texture, including
    color space and alpha premultiplication information.

[CoreCounters](https://docs.rs/wgpu/latest/wgpu/struct.CoreCounters.html "struct wgpu::CoreCounters")
:   `wgpu-core`’s part of [`InternalCounters`](https://docs.rs/wgpu/latest/wgpu/struct.InternalCounters.html "struct wgpu::InternalCounters").

[CreateSurfaceError](https://docs.rs/wgpu/latest/wgpu/struct.CreateSurfaceError.html "struct wgpu::CreateSurfaceError")
:   [`Instance::create_surface()`](https://docs.rs/wgpu/latest/wgpu/struct.Instance.html#method.create_surface "method wgpu::Instance::create_surface") or a related function failed.

[DepthBiasState](https://docs.rs/wgpu/latest/wgpu/struct.DepthBiasState.html "struct wgpu::DepthBiasState")
:   Describes the biasing setting for the depth target.

[DepthStencilState](https://docs.rs/wgpu/latest/wgpu/struct.DepthStencilState.html "struct wgpu::DepthStencilState")
:   Describes the depth/stencil state in a render pipeline.

[Device](https://docs.rs/wgpu/latest/wgpu/struct.Device.html "struct wgpu::Device")
:   Open connection to a graphics and/or compute device.

[DisplayChromaticity](https://docs.rs/wgpu/latest/wgpu/struct.DisplayChromaticity.html "struct wgpu::DisplayChromaticity")
:   CIE 1931 xy chromaticity of a display’s primaries and white point. Each
    coordinate is `[x, y]`; a coordinate the platform omits is `None`.

[DisplayCoarseRange](https://docs.rs/wgpu/latest/wgpu/struct.DisplayCoarseRange.html "struct wgpu::DisplayCoarseRange")
:   Coarse, boolean dynamic-range and gamut signal.

[DisplayHdrInfo](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHdrInfo.html "struct wgpu::DisplayHdrInfo")
:   HDR and luminance characteristics of the display backing a [`Surface`](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html), as
    reported by the platform at query time.

[DisplayHeadroom](https://docs.rs/wgpu/latest/wgpu/struct.DisplayHeadroom.html "struct wgpu::DisplayHeadroom")
:   Relative EDR headroom (Apple): unitless multipliers over current SDR white,
    where `1.0` means no headroom. Moves with brightness, ambient light, battery,
    and which display the window is on. Apple exposes no absolute-nit equivalent,
    so this is separate from [`DisplayLuminance`](https://docs.rs/wgpu/latest/wgpu/struct.DisplayLuminance.html "struct wgpu::DisplayLuminance") and can’t be converted to nits.

[DisplayLuminance](https://docs.rs/wgpu/latest/wgpu/struct.DisplayLuminance.html "struct wgpu::DisplayLuminance")
:   Absolute luminance levels in nits (cd/m²). Populated only on Windows (via
    DXGI); `None` on every other platform.

[DownlevelCapabilities](https://docs.rs/wgpu/latest/wgpu/struct.DownlevelCapabilities.html "struct wgpu::DownlevelCapabilities")
:   Lists various ways the underlying platform does not conform to the WebGPU standard.

[DownlevelFlags](https://docs.rs/wgpu/latest/wgpu/struct.DownlevelFlags.html "struct wgpu::DownlevelFlags")
:   Binary flags listing features that may or may not be present on downlevel adapters.

[DownlevelLimits](https://docs.rs/wgpu/latest/wgpu/struct.DownlevelLimits.html "struct wgpu::DownlevelLimits")
:   Represents the sets of additional limits on an adapter,
    which take place when running on downlevel backends.

[Dx12BackendOptions](https://docs.rs/wgpu/latest/wgpu/struct.Dx12BackendOptions.html "struct wgpu::Dx12BackendOptions")
:   Configuration for the DX12 backend.

[ErrorScopeGuard](https://docs.rs/wgpu/latest/wgpu/struct.ErrorScopeGuard.html "struct wgpu::ErrorScopeGuard")
:   Guard for an error scope pushed with [`Device::push_error_scope()`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.push_error_scope "method wgpu::Device::push_error_scope").

[ExperimentalFeatures](https://docs.rs/wgpu/latest/wgpu/struct.ExperimentalFeatures.html "struct wgpu::ExperimentalFeatures")
:   Token of the user agreeing to access experimental features.

[Extent3d](https://docs.rs/wgpu/latest/wgpu/struct.Extent3d.html "struct wgpu::Extent3d")
:   Extent of a texture related operation.

[ExternalTexture](https://docs.rs/wgpu/latest/wgpu/struct.ExternalTexture.html "struct wgpu::ExternalTexture")
:   Handle to an external texture on the GPU.

[ExternalTextureTransferFunction](https://docs.rs/wgpu/latest/wgpu/struct.ExternalTextureTransferFunction.html "struct wgpu::ExternalTextureTransferFunction")
:   Parameters describing a gamma encoding transfer function in the form
    tf = { k \* linear | linear < b
    { a \* pow(linear, 1/g) - (a-1) | linear >= b

[Features](https://docs.rs/wgpu/latest/wgpu/struct.Features.html "struct wgpu::Features")
:   Features that are not guaranteed to be supported.

[FeaturesWGPU](https://docs.rs/wgpu/latest/wgpu/struct.FeaturesWGPU.html "struct wgpu::FeaturesWGPU")
:   Features that are not guaranteed to be supported.

[FeaturesWebGPU](https://docs.rs/wgpu/latest/wgpu/struct.FeaturesWebGPU.html "struct wgpu::FeaturesWebGPU")
:   Features that are not guaranteed to be supported.

[ForceShaderModelToken](https://docs.rs/wgpu/latest/wgpu/struct.ForceShaderModelToken.html "struct wgpu::ForceShaderModelToken")
:   Used to force wgpu to expose certain features on passthrough shaders even when
    those features aren’t present on runtime-compiled shaders

[FragmentState](https://docs.rs/wgpu/latest/wgpu/struct.FragmentState.html "struct wgpu::FragmentState")
:   Describes the fragment processing in a render pipeline.

[GlBackendOptions](https://docs.rs/wgpu/latest/wgpu/struct.GlBackendOptions.html "struct wgpu::GlBackendOptions")
:   Configuration for the OpenGL/OpenGLES backend.

[HalCounters](https://docs.rs/wgpu/latest/wgpu/struct.HalCounters.html "struct wgpu::HalCounters")
:   `wgpu-hal`’s part of [`InternalCounters`](https://docs.rs/wgpu/latest/wgpu/struct.InternalCounters.html "struct wgpu::InternalCounters").

[ImageSubresourceRange](https://docs.rs/wgpu/latest/wgpu/struct.ImageSubresourceRange.html "struct wgpu::ImageSubresourceRange")
:   Subresource range within an image

[Instance](https://docs.rs/wgpu/latest/wgpu/struct.Instance.html "struct wgpu::Instance")
:   Contains the various entry points to start interacting with the system’s GPUs.

[InstanceDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.InstanceDescriptor.html "struct wgpu::InstanceDescriptor")
:   Options for creating an instance.

[InstanceFlags](https://docs.rs/wgpu/latest/wgpu/struct.InstanceFlags.html "struct wgpu::InstanceFlags")
:   Instance debugging flags.

[InternalCounters](https://docs.rs/wgpu/latest/wgpu/struct.InternalCounters.html "struct wgpu::InternalCounters")
:   All internal counters, exposed for debugging purposes.

[Limits](https://docs.rs/wgpu/latest/wgpu/struct.Limits.html "struct wgpu::Limits")
:   Represents the sets of limits an adapter/device supports.

[LoadOpDontCare](https://docs.rs/wgpu/latest/wgpu/struct.LoadOpDontCare.html "struct wgpu::LoadOpDontCare")
:   Token of the user agreeing to use [`LoadOp::DontCare`](https://docs.rs/wgpu/latest/wgpu/enum.LoadOp.html#variant.DontCare "variant wgpu::LoadOp::DontCare").

[MapRangeError](https://docs.rs/wgpu/latest/wgpu/struct.MapRangeError.html "struct wgpu::MapRangeError")
:   Error returned by [`BufferSlice::get_mapped_range`](https://docs.rs/wgpu/latest/wgpu/struct.BufferSlice.html#method.get_mapped_range "method wgpu::BufferSlice::get_mapped_range") and [`BufferSlice::get_mapped_range_mut`](https://docs.rs/wgpu/latest/wgpu/struct.BufferSlice.html#method.get_mapped_range_mut "method wgpu::BufferSlice::get_mapped_range_mut").

[MemoryBudgetThresholds](https://docs.rs/wgpu/latest/wgpu/struct.MemoryBudgetThresholds.html "struct wgpu::MemoryBudgetThresholds")
:   Memory budget thresholds used by backends to try to avoid high memory pressure situations.

[MeshPipelineDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.MeshPipelineDescriptor.html "struct wgpu::MeshPipelineDescriptor")
:   Describes a mesh shader (graphics) pipeline.

[MeshState](https://docs.rs/wgpu/latest/wgpu/struct.MeshState.html "struct wgpu::MeshState")
:   Describes the mesh shader stage in a mesh shader pipeline.

[MultisampleState](https://docs.rs/wgpu/latest/wgpu/struct.MultisampleState.html "struct wgpu::MultisampleState")
:   Describes the multi-sampling state of a render pipeline.

[NoopBackendOptions](https://docs.rs/wgpu/latest/wgpu/struct.NoopBackendOptions.html "struct wgpu::NoopBackendOptions")
:   Configuration for the noop backend.

[Operations](https://docs.rs/wgpu/latest/wgpu/struct.Operations.html "struct wgpu::Operations")
:   Pair of load and store operations for an attachment aspect.

[Origin2d](https://docs.rs/wgpu/latest/wgpu/struct.Origin2d.html "struct wgpu::Origin2d")
:   Origin of a copy from a 2D image.

[Origin3d](https://docs.rs/wgpu/latest/wgpu/struct.Origin3d.html "struct wgpu::Origin3d")
:   Origin of a copy to/from a texture.

[PassthroughShaderEntryPoint](https://docs.rs/wgpu/latest/wgpu/struct.PassthroughShaderEntryPoint.html "struct wgpu::PassthroughShaderEntryPoint")
:   Describes a single entry point in a passthrough shader descriptor.

[PipelineCache](https://docs.rs/wgpu/latest/wgpu/struct.PipelineCache.html "struct wgpu::PipelineCache")
:   Handle to a pipeline cache, which is used to accelerate
    creating [`RenderPipeline`](https://docs.rs/wgpu/latest/wgpu/struct.RenderPipeline.html "struct wgpu::RenderPipeline")s and [`ComputePipeline`](https://docs.rs/wgpu/latest/wgpu/struct.ComputePipeline.html "struct wgpu::ComputePipeline")s
    in subsequent executions

[PipelineCacheDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.PipelineCacheDescriptor.html "struct wgpu::PipelineCacheDescriptor")
:   Describes a pipeline cache, which allows reusing compilation work
    between program runs.

[PipelineCompilationOptions](https://docs.rs/wgpu/latest/wgpu/struct.PipelineCompilationOptions.html "struct wgpu::PipelineCompilationOptions")
:   Advanced options for use when a pipeline is compiled

[PipelineLayout](https://docs.rs/wgpu/latest/wgpu/struct.PipelineLayout.html "struct wgpu::PipelineLayout")
:   Handle to a pipeline layout.

[PipelineLayoutDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.PipelineLayoutDescriptor.html "struct wgpu::PipelineLayoutDescriptor")
:   Describes a [`PipelineLayout`](https://docs.rs/wgpu/latest/wgpu/struct.PipelineLayout.html "struct wgpu::PipelineLayout").

[PipelineStatisticsTypes](https://docs.rs/wgpu/latest/wgpu/struct.PipelineStatisticsTypes.html "struct wgpu::PipelineStatisticsTypes")
:   Flags for which pipeline data should be recorded in a query.

[PresentationTimestamp](https://docs.rs/wgpu/latest/wgpu/struct.PresentationTimestamp.html "struct wgpu::PresentationTimestamp")
:   Nanosecond timestamp used by the presentation engine.

[PrimitiveState](https://docs.rs/wgpu/latest/wgpu/struct.PrimitiveState.html "struct wgpu::PrimitiveState")
:   Describes the state of primitive assembly and rasterization in a render pipeline.

[QuerySet](https://docs.rs/wgpu/latest/wgpu/struct.QuerySet.html "struct wgpu::QuerySet")
:   Handle to a query set.

[Queue](https://docs.rs/wgpu/latest/wgpu/struct.Queue.html "struct wgpu::Queue")
:   Handle to a command queue on a device.

[QueueWriteBufferView](https://docs.rs/wgpu/latest/wgpu/struct.QueueWriteBufferView.html "struct wgpu::QueueWriteBufferView")
:   A write-only view into a staging buffer.

[RenderBundle](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundle.html "struct wgpu::RenderBundle")
:   Pre-prepared reusable bundle of GPU operations.

[RenderBundleDepthStencil](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundleDepthStencil.html "struct wgpu::RenderBundleDepthStencil")
:   Describes the depth/stencil attachment for render bundles.

[RenderBundleEncoder](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundleEncoder.html "struct wgpu::RenderBundleEncoder")
:   Encodes a series of GPU operations into a reusable “render bundle”.

[RenderBundleEncoderDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundleEncoderDescriptor.html "struct wgpu::RenderBundleEncoderDescriptor")
:   Describes a [`RenderBundleEncoder`](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundleEncoder.html "struct wgpu::RenderBundleEncoder").

[RenderPass](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html "struct wgpu::RenderPass")
:   In-progress recording of a render pass: a list of render commands in a [`CommandEncoder`](https://docs.rs/wgpu/latest/wgpu/struct.CommandEncoder.html "struct wgpu::CommandEncoder").

[RenderPassColorAttachment](https://docs.rs/wgpu/latest/wgpu/struct.RenderPassColorAttachment.html "struct wgpu::RenderPassColorAttachment")
:   Describes a color attachment to a [`RenderPass`](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html "struct wgpu::RenderPass").

[RenderPassDepthStencilAttachment](https://docs.rs/wgpu/latest/wgpu/struct.RenderPassDepthStencilAttachment.html "struct wgpu::RenderPassDepthStencilAttachment")
:   Describes a depth/stencil attachment to a [`RenderPass`](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html "struct wgpu::RenderPass").

[RenderPassDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.RenderPassDescriptor.html "struct wgpu::RenderPassDescriptor")
:   Describes the attachments of a render pass.

[RenderPassTimestampWrites](https://docs.rs/wgpu/latest/wgpu/struct.RenderPassTimestampWrites.html "struct wgpu::RenderPassTimestampWrites")
:   Describes the timestamp writes of a render pass.

[RenderPipeline](https://docs.rs/wgpu/latest/wgpu/struct.RenderPipeline.html "struct wgpu::RenderPipeline")
:   Handle to a rendering (graphics) pipeline.

[RenderPipelineDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.RenderPipelineDescriptor.html "struct wgpu::RenderPipelineDescriptor")
:   Describes a render (graphics) pipeline.

[RequestAdapterOptionsBase](https://docs.rs/wgpu/latest/wgpu/struct.RequestAdapterOptionsBase.html "struct wgpu::RequestAdapterOptionsBase")
:   Options for requesting adapter.

[RequestDeviceError](https://docs.rs/wgpu/latest/wgpu/struct.RequestDeviceError.html "struct wgpu::RequestDeviceError")
:   Requesting a device from an [`Adapter`](https://docs.rs/wgpu/latest/wgpu/struct.Adapter.html "struct wgpu::Adapter") failed.

[Sampler](https://docs.rs/wgpu/latest/wgpu/struct.Sampler.html "struct wgpu::Sampler")
:   Handle to a sampler.

[ShaderModule](https://docs.rs/wgpu/latest/wgpu/struct.ShaderModule.html "struct wgpu::ShaderModule")
:   Handle to a compiled shader module.

[ShaderModuleDescriptor](https://docs.rs/wgpu/latest/wgpu/struct.ShaderModuleDescriptor.html "struct wgpu::ShaderModuleDescriptor")
:   Descriptor for use with [`Device::create_shader_module`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.create_shader_module "method wgpu::Device::create_shader_module").

[ShaderRuntimeChecks](https://docs.rs/wgpu/latest/wgpu/struct.ShaderRuntimeChecks.html "struct wgpu::ShaderRuntimeChecks")
:   Describes how shader bound checks should be performed.

[ShaderStages](https://docs.rs/wgpu/latest/wgpu/struct.ShaderStages.html "struct wgpu::ShaderStages")
:   Describes the shader stages that a binding will be visible from.

[SourceLocation](https://docs.rs/wgpu/latest/wgpu/struct.SourceLocation.html "struct wgpu::SourceLocation")
:   A human-readable representation for a span, tailored for text source.

[StencilFaceState](https://docs.rs/wgpu/latest/wgpu/struct.StencilFaceState.html "struct wgpu::StencilFaceState")
:   Describes stencil state in a render pipeline.

[StencilState](https://docs.rs/wgpu/latest/wgpu/struct.StencilState.html "struct wgpu::StencilState")
:   State of the stencil operation (fixed-pipeline stage).

[SubmissionIndex](https://docs.rs/wgpu/latest/wgpu/struct.SubmissionIndex.html "struct wgpu::SubmissionIndex")
:   Identifier for a particular call to [`Queue::submit`](https://docs.rs/wgpu/latest/wgpu/struct.Queue.html#method.submit "method wgpu::Queue::submit"). Can be used
    as part of an argument to [`Device::poll`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.poll "method wgpu::Device::poll") to block for a particular
    submission to finish.

[Surface](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html "struct wgpu::Surface")
:   Handle to a presentable surface.

[SurfaceCapabilities](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceCapabilities.html "struct wgpu::SurfaceCapabilities")
:   Defines the capabilities of a given surface and adapter.

[SurfaceColorSpaces](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceColorSpaces.html "struct wgpu::SurfaceColorSpaces")
:   A set of [`SurfaceColorSpace`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html "enum wgpu::SurfaceColorSpace")s supported by a surface for a particular
    texture format.

[SurfaceFormatCapabilities](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceFormatCapabilities.html "struct wgpu::SurfaceFormatCapabilities")
:   A texture format supported by a surface, together with the color spaces
    in which the surface can present it.

[SurfaceTexture](https://docs.rs/wgpu/latest/wgpu/struct.SurfaceTexture.html "struct wgpu::SurfaceTexture")
:   Surface texture that can be rendered to.
    Result of a successful call to [`Surface::get_current_texture`](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html#method.get_current_texture "method wgpu::Surface::get_current_texture").

[TaskState](https://docs.rs/wgpu/latest/wgpu/struct.TaskState.html "struct wgpu::TaskState")
:   Describes the task shader stage in a mesh shader pipeline.

[TexelCopyBufferInfoBase](https://docs.rs/wgpu/latest/wgpu/struct.TexelCopyBufferInfoBase.html "struct wgpu::TexelCopyBufferInfoBase")
:   View of a buffer which can be used to copy to/from a texture.

[TexelCopyBufferLayout](https://docs.rs/wgpu/latest/wgpu/struct.TexelCopyBufferLayout.html "struct wgpu::TexelCopyBufferLayout")
:   Layout of a texture in a buffer’s memory.

[TexelCopyTextureInfoBase](https://docs.rs/wgpu/latest/wgpu/struct.TexelCopyTextureInfoBase.html "struct wgpu::TexelCopyTextureInfoBase")
:   View of a texture which can be used to copy to/from a buffer/texture.

[Texture](https://docs.rs/wgpu/latest/wgpu/struct.Texture.html "struct wgpu::Texture")
:   Handle to a texture on the GPU.

[TextureChannel](https://docs.rs/wgpu/latest/wgpu/struct.TextureChannel.html "struct wgpu::TextureChannel")
:   Represents the different format channels of a texture format.

[TextureFormatFeatureFlags](https://docs.rs/wgpu/latest/wgpu/struct.TextureFormatFeatureFlags.html "struct wgpu::TextureFormatFeatureFlags")
:   Feature flags for a texture format.

[TextureFormatFeatures](https://docs.rs/wgpu/latest/wgpu/struct.TextureFormatFeatures.html "struct wgpu::TextureFormatFeatures")
:   Features supported by a given texture format

[TextureTransition](https://docs.rs/wgpu/latest/wgpu/struct.TextureTransition.html "struct wgpu::TextureTransition")
:   A texture transition for use with `CommandEncoder::transition_resources`.

[TextureUsages](https://docs.rs/wgpu/latest/wgpu/struct.TextureUsages.html "struct wgpu::TextureUsages")
:   Different ways that you can use a texture.

[TextureUses](https://docs.rs/wgpu/latest/wgpu/struct.TextureUses.html "struct wgpu::TextureUses")
:   Similar to `TextureUsages`, but used only for `CommandEncoder::transition_resources`.

[TextureView](https://docs.rs/wgpu/latest/wgpu/struct.TextureView.html "struct wgpu::TextureView")
:   Handle to a texture view.

[Tlas](https://docs.rs/wgpu/latest/wgpu/struct.Tlas.html "struct wgpu::Tlas")
:   Top Level Acceleration Structure (TLAS).

[TlasInstance](https://docs.rs/wgpu/latest/wgpu/struct.TlasInstance.html "struct wgpu::TlasInstance")
:   Safe instance for a [Tlas](https://docs.rs/wgpu/latest/wgpu/struct.Tlas.html "struct wgpu::Tlas").

[VertexAttribute](https://docs.rs/wgpu/latest/wgpu/struct.VertexAttribute.html "struct wgpu::VertexAttribute")
:   Vertex inputs (attributes) to shaders.

[VertexBufferLayout](https://docs.rs/wgpu/latest/wgpu/struct.VertexBufferLayout.html "struct wgpu::VertexBufferLayout")
:   Specifies an interpretation of the bytes of a vertex buffer as vertex attributes.

[VertexState](https://docs.rs/wgpu/latest/wgpu/struct.VertexState.html "struct wgpu::VertexState")
:   Describes the vertex processing in a render pipeline.

[WgslLanguageFeatures](https://docs.rs/wgpu/latest/wgpu/struct.WgslLanguageFeatures.html "struct wgpu::WgslLanguageFeatures")
:   WGSL language extensions.

[WriteOnly](https://docs.rs/wgpu/latest/wgpu/struct.WriteOnly.html "struct wgpu::WriteOnly")
:   Like `&'a mut T`, but allows only write operations.

[WriteOnlyIter](https://docs.rs/wgpu/latest/wgpu/struct.WriteOnlyIter.html "struct wgpu::WriteOnlyIter")
:   Iterator over the elements of [`WriteOnly<[T]>`](https://docs.rs/wgpu/latest/wgpu/struct.WriteOnly.html "struct wgpu::WriteOnly").

<a id="enums"></a>

## Enums[§](#enums)

[AddressMode](https://docs.rs/wgpu/latest/wgpu/enum.AddressMode.html "enum wgpu::AddressMode")
:   How edges should be handled in texture addressing.

[AstcBlock](https://docs.rs/wgpu/latest/wgpu/enum.AstcBlock.html "enum wgpu::AstcBlock")
:   ASTC block dimensions

[AstcChannel](https://docs.rs/wgpu/latest/wgpu/enum.AstcChannel.html "enum wgpu::AstcChannel")
:   ASTC RGBA channel

[Backend](https://docs.rs/wgpu/latest/wgpu/enum.Backend.html "enum wgpu::Backend")
:   Backends supported by wgpu.

[BindingResource](https://docs.rs/wgpu/latest/wgpu/enum.BindingResource.html "enum wgpu::BindingResource")
:   Resource to be bound by a [`BindGroup`](https://docs.rs/wgpu/latest/wgpu/struct.BindGroup.html "struct wgpu::BindGroup") for use with a pipeline.

[BindingType](https://docs.rs/wgpu/latest/wgpu/enum.BindingType.html "enum wgpu::BindingType")
:   Type of a binding in a [bind group layout](https://docs.rs/wgpu/latest/wgpu/struct.BindGroupLayoutEntry.html "struct wgpu::BindGroupLayoutEntry").

[BlasGeometries](https://docs.rs/wgpu/latest/wgpu/enum.BlasGeometries.html "enum wgpu::BlasGeometries")
:   Contains the sets of geometry that go into a [Blas](https://docs.rs/wgpu/latest/wgpu/struct.Blas.html "struct wgpu::Blas").

[BlendFactor](https://docs.rs/wgpu/latest/wgpu/enum.BlendFactor.html "enum wgpu::BlendFactor")
:   Alpha blend factor.

[BlendOperation](https://docs.rs/wgpu/latest/wgpu/enum.BlendOperation.html "enum wgpu::BlendOperation")
:   Alpha blend operation.

[BufferBindingType](https://docs.rs/wgpu/latest/wgpu/enum.BufferBindingType.html "enum wgpu::BufferBindingType")
:   Specific type of a buffer binding.

[CompareFunction](https://docs.rs/wgpu/latest/wgpu/enum.CompareFunction.html "enum wgpu::CompareFunction")
:   Comparison function used for depth and stencil operations.

[CompilationMessageType](https://docs.rs/wgpu/latest/wgpu/enum.CompilationMessageType.html "enum wgpu::CompilationMessageType")
:   The type of a compilation message.

[CompositeAlphaMode](https://docs.rs/wgpu/latest/wgpu/enum.CompositeAlphaMode.html "enum wgpu::CompositeAlphaMode")
:   Specifies how the alpha channel of the textures should be handled during
    compositing.

[ContextBlasGeometries](https://docs.rs/wgpu/latest/wgpu/enum.ContextBlasGeometries.html "enum wgpu::ContextBlasGeometries")
:   Context version of [BlasGeometries](https://docs.rs/wgpu/latest/wgpu/enum.BlasGeometries.html "enum wgpu::BlasGeometries").

[CooperativeScalarType](https://docs.rs/wgpu/latest/wgpu/enum.CooperativeScalarType.html "enum wgpu::CooperativeScalarType")
:   The underlying scalar type of the cooperative matrix component.

[CurrentSurfaceTexture](https://docs.rs/wgpu/latest/wgpu/enum.CurrentSurfaceTexture.html "enum wgpu::CurrentSurfaceTexture")
:   Result of a call to [`Surface::get_current_texture`](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html#method.get_current_texture "method wgpu::Surface::get_current_texture").

[DeviceLostReason](https://docs.rs/wgpu/latest/wgpu/enum.DeviceLostReason.html "enum wgpu::DeviceLostReason")
:   Corresponds to a [`GPUDeviceLostReason`](https://www.w3.org/TR/webgpu/#enumdef-gpudevicelostreason).

[DeviceType](https://docs.rs/wgpu/latest/wgpu/enum.DeviceType.html "enum wgpu::DeviceType")
:   Supported physical device types.

[DisplayGamut](https://docs.rs/wgpu/latest/wgpu/enum.DisplayGamut.html "enum wgpu::DisplayGamut")
:   Coarse gamut classification, mirroring CSS `color-gamut`.

[Dx12Compiler](https://docs.rs/wgpu/latest/wgpu/enum.Dx12Compiler.html "enum wgpu::Dx12Compiler")
:   Selects which DX12 shader compiler to use.

[Dx12SwapchainKind](https://docs.rs/wgpu/latest/wgpu/enum.Dx12SwapchainKind.html "enum wgpu::Dx12SwapchainKind")
:   Selects which kind of swapchain to use on DX12.

[Dx12UseFrameLatencyWaitableObject](https://docs.rs/wgpu/latest/wgpu/enum.Dx12UseFrameLatencyWaitableObject.html "enum wgpu::Dx12UseFrameLatencyWaitableObject")
:   Whether and how to use a waitable handle obtained from `GetFrameLatencyWaitableObject`.

[DxcShaderModel](https://docs.rs/wgpu/latest/wgpu/enum.DxcShaderModel.html "enum wgpu::DxcShaderModel")
:   DXC shader model.

[Error](https://docs.rs/wgpu/latest/wgpu/enum.Error.html "enum wgpu::Error")
:   Errors resulting from usage of GPU APIs.

[ErrorFilter](https://docs.rs/wgpu/latest/wgpu/enum.ErrorFilter.html "enum wgpu::ErrorFilter")
:   Kinds of [`Error`](https://docs.rs/wgpu/latest/wgpu/enum.Error.html "enum wgpu::Error")s a [`Device::push_error_scope()`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.push_error_scope "method wgpu::Device::push_error_scope") may be configured to catch.

[ExternalTextureFormat](https://docs.rs/wgpu/latest/wgpu/enum.ExternalTextureFormat.html "enum wgpu::ExternalTextureFormat")
:   Format of an `ExternalTexture`. This indicates the number of underlying
    planes used by the `ExternalTexture` as well as each plane’s format.

[Face](https://docs.rs/wgpu/latest/wgpu/enum.Face.html "enum wgpu::Face")
:   Face of a vertex.

[FilterMode](https://docs.rs/wgpu/latest/wgpu/enum.FilterMode.html "enum wgpu::FilterMode")
:   Texel mixing mode when sampling between texels.

[FrontFace](https://docs.rs/wgpu/latest/wgpu/enum.FrontFace.html "enum wgpu::FrontFace")
:   Vertex winding order which classifies the “front” face of a triangle.

[GlDebugFns](https://docs.rs/wgpu/latest/wgpu/enum.GlDebugFns.html "enum wgpu::GlDebugFns")
:   Controls whether OpenGL debug functions are enabled.

[GlFenceBehavior](https://docs.rs/wgpu/latest/wgpu/enum.GlFenceBehavior.html "enum wgpu::GlFenceBehavior")
:   Dictate the behavior of fences in OpenGL.

[Gles3MinorVersion](https://docs.rs/wgpu/latest/wgpu/enum.Gles3MinorVersion.html "enum wgpu::Gles3MinorVersion")
:   Selects which OpenGL ES 3 minor version to request.

[IndexFormat](https://docs.rs/wgpu/latest/wgpu/enum.IndexFormat.html "enum wgpu::IndexFormat")
:   Format of indices used with pipeline.

[LoadOp](https://docs.rs/wgpu/latest/wgpu/enum.LoadOp.html "enum wgpu::LoadOp")
:   Operation to perform to the output attachment at the start of a render pass.

[MapMode](https://docs.rs/wgpu/latest/wgpu/enum.MapMode.html "enum wgpu::MapMode")
:   Type of buffer mapping.

[MemoryHints](https://docs.rs/wgpu/latest/wgpu/enum.MemoryHints.html "enum wgpu::MemoryHints")
:   Hints to the device about the memory allocation strategy.

[MipmapFilterMode](https://docs.rs/wgpu/latest/wgpu/enum.MipmapFilterMode.html "enum wgpu::MipmapFilterMode")
:   Texel mixing mode when sampling between texels.

[PollError](https://docs.rs/wgpu/latest/wgpu/enum.PollError.html "enum wgpu::PollError")
:   Error states after a device poll.

[PollStatus](https://docs.rs/wgpu/latest/wgpu/enum.PollStatus.html "enum wgpu::PollStatus")
:   Status of device poll operation.

[PolygonMode](https://docs.rs/wgpu/latest/wgpu/enum.PolygonMode.html "enum wgpu::PolygonMode")
:   Type of drawing mode for polygons

[PowerPreference](https://docs.rs/wgpu/latest/wgpu/enum.PowerPreference.html "enum wgpu::PowerPreference")
:   Power Preference when choosing a physical adapter.

[PredefinedColorSpace](https://docs.rs/wgpu/latest/wgpu/enum.PredefinedColorSpace.html "enum wgpu::PredefinedColorSpace")
:   Color spaces supported on the web.

[PresentMode](https://docs.rs/wgpu/latest/wgpu/enum.PresentMode.html "enum wgpu::PresentMode")
:   Timing and queueing with which frames are actually displayed to the user.

[PrimitiveTopology](https://docs.rs/wgpu/latest/wgpu/enum.PrimitiveTopology.html "enum wgpu::PrimitiveTopology")
:   Primitive type the input mesh is composed of.

[QueryType](https://docs.rs/wgpu/latest/wgpu/enum.QueryType.html "enum wgpu::QueryType")
:   Type of queries contained in a [`QuerySet`](https://docs.rs/wgpu/latest/wgpu/struct.QuerySet.html).

[RequestAdapterError](https://docs.rs/wgpu/latest/wgpu/enum.RequestAdapterError.html "enum wgpu::RequestAdapterError")
:   Error when [`Instance::request_adapter()`](https://docs.rs/wgpu/latest/wgpu/struct.Instance.html#method.request_adapter) fails.

[SamplerBindingType](https://docs.rs/wgpu/latest/wgpu/enum.SamplerBindingType.html "enum wgpu::SamplerBindingType")
:   Specific type of a sampler binding.

[SamplerBorderColor](https://docs.rs/wgpu/latest/wgpu/enum.SamplerBorderColor.html "enum wgpu::SamplerBorderColor")
:   Color variation to use when sampler addressing mode is [`AddressMode::ClampToBorder`](https://docs.rs/wgpu/latest/wgpu/enum.AddressMode.html#variant.ClampToBorder "variant wgpu::AddressMode::ClampToBorder")

[ShaderModel](https://docs.rs/wgpu/latest/wgpu/enum.ShaderModel.html "enum wgpu::ShaderModel")
:   Collections of shader features a device supports if they support less than WebGPU normally allows.

[ShaderSource](https://docs.rs/wgpu/latest/wgpu/enum.ShaderSource.html "enum wgpu::ShaderSource")
:   Source of a shader module.

[StencilOperation](https://docs.rs/wgpu/latest/wgpu/enum.StencilOperation.html "enum wgpu::StencilOperation")
:   Operation to perform on the stencil value.

[StorageTextureAccess](https://docs.rs/wgpu/latest/wgpu/enum.StorageTextureAccess.html "enum wgpu::StorageTextureAccess")
:   Specific type of a sample in a texture binding.

[StoreOp](https://docs.rs/wgpu/latest/wgpu/enum.StoreOp.html "enum wgpu::StoreOp")
:   Operation to perform to the output attachment at the end of a render pass.

[SurfaceColorSpace](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceColorSpace.html "enum wgpu::SurfaceColorSpace")
:   The color space in which the presentation engine interprets the values
    written to a surface texture.

[SurfaceStatus](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceStatus.html "enum wgpu::SurfaceStatus")
:   Status of the received surface image.

[SurfaceTarget](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceTarget.html "enum wgpu::SurfaceTarget")
:   The window/canvas/surface/swap-chain/etc. a surface is attached to, for use with safe surface creation.

[SurfaceTargetUnsafe](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceTargetUnsafe.html "enum wgpu::SurfaceTargetUnsafe")
:   The window/canvas/surface/swap-chain/etc. a surface is attached to, for use with unsafe surface creation.

[TextureAspect](https://docs.rs/wgpu/latest/wgpu/enum.TextureAspect.html "enum wgpu::TextureAspect")
:   Selects a subset of the data a [`Texture`](https://docs.rs/wgpu/latest/wgpu/struct.Texture.html) holds.

[TextureDimension](https://docs.rs/wgpu/latest/wgpu/enum.TextureDimension.html "enum wgpu::TextureDimension")
:   Dimensionality of a texture.

[TextureFormat](https://docs.rs/wgpu/latest/wgpu/enum.TextureFormat.html "enum wgpu::TextureFormat")
:   Format in which a texture’s texels are stored in GPU memory.

[TextureSampleType](https://docs.rs/wgpu/latest/wgpu/enum.TextureSampleType.html "enum wgpu::TextureSampleType")
:   Specific type of a sample in a texture binding.

[TextureViewDimension](https://docs.rs/wgpu/latest/wgpu/enum.TextureViewDimension.html "enum wgpu::TextureViewDimension")
:   Dimensions of a particular texture view.

[Trace](https://docs.rs/wgpu/latest/wgpu/enum.Trace.html "enum wgpu::Trace")
:   Controls API call tracing and specifies where the trace is written.

[VertexFormat](https://docs.rs/wgpu/latest/wgpu/enum.VertexFormat.html "enum wgpu::VertexFormat")
:   Corresponds to [WebGPU `GPUVertexFormat`](https://gpuweb.github.io/gpuweb/#enumdef-gpuvertexformat).

[VertexStepMode](https://docs.rs/wgpu/latest/wgpu/enum.VertexStepMode.html "enum wgpu::VertexStepMode")
:   Whether a vertex buffer is indexed by vertex or by instance.

<a id="constants"></a>

## Constants[§](#constants)

[AABB\_GEOMETRY\_MIN\_STRIDE](https://docs.rs/wgpu/latest/wgpu/constant.AABB_GEOMETRY_MIN_STRIDE.html "constant wgpu::AABB_GEOMETRY_MIN_STRIDE")
:   Minimum stride for AABB geometry (24 bytes: two `vec3<f32>`).
    Minimum stride for AABB geometry (24 bytes: two `vec3<f32>`).

[COPY\_BUFFER\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.COPY_BUFFER_ALIGNMENT.html "constant wgpu::COPY_BUFFER_ALIGNMENT")
:   Buffer to buffer copy as well as buffer clear offsets and sizes must be aligned to this number.

[COPY\_BYTES\_PER\_ROW\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.COPY_BYTES_PER_ROW_ALIGNMENT.html "constant wgpu::COPY_BYTES_PER_ROW_ALIGNMENT")
:   Buffer-texture copies must have [`bytes_per_row`](https://docs.rs/wgpu/latest/wgpu/struct.TexelCopyBufferLayout.html#structfield.bytes_per_row "field wgpu::TexelCopyBufferLayout::bytes_per_row") aligned to this number.

[IMMEDIATE\_DATA\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.IMMEDIATE_DATA_ALIGNMENT.html "constant wgpu::IMMEDIATE_DATA_ALIGNMENT")
:   Ranges of [writes to immediate data](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html#method.set_immediates) must be at least this aligned.

[MAP\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.MAP_ALIGNMENT.html "constant wgpu::MAP_ALIGNMENT")
:   Minimum alignment of buffer mappings.

[MAXIMUM\_SUBGROUP\_MAX\_SIZE](https://docs.rs/wgpu/latest/wgpu/constant.MAXIMUM_SUBGROUP_MAX_SIZE.html "constant wgpu::MAXIMUM_SUBGROUP_MAX_SIZE")
:   The maximum allowed value for [`AdapterInfo::subgroup_max_size`](https://docs.rs/wgpu/latest/wgpu/struct.AdapterInfo.html#structfield.subgroup_max_size "field wgpu::AdapterInfo::subgroup_max_size").

[MINIMUM\_SUBGROUP\_MIN\_SIZE](https://docs.rs/wgpu/latest/wgpu/constant.MINIMUM_SUBGROUP_MIN_SIZE.html "constant wgpu::MINIMUM_SUBGROUP_MIN_SIZE")
:   The minimum allowed value for [`AdapterInfo::subgroup_min_size`](https://docs.rs/wgpu/latest/wgpu/struct.AdapterInfo.html#structfield.subgroup_min_size "field wgpu::AdapterInfo::subgroup_min_size").

[QUERY\_RESOLVE\_BUFFER\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.QUERY_RESOLVE_BUFFER_ALIGNMENT.html "constant wgpu::QUERY_RESOLVE_BUFFER_ALIGNMENT")
:   An [offset into the query resolve buffer](https://docs.rs/wgpu/latest/wgpu/struct.CommandEncoder.html#method.resolve_query_set) has to be aligned to this.

[QUERY\_SET\_MAX\_QUERIES](https://docs.rs/wgpu/latest/wgpu/constant.QUERY_SET_MAX_QUERIES.html "constant wgpu::QUERY_SET_MAX_QUERIES")
:   Maximum number of query result slots that can be requested in a [`QuerySetDescriptor`](https://docs.rs/wgpu-types/30.0.1/x86_64-unknown-linux-gnu/wgpu_types/struct.QuerySetDescriptor.html "struct wgpu_types::QuerySetDescriptor").

[QUERY\_SIZE](https://docs.rs/wgpu/latest/wgpu/constant.QUERY_SIZE.html "constant wgpu::QUERY_SIZE")
:   Size in bytes of a single piece of [query](https://docs.rs/wgpu/latest/wgpu/struct.QuerySet.html) data.

[VERTEX\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.VERTEX_ALIGNMENT.html "constant wgpu::VERTEX_ALIGNMENT")
:   [Vertex buffer offsets](https://docs.rs/wgpu/latest/wgpu/util/trait.RenderEncoder.html#tymethod.set_vertex_buffer) and [strides](https://docs.rs/wgpu/latest/wgpu/struct.VertexBufferLayout.html#structfield.array_stride) have to be a multiple of this number.

[VERTEX\_STRIDE\_ALIGNMENT](https://docs.rs/wgpu/latest/wgpu/constant.VERTEX_STRIDE_ALIGNMENT.html "constant wgpu::VERTEX_STRIDE_ALIGNMENT")Deprecated
:   [Vertex buffer strides](https://docs.rs/wgpu/latest/wgpu/struct.VertexBufferLayout.html#structfield.array_stride) have to be a multiple of this number.

<a id="traits"></a>

## Traits[§](#traits)

[DisplayAndWindowHandle](https://docs.rs/wgpu/latest/wgpu/trait.DisplayAndWindowHandle.html "trait wgpu::DisplayAndWindowHandle")
:   Super trait for a pair of display and window handles as used in [`SurfaceTarget`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceTarget.html "enum wgpu::SurfaceTarget").

[UncapturedErrorHandler](https://docs.rs/wgpu/latest/wgpu/trait.UncapturedErrorHandler.html "trait wgpu::UncapturedErrorHandler")
:   The callback of [`Device::on_uncaptured_error()`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.on_uncaptured_error "method wgpu::Device::on_uncaptured_error").

[WasmNotSend](https://docs.rs/wgpu/latest/wgpu/trait.WasmNotSend.html "trait wgpu::WasmNotSend")

[WasmNotSendSync](https://docs.rs/wgpu/latest/wgpu/trait.WasmNotSendSync.html "trait wgpu::WasmNotSendSync")

[WasmNotSync](https://docs.rs/wgpu/latest/wgpu/trait.WasmNotSync.html "trait wgpu::WasmNotSync")

[WindowHandle](https://docs.rs/wgpu/latest/wgpu/trait.WindowHandle.html "trait wgpu::WindowHandle")
:   [`Send`](https://doc.rust-lang.org/nightly/core/marker/trait.Send.html "trait core::marker::Send")/[`Sync`](https://doc.rust-lang.org/nightly/core/marker/trait.Sync.html "trait core::marker::Sync") blanket trait for [`HasWindowHandle`](https://docs.rs/raw-window-handle/0.6.2/x86_64-unknown-linux-gnu/raw_window_handle/borrowed/trait.HasWindowHandle.html "trait raw_window_handle::borrowed::HasWindowHandle") used in [`SurfaceTarget`](https://docs.rs/wgpu/latest/wgpu/enum.SurfaceTarget.html "enum wgpu::SurfaceTarget").

<a id="types"></a>

## Type Aliases[§](#types)

[AccelerationStructureFlags](https://docs.rs/wgpu/latest/wgpu/type.AccelerationStructureFlags.html "type wgpu::AccelerationStructureFlags")
:   Flags for an acceleration structure.

[AccelerationStructureGeometryFlags](https://docs.rs/wgpu/latest/wgpu/type.AccelerationStructureGeometryFlags.html "type wgpu::AccelerationStructureGeometryFlags")
:   Flags for a geometry inside a bottom level acceleration structure.

[AccelerationStructureUpdateMode](https://docs.rs/wgpu/latest/wgpu/type.AccelerationStructureUpdateMode.html "type wgpu::AccelerationStructureUpdateMode")
:   Update mode for acceleration structure builds.

[BlasAABBGeometrySizeDescriptor](https://docs.rs/wgpu/latest/wgpu/type.BlasAABBGeometrySizeDescriptor.html "type wgpu::BlasAABBGeometrySizeDescriptor")
:   Descriptor for the size defining attributes of an AABB geometry group in a bottom level acceleration structure.

[BlasGeometrySizeDescriptors](https://docs.rs/wgpu/latest/wgpu/type.BlasGeometrySizeDescriptors.html "type wgpu::BlasGeometrySizeDescriptors")
:   Descriptor for the size defining attributes, for a bottom level acceleration structure.

[BlasTriangleGeometrySizeDescriptor](https://docs.rs/wgpu/latest/wgpu/type.BlasTriangleGeometrySizeDescriptor.html "type wgpu::BlasTriangleGeometrySizeDescriptor")
:   Descriptor for the size defining attributes of a triangle geometry, for a bottom level acceleration structure.

[BufferAddress](https://docs.rs/wgpu/latest/wgpu/type.BufferAddress.html "type wgpu::BufferAddress")
:   Integral type used for [`Buffer`](https://docs.rs/wgpu/latest/wgpu/struct.Buffer.html) offsets and sizes.

[BufferDescriptor](https://docs.rs/wgpu/latest/wgpu/type.BufferDescriptor.html "type wgpu::BufferDescriptor")
:   Describes a [`Buffer`](https://docs.rs/wgpu/latest/wgpu/struct.Buffer.html "struct wgpu::Buffer").

[BufferSize](https://docs.rs/wgpu/latest/wgpu/type.BufferSize.html "type wgpu::BufferSize")
:   Integral type used for [`BufferSlice`](https://docs.rs/wgpu/latest/wgpu/struct.BufferSlice.html) sizes.

[CommandEncoderDescriptor](https://docs.rs/wgpu/latest/wgpu/type.CommandEncoderDescriptor.html "type wgpu::CommandEncoderDescriptor")
:   Describes a [`CommandEncoder`](https://docs.rs/wgpu/latest/wgpu/struct.CommandEncoder.html "struct wgpu::CommandEncoder").

[CreateBlasDescriptor](https://docs.rs/wgpu/latest/wgpu/type.CreateBlasDescriptor.html "type wgpu::CreateBlasDescriptor")
:   Descriptor to create bottom level acceleration structures.

[CreateTlasDescriptor](https://docs.rs/wgpu/latest/wgpu/type.CreateTlasDescriptor.html "type wgpu::CreateTlasDescriptor")
:   Descriptor to create top level acceleration structures.

[DeviceDescriptor](https://docs.rs/wgpu/latest/wgpu/type.DeviceDescriptor.html "type wgpu::DeviceDescriptor")
:   Describes a [`Device`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html "struct wgpu::Device").

[DynamicOffset](https://docs.rs/wgpu/latest/wgpu/type.DynamicOffset.html "type wgpu::DynamicOffset")
:   Integral type used for
    [dynamic bind group offsets](https://docs.rs/wgpu/latest/wgpu/struct.RenderPass.html#method.set_bind_group).

[ErrorSource](https://docs.rs/wgpu/latest/wgpu/type.ErrorSource.html "type wgpu::ErrorSource")
:   Lower level source of the error.

[ExternalTextureDescriptor](https://docs.rs/wgpu/latest/wgpu/type.ExternalTextureDescriptor.html "type wgpu::ExternalTextureDescriptor")
:   Describes an [`ExternalTexture`](https://docs.rs/wgpu/latest/wgpu/struct.ExternalTexture.html "struct wgpu::ExternalTexture").

[Label](https://docs.rs/wgpu/latest/wgpu/type.Label.html "type wgpu::Label")
:   Object debugging label.

[PollType](https://docs.rs/wgpu/latest/wgpu/type.PollType.html "type wgpu::PollType")
:   Passed to [`Device::poll`](https://docs.rs/wgpu/latest/wgpu/struct.Device.html#method.poll "method wgpu::Device::poll") to control how and if it should block.

[QuerySetDescriptor](https://docs.rs/wgpu/latest/wgpu/type.QuerySetDescriptor.html "type wgpu::QuerySetDescriptor")
:   Describes a [`QuerySet`](https://docs.rs/wgpu/latest/wgpu/struct.QuerySet.html "struct wgpu::QuerySet").

[RenderBundleDescriptor](https://docs.rs/wgpu/latest/wgpu/type.RenderBundleDescriptor.html "type wgpu::RenderBundleDescriptor")
:   Describes a [`RenderBundle`](https://docs.rs/wgpu/latest/wgpu/struct.RenderBundle.html "struct wgpu::RenderBundle").

[RequestAdapterOptions](https://docs.rs/wgpu/latest/wgpu/type.RequestAdapterOptions.html "type wgpu::RequestAdapterOptions")
:   Additional information required when requesting an adapter.

[SamplerDescriptor](https://docs.rs/wgpu/latest/wgpu/type.SamplerDescriptor.html "type wgpu::SamplerDescriptor")
:   Describes a [`Sampler`](https://docs.rs/wgpu/latest/wgpu/struct.Sampler.html "struct wgpu::Sampler").

[ShaderLocation](https://docs.rs/wgpu/latest/wgpu/type.ShaderLocation.html "type wgpu::ShaderLocation")
:   Integral type used for binding locations in shaders.

[ShaderModuleDescriptorPassthrough](https://docs.rs/wgpu/latest/wgpu/type.ShaderModuleDescriptorPassthrough.html "type wgpu::ShaderModuleDescriptorPassthrough")
:   Descriptor for a shader module given by any of several sources.
    At least one of the shader types that may be used by the backend must be `Some`

[SurfaceConfiguration](https://docs.rs/wgpu/latest/wgpu/type.SurfaceConfiguration.html "type wgpu::SurfaceConfiguration")
:   Describes a [`Surface`](https://docs.rs/wgpu/latest/wgpu/struct.Surface.html "struct wgpu::Surface").

[TexelCopyBufferInfo](https://docs.rs/wgpu/latest/wgpu/type.TexelCopyBufferInfo.html "type wgpu::TexelCopyBufferInfo")
:   View of a buffer which can be used to copy to/from a texture.

[TexelCopyTextureInfo](https://docs.rs/wgpu/latest/wgpu/type.TexelCopyTextureInfo.html "type wgpu::TexelCopyTextureInfo")
:   View of a texture which can be used to copy to/from a buffer/texture.

[TextureDescriptor](https://docs.rs/wgpu/latest/wgpu/type.TextureDescriptor.html "type wgpu::TextureDescriptor")
:   Describes a [`Texture`](https://docs.rs/wgpu/latest/wgpu/struct.Texture.html "struct wgpu::Texture").

[TextureViewDescriptor](https://docs.rs/wgpu/latest/wgpu/type.TextureViewDescriptor.html "type wgpu::TextureViewDescriptor")
:   Describes a [`TextureView`](https://docs.rs/wgpu/latest/wgpu/struct.TextureView.html "struct wgpu::TextureView").

Source: [https://docs.rs/wgpu/latest/wgpu/index.html](<https://docs.rs/wgpu/latest/wgpu/index.html>)
