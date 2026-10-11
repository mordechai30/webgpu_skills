---
name: WebGpuFull
description: >-
  Design, review, and optimize WebGPU and WGSL, primarily through Rust wgpu.
  Use for GPU resource contracts, pipeline and pass design, shader data paths,
  textures, render targets, half precision, and measured performance.
---

# WebGpuFull

Use this skill for decisions that affect GPU correctness, portability, or performance.
Use Rust `wgpu` examples for Rust work. Translate browser examples by intent, not by API spelling.

## Establish the contract

- Read the project's `Cargo.toml` and resolved `wgpu` version before selecting Rust APIs.
- Identify the execution path: native backend, browser WebGPU, or an explicit compatibility fallback.
- Check adapter capabilities, then request only the features and limits the selected path needs.
  Use the granted device capabilities for allocation and dispatch decisions.
- Record each resource's byte layout, usages, logical count, and writer/reader order.
  Include texture mip, layer, and aspect ranges where hazards depend on them.
- Ask for missing target, precision, or workload requirements when they change the design.
  Do not infer performance from an API name or a tutorial's result on another GPU.

## Read only the required detail

| Task | Read |
| --- | --- |
| Pipeline layouts, variants, reuse, creation cost | [Pipelines](references/pipelines.md) |
| Dependencies, barriers, bundles, indirect work | [Passes and state](references/passes-state.md) |
| Rust/WGSL layout, uploads, mapping, copies | [Data passage](references/data-passage.md) |
| Constants, overrides, uniforms, storage, immediate data | [Shader settings](references/shader-settings.md) |
| Attachments, MSAA, formats, sampling, mips | [Targets and textures](references/targets-textures.md) |
| Adapter/device/shader f16 setup and precision tradeoffs | [Half precision](references/half-precision.md) |
| Timing, diagnostics, surface/device loss | [Profiling and lifecycle](references/profiling-lifecycle.md) |
| Conflicting advice, version changes, source selection | [Verification and sources](references/sources.md) |

Keep references one level below this file. Do not load the full collection for a narrow task.
Use each reference's direct primary-source links to verify a disputed or version-sensitive rule.

## Keep these invariants

- Treat WGSL declarations, Rust layouts, binding layouts, and binding ranges as one contract.
  A Rust `Pod` type proves byte safety; it does not prove WGSL compatibility.
- Upload distinct parameter records for distinct draws or dispatches. Repeated queue writes to one
  offset before one submission do not capture per-command values.
- Reuse pipelines, layouts, bind groups, samplers, and persistent resources while their contracts hold.
  Update buffer contents without rebuilding the bind group. Rebuild it when its resource changes.
- Use separate input/output state when parallel updates must read the previous generation.
  A workgroup barrier cannot synchronize different workgroups.
- Let `wgpu` manage ordinary resource transitions. Validate usage scopes and binding aliasing instead
  of inventing native API barriers in safe `wgpu` code.
- Keep GPU-produced state on the GPU when the next consumer is a shader or indirect command.
  Read back bounded results asynchronously when the CPU actually needs them.
- Treat `f16` arithmetic, half-float buffer storage, half-float textures, and compressed vertex formats
  as separate decisions. Enabling one does not automatically enable or improve the others.

## Verify the result

Compile against the project's pinned `wgpu`. Validate generated WGSL and all selected entry points.
Test the host/shader contract with known values, including a partial final workgroup and distinct
per-object parameters. Exercise the relevant resize, loss, or capability fallback path.

Measure CPU preparation/encoding, GPU work, transfer cost, and frame latency separately.
Keep workload and output quality fixed. State unmeasured tradeoffs as hypotheses.

The concrete Rust snippets target `wgpu 30.0.1`. Recheck signatures for other versions; do not upgrade
the user's project merely to match an example. Preserve the requested skill name `WebGpuFull`.
