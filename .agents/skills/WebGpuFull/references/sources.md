# Verification and sources

## Use the right authority

Verify WebGPU validity and browser API rules against [published WebGPU][api]. Verify shader syntax,
types, layout, synchronization, and language features against [published WGSL][wgsl]. Use the
[explainer][explainer] for design intent. Resolve a conflict with the specifications, not a tutorial.
Verify Rust names, descriptors, features, and backend restrictions against the project's exact
[wgpu release][rust]. A W3C JavaScript signature cannot validate a Rust descriptor.

This skill was reviewed on 2026-10-11 against WebGPU CRD 2026-09-15, WGSL CRD 2026-09-21,
the explainer, and released `wgpu 30.0.1` documentation and source. These are review baselines,
not guarantees that a user's browser, backend, or older crate implements every published feature.
Recheck changing features and signatures before adapting examples.

## Retain decision-changing corrections

- An ID-bearing override uses its decimal ID key. Older name-or-ID examples need correction.
- Natural WGSL layout, uniform address-space restrictions, vertex decoding, and dynamic binding
  alignment are separate contracts. A universal 16-byte rounding helper cannot prove all four.
- Queue writes before one submission are transfers before its commands, not per-draw snapshots.
- Compute usage scopes belong to dispatches; render scopes cover the entire pass, including state
  commands whose bindings are later replaced. Pass breaks do not solve shader-global barriers.
- Bundle execution clears binding/pipeline state. Referenced buffer contents may still change.
- Safe `wgpu` retains submitted resources. A fixed deferred-deletion frame count is not a required
  safe-handle lifetime rule, nor proof that an explicit staging slot is reusable.
- Half-float textures, compact vertex formats, packed half words, and typed f16 arithmetic differ.
- Immediate data, error scopes, surface acquisition/presentation, and staging belts have changed
  Rust APIs. Keep the examples pinned; do not combine fields from different releases.
- Branch removal, large workgroups, render bundles, pass fusion, and f16 are performance hypotheses
  until measured on the target workload. Tutorial measurements do not establish universal rankings.

## Material reviewed and how to use it

The source skills reviewed were `webgpu`, `webgpu-specs`, `webgpu-claude-skill`,
`rust-wgpu-functional`, and `webgpu-syntax-render-pipeline`, including their references, scripts,
starter GPU code, and the stub's upstream material. They informed coverage; this skill does not
inherit their unrelated architecture, commands, or blanket performance rules.

The local `/Users/motyzas/Downloads/WGPU` document collection supplied codelab stages 0–7,
bind groups, uploads, packed compute/vertex data, bundles, indirect draws, dynamic shaders,
textures, device loss, errors, raw WebGPU, comparisons, profiling, MDN, Rust API documentation,
and a specification snapshot. Treat snapshots as historical inputs; check current normative rules.

Use these direct references when the related task needs deeper evidence:

| Material | Relevant analysis |
| --- | --- |
| [wgpu repository][repo], [release docs][rust] | Ownership, descriptors, features, backend differences |
| [GPUWeb explainer][explainer] | Explicit resource contracts, bindings, pipelines, queue design |
| [LearnWebGPU][learn] | Uniforms, mips, MSAA, color, timing; translate C++ calls to Rust |
| [Official samples][samples] | Boids, blur, particles, instancing, bundles, textures, timestamps |
| [luma.gl][luma] | Higher-level resource and pipeline abstractions; optional architecture examples |
| [WebGPU.org][org] and [WebKit demos][webkit] | Capability discovery and examples of GPU workloads |
| [Showcase][showcase] and [games][games] | Workload ideas; no authority for API validity or performance |
| [WGSL editor][editor] | Shader experiments; validate on the intended Rust/Naga/backend path |

Official sample implementations were inspected at commit
`e040ec1a20dbbe01afed30c812e825b2639cfe5b`. The Three.js-oriented stub's upstream materials
were inspected at commit `af2319bd01bb7cc881267a9ef42cafdaf5e9029d` of
`dgreenheck/webgpu-claude-skill`. Transfer principles where valid; do not require Three.js in Rust.

The supplied WebGPU Fundamentals lessons were reviewed as worked explanations:

| Lesson | Use for |
| --- | --- |
| [Inter-stage variables][varyings] | Location/type matching and interpolation |
| [Uniforms][uniforms] | Per-frame/per-object parameter lifetime |
| [Storage buffers][storage] | Dense indexed tables and GPU-produced data |
| [Vertex buffers][vertex] | Fixed attribute decoding and instance streams |
| [Copying data][copies] | Queue writes, ordered copies, and mapped staging |
| [Bind group layouts][bindings] | Explicit reuse and inferred-layout limitations |
| [Memory layout][memory] | Member offsets, matrix columns, array strides |
| [Immediates][immediate] | Tiny command-local inputs; check current Rust support |
| [Constants][constants] | Source constants versus pipeline overrides |
| [Miscellaneous input][misc] | Built-ins and execution-provided values |

Related chapters and links were inspected where they affected the requested choices: compute,
texture sampling/storage, mip generation, MSAA, transparency, timing, and optimization. Preserve
their useful examples without importing browser-only setup, old signatures, or measured rankings
as mandatory Rust practice. The supplied `.htm` bind-group link was confirmed as `.html`.

[api]: https://www.w3.org/TR/webgpu/
[wgsl]: https://www.w3.org/TR/WGSL/
[explainer]: https://gpuweb.github.io/gpuweb/explainer/
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/
[repo]: https://github.com/gfx-rs/wgpu
[learn]: https://eliemichel.github.io/LearnWebGPU/
[samples]: https://webgpu.github.io/webgpu-samples/
[luma]: https://luma.gl/
[org]: https://webgpu.org/
[webkit]: https://webkit.org/demos/webgpu/
[showcase]: https://www.webgpu.com/showcase/
[games]: https://webgpu-games.com/
[editor]: https://www.wgsl.dev/editor?example=triangle&canvas=webgpu
[varyings]: https://webgpufundamentals.org/webgpu/lessons/webgpu-inter-stage-variables.html
[uniforms]: https://webgpufundamentals.org/webgpu/lessons/webgpu-uniforms.html
[storage]: https://webgpufundamentals.org/webgpu/lessons/webgpu-storage-buffers.html
[vertex]: https://webgpufundamentals.org/webgpu/lessons/webgpu-vertex-buffers.html
[copies]: https://webgpufundamentals.org/webgpu/lessons/webgpu-copying-data.html
[bindings]: https://webgpufundamentals.org/webgpu/lessons/webgpu-bind-group-layouts.html
[memory]: https://webgpufundamentals.org/webgpu/lessons/webgpu-memory-layout.html
[immediate]: https://webgpufundamentals.org/webgpu/lessons/webgpu-immediates.html
[constants]: https://webgpufundamentals.org/webgpu/lessons/webgpu-constants.html
[misc]: https://webgpufundamentals.org/webgpu/lessons/webgpu-miscellaneous-input.html
