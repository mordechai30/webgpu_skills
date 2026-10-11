# Sources

## Authority and support

- Verify API validity against [WebGPU][api] and shader rules against [WGSL][wgsl].
- Use the [explainer][explainer] for design intent and the project's exact [wgpu release][rust] for Rust APIs.
- Use the reviewed baseline: WebGPU CRD 2026-09-15, WGSL CRD 2026-09-21, and `wgpu 30.0.1`.
- Recheck version-sensitive features/signatures; do not infer implementation support from publication alone.
- Treat tutorials and samples as optional examples, not required architecture or performance evidence.
- Consult [official samples][samples], [LearnWebGPU][learn], or [Fundamentals][fundamentals] for worked cases.
- Preserve target-specific measurements and verified contracts when adapting an example.

[api]: https://www.w3.org/TR/webgpu/
[wgsl]: https://www.w3.org/TR/WGSL/
[explainer]: https://gpuweb.github.io/gpuweb/explainer/
[rust]: https://docs.rs/wgpu/30.0.1/wgpu/
[samples]: https://webgpu.github.io/webgpu-samples/
[learn]: https://eliemichel.github.io/LearnWebGPU/
[fundamentals]: https://webgpufundamentals.org/
