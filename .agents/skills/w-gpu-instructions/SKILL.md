---
name: w-gpu-instructions
description: >-
  Guide Rust wgpu and WGSL implementation and review with practical pipeline, pass, state,
  data-transfer, shader-parameter, texture, and precision choices.
---

# WGPU Instructions

Use the project's resolved `wgpu` version and target backend. Use `wgpu 30.0.1` as the reviewed
API baseline; preserve the project's version. Read supporting detail only when it governs the decision.

## Capabilities and resource contracts

- Check adapter features and limits; request those needed by the selected implementation.
- Use granted device features and limits for shader selection, allocation, and dispatch.
- Specify buffer layout, binding range, logical count, usages, and producer/consumer order together.
- Specify texture format, sample count, view range, usages, and lifetime together.
- Ask for missing target or precision requirements when they change a design choice. Do not guess.
- Read [authority and support checks](references/sources.md#authority-and-support).

## Pipelines and bindings

- Reuse pipelines, layouts, bind groups, and samplers while their resource contracts remain valid.
- Use explicit layouts for shared bindings. Use inferred layouts for isolated pipelines.
- Group bindings by reuse and update frequency; put widely shared bindings at low group indices.
- Set fixed-record `min_binding_size`; match WGSL types, access, stages, and ranges to bindings.
- Cache complete pipeline identities; bound specialized variants and create them outside hot paths.
- Match attachment view formats and sample counts. Keep compatible pipelines through a size-only resize.
- Read [layout and cache details](references/pipelines.md#layouts-and-bindings).

## Passes and synchronization

- Order dependent dispatches, copies, and draws on the queue; avoid CPU waits between GPU-only phases.
- Validate compute hazards per dispatch and render hazards across the whole pass, including state commands.
- Let safe `wgpu` handle resource transitions. Split passes when incompatible usage requires it.
- Use ping-pong state for neighbor-dependent updates; advance it for every encoded simulation step.
- Use workgroup barriers only for workgroup synchronization. Split global phases into dispatches.
- Keep barriers in uniform control flow; guard loads and final writes for partial workgroups.
- Read [usage scopes and compute rules](references/passes-state.md#usage-scopes).

## State management and repeated draws

- Keep CPU binding caches local to each pass; compare dynamic offsets as well as bind group handles.
- Restore incompatible bindings after pipeline changes; rebind required state after bundle execution.
- Sort opaque work to reduce state changes and overdraw; preserve transparent composition order.
- Use bundles for stable repeated commands when CPU encoding cost matters.
- Keep GPU-produced parameters on the GPU; use indirect commands when the next consumer is a draw.
- Gate indirect and multi-draw extensions independently.
- Read [state, bundles, and indirect details](references/passes-state.md#pass-state).

## Data layout and passage

- Calculate WGSL member offsets, array strides, and address-space restrictions; match Rust bytes exactly.
- Check `size_of` and `offset_of`. Do not treat `Pod` as proof of shader layout compatibility.
- Keep vertex decoding, storage layout, uniform layout, and binding alignment separate.
- Upload distinct records for distinct commands; do not overwrite one offset for several queued draws.
- Start small CPU updates with `Queue::write_buffer`; batch adjacent dirty records.
- Reuse staging allocations when measured upload/allocation cost justifies them.
- Reclaim mapped staging/readback slots from completion, not elapsed frame counts.
- Read [layout, upload, and mapping details](references/data-passage.md#host-and-shader-layout).

## Constant settings and shader inputs

- Use WGSL `const` for shader invariants and `override` for a bounded set of pipeline-time choices.
- Use uniforms for small runtime parameters; use storage indexing for dense or GPU-produced tables.
- Compare dynamic uniforms with indexed storage for many objects; account for padded allocation stride.
- Use immediate data for tiny command parameters when the feature and byte limit are granted.
- Use vertex/instance streams for regular attributes and textures for spatial or filtered data.
- Use built-ins for execution-provided values; use flat interpolation for integer inter-stage IDs.
- Read [input selection and performance tradeoffs](references/shader-settings.md#select-the-input-path).

## Render targets and textures

- Choose format, resolution, and sample count from required quality and the next consumer.
- Clear unused prior contents; store only results needed after the pass. Discard dead attachments.
- Use transient attachments only for pass-local contents with clear/discard operations.
- Reuse compatible targets; rebuild dependent views/bindings when their resource changes.
- Check format usage, filtering, blending, storage access, and sample-count support.
- Generate or upload allocated mip levels; preserve color space and data-specific reduction rules.
- Perform lighting/blending in linear space; avoid duplicate sRGB conversion.
- Read [attachment and texture details](references/targets-textures.md#attachment-lifetime).

## Half precision

- Check adapter `SHADER_F16`; request it in `DeviceDescriptor::required_features`.
- Verify the granted device feature; put `enable f16;` before WGSL declarations.
- Provide a validated f32 path when f16 is optional; reject unsupported adapters when it is required.
- Choose half-float storage and f16 arithmetic separately; match the selected host/shader layout.
- Retain f32 where reductions, range, or repeated updates exceed the error budget.
- Measure conversion cost, bandwidth, and GPU time at equal output quality.
- Read [f16 setup and precision choices](references/half-precision.md#enable-f16).

## Performance and recovery

- Measure CPU preparation/encoding, GPU execution, transfers, and frame latency separately.
- Compare equal workloads and output quality; separate startup from steady state.
- Test partial workgroups, distinct per-command data, and relevant capability fallbacks.
- Capture validation errors around fallible GPU creation; keep error scopes narrow.
- Suspend zero-sized surfaces; rebuild size-dependent targets on resize.
- Rebuild device-owned state after device loss; invalidate caches tied to the old device.
- Read [profiling and lifecycle details](references/profiling-lifecycle.md#measure-the-workload).
