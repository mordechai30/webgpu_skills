# Passes and state

## Usage scopes

- Treat each compute dispatch as a usage scope; keep dependent dispatches in one pass when valid.
- Treat the entire render pass as a usage scope, including replaced or unused state bindings.
- End an active pass before encoding copies with its encoder.
- Do not sample a texture subresource while writing it in the same incompatible scope.
- Restrict views to required mips/layers/aspects; use separate passes for render-target feedback.
- Sample attached depth only when the relevant attachment aspects satisfy read-only rules.
- Do not bind overlapping writable buffer ranges to multiple bindings in one draw/dispatch.
- Check buffer usage compatibility even when bound ranges are disjoint.

## Compute dependencies

- Use `workgroupBarrier` for workgroup memory and `storageBarrier` for storage ordering within a workgroup.
- Split dispatch-wide phases into separate dispatches; do not use atomics as a global barrier.
- Keep all lanes through barriers. Load neutral values for inactive lanes; guard final external writes.
- Use `if` for guarded loads; do not use `select` to hide an out-of-bounds expression.
- Use ceiling division for dispatch counts; check axis/product, storage, and dispatch limits.
- Tune legal workgroup sizes against occupancy and memory traffic; do not assume a fixed optimum.
- Use ping-pong buffers for neighbor-dependent updates; select the final destination for rendering.
- Clear accumulation counters before use; define ownership for counts, totals, and indirect arguments.

## Pass state

- Set required state at pass start. Invalidate CPU state caches after external helpers or bundles.
- Compare binding handles, buffer ranges, and dynamic offsets before omitting state commands.
- Select pipelines for shader, topology, blend/depth, or sample changes.
- Set viewport, scissor, blend constant, stencil reference, and command data through pass commands.
- Retain bindings across pipeline changes only when layout compatibility permits it.

## Bundles and indirect commands

- Use bundles for stable command structure when measured CPU encoding cost justifies them.
- Match bundle attachment formats, samples, and depth/stencil read-only policy to the pass.
- Rebind pipeline, bind groups, immediate data, and vertex/index buffers after bundle execution.
- Set viewport/scissor and blend/stencil dynamic values on the pass, not in a bundle.
- Update referenced buffer contents without rebuilding; rerecord changed handles or recorded arguments.
- Create GPU-produced argument buffers with `STORAGE | INDIRECT` and required copy usages.
- Use exact indirect record layouts, including signed indexed `base_vertex`.
- Keep indirect `first_instance` zero unless its feature is granted; gate multi-draw/count separately.
- Group indirect records where useful; measure hidden validation cost across the whole frame.

Sources: [usage scopes][scopes], [aliasing][alias], [bundles][bundles],
[barriers][barriers], [Rust passes][rust].

[scopes]: https://www.w3.org/TR/webgpu/#programming-model-synchronization
[alias]: https://www.w3.org/TR/webgpu/#abstract-opdef-encoder-bind-groups-alias-a-writable-resource
[bundles]: https://www.w3.org/TR/webgpu/#bundles
[barriers]: https://www.w3.org/TR/WGSL/#sync-builtin-functions
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.RenderPass.html
