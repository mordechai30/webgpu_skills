# Passes and state management

## Pass ownership and synchronization

Describe producer/consumer order before selecting pass boundaries. Use one writer for aggregate
outputs such as counts, scan totals, and indirect arguments. Initialize counters before accumulation.
Order dependent dispatches/copies/draws on the queue; avoid CPU waits between GPU-only phases.

Distinguish API usage scopes:

- Each compute dispatch is a scope. Different dispatches in one compute pass can consume prior
  output with different bindings; an extra compute pass is not required solely for that dependency.
- The whole render pass is a scope, including resources set by binding and vertex/index commands.
  Overwriting a binding later does not remove its earlier usage from the scope.
- Copies have their own validation. End the active pass before encoding a copy with its encoder.

Do not read a texture subresource as sampled input while writing it as an attachment or storage
output in the same incompatible scope. Restrict views to the mips/layers/aspects actually needed.
Use separate passes for render-target feedback. The read-only depth exception permits sampling a
depth attachment when the relevant attachment aspects are also read-only.

Do not bind overlapping writable ranges of one buffer to multiple bindings in a draw/dispatch.
Splitting one allocation into disjoint ranges can avoid that alias, but does not make a buffer's
otherwise incompatible usage combination legal within one scope.

Use ping-pong buffers/textures when an update reads neighbors from generation N and writes N+1.
In-place updates are suitable only when ownership proves that no invocation reads another's changing
state. Advance the authoritative index for every encoded simulation step, including multiple steps
in one frame. Render the final destination; do not swap only once per displayed frame.

## Compute barriers and bounds

Use `workgroupBarrier` for workgroup memory and `storageBarrier` for storage memory ordering within
a workgroup. Neither is a dispatch-wide barrier. Split global phases into separate dispatches.
An atomic counter does not supply a global barrier or deterministic ordering of unrelated writes.

Keep barriers in uniform control flow. For a partial workgroup, load a neutral value for inactive
lanes, let all lanes reach the barrier, and guard the final external write. Use a guarded load:

```wgsl
// Each lane owns one tile slot; all 64 lanes reach the barrier.
var<workgroup> tile: array<f32, 64>;

@group(0) @binding(0) var<storage, read> input: array<f32>;

// Load one guarded tile for a later workgroup operation.
// Neutral values preserve the algorithm for a partial final group.
@compute @workgroup_size(64)
fn load_tile(@builtin(local_invocation_index) lane: u32,
             @builtin(global_invocation_id) gid: vec3<u32>) {
  var value = 0.0;
  if (gid.x < arrayLength(&input)) {
    value = input[gid.x];
  }
  tile[lane] = value;
  workgroupBarrier();
}
```

Do not use `select(0.0, input[gid.x], in_bounds)` as a guarded read: arguments are evaluated before
selection. Do not return inactive lanes before a later barrier.

Use ceiling division for dispatch counts and a shader bounds guard. Check per-axis workgroup sizes,
their product, workgroup storage, and dispatch limits. Start tuning with a small legal workgroup,
such as 64 lanes or 8 by 8, rather than treating either as a hardware optimum. Tile large workloads
with a base index; keep host arithmetic and shader indexing within their integer ranges.

## Pass-local state

Start each pass with explicit required state. Keep any CPU state cache local to that pass and
invalidate it after external encoding helpers or bundle execution. Compare bind group **and dynamic
offsets**, not just the bind group handle, before skipping a state command.

Separate pipeline state from dynamic pass state. Changing shader entry point, blend/depth policy,
topology, or sample count selects another pipeline. Set viewport, scissor, blend constant, stencil
reference, buffer ranges, and per-draw data through their appropriate pass commands.

Retain compatible bindings across pipeline switches when layout compatibility proves reuse.
Restore bindings for incompatible layouts. Sort opaque work to reduce pipeline/material changes
and overdraw; preserve transparent composition order instead of forcing a material sort.

## Bundles and indirect work

Use render bundles when repeated command encoding is a measured CPU cost and command structure is
stable. They reduce encoding/validation overhead, not shader work or overdraw. Match bundle color
formats, depth/stencil format/read-only policy, and sample count to the executing pass.

Treat pipelines, bindings, immediate data, and vertex/index state as cleared after bundle execution.
Rebind what subsequent direct draws need. Bundles do not inherit those bindings from the pass.
Viewport/scissor and blend/stencil dynamic values are pass state; bundles cannot set them.

Update contents of referenced buffers without rebuilding a bundle. Rebuild when recorded resource
handles, offsets, immediate values, or command structure change. Use indirect arguments for a
GPU-generated draw count; recorded direct counts are fixed. Keep argument records together in a
shared buffer when practical: some implementations batch indirect validation by buffer. Measure
the whole frame because hidden validation work may fall outside a pass timestamp.

Create generated draw data with `STORAGE | INDIRECT` and any required copy usage. End its producer
phase before consuming it in rendering. Use the exact `wgpu::util` indirect record layout, including
signed `base_vertex` for indexed draws. Keep indirect `first_instance` zero unless its optional
feature is enabled. Gate multi-draw/count extensions separately; do not assume baseline support.

## Verify

Use [usage scopes][sync], [binding aliasing][binding], [bundles][bundles], and
[WGSL synchronization][barriers]. Check [Rust render pass commands][rust] for the pinned signatures.

[sync]: https://www.w3.org/TR/webgpu/#programming-model-synchronization
[binding]: https://www.w3.org/TR/webgpu/#abstract-opdef-encoder-bind-groups-alias-a-writable-resource
[bundles]: https://www.w3.org/TR/webgpu/#bundles
[barriers]: https://www.w3.org/TR/WGSL/#sync-builtin-functions
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/struct.RenderPass.html
